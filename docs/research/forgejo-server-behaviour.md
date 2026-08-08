# Forgejo Server Behaviour

Research pass 2026-08-08 against **Forgejo 10.0.0+gitea-1.22.0** (the deployed
version). Source citations are against the `v10.0.0` tag on Codeberg; live
probes were read-only.

Two findings here change the design rather than inform it: LFS leaks the
injected credential straight back to the caller, and a caller can override
BOBBIN's injected identity via a query parameter.

## 1. Authentication — use HTTP Basic, and only Basic

The username is **irrelevant**. `services/auth/basic.go` decodes the pair,
treats the password as the token, looks it up globally, and derives identity
from `token.UID`. `uname` is never compared to anything. Verified live: bogus
username, empty username, token-as-username, `Authorization: token`,
`Authorization: Bearer`, and `?access_token=` **all return 200**.

Despite that, **BOBBIN must use Basic specifically**, for three independent
reasons:

1. **Token scopes are only enforced under Basic.** `services/context/permission.go`
   returns early from `CheckRepoScopedToken` unless `IsBasicAuth` is true.
   Header-token auth registers as `oauth2`, so `read:` vs `write:repository` is
   **never checked** on git-over-HTTP. Underlying repo permissions still apply,
   so this is not privilege escalation — but token scope is not a confinement
   boundary unless you authenticate with Basic.
2. `Authorization: token` **500s** on LFS paths (verified).
3. Only Basic's path regex covers LFS at all (`isGitRawOrAttachOrLFSPath`
   vs OAuth2's `isGitRawOrAttachPath`).

Token format is exactly **40 lowercase hex characters**; anything containing a
`.` is a JWT. Useful for input validation.

### The identity-override bypass BOBBIN must close

Auth methods are tried in order and **the first one returning a non-nil user
wins**; errors fall through. OAuth2 is registered *before* Basic, and
`DISABLE_QUERY_AUTH_TOKEN` defaults to **false** in this version.

So a pod that supplies `?access_token=<its own valid token>` authenticates **as
itself**, overriding BOBBIN's injected Basic header — defeating the allowlist,
because policy would be evaluated for one identity and the request executed as
another.

**BOBBIN must strip inbound `Authorization` headers and `token` /
`access_token` query parameters before injecting its own.** This is not
optional hardening; it is the difference between a policy boundary and a
suggestion.

### Protocol v2

Verified live: with `Git-Protocol: version=2` the server advertises `version 2`,
`ls-refs`, `fetch=shallow wait-for-done filter`; without it, v0. The header must
be forwarded verbatim or every clone silently degrades — and `filter` is
advertised, so partial clone works.

## 2. Token scopes — per-repo scoping does not exist here

Nine categories × {read, write}, plus `all` and `public-only`. Write implies
read. The `AccessToken` struct carries no repository list and **no expiry
field** — confirming that PATs cannot be time-bounded.

**`write:repository` cannot be narrowed in Forgejo 10.x.** Effective access is
the union of everything the owning user can reach.

**Per-repo token scoping shipped in Forgejo v15.0 (April 2026)** — restricted to
`read/write:repository` and `read/write:issue`. Note that the *latest* docs
describe "Specific repositories" while the v10.0 docs do not, which is an easy
trap when reading documentation for a deployed older version.

> **Strategically, upgrading Forgejo to ≥v15 is the single highest-leverage
> change for BOBBIN's threat model.** It converts "one token that reaches
> everything" into a genuine per-repo credential. BOBBIN's allowlist would then
> be defence-in-depth rather than the only boundary.

The sole narrowing available today is `public-only`, which 403s on private repos.

## 3. Mint-and-revoke short-lived tokens — do not build this

The endpoints exist (`POST`/`DELETE /api/v1/users/{u}/tokens`) and require
`write:user` **via Basic auth only**. Revocation propagates immediately: the
token cache re-reads the row by ID on every hit specifically so deletions take
effect at once, so there is no stale-auth window.

It still should not be built, in order of severity:

1. **It builds on behaviour upstream classified as a vulnerability and removed.**
   PR #9079 (landed v11.0.4, backported to v12.0.2/v15/v16) makes token creation
   require the *actual user password*, explicitly because obtaining a PAT while
   authenticated by another token was privilege escalation. It works in v10.0.0
   — and breaks on the next upgrade.
2. **The bootstrap credential is worse than what it replaces.** Minting needs
   `write:user`, which can mint `all`-scoped tokens. BOBBIN would hold something
   strictly more dangerous than the `write:repository` PAT it exists to remove.
3. **No TTL backstop.** Tokens have no expiry, so any minted token that outlives
   a BOBBIN crash leaks permanently. You would need a reconciliation sweeper.
4. **No confinement gained.** Before v15 a minted token is not repo-scoped, so a
   short-lived token has the *same blast radius* — you buy rotation, not
   least-privilege, at significant complexity.

Also worth knowing: Forgejo does `AllCols().Update()` on the token row for
**every authenticated request**, so a pod fleet sharing one token hammers a
single row.

## 4. Deploy keys — unusable, doubly

SSH only. Deploy keys are resolved exclusively in the SSH `serv` path; there are
**zero** occurrences of `DeployKey` in the git-over-HTTP router, and the auth
group registers no deploy-key method.

Independently blocked: `git.ardenone.com` is Cloudflare anycast and **port 22
does not connect**. Forgejo still advertises an `ssh_url` that is dead at that
hostname.

## 5. Git LFS leaks the injected credential — verbatim

**This is the finding that most threatens BOBBIN's premise.** LFS is enabled on
the target.

`services/lfs/server.go` copies the inbound `Authorization` header into the
batch response body:

```go
Authorization: ctx.Req.Header.Get("Authorization"),          // L419
if len(rc.Authorization) > 0 { header["Authorization"] = rc.Authorization }   // L480-481
```

So a batch response contains:

```json
"actions": { "upload": { "href": "…", "header": { "Authorization": "Basic <credential>" } } }
```

Verified by decoding: the returned credential is **byte-identical** to the one
sent, and the decoded password is the 40-char PAT.

**Any `git lfs` operation through BOBBIN hands the injected PAT straight back to
the pod, in the response body.** That defeats "pods never hold credentials"
completely.

There is a second, independent problem: `href` is built from `setting.AppURL`,
not the request Host, so clients receive URLs pointing at the **real Forgejo
origin** — routing LFS object traffic around the proxy entirely.

So LFS forces a real decision: either BOBBIN parses and rewrites batch JSON
(stripping `header.Authorization`, substituting its own handle, and rewriting
`href`), or it **blocks `/info/lfs/` outright** with a legible error. Doing
nothing is not an option — it is a silent credential leak.

*Not verified:* LFS lock endpoints and chunking behaviour.

## 6. Cloudflare — exact limits, and a buffering trap

Bisected live against the zone:

- **Request body cap: exactly 104,857,600 bytes (100 MiB), inclusive.**
  104,857,600 passes; 104,857,601 returns 413 from Cloudflare. That places the
  zone on Free/Pro.
- **524 proxy read timeout: 125 s.** Origin connect 19 s (522); proxy idle
  900 s (520).
- The 413 is an HTML page, so git surfaces only `RPC failed; HTTP 413`.

Two consequences that directly constrain the implementation:

**Buffering the request body converts a working push into an instant 413.**
Cloudflare enforces the cap pre-emptively off `Content-Length` (413 returned in
0.26 s after 3 MB sent). Real `git push` uses chunked encoding above
`http.postBuffer` and sends *no* `Content-Length`, so pushes normally dodge the
pre-check. If BOBBIN does `io.ReadAll`, it sets a real `Content-Length` and
re-arms it. Keep streaming and `ContentLength: -1`; never raise
`http.postBuffer`.

**Buffering the response manufactures 524s.** Git emits sideband progress or
empty keepalives every 5 s (`uploadpack.keepAlive`), which resets the 125 s
read timer. Forgejo streams natively — it pipes `git upload-pack` stdout
directly to the response — so BOBBIN is the only place buffering could be
introduced.

Also: don't synthesize `Expect: 100-continue` on an HTTP/2 upstream (Cloudflare
400s it), and if Cache Rules are ever added to this zone, exclude
`/info/refs`, `/git-upload-pack`, `/git-receive-pack` explicitly. Bot Fight Mode
is currently off; enabling it would break every pod clone and **cannot be
excepted by WAF Skip rules**.

The clean structural fix, per Cloudflare's own docs, is a **DNS-only
(grey-clouded) hostname for git traffic** that BOBBIN targets — removing both
the 100 MiB cap and the 125 s ceiling. Cloudflare Tunnel is not a workaround;
it carries the same class of limit.

## 7. Rate limiting is BOBBIN's job

**Forgejo has none** — verified against the config schema, the middleware chain,
and live headers: no per-IP or per-token limiter, no `X-RateLimit-*`, no 429.
Forgejo's own reverse-proxy documentation states this is deliberate and that
rate limiting belongs in the proxy.

(`REVERSE_PROXY_LIMIT` is a trusted-hop count, not a rate limit.)

Forgejo-side timeouts that interact: `PER_WRITE_TIMEOUT` 30 s,
`PER_WRITE_PER_KB_TIMEOUT` 10 s. Note `[git.timeout]` does **not** bound
smart-HTTP RPC — Cloudflare's 125 s is the binding constraint.

## 8. Doing better than a shared PAT

**`ENABLE_REVERSE_PROXY_AUTHENTICATION` is the architecturally right answer**,
and BOBBIN is exactly the shape it expects. Forgejo trusts a header
(`X-WEBAUTH-USER` by default) naming the authenticated user; **no token exists
anywhere** — nothing to leak, rotate, or revoke. It explicitly handles git and
LFS paths, so it works for git-over-HTTP. Each pod could authenticate as a
distinct Forgejo user, making per-user repo permissions the enforcement layer —
real least-privilege, which PAT scopes cannot give before v15.

**Two serious conditions.** Forgejo blindly trusts the header ("verification of
header data is not performed as it should have already been done by the reverse
proxy"), and — verified — **`REVERSE_PROXY_TRUSTED_PROXIES` does not gate it**;
that setting feeds only X-Forwarded-For handling. There is no source-IP check on
the auth header.

So adopting it requires Forgejo to be unreachable except through BOBBIN. **On a
public Cloudflare hostname it would be a critical remote authentication
bypass.** Currently disabled (verified). If ever adopted, move Forgejo off
public ingress first and leave auto-registration off.

Options that do **not** work: OAuth2 app tokens (no `client_credentials` grant;
builtin apps are public clients with loopback redirects — interactive only),
impersonation/`sudo` (admin-only and API-only, not applied to git routes),
`INTERNAL_TOKEN` (full-trust backdoor with no per-user semantics, and it *is*
routed publicly), and per-request signed access (nothing exists for git).

## Recommended posture

1. Inject via **HTTP Basic** with a constant username, token as password.
2. **Strip inbound `Authorization` and `token`/`access_token` query params.**
3. **Handle or block LFS** — untouched it leaks the PAT to every pod.
4. Forward `Git-Protocol`; stream both directions; never `io.ReadAll` a body.
5. Translate Cloudflare 413/524/502 into legible git errors.
6. **Rate-limit in BOBBIN** — Forgejo has none by design.
7. Strategic: upgrade Forgejo to ≥v15 for per-repo scoping, and/or move it
   behind private ingress and adopt reverse-proxy auth.
8. Optional hardening: `ENABLE_BASIC_AUTHENTICATION=false` is safe (tokens are
   checked *before* that flag, so PAT-basic keeps working while password-basic
   is disabled), and `DISABLE_QUERY_AUTH_TOKEN=true`.
