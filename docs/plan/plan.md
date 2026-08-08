# BOBBIN Plan

## Overview

A credential-injecting git proxy for the needle-pod worker fleet. Workers clone
and push through BOBBIN; BOBBIN holds the Forgejo credential and enforces which
repos may be read and which may be written.

Its reason for existing is a specific, measured gap: worker pods currently mount
a Forgejo `write:repository` token that **cannot expire** and is **account-wide**,
into the environment of a process running model-generated commands. See
`../notes/features.md` for the full statement of the problem.

> **Status: design in progress.** Four research tracks are open (protocol,
> Forgejo server behaviour, prior art, deployment/credential sourcing). Sections
> marked _pending research_ will be settled by their findings — in particular,
> whether BOBBIN is written at all or an existing proxy is adopted.

## Architecture

```
needle-pod worker (no credential)
        │  git clone / push over plain HTTP, ClusterIP
        ▼
     BOBBIN ── policy: repo allowlist + fetch/push
        │   ── injects Forgejo credential
        ▼
  git.ardenone.com (Forgejo, via Cloudflare)
```

Three properties define it:

1. **The pod holds nothing.** No token, no credential helper, no `.git-credentials`.
2. **Policy is by construction.** BOBBIN forwards only what its configuration
   permits; a caller cannot widen its own scope through any request it can make.
3. **The wire is untouched.** Every byte of pkt-line and packfile passes through
   unaltered and unbuffered, so a stock `git` client cannot tell the difference.

### Precedent

`warden` is the same shape for the Rackspace Spot API: a small Go service holding
an org credential agents must never see, with a pure, exhaustively tested policy
core and an intent-level API that makes forbidden operations unrepresentable.
BOBBIN should copy its structure — and, like warden, is intended for eventual
absorption into SEAM rather than permanent independent existence.

## Components

1. **HTTP front end** — accepts git smart-HTTP from in-cluster callers.
   Streaming, no buffering, faithful to the protocol. _Exact requirements
   pending research._

2. **Policy core** — pure, no I/O, exhaustively tested. Input: caller identity,
   repo path, service (fetch/push). Output: allow or deny with reason. Modelled
   on warden's `internal/policy`.

3. **Credential injector** — attaches the Forgejo credential to the upstream
   request and guarantees it cannot leak back to the caller, including via
   redirects. _Credential source pending research_ — the usual fleet answer
   (ExternalSecrets from rs-manager OpenBao) is unavailable, because that
   OpenBao is Tailscale-only and agent-sandbox has no Tailscale operator.

4. **Audit log** — one structured record per decision, matching the single-line
   JSON shape needle-pod already emits.

5. **Deployment** — Deployment + ClusterIP Service on agent-sandbox. No ingress:
   exposing a service that holds a write-scoped credential is the thing to
   avoid. `kind: Job`/`CronJob` are banned fleet-wide; an image tag must be a
   pinned semver, never `:latest`.

## Data Models

None persisted. BOBBIN is stateless: configuration in, decisions and logs out.
Anything that would require state (rate accounting, caching) is out of scope.

## Implementation Phases

- [x] **P0 — Research.** Protocol requirements, Forgejo specifics, prior art,
      deployment and credential sourcing. **Complete** — four tracks, all in
      `../research/`.
- [ ] **P1 — Decide the shape.** No longer "build vs adopt": nothing is
      adoptable (every credible implementation hides its git-HTTP logic under Go
      `internal/`), so the real fork is **broker vs proxy**, and it depends on a
      decision outside this repo. See "The decision that sizes this project"
      below. **Blocking.**
- [ ] **P2 — Policy core.** Pure, tested first, before any networking.
- [ ] **P3 — Proxy with credential injection.** Verified against a real clone
      and a real push, including a large repo and a shallow clone.
- [ ] **P4 — Deploy to agent-sandbox.** Direct manifests; the cluster is not
      ArgoCD-managed, matching the openbao-experiment precedent.
- [ ] **P5 — Cut needle-pod over.** Point `NEEDLE_POD_GIT_BASE_URL` at BOBBIN,
      remove `NEEDLE_POD_GIT_TOKEN`, delete the pod's git Secret. Confirm the
      worker still clones and pushes with no credential of its own.
- [ ] **P6 — Revoke the direct token.** The change is only real once the
      account-wide token is gone from the cluster.

## The decision that sizes this project

Research turned the central question from "build or adopt" into something
sharper, because **nothing is adoptable** — every credible implementation buries
its git-HTTP logic under Go `internal/`, so reuse means reading code, not
importing it.

The real fork is **broker vs proxy**, and it hinges on the Forgejo version:

**Forgejo v15.0 (2026-04-16) shipped per-repository scoped tokens**, enforced on
the git smart-HTTP path, not just the API. On such a server, the per-repo
allowlist and the fetch/push split are enforced **by Forgejo** — and most of
this repo's reason to exist evaporates.

`git.ardenone.com` runs **10.0.0**. Current stable is 16.0.2.

| | **Broker** (needs Forgejo ≥v15) | **Proxy** (works on v10 today) |
|---|---|---|
| Size | ~200 lines | ~2,000 lines |
| Enforces policy | Forgejo | BOBBIN |
| Git protocol parsing | none | all of it |
| On the critical path | no | **yes — new SPOF** |
| LFS gap | none | must block or rewrite |
| Attack surface from §5 research | none | redirect, smuggling, traversal |
| Residual exposure | a single-repo, minimally-scoped, non-expiring token in the pod for its lifetime | no token in the pod at all |

The broker mints a repo-scoped token per pod, serves it over the **`git-sync
--askpass-url` wire format**, authenticates the pod with a **projected SA token
+ `TokenReview`**, and revokes on teardown. Its one weakness: the minter needs a
user password or site-admin rights — a credential *more* powerful than the PAT
it replaces — so it must be isolated.

**Build the proxy anyway if** push policy must be finer than "which repo"
(per-branch or per-path — precisely why Anthropic built theirs), or a
never-expiring token in the pod is unacceptable even scoped to one repo, or one
audited chokepoint for all fleet git traffic is itself the goal.

**Before either, run the 10-minute experiment** (`../research/prior-art-and-build-vs-adopt.md`
§7): on a v15+ server, verify a repo-scoped token clones, cannot push, cannot
reach a second private repo, and cannot upload LFS objects. Given
**CVE-2026-28744** — where sending a token as `Bearer` instead of Basic bypassed
repo scope checks entirely on the git path, two months ago — this must be tested,
not assumed.

## Open Questions

- **Upgrade Forgejo, or build the proxy?** The question above, and the only one
  that matters right now. It is an infrastructure decision outside this repo.

### Resolved by research (see `../research/deployment-and-credentials.md`)

- **Credential source** → a plain Kubernetes Secret, materialised once from
  OpenBao on the EX44, consumed via `envFrom: secretRef`. Named so that an
  `ExternalSecret` producing the same name and keys replaces it under M0 with
  **zero change to the Deployment**. The in-cluster `openbao-experiment` was
  rejected despite working: shamir seal, no auto-unseal, on a churning Spot
  node — a restart would put every worker's ability to clone behind a manual
  3-of-5 unseal, and the rig is slated for teardown.

- **Caller authentication** → NetworkPolicy as the real boundary (Calico
  enforcement confirmed empirically), plus warden's shared bearer for audit
  attribution. Per-pod ServiceAccount identity is a well-scoped phase 2.

- **Deployment shape** → namespace `bobbin`, Deployment + ClusterIP, 2 replicas
  with anti-affinity (Spot churn is active), no ingress, no PVC,
  `imagePullSecrets` mandatory because `ronaldraygun/*` is private.

- **Git LFS** → **resolved: block `/info/lfs/` by default.** Not a compatibility
  question after all. Forgejo's batch endpoint copies the inbound
  `Authorization` header into the response body verbatim, so an untouched LFS
  request hands the injected PAT straight back to the pod. Rewriting batch JSON
  is possible but makes BOBBIN protocol-aware; blocking is honest and safe, and
  no repo in the current worker set uses LFS.

- **Should Forgejo be upgraded to ≥v15?** Per-repo token scoping does not exist
  in 10.x — it shipped in v15.0 (April 2026). That upgrade is the single
  highest-leverage change available to this threat model, because it turns the
  account-wide token into a genuinely per-repo credential and demotes BOBBIN's
  allowlist to defence-in-depth. Out of BOBBIN's scope to decide, but it should
  be on someone's list.

- **Should git move to a DNS-only hostname?** Cloudflare imposes a hard
  100 MiB request cap and a 125 s read timeout on the current hostname. A
  grey-clouded hostname for git traffic removes both. Cloudflare Tunnel is not
  a workaround — same class of limit.

- **Is `git-receive-pack` genuinely the only write path?** → **Resolved: NO,
  and this changes the policy model.** Confirmed write paths that bypass it:
  WebDAV via `git http-push` (observed on the wire — `GIT_SMART_HTTP=0 git push`
  issued `PROPFIND` and never touched `git-receive-pack`), LFS object `PUT`,
  Forgejo's REST API and web UI under the injected credential, and `{repo}.wiki`
  routed through the same handlers. Dumb HTTP is likewise a full read path that
  never touches `git-upload-pack`. The policy is therefore a **default-deny
  allowlist of (method, exact path shape)**, answering "is this a known read
  endpoint?" rather than "is this a push?".

- **Does this survive Cloudflare?** Partly answered: fleet history records
  **413** on large bodies and **504 at roughly a 240 s per-request ceiling**.
  BOBBIN must therefore set no response timeout upstream, stream both
  directions, and pass those statuses through verbatim. The remaining unknown is
  what repo size that bounds in practice.

### New requirements surfaced by research

- **Strip inbound `Authorization` headers and `token`/`access_token` query
  params.** Forgejo tries auth methods in order, first-user-wins, with OAuth2
  before Basic and query-param tokens still enabled. A caller supplying its own
  token would execute as itself while BOBBIN evaluated policy for another
  identity — the allowlist becomes advisory. This is a correctness requirement.
- **Inject with HTTP Basic, not `Authorization: token`.** Forgejo only enforces
  token scopes when `IsBasicAuth` is true; header-token auth 500s on LFS paths;
  and only Basic's path matching covers LFS.
- **Rate-limit in BOBBIN.** Forgejo has none, by deliberate design.
- **Strip `Set-Cookie` from responses.** Forgejo returns an `i_like_gitea`
  session cookie on git HTTP endpoints — a second credential-leak channel,
  independent of LFS.
- **Use `httputil.ReverseProxy`'s `Rewrite`, never `Director`.** `Director` runs
  *before* hop-by-hop header stripping, so a caller sending
  `Connection: Authorization` makes Go delete the credential BOBBIN just
  injected. `Rewrite` also drops client-supplied `X-Forwarded-*`.
- **Set `FlushInterval = -1` explicitly**; it currently auto-flushes only
  because git responses happen to be chunked.
- **Decide allow/deny before reading the body** — ideally at `info/refs`, so a
  rejected push never uploads a pack. Go reads at most 256 KiB of an unread body
  before closing, which turns a 403 into an opaque transport error.
- **Pin the upstream host and never relay a 3xx.** Observed: git re-bases its
  URL on redirect and sends every subsequent request directly to the new host,
  silently leaving the proxy.
- **Never set `Content-Length` on a proxied push.** Cloudflare enforces its
  100 MiB cap pre-emptively off that header; real pushes are chunked and dodge
  the check, so buffering re-arms it and turns a working push into a 413.
- **Canonicalise repo paths before matching.** Forgejo matches
  case-insensitively and treats `.git` as optional (both verified live), so an
  exact-string allowlist is bypassable by capitalisation. This is the highest-risk
  detail in the policy core.
- **Forward the `Git-Protocol` header.** Protocol v2 is live upstream; dropping
  the header silently downgrades every clone to v0.
- **Strip `WWW-Authenticate` from upstream 401s**, or git attempts its
  credential helper and — on any client without `GIT_TERMINAL_PROMPT=0` — hangs.
- **Forward upstream error bodies byte-for-byte on push failure.** Forgejo's
  pre-receive hook rejects blobs >100 MB with a message naming the `size:allow`
  escape hatch; swallowing it turns a self-explaining failure into a mystery.

### Prerequisite fix in needle-pod

`lib/workspaces.sh` hardcodes `https://` when writing `.git-credentials`. Git's
`store` helper matches on **scheme + host**, so that entry will not match an
`http://bobbin…` clone and every clone would 401. The scheme must be derived
from `NEEDLE_POD_GIT_BASE_URL`. Separately, `clone_workspaces()` accepts a full
URL as a workspace entry, which would bypass BOBBIN entirely — forbid it or
rewrite to the BOBBIN base.
