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

- [ ] **P0 — Research.** Protocol requirements, Forgejo specifics, prior art,
      deployment and credential sourcing. **In progress.**
- [ ] **P1 — Decide build vs adopt.** If a maintained proxy already does this
      safely, adopt it and write only the policy layer. This decision gates
      everything below and is a real fork, not a formality.
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

## Open Questions

- **Build or adopt?** The most consequential question, and deliberately first.
  Writing a git proxy means owning protocol edge cases indefinitely.

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

- **Is `git-receive-pack` genuinely the only write path?** The fetch/push
  distinction is the security boundary, so this needs proving rather than
  assuming. _Protocol research pending._

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
