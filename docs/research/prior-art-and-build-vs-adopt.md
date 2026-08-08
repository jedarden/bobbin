# Prior Art, and Build vs Adopt

Research pass 2026-08-08. Dates verified via GitHub/Codeberg APIs, not recalled.

**Bottom line: nothing is a drop-in, and the right thing to build may not be a
proxy at all.** The recommendation is a token-minting *broker*, with the
reverse proxy reserved for cases the broker cannot cover — and which of those
applies is decided by one experiment and one infrastructure decision.

## 1. Two facts that reframe the problem

### Forgejo can now enforce most of BOBBIN's policy natively

**Forgejo v15.0.0 (released 2026-04-16) shipped per-repository scoped tokens.**
A repo-restricted token may hold only `read:repository`, `write:repository`,
`read:issue`, `write:issue`, with repository selection of All / Public-only /
Specific. Current stable is **v16.0.2 (2026-07-30)**.

Enforcement reaches the git smart-HTTP path, not just the API — Gitea PR #24362
(2023, Gitea 1.20) added scoped-token checks to the web routes that accept basic
auth, including git HTTP and LFS. Forgejo inherits it.

**This collapses most of BOBBIN's stated value.** The per-repo allowlist and the
fetch-vs-push split would be enforced *by Forgejo*, not by a proxy we maintain.

Four caveats keep something alive:

| Caveat | Evidence |
|---|---|
| Forgejo PATs **never expire** | Issue #8837 "Expiration of API token", opened 2025-08-09, still open |
| Forgejo's *expiring* tokens are **unscoped** | OAuth2 tokens have `expires_in: 3600` but "scopes are not implemented… they can be used to execute any actions on behalf of the user" |
| Minting needs a **stronger credential** | `/users/:name/tokens` requires BasicAuth **with a password**, so the minter holds the user password or site-admin `Sudo:` |
| Deploy keys are **SSH-only** | Issue #7763 "HTTPS Deploy Keys", opened 2025-05-02, still open |

**You cannot get scoped *and* short-lived from Forgejo today. That is the entire
residual case for BOBBIN.**

> **Blocking constraint for us: `git.ardenone.com` runs Forgejo 10.0.0.**
> Per-repo scoping does not exist there. Everything in the "broker" branch below
> requires an upgrade to ≥v15 first.

### Anthropic built this exact service

Anthropic's Claude Code sandboxing writeup (2025-10-20) describes it verbatim:
credentials never inside the sandbox; the git client authenticates to a proxy
with a scoped credential; the proxy *"verifies this credential **and the
contents of the git interaction** (e.g. ensuring it is only pushing to the
configured branch), then attaches the right authentication token"*.

Not open source. But it validates the design — and note the justification is
**branch-level push policy**, which no Forgejo token can express. If our policy
ever needs to be finer than "which repo", the proxy becomes necessary again.

## 2. Candidate inventory

| Project | Lang | Licence | Last commit | Injects credential | Verdict |
|---|---|---|---|---|---|
| `dependabot/proxy` | Go | MIT | 2026-08-07 | **Yes** | **Read this first** — fetch-only |
| harness/harness (ex-Gitness) | Go | Apache-2.0 | 2026-08-06 | n/a (is a forge) | Best licence:quality for route-table reference |
| Gitea `githttp.go` | Go | **MIT** | 2026-08-08 | n/a | Copy from here |
| Forgejo `githttp.go` | Go | **GPL-3.0-or-later** | 2026-08-08 | n/a | **Licence trap** — near-identical to Gitea's; take the MIT one |
| GitLab Workhorse | Go | MIT Expat | 2026-08-08 | indirectly | Architecture reference (pre-authorize then stream) |
| finos/git-proxy | TypeScript | Apache-2.0 | 2026-08-07 | **No — pass-through** | **Reject** — see §3 |
| gomods/athens | Go | MIT | 2026-07-25 | Yes (`.netrc`) | Allowlist ergonomics worth copying |
| kubernetes/git-sync | Go | Apache-2.0 | 2026-07-28 | via `--askpass-url` | **Adopt the wire format** |
| Envoy `credential_injector` | C++ | Apache-2.0 | active | **Yes** | Spike only — see §5 |
| git-lfs-authenticate | — | — | 2026-08-07 | Yes (capability model) | Strong conceptual model |
| sosedoff/gitkit | Go | MIT | 2025-01-21 | no | Marginal — 18 months idle |
| AaronO/go-git-http | Go | Apache-2.0 | **2016** | no | Dead |
| google/goblet | Go | Apache-2.0 | **archived 2024-12-11** | caching only | Dead |
| asim/git-http-backend, schacon/grack | — | **no LICENCE file** | — | — | Legally unusable |
| Sourcegraph gitserver | Go | **repo private** | archived 2024-09 | — | Not applicable |
| oauth2-proxy | Go | MIT | 2026-08-01 | **No** — forwards identity | Not applicable |

**Nothing is importable.** Every credible implementation buries its git-HTTP
logic under Go `internal/`, so "reuse" means reading code, not `go get`.

## 3. finos/git-proxy — assessed seriously, and rejected

It is alive and well-governed (FINOS Graduated, Citi/RBC/NatWest backing,
v2.0.0 on 2026-05-08, commits through 2026-08-07, Apache-2.0), and it does have
a real per-repo allowlist. Three disqualifiers:

1. **Credentials are pass-through, not injected.** Its own docs: *"GitProxy does
   not use your Personal Access Token other than to authenticate with GitHub
   when pushing code."* The developer supplies the PAT at push time. **The token
   still lives in the pod** — the single thing BOBBIN exists to prevent. It
   solves the policy half and none of the credential half.
2. **Push-only, built around human approval.** *"All pushes… require an approval,
   and until a push is approved, GitProxy will block the commits."* That is a
   compliance-review workflow for regulated finance. A fleet that needs a human
   to click approve on every push is not a fleet. Clone/fetch is not proxied.
3. **TypeScript/Node + MongoDB** — heavier operational surface, in a stack we do
   not otherwise run.

Worth taking from it: the deployment model. Workers repoint the remote at the
proxy rather than being transparently intercepted, which validates that git
clients tolerate the approach fine.

## 4. How the Kubernetes ecosystem actually does this

- **Client-side credential helper / askpass — dominant.** Argo CD is the best
  instance: it passes only a UUID nonce plus a Unix socket path, and the
  credential is fetched over gRPC. **But it is not a security boundary against a
  hostile workload** — anything that can reach the socket gets the credential.
  It defends against *accidental* leakage (a Kustomize bug that dumped env vars
  into manifests). Our adversary is a prompt-injectable agent. Different
  property; do not cite it as precedent for safety.
- **Tekton** materialises `~/.git-credentials` containing `https://user:pass@host`
  into the workspace — plaintext token in the workload, the worst case here.
  **Jenkins X** does likewise and is decaying (docs last updated 2021).
- **Short-lived minted tokens — the clear direction of travel**, but
  GitHub-App-shaped: Flux exchanges a signed JWT for a 1-hour installation
  token; External Secrets has a `GithubAccessToken` generator. **No Vault or
  OpenBao secrets engine exists for Gitea/Forgejo** — the integration runs the
  other way.
- **HTTP-layer proxying with server-side injection is genuinely rare** — only
  `dependabot/proxy` (OSS, fetch-only) and Anthropic's (closed). Both were built
  by organisations running **untrusted code**. That is why it is rare: it is what
  you reach for when the workload is the adversary, and almost no CI system
  treats its own workloads that way. **This fleet does meet that bar.**

Two primitives to adopt regardless of which path we take:

- **`git-sync`'s `--askpass-url` contract** — a URL returning `200` with
  `username=…` / `password=…` lines. A `git-credential`-shaped broker endpoint
  that already has a maintained client in the Kubernetes org.
- **Projected ServiceAccount tokens + `TokenReview`** for pod→BOBBIN auth.
  Audience-bound, auto-rotated, and revoked when the Pod object dies. Do not
  invent a credential format; do not use a static shared bearer.

## 5. Security review — with recent CVEs, all reachable from this design

- **Redirect-following defeats the allowlist — the top risk.**
  **CVE-2026-57894** (2026-07-21, CVSS 8.5, Gitea <1.27.0): Gitea validated a
  migration URL against an allowlist, then shelled out to `git`, which follows
  the first redirect by default. Validation passed, git followed, internal
  repos were exfiltrated. **That is exactly BOBBIN's shape** — validate path,
  then forward with an injected credential. Also **CVE-2026-41506** (go-git
  reused credentials against a new host after a cross-host redirect).
  `httputil.ReverseProxy` does not follow redirects, which is the safe default —
  but `Location` headers must still be rewritten or rejected in
  `ModifyResponse`, and no credentialed `http.Client` may follow redirects.
- **Query-parameter smuggling defeats a service-based decision.**
  **CVE-2022-2880**: ReverseProxy forwarded raw query parameters, so the proxy
  and the upstream could parse different values from one string. Using `Rewrite`
  fixes it — unparsable parameters are removed *before* the hook runs.
- **Header smuggling** — `Director` runs before hop-by-hop stripping, so
  `Connection: Authorization` deletes the injected credential. Use `Rewrite`.
  Also keep the Go toolchain current (**CVE-2025-22871**, bare-LF chunk
  smuggling).
- **Path traversal / allowlist confusion** — `URL.Path` is decoded, `RawPath`
  is not. Match a canonical decoded identifier, then **reconstruct the upstream
  path from the allowlist entry** rather than forwarding the client's string.
- **Forge-side enforcement has itself been bypassed recently.**
  **CVE-2026-28744** (2026-06-16, CVSS 8.1, Gitea <1.26.2): `CheckRepoScopedToken()`
  returns early unless `IsBasicAuth`, so sending the same token as
  `Authorization: Bearer` **bypassed repository scope checks entirely** on the
  git path. Two months old. This cuts *for* defence-in-depth — and it
  independently corroborates the Basic-vs-Bearer finding in the Forgejo research.
- **Reverse proxy beats forward MITM** — no CA to distribute into every pod's
  trust store, no `HTTP_PROXY`-respect problem. This is the design's biggest
  architectural advantage; do not trade it away.
- **Operationally**, BOBBIN is a new SPOF on every git operation in the fleet,
  plus a new TLS surface and a new thing to patch.

## 6. Go implementation notes

`httputil.ReverseProxy` **is** adequate for git streaming — the folklore that it
is not concerns SSE. Git responses are chunked with `ContentLength == -1`, which
triggers immediate flushing. Rules: use `Rewrite` + `ProxyRequest` (never
`Director`), set `FlushInterval: -1` belt-and-braces, never buffer in
`ModifyResponse`, and give the `Transport` no redirect-following and no
`ResponseHeaderTimeout` that slow pack generation would trip.

**Code to read, in order:** `dependabot/proxy/internal/gitproto` (MIT — note its
`IsUploadPackRequest` requires **all three** of method, path suffix, *and*
`Content-Type` to match, deliberately so an unrelated POST sharing the suffix is
not routed through); `harness/harness app/router/git.go` (Apache-2.0, cleanest
route table, explicitly stubs every dumb-protocol path); GitLab Workhorse
(pre-authorize-then-stream, where one callback returns repo + allowed ops +
credential and the proxy holds no policy of its own); Athens' download-mode
globs; and `git-lfs-authenticate`'s capability model
(`{href, header, expires_in}` scoped to one repo *and* one operation).

**Licence hazard: copy from Gitea (MIT), never Forgejo (GPL-3.0-or-later since
v9.0).** The two `githttp.go` files are near-identical, so there is no reason to
take the GPL one.

## 7. Recommendation

### Step 0 — the experiment that decides the size of this

On a Forgejo ≥v15, create a token with `read:repository` restricted to one repo,
then confirm: clone works; push is rejected; a second private repo is
unreachable; and `POST /info/lfs/objects/batch` with `operation: upload` is
rejected. **This single test decides whether BOBBIN is ~200 lines or ~2,000** —
and given CVE-2026-28744, it must be tested rather than assumed.

**It cannot be run against our server today, which is v10.0.0.**

### If Forgejo is upgraded and the experiment passes — build a broker, not a proxy

A small Go controller that mints a repo-scoped token per worker pod, serves it
over the **`git-sync --askpass-url` wire format**, authenticates the pod via
**projected SA token + `TokenReview`**, and revokes on teardown.

Allowlist and fetch/push policy are then enforced **by Forgejo**: no git
protocol parsing, no TLS termination, no SPOF on the critical path, no LFS gap,
and none of the redirect/smuggling/traversal surface in §5. The residual
exposure is a never-expiring but single-repo, minimally-scoped token present for
the pod's lifetime, mitigated by mint-on-start/revoke-on-stop.

One problem to solve: the minter needs a user password or site-admin `Sudo:`
rights — a credential *more* powerful than the PAT it replaces. Isolate it in
its own namespace and ServiceAccount.

### Build the reverse proxy only if one of these holds

1. **Push policy must be finer than Forgejo can express** — per-branch,
   per-path, or commit-content. This is exactly why Anthropic built theirs.
   Check Forgejo branch protection first.
2. **A never-expiring token in the pod is unacceptable**, and mint/revoke is not
   sufficient mitigation.
3. **One audited chokepoint** for all fleet git traffic is itself the goal.

Before writing Go, spend an hour on **Envoy `credential_injector`** — route
allowlist plus a generic credential from an SDS secret plus a deny route is
BOBBIN v0 with no new code. But Envoy's own docs caveat it as *"functional but
has not had substantial production burn time"* and *"should only be used in
deployments where both the downstream and upstream are trusted"* — and our
downstream is explicitly untrusted. Treat it as a spike, not a destination.

### Reject outright

finos/git-proxy (pass-through credentials + human approval), Tekton creds-init
(plaintext credentials in the workload), Jenkins X (2021-era docs, rebranding
away), **Argo CD askpass as a security boundary** (it is not one against a
hostile workload), oauth2-proxy (forwards identity, injects nothing),
Sourcegraph (private since 2024), goblet (archived).
