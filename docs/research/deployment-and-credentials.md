# Deployment and Credential Sourcing on agent-sandbox

Research pass 2026-08-08, verified against the live cluster with read-only
`kubectl` (plus `exec`) and live Forgejo. Nothing was created or applied.

Items below marked **VERIFIED** were measured, not inferred. Re-check before
relying on them — a Spot cloudspace changes under you, and one node in this
pool had already turned over within 8 hours at probe time.

## Verified environment facts

| Fact | How it was established |
|---|---|
| Cluster is **not** ArgoCD-managed | No `argocd.argoproj.io/*` on any namespace; every install is raw helm (`sh.helm.release.v1.{traefik,authentik,openbao,…}`) or plain `kubectl apply`. `k8s/agent-sandbox/` exists in declarative-config but is inert — nothing globs it and none of it is on the cluster. |
| No SealedSecrets, ESO, cert-manager, Tailscale | CRD count for each → **0** |
| **Calico genuinely enforces NetworkPolicy** | Timing probe to a closed port: netpol-selected pod **dropped** (6005 ms timeout, exit 124); non-selected pod **refused in 9 ms**. Enforcement is real, not decorative. |
| rs-manager OpenBao unreachable from pods | `curl …:8200/v1/sys/health` from a pod → **000**; from the EX44 → **200** |
| In-cluster `openbao-experiment` reachable and unsealed | `/v1/sys/health` → 200; `Sealed false`, `Seal Type shamir`, `Storage file` |
| The 2026-08-02 SA-JWT rig still works | Live login from `needle-pod/bao-test` → token issued, `ttl=3600`, policy `needle-pod-ro`; allowed path read OK, other paths denied |
| `git.ardenone.com` reachable from pods | Resolves and reaches Cloudflare (`cf-ray: …-ORD`) |
| `ronaldraygun/*` is **private** on Docker Hub | Anonymous `tags/list` → **401**. An `imagePullSecret` is mandatory. |
| Nodes are Spot and churn | Both `servers.ngpc.rxt.io/type=spot`; ages 11d and **8h** at probe |
| Worker runs as the **`default`** SA | Not `needle-pod` — matters for any future TokenReview identity work |

## Protocol- and server-level findings that shape the policy layer

These are the findings that change the design, all measured against live Forgejo:

1. **Forgejo requires auth for FETCH, not just push.** Anonymous
   `info/refs?service=git-upload-pack` → **401**. So BOBBIN must inject the
   credential on *both* services; a design that only injects for
   `git-receive-pack` breaks every clone.

2. **Path matching is case-insensitive and the `.git` suffix is optional.**
   `JedArden/Warden.git/info/refs` → **200**, and `jedarden/warden/info/refs`
   (no `.git`) → **200**. An exact-string allowlist is therefore **trivially
   bypassable by changing capitalisation**. Canonicalisation is mandatory:
   URL-decode → reject `..` and `%2f` → strip a trailing `.git` → lowercase →
   compare against a lowercased allowlist. This is the single most important
   correctness detail in the policy core, and it is empirical, not theoretical.

3. **Protocol v2 is live** — `info/refs` advertises `version 2`, `ls-refs`,
   `fetch=shallow wait-for-done`. BOBBIN must forward the `Git-Protocol`
   request header; dropping it silently downgrades every clone to v0.

4. **Upstream 401s carry `WWW-Authenticate: Basic realm="Gitea"`.** That header
   must be stripped rather than passed through: git would attempt its
   credential helper and, on any client without `GIT_TERMINAL_PROMPT=0`, hang.
   Deny with a plain 403 and a legible body.

5. **Cloudflare bounds long requests** — documented fleet history of **413** on
   large bodies and **504 at roughly a 240 s per-request ceiling**. BOBBIN must
   set no response timeout on the upstream client (only `ReadHeaderTimeout`
   server-side), stream both directions, and pass 413/504 through verbatim so
   the failure is diagnosable rather than looking like "BOBBIN broke".

6. **Forgejo's pre-receive hook rejects any blob >100 MB** unless the commit
   message contains `size:allow`. The upstream error body must be forwarded
   byte-for-byte — that message is how an agent discovers the escape hatch.
   Swallowing it turns a self-explaining failure into a mystery.

## Credential sourcing — ranked

**Recommended: a plain Kubernetes Secret, materialised once from OpenBao on the
EX44.**

```
EX44 (has Tailscale) → read secret/rs-manager/agent-sandbox/forgejo/token
                     → kubectl create secret generic bobbin-forgejo-credentials
```

It wins on three grounds: zero new dependencies; it lives in etcd so Spot
preemption does not affect it; and it is structurally identical to warden's
`warden-spot-credentials`, consumed via `envFrom: secretRef`. **Choose the
Secret name and keys now so that when M0 lands, an `ExternalSecret` producing
the same name and keys replaces it with zero change to the Deployment.** That
substitution property is the design decision worth locking in.

Stated honestly: the token sits at rest in etcd, unrotated and not represented
in git. That is still strictly better than today, where the same token is in a
Secret *and* written to `$HOME/.git-credentials` inside a pod running
model-generated commands.

**Rejected — in-cluster `openbao-experiment` + Kubernetes auth.** The rig
genuinely works today (proven live). It is disqualified because it has a
**shamir seal and no auto-unseal**: its config has no `seal` stanza, transit
auto-unseal was prepared but never migrated, and it runs on a Spot node in a
pool that demonstrably churns. A restart brings it back **sealed**, putting
every worker's ability to clone behind a manual 3-of-5 unseal. It is also
explicitly slated for teardown in needle-pod's own notes, and would cost real
code (login, 1 h renewal, re-read on failure).

**Rejected — SealedSecrets.** Zero CRDs here, and a SealedSecret is encrypted to
*the target cluster's* controller key, so a controller elsewhere cannot help.
This is "install a new controller", not "find the existing one".

**Rejected for now — wait for M0.** The right end state, but weeks-scale and
human-gated. Build so M0 is a drop-in substitution instead of a blocker.

## What to copy from warden

warden is ~849 lines of stdlib-only Go. Copy the split
(`cmd/` + `internal/{config,policy,proxy,server,audit}` + `deploy/`) and these
behaviours verbatim:

- **The credential never leaves the upstream client package.** Only that layer
  sets the `Authorization` header, so the server layer cannot leak what it does
  not hold.
- **Config from env, fail-closed, collecting *every* error** before refusing to
  start.
- **Caller auth by SHA-256 digest with `subtle.ConstantTimeCompare`**, logging
  only a 12-hex fingerprint, never the token. ~30 lines, lift as-is.
- **Audit every decision, allow and deny**, one structured line with a stable
  key set.
- **`/healthz` unauthenticated and outside the auth middleware.**
- **The hardened security context verbatim** — distroless `static:nonroot`,
  `runAsNonRoot` *plus* explicit `runAsUser: 65532` (kubelet cannot verify a
  non-numeric user), `readOnlyRootFilesystem`, all capabilities dropped,
  `seccompProfile: RuntimeDefault`. Raise the memory limit to ~256–512 Mi,
  since BOBBIN streams packfiles.
- **`VERSION` file drives a pinned semver image tag.**

Do **not** copy warden's ingress (BOBBIN needs none), its ExternalSecret (no ESO
here), or its `deploy/README.md` instruction that nothing is applied with
kubectl — that rule is true for warden on rs-manager and **false** for BOBBIN on
agent-sandbox. State the opposite explicitly, with the raw-helm precedent, or a
future agent will refuse to deploy it.

One place BOBBIN must be *better*: warden decodes bounded JSON bodies. BOBBIN
must stream, and gets no prior art from warden for it.

## Deployment shape

Namespace `bobbin`, Deployment, ClusterIP Service on 80→8080. **No ingress** —
and that is a security requirement, not a simplification: cloudflared routes
exactly two hostnames and 404s everything else, so adding BOBBIN would expose a
credential-injecting proxy to the public internet. Workers reach it at
`http://bobbin.bobbin.svc.cluster.local`.

- **No PVC**, so the `sata`/`ssd` rule never engages. Do not add a cache volume.
- **2 replicas with pod anti-affinity.** Spot preemption is active; a single
  replica means every in-flight clone dies on reschedule. Nearly free at ~30 MiB.
- **`imagePullSecrets: [docker-hub-registry]` mandatory** — copy the existing
  Secret from `needle-pod`.
- **NetworkPolicy**: default-deny ingress, allow from
  `namespaceSelector: kubernetes.io/metadata.name=needle-pod` on 8080; egress
  restricted to DNS and 443.

Image build: local Docker on the EX44 for 0.1.0 (exactly how needle-worker
0.1.0–0.1.7 were produced), then a copy of `warden-build.workflowtemplate.yaml`
on iad-ci. Do **not** use the generic `container-build` template — it clones
from GitHub and expects a `containers/<name>/` layout BOBBIN does not have.

## Caller authentication

**NetworkPolicy as the primary control, warden's shared bearer for attribution.**

The bearer is explicitly *not* a defence against the agent — an agent running
model-generated commands can read anything its pod can read, which is precisely
why the Forgejo PAT must not be there. Its value is audit attribution,
defence-in-depth, and that the implementation already exists. Crucially, what
the pod then holds is a token valid only against an in-cluster ClusterIP and
bounded by BOBBIN's allowlist — categorically weaker than an account-wide,
non-expiring Forgejo PAT.

Per-pod ServiceAccount identity is a clean phase 2: the worker already has a
**projected, auto-rotating** SA token (verified: re-stamped ~49 min into the
pod's life), `create tokenreviews` is permitted, and the `system:auth-delegator`
pattern is already proven on this cluster. A git credential helper that reads
the projected token gives BOBBIN a fresh kubelet-issued credential per git
invocation — something a once-written `.git-credentials` can never provide.

## Cutting the worker over

The needle-worker entrypoint is already most of the way there:

- `configure_git()` sets `core.askPass /bin/true` and `GIT_TERMINAL_PROMPT=0`
  **unconditionally**, so "git must never prompt" is already satisfied — a
  denied request fails fast rather than hanging (reproduced live).
- Dropping `NEEDLE_POD_GIT_TOKEN` makes it write no `.git-credentials` and set
  no helper. Because anonymous Forgejo is 401 even for reads, that is
  **fail-closed by construction**: a worker that bypasses BOBBIN cannot clone.

Changes needed:

1. `NEEDLE_POD_GIT_BASE_URL` → `http://bobbin.bobbin.svc.cluster.local/jedarden`
2. Delete `NEEDLE_POD_GIT_TOKEN`/`NEEDLE_POD_GIT_USER` and the
   `needle-worker-git` Secret, then **revoke that PAT** — it has been in a
   pod's `$HOME` and cannot expire.

**One real bug to fix in needle-pod first:** `lib/workspaces.sh` hardcodes the
scheme when writing the credential file:

```bash
printf 'https://%s:%s@%s\n' "$user" "$NEEDLE_POD_GIT_TOKEN" "$host"
```

Git's `store` helper matches on **scheme + host**, so an `https://` entry will
not match an `http://bobbin…` clone — every clone would 401. Derive the scheme
from `NEEDLE_POD_GIT_BASE_URL`.

Also: `clone_workspaces()` accepts a full URL as a workspace entry
(`case "$repo" in *://*)`), which would bypass BOBBIN entirely and then fail
hard. Forbid full URLs, or rewrite them to the BOBBIN base.

## Watch items

- **IPv6 ordering** — `getent hosts git.ardenone.com` returns AAAA first in-pod
  while pod networking is IPv4-only with `natOutgoing`. `curl` succeeded via
  happy-eyeballs, but Go's dialer may stall on the v6 attempt; consider
  `FallbackDelay` tuning or forcing `tcp4` if latency appears.
- **Interrupted pushes** can leave `receive-pack`/`index-pack` running and a
  `tmp_objdir-incoming-*` quarantine directory on the Forgejo side — a known
  fleet failure mode that BOBBIN will now sit in the path of.
