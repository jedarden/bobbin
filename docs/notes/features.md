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

## Fetch and push are separate permissions

A worker that only needs to read should not be able to write, and that
distinction has to be enforced on the request itself.

Git makes this legible: the service is named in the `?service=` query parameter
on `info/refs`, and in the POST path (`git-upload-pack` = fetch,
`git-receive-pack` = push). BOBBIN keys its decision on those.

Note that the credential must be injected for **both** services, not just push.
Forgejo runs with `REQUIRE_SIGNIN_VIEW`, so an anonymous
`info/refs?service=git-upload-pack` returns 401 even for a public repo — a
design that only authenticates pushes would break every clone. The distinction
governs *authorisation*, not whether to authenticate.

This is security-critical and therefore the thing to get *provably* right: the
research task is to confirm there is no path by which a client can write without
transiting `git-receive-pack`. Until that is confirmed, treat it as unproven.

## Transparency to a stock git client

BOBBIN is only useful if `git clone` and `git push` work unmodified. No custom
client, no wrapper script, no special remote helper.

That means it must be faithful to the smart-HTTP protocol:

- Preserve pkt-line framing byte-for-byte, including the `# service=`
  advertisement and flush packets.
- **Stream, never buffer.** A clone or push body can be very large; buffering it
  breaks memory bounds and stalls sideband progress reporting. This is the
  requirement most likely to be violated by a naive `httputil.ReverseProxy`.
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

## Git LFS

Unresolved. LFS uses a separate batch API and object transfer, so a
smart-HTTP-only proxy may silently break any repo using it. Needs a decision:
support it, or detect and refuse it with a clear error rather than a confusing
partial failure. Under research.

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
