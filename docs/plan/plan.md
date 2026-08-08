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

- **Where does BOBBIN's own credential come from?** rs-manager OpenBao is
  unreachable from this cluster's pods. Candidates: a plain Secret, a
  SealedSecret, the in-cluster `openbao-experiment` instance with Kubernetes
  auth (a NEEDLE-pod-style SA-JWT login was proven on this cluster on
  2026-08-02), or waiting for Tailscale + ESO under needle-pod's M0. It must
  work today without M0.

- **How do callers authenticate to BOBBIN?** Nothing, a shared bearer, or
  per-pod ServiceAccount identity — partly determined by whether Calico is
  actually enforcing NetworkPolicy here.

- **Git LFS.** Support, or detect and refuse clearly? Silently breaking LFS
  repos is the outcome to avoid.

- **Is `git-receive-pack` genuinely the only write path?** The fetch/push
  distinction is the security boundary, so this needs proving rather than
  assuming.

- **Does this survive Cloudflare?** Long-running clone and push POSTs pass
  through Cloudflare to Forgejo; proxy timeouts and buffering behaviour there
  could bound the maximum usable repo size.
