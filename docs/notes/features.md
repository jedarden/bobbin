# Features

What BOBBIN must actually do, as distinct from how it will be built. Design
decisions live inline where they affect the feature; open questions are marked
and carried into `../plan/plan.md`.

The consistent principle, inherited from warden: **make the unsafe thing
impossible by construction, not by deny-list.** A worker cannot misuse a
credential it never receives, and cannot push to a repo the proxy will not
forward for.

## Server-side credential injection

The defining feature. A worker pod's git config carries no token, no
`.git-credentials`, and no credential helper that can produce one. It points at
BOBBIN, BOBBIN attaches the Forgejo credential, and the response comes back
unchanged.

Requirements:

- The injected credential must **never appear in anything the pod can read** —
  not in a redirect the client follows, not in an error body, not in a response
  header.
- Redirects are the specific hazard: a naive proxy that follows an upstream
  redirect to another host can forward the `Authorization` header with it.
  Cross-host redirects must be refused or stripped, not followed blindly.
- The pod must never be prompted for credentials. `git` prompting in a pod hangs
  the worker forever, so BOBBIN must not return a bare `401` with a
  `WWW-Authenticate` challenge to the client — a denied request should fail
  cleanly and legibly instead.

## Per-repo allowlist

The feature that actually shrinks blast radius. Today's token can push to every
repo the account reaches; BOBBIN should forward only for repos it has been told
about.

- The allowlist is **configuration, not a request parameter** — a caller cannot
  widen its own scope.
- It should be derivable from the same list the worker is given
  (`NEEDLE_POD_WORKSPACES`), so a repo a worker cannot clone is also a repo it
  cannot push to. Divergence between those two lists is a bug, not a feature.
- **Canonicalisation is mandatory, and this is measured rather than theoretical.**
  Forgejo matches repo paths **case-insensitively** and treats the `.git` suffix
  as optional: `JedArden/Warden.git/info/refs` returns 200, and
  `jedarden/warden/info/refs` returns 200. So a naive exact-string allowlist is
  **bypassable by changing capitalisation**. Required order: URL-decode → reject
  `..` and `%2f` → strip a trailing `.git` → lowercase → compare against a
  lowercased allowlist.
- **Resolved:** the allowlist is static configuration. A restart to add a repo
  is acceptable, and an operator endpoint that mutates policy is more surface
  than the convenience is worth.

## Default-deny allowlist, not push detection

The original framing of this feature — *detect a push and gate it* — was wrong,
and research disproved it empirically. **`git-receive-pack` is not the only way
to write.** Confirmed write paths that never touch it:

- **WebDAV.** If the `info/refs?service=git-receive-pack` response is not a smart
  advertisement, git falls back to `git http-push`, which writes with `PROPFIND`,
  `MKCOL`, `PUT`, `MOVE`, `LOCK`, `DELETE`. Observed on the wire:
  `GIT_SMART_HTTP=0 git push` issued `PROPFIND` and never touched
  `git-receive-pack`. Forgejo would reject it — but a worker controls its own
  client, so BOBBIN must not delegate that refusal upstream.
- **Git LFS.** `PUT /info/lfs/objects/{oid}/{size}` writes object content; the
  lock endpoints mutate state.
- **Forgejo's REST API and web UI.** The injected credential is valid for the
  whole instance, so any proxied URL space broader than the git endpoints lets a
  worker delete repos or mint tokens — authenticated, and not a git push.
- **Wiki repos.** `{repo}.wiki` routes through the same git handlers, so it is a
  write to a *different repository* than `{repo}`.

There is a symmetric read-path surprise: **dumb HTTP** clones fully without ever
touching `git-upload-pack`, using only `GET /objects/...`. And the fallback
triggers on any `info/refs` response whose `Content-Type` is not the expected
advertisement type — so mangling one header silently converts every client to
dumb HTTP and every push to WebDAV.

**So the policy is an allowlist of (method, exact path shape), default-deny.**
The question BOBBIN answers is *"is this one of the few known read endpoints?"*,
never *"is this a push?"*. Anything unrecognised and not GET/HEAD is treated as
a write and refused — which is also Forgejo's own internal default.

Authentication is separate from authorisation here: the credential is injected
for **both** services, because Forgejo's `REQUIRE_SIGNIN_VIEW` makes even an
anonymous fetch return 401. The allowlist governs what is permitted, not whether
to authenticate.

## Never hand the caller anything that authenticates

Two leak channels were found, both live, and neither is obvious:

- **`Set-Cookie`.** Forgejo returns `i_like_gitea=<session-id>` on git HTTP
  endpoints. That is a session credential; passing it through hands the worker
  exactly what BOBBIN exists to withhold. The git spec explicitly permits
  stripping it — servers must not require cookies to function.
- **LFS batch responses**, which echo the injected `Authorization` header back
  in the response body (see the LFS section).

BOBBIN strips `Set-Cookie`, `Set-Cookie2`, `WWW-Authenticate` and
`Authentication-Info` from responses, and `Cookie`/`Authorization` from
requests.

Also: **never return `401`.** A 401 invites git into a credential-helper retry
loop against a proxy that will never accept client credentials, and an upstream
401 relayed verbatim points the investigation at the *worker's* missing
credentials rather than at BOBBIN's rejected token. Deny with `403`; report an
upstream credential failure as `502`. Both with a `text/plain` body, which git
surfaces as `remote: <text>`.

## Transparency to a stock git client

BOBBIN is only useful if `git clone` and `git push` work unmodified. No custom
client, no wrapper script, no special remote helper.

That means it must be faithful to the smart-HTTP protocol:

- Preserve pkt-line framing byte-for-byte, including the `# service=`
  advertisement and flush packets.
- **Stream, never buffer** — and the reason is sharper than memory bounds.
  Cloudflare sits in front of Forgejo with a **100 MiB request cap enforced
  pre-emptively off `Content-Length`**, and a **125 s read timeout**. Real
  pushes use chunked encoding and send no `Content-Length`, so they dodge the
  size pre-check; if BOBBIN buffers the request body it *sets* one and converts
  a working push into an instant 413. Buffering the response is equally bad:
  git's sideband keepalives every 5 s are what keep the 125 s timer from
  firing, and buffering collapses them into silence.
- Preserve `Content-Type`s and chunked transfer-encoding exactly.
- Tolerate long-running requests. A large clone can outlast a default HTTP
  timeout, and Cloudflare sits in front of the upstream.

## Audit trail

One place that sees every clone and every push, which does not exist today.

Each decision should emit a structured record: caller identity, repo, service
(fetch/push), allow/deny with reason, upstream status, bytes, duration. Same
single-line JSON shape the needle-pod entrypoint uses, so one log query spans
both.

Denials matter more than successes here — a denied push is the signal that a
worker tried to exceed its scope.

## Caller identity

BOBBIN needs to know who is asking, or the allowlist is per-deployment rather
than per-worker.

**Resolved: NetworkPolicy as the primary control, plus a shared bearer for
attribution.** Calico enforcement on agent-sandbox was confirmed empirically —
traffic to a netpol-selected pod is *dropped* (6 s timeout) while an unselected
pod *refuses* in 9 ms. So a namespace-scoped ingress policy is a real boundary,
not decoration.

The bearer is explicitly **not** a defence against the agent: a process running
model-generated commands can read anything its pod can read, which is exactly
why the Forgejo PAT must not be there. Its value is audit attribution and
defence-in-depth, and warden's implementation already exists. What the pod ends
up holding is a token valid only against an in-cluster ClusterIP and bounded by
the allowlist — categorically weaker than an account-wide, non-expiring PAT.

Per-pod ServiceAccount identity is a clean phase 2: the worker already carries a
projected, auto-rotating SA token, and a git credential helper that reads it
would give BOBBIN a fresh kubelet-issued credential per git invocation — which a
once-written `.git-credentials` can never do.

## Fail closed

Every ambiguous case denies. An unparseable path, an unknown service, a repo
that does not match the allowlist, a missing credential at startup: all refuse
rather than forward.

Specifically: if BOBBIN cannot load its own credential it must **fail to
start**, not start and serve 500s. A proxy that is up but cannot authenticate
looks healthy to Kubernetes while every worker silently fails to clone.

## Reject any credential the caller supplies

A caller must not be able to authenticate as anyone but the identity BOBBIN
assigns it. Forgejo tries auth methods in order and **the first one yielding a
user wins**, with OAuth2 registered before Basic and query-parameter tokens
still enabled in this version.

So a pod supplying `?access_token=<its own token>` would execute the request as
*itself*, while BOBBIN evaluated policy for a different identity. That turns the
allowlist from a boundary into a suggestion.

**BOBBIN strips inbound `Authorization` headers and `token` / `access_token`
query parameters before injecting its own.** Not hardening — a correctness
requirement.

Relatedly, injection must use **HTTP Basic**, not `Authorization: token`.
Forgejo only enforces token scopes when `IsBasicAuth` is true, header-token auth
returns 500 on LFS paths, and only Basic's path matching covers LFS at all.

## Git LFS is a credential leak, not a compatibility gap

Measured, not theorised: Forgejo's LFS batch endpoint **copies the inbound
`Authorization` header into the response body**, so the client is handed back
the exact credential used upstream. Decoded and compared byte-for-byte against
what was sent — identical.

Left untouched, a single `git lfs` operation hands the injected PAT to the pod
and defeats the entire premise. A second, independent problem: batch responses
build object URLs from the server's configured base URL rather than the request
Host, so clients get URLs pointing at the real Forgejo origin and route object
traffic around the proxy.

So LFS forces a decision, and "not yet implemented" is not one of the options:

- **Rewrite** — parse batch JSON, strip `header.Authorization`, substitute a
  BOBBIN-issued handle, rewrite `href` to point back at BOBBIN. Correct, but
  makes BOBBIN a protocol-aware participant rather than a passthrough.
- **Block** — refuse `/info/lfs/` with a legible error. Honest, trivially safe,
  and breaks any repo that uses LFS.

**Default to blocking until a repo actually needs LFS.** A loud, understood
failure beats a silent credential leak, and no repo in the current worker set
uses LFS.

## Rate limiting

**Forgejo has no rate limiting at all** — verified against its config schema,
middleware chain, and live response headers, and its own documentation says this
is deliberate because rate limiting belongs in the reverse proxy.

That makes it BOBBIN's job. A fleet of workers retrying a failing clone in a
restart loop is exactly the shape of traffic that would otherwise hit Forgejo
unbounded.

## Designed for absorption into SEAM

BOBBIN exists now because [SEAM](https://git.ardenone.com/jedarden/SEAM) is not
deployed and its plan does not cover the git protocol. It should not become a
permanent parallel system.

So: keep the credential in the same place SEAM would read it from, keep the
policy layer pure and separable (warden's `internal/policy` is the model), and
keep the caller-facing surface small enough to become a SEAM route. Migration
for a worker should be one environment variable.

## Explicitly not in scope

- **Caching or mirroring git objects.** BOBBIN is an auth and policy boundary,
  not a CDN. Caching invites correctness bugs for no current benefit.
- **Rewriting git contents.** No commit signing, no push rewriting, no
  content inspection. Every byte of the packfile passes through untouched.
- **Serving git itself.** Forgejo remains the server; BOBBIN never becomes the
  source of truth.
- **Being an ingress.** Workers reach it on a ClusterIP inside the cluster.
  Exposing it publicly would make it an attractive target holding a
  write-scoped credential.
