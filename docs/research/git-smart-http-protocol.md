# Git Smart-HTTP: What a Transparent Proxy Must Do

Research pass 2026-08-08 against git documentation and source (`remote-curl.c`,
`http.c`, `http-backend.c`, `http-push.c`), Forgejo's route tables, Go's
`net/http` and `httputil` source, plus **empirical wire captures** from a local
`git-http-backend` harness driven by stock `git 2.47.3`. Items marked
*[observed]* were captured on the wire, not read from a spec.

The headline correction: **`git-receive-pack` is not the only write path.** The
feature note's original framing — detect a push and gate it — is unsafe. See §3.

## 1. The request sequences

**Fetch/clone (v0)** — `GET {repo}.git/info/refs?service=git-upload-pack`
(`200`, `Content-Type: application/x-git-upload-pack-advertisement`), then one
or more `POST {repo}.git/git-upload-pack`
(`application/x-git-upload-pack-request` → `-result`). Many POSTs may occur, one
per negotiation round, each an independent stateless request.

**Fetch/clone (v2, default since git 2.26)** — same URLs, but the request
carries `Git-Protocol: version=2` and each POST carries one command:
`ls-refs`, `fetch`, `object-info`, or `bundle-uri`. *[observed]* a trivial clone
is 1 GET + 2 POSTs. **All four v2 commands are read-only; v2 has no push
command at all** — `remote-curl.c` forces v0 for anything that is not
`git-upload-pack`.

**Push** — `GET …/info/refs?service=git-receive-pack` (no `Git-Protocol`; push
is v0 only), then optionally a **probe POST** with `Content-Length: 4` and body
`0000`, then the real `POST …/git-receive-pack`.

The probe is `probe_rpc()` — a 4-byte flush-packet body used to discover the
auth scheme before committing to an unrewindable chunked upload. *[observed]*
exactly as described. **BOBBIN must forward it and must authenticate it**, and
must not reject a POST whose body is only a flush packet (it is also a legal v2
`empty-request`).

Status codes worth preserving: `200`, `304` (permitted on `info/refs`, clients
must treat as `200`), `403` (exists but denied — also *required* for an
unrecognised service), `404`/`410` (absent; the spec forbids answering `200`),
`415` (wrong POST `Content-Type`).

## 2. pkt-line and the advertisement — do not touch the body

4 hex length bytes counting themselves; `0000` flush, `0001` delim, `0002`
response-end. Max data 65516, max line 65520. **"A pkt-line MAY contain binary
data… implementors MUST ensure pkt-line parsing/formatting routines are 8-bit
clean."**

Concretely, BOBBIN must not:

1. **Add, remove, or rewrite the `# service=` line.** Its presence is version-
   and server-dependent: `git-http-backend` omits it under v2, while Forgejo
   emits it unconditionally even under v2. The client tolerates both *only
   because both are byte-exact*. Any normalisation breaks one shape.
2. **Recompute pkt-line lengths or re-frame.** A ref name lives inside the
   length field; one changed byte desynchronises the stream.
3. **Touch the NUL-delimited capability list.** Stripping `side-band-64k` or
   `object-format=sha1` silently changes negotiation semantics.
4. **Strip or reorder the trailing `0000`.**
5. **Transcode.** No gzip/br re-encoding, no `Content-Type` rewriting, and never
   inject an HTML error page into an `application/x-git-*` stream.
6. **Filter refs.** Clients must not `want` an id absent from the
   advertisement; proxy-side hiding breaks that invariant.

Denials must happen **before any body byte is proxied**, with a non-200 status
and a `text/plain` body — *[observed]*, git prints such bodies as
`remote: <text>`, which is the correct channel for a policy message.

## 3. The write paths — plural

**Answer to the open question: no, `git-receive-pack` is not the only way to
write.** Over smart HTTP alone it is; but "over smart HTTP" is the wrong scope,
because the caller chooses the protocol.

1. **Smart HTTP v0/v2 — no other write.** The complete service table is three
   POST endpoints: `git-upload-pack`, `git-upload-archive`, `git-receive-pack`;
   only the last writes. v2's four commands are all reads.

2. **WebDAV — YES, confirmed empirically.** If the `info/refs?service=git-receive-pack`
   response is not a smart advertisement, `remote-curl.c` falls back to
   `push_dav()` → `git http-push`, which writes with `PROPFIND`, `MKCOL`, `PUT`,
   `MOVE`, `LOCK`, `UNLOCK`, `DELETE`. *[observed]*: `GIT_SMART_HTTP=0 git push`
   issued `PROPFIND` and **never touched `git-receive-pack`**. `git-http-push`
   ships in git 2.47.3. Forgejo does not implement DAV so it would fail — **but
   BOBBIN must not delegate that refusal upstream**, because a worker controls
   its own client.

3. **Git LFS — YES, and the most likely real bypass.** `PUT /info/lfs/objects/{oid}/{size}`
   writes object content; the lock endpoints mutate state. None of it touches
   `git-receive-pack`.

4. **Forgejo's REST API and web UI — YES.** BOBBIN injects a credential valid
   for the *whole instance*. If the proxied URL space is broader than the git
   endpoints, a worker can `DELETE /api/v1/repos/...`, create tokens, or drive
   the UI — all authenticated, none of it a git push.

5. **Wiki repos.** Forgejo routes `{repo}.wiki` through the *same* git handlers.
   `POST /{owner}/{repo}.wiki.git/git-receive-pack` writes to a **different
   repository** than `{repo}`. The allowlist must decide explicitly whether
   `<repo>.wiki` is in scope.

**Consequence: the policy must be an allowlist of (method, exact path shape),
default-deny.** The question to answer is *"is this one of the few known read
endpoints?"* — never *"is this a push?"*. Forgejo's own handler shows the right
default: anything unrecognised that is not GET/HEAD is treated as a write.

**Dumb HTTP is a full read path that never touches `git-upload-pack`.**
*[observed]*: `GIT_SMART_HTTP=0 git clone` completed using only `GET /info/refs`
(no query), `GET /objects/info/packs`, and hundreds of `GET /objects/{2}/{38}`
and `/objects/pack/pack-*.{pack,idx}` — all of which Forgejo serves. The
fallback triggers on **any `info/refs` response whose `Content-Type` is not
`application/x-$service-advertisement`**, so mangling that one header converts
every client to dumb HTTP and every push to WebDAV. Decide explicitly whether to
allow or deny the dumb routes; do not leave them unspecified.

### Path-matching hazards

- `.git` is optional against Forgejo — match both forms.
- Use **exact final-segment equality**, not `HasSuffix`: a repo literally named
  `git-receive-pack` yields `/owner/git-receive-pack`.
- Reject `..`, `.`, and empty segments **after** percent-decoding. Go's
  `net/http` does **not** clean `r.URL.Path`, so
  `/r.git/git-upload-pack/%2e%2e/git-receive-pack` decodes to a traversal while
  `RawPath` still ends in `git-upload-pack`. **Match and forward the same
  representation.**
- The `service=` param appears only on `info/refs`, and `remote-curl.c` appends
  it with `&` if the base URL already has a query — so parse `?a=b&service=…`,
  not just `?service=…`. The POST carries no such param: **classify POSTs by
  path only**, or `POST …/git-receive-pack?service=git-upload-pack` defeats you.

## 4. Streaming

**Request bodies.** The client buffers up to `http.postBuffer` (default
1,048,320 bytes) and sends `Content-Length`; above that it switches to
`Transfer-Encoding: chunked` preceded by the probe POST. *[observed]* with a
3 MB push. Note this host's `~/.gitconfig` sets `http.postBuffer=524288000`,
which suppresses chunking up to 500 MB — **a proxy tested only here will never
exercise the chunked path.** Test both.

Fetch POST bodies may be **`Content-Encoding: gzip`** (*[observed]*: a clone
sent `Content-Encoding: gzip`, `Content-Length: 218`). Anything parsing pkt-lines
out of a request body will see garbage. The correct answer is not to parse it.

`Expect: 100-continue` is *not* sent normally — git explicitly suppresses
libcurl's automatic header.

**Response bodies.** Servers stream: `git-http-backend` ends headers before
running the service and never sets `Content-Length`; Forgejo pipes stdout
straight to the `ResponseWriter`. Fetch responses use **side-band-64k** — 4-byte
length, 1-byte stream code (`1` pack, `2` progress, `3` fatal), payload.

**Why buffering is wrong**, beyond memory: `uploadpack.keepAlive` and
`receive.keepAlive` both default to **5 s**, emitting keepalives during
`pack-objects` preparation and post-push indexing precisely so intermediaries do
not consider the connection hung. A buffering proxy absorbs those keepalives and
then presents both its links with exactly the dead air they exist to prevent.
Progress reporting also disappears, and latency doubles.

**Timeouts.** The git client has **no default overall timeout**
(`http.lowSpeedLimit`/`Time` unset), so it waits indefinitely while bytes
trickle. Proxy timeouts must therefore be **per-read, not per-request**, ≥60 s,
with absolute request deadlines disabled.

## 5. Authentication behaviour

Git sends the first request with **no** `Authorization`, and on 401 invokes
credential helpers then retries (up to 3×). *[observed]* with
`GIT_TERMINAL_PROMPT=0` and no helper: two unauthenticated GETs then
`fatal: Authentication failed`.

**So an upstream 401 must never be relayed to the worker.** It would point the
investigation at the *worker's* missing credentials rather than at BOBBIN's
rejected token. Translate to `502` (or `403`) with a `text/plain` explanation.

Injecting Basic unconditionally is correct, with four constraints:

1. **`Set`, never `Add`** — and unconditionally delete inbound `Authorization`
   and `Proxy-Authorization` first. Two `Authorization` headers is undefined.
2. **Never emit `401` yourself** — it invites a credential-helper retry loop
   against a proxy that will never accept client credentials. Use `403`.
3. **Authenticate the probe POST too**, uniformly.
4. **Strip `Set-Cookie` from responses.** *[observed live]* Forgejo returns
   `set-cookie: i_like_gitea=<session-id>` on git HTTP endpoints. **That is a
   session credential — handing it to the worker defeats BOBBIN entirely.** The
   spec explicitly permits stripping: servers must not require cookies to
   function. Strip inbound `Cookie` too.

### Redirects re-base the client

`http.followRedirects` defaults to `initial`. *[observed]*: a `302` on
`info/refs` pointing at another host produced
`warning: redirecting to http://…` and **every subsequent request went directly
to the new host** — the redirecting server saw one request for the whole clone.

Two consequences: any relayed upstream 3xx silently takes the client *outside
the proxy* (where it has no credential and probably no egress), and if BOBBIN's
upstream leg uses an `http.Client` that injects the credential inside a wrapping
`RoundTripper`, the header is re-added on **every** hop including an
attacker-controlled `Location`. Go's own stripping only protects headers set on
the initial request, and its subdomain rule would keep the header for
`evil.git.ardenone.com`.

**Requirement:** pin the upstream host, set `CheckRedirect` to
`http.ErrUseLastResponse` (or use a bare `Transport`), and convert upstream 3xx
into 502.

## 6. LFS, archive, shallow, partial

**LFS** uses entirely separate endpoints under `{repo}.git/info/lfs`, with Basic
auth via the same credential helper. If BOBBIN handles only smart-HTTP, a clone
of an LFS repo **appears to succeed** while leaving pointer files instead of
content — a confusing failure. If BOBBIN blanket-allows `/info/lfs/*`, it has
handed workers an authenticated write path. Batch `href` values may also point
at a different host under an external LFS backend, bypassing the proxy entirely.

**`git-upload-archive`** is read-only and v2-only, and **Forgejo does not route
it** — so `git archive --remote` fails regardless.

**Shallow/partial**: nothing structurally new, but `--depth` adds a round trip
(more requests, so per-request policy cost is paid more often), and `--filter`
turns the remote into a **promisor** — the repo makes additional
`git-upload-pack` POSTs lazily during ordinary commands like `git log -p`. Expect
fetch traffic long outside the clone window.

Two capabilities can hand the client **out-of-band URLs** whose objects never
traverse BOBBIN: v2 `packfile-uris` and `bundle-uri`. Forgejo does not advertise
them today; if that changes, strip the capability (a rare, deliberate exception
to §2) or block the feature.

## 7. Concrete Go failure modes

Each is a real defect in the obvious implementation.

- **`Director` lets the client delete the injected credential.** `ReverseProxy`
  calls `Director` *before* `removeHopByHopHeaders`, which deletes every header
  named in the request's own `Connection` header. A worker sending
  `Connection: Authorization` strips the header BOBBIN just injected. **Use
  `Rewrite`, never `Director`/`NewSingleHostReverseProxy`** — with `Rewrite`,
  hop-by-hop stripping happens first, and client-supplied `X-Forwarded-*` is
  dropped automatically.
- **`WriteTimeout`/`ReadTimeout` kill long transfers** (absolute, from request
  start). Set both to 0; use `ReadHeaderTimeout` and `IdleTimeout`, plus
  per-read deadlines via `http.ResponseController`.
- **`FlushInterval` defaults to 0.** It only auto-flushes because git responses
  are chunked (`ContentLength == -1`). Any upstream that sets a `Content-Length`
  falls back to buffering. **Set `FlushInterval = -1` explicitly.**
- **Rejecting a push mid-upload loses the error.** Go reads at most 256 KiB of
  unread body then sets `Connection: close`; the client reports a transport
  error rather than the 403 body. **Decide before touching `Body` — ideally deny
  at `info/refs` so the pack is never uploaded.**
- **`Content-Type` mutation yields a misleading 401.** Forgejo compares exactly
  and returns **401** (not 415) on mismatch, which git reports as an
  authentication failure — pointing the investigation the wrong way. Forward it
  byte-for-byte.
- **`http.ServeMux` path cleaning emits 301s**, which git follows and re-bases
  from. Route with a mux that does not clean; never emit 3xx.
- **Path decode/forward mismatch** between `r.URL.Path` and `RawPath` is an
  authorization bypass. Normalise once, reject anything that changes, then match
  and forward the same string.
- **Query re-encoding**: `ReverseProxy` may re-encode `RawQuery`, so **authorize
  on the outbound query, after rewriting**.
- **HTTP/2**: Forgejo answers h2, where there is no `Transfer-Encoding`. Branch
  on `ContentLength == -1`, never on the header.
- **`MaxIdleConnsPerHost` defaults to 2** — a worker fleet will thrash TLS
  handshakes. Use a dedicated `Transport` sized to expected concurrency.

## 8. Test matrix

Must include: protocol v0 **and** v2; `http.postBuffer` at the default (chunked
above ~1 MB) **and** at this fleet's 500 MB setting (never chunked); shallow and
`--filter` clones; a repo with LFS objects; an empty repo (`capabilities^{}`
advertisement); and a push large enough to cross the Cloudflare body limit.

## Sources

Git: `gitprotocol-http`, `-common`, `-pack`, `-capabilities`, `-v2` (git-scm.com
/docs); source `remote-curl.c`, `http.c`, `http-backend.c`, `http-push.c`,
`upload-pack.c`. Git LFS: server-discovery, batch, authentication,
basic-transfers, locking docs. Forgejo: `routers/web/repo/githttp.go`,
`routers/web/web.go`, `routers/common/lfs.go`. Go: `httputil/reverseproxy.go`,
`http/client.go` (`shouldCopyHeaderOnRedirect`), `http/server.go`
(`maxPostHandlerReadBytes`).
