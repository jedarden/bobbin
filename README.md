# BOBBIN

A credential-injecting git proxy. Worker pods clone and push through BOBBIN;
BOBBIN holds the Forgejo token and they never see it.

Named for the part that holds the thread and feeds the needle — which is
literally what it does for [NEEDLE](https://git.ardenone.com/jedarden/NEEDLE)
workers.

## Why

`needle-pod` runs autonomous coding agents as Kubernetes pods. Those agents
must clone repos and push their work, so today each pod carries a Forgejo
personal access token mounted as a Secret.

That token is the problem:

- **Forgejo PATs cannot expire.** The token object exposes only `id`, `name`,
  `scopes`, `sha1`, `token_last_eight` — there is no TTL field. A leaked token
  is valid until someone notices and revokes it.
- **`write:repository` is account-wide.** There is no per-repo PAT scoping, so a
  worker that only ever touches one repo still holds a credential that can push
  to every repo the account can reach.
- **The agent is the one holding it.** NEEDLE delegates pushing to the
  dispatched coding agent via prompt text, so the credential sits in the
  environment of a process that is executing model-generated commands.

BOBBIN removes the credential from that blast radius. Pods get no token at all;
they point `git` at BOBBIN, which injects auth server-side and enforces what
they are allowed to do with it.

This is [warden](https://git.ardenone.com/jedarden/warden)'s pattern — a policy
proxy holding a credential agents must never see — applied to git instead of the
Rackspace Spot API.

## Status

Research and design. No implementation yet.

## Structure

- `docs/notes/` — features, constraints, design decisions
- `docs/research/` — protocol, server, prior-art and deployment research
- `docs/plan/plan.md` — complete application plan
