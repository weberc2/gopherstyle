# External-API clients

House conventions for Go code that wraps an external HTTP API. Read this when
you're building one; skip it otherwise. It extends [core.md](core.md) — the
general rules there still apply, and in particular the failure rules: a
client is exactly the *adapter boundary* where `error` stops and `Cause`
result unions begin.

## Two client shapes — pick by how many backends you talk to

There are two constructor shapes for external-API clients, and they are not
interchangeable. Pick by whether the *set of backends is fixed at startup* or
*discovered at runtime*.

### Single backend, long-lived — the default for a service

A plain service (daemon, MCP server, CLI) talks to **one** backend whose
endpoint and credentials are known at startup. Construct **one long-lived
client once** and reuse it, using the ordinary convention from
[core.md](core.md) (*Initialization and arguments*): required arguments
positional, everything else a functional `Option`.

```go
func New(apiKey, model string, opts ...Option) Client { … }
func WithMaxTokens(n int64) Option { … }
func WithBaseURL(u string) Option { … }
```

There is **no `Factory` and no `Target`** here — those solve a problem this
shape doesn't have (see below). The client holds its own connection (or
`*http.Client`) and, for OAuth, its own token source, both good for the life
of the process.

### Many backends, resolved at runtime — the `Factory`/`Target` seam

A controller or operator that serves many resources, **each naming a
different backend connection** (different URL + credentials, resolved per
request or reconcile), uses a three-type seam instead of a bare constructor:

- an `API` interface (the operation subset callers need),
- a concrete `*Client`, and
- a `Factory` with `New(Target) (API, error)`, where **`Target`** is the
  per-resource *resolved connection* (`BaseURL` — no trailing slash — plus
  credentials).

The `Factory` is long-lived (one per controller) and builds a fresh
per-`Target` client each pass; crucially it lets those clients **share a
token cache across passes** so an expensive login isn't repeated per pass
(see *Auth caching*). If you are not resolving a backend per call, you do not
need this — use the single-backend shape above.

## Conventions common to both shapes

- **One package per backend, named after it** (`pkg/cloudflare`, `pkg/ebay`),
  opening with a package doc that pins the wire facts against the real API
  (base URLs, auth, quirks, rate limits).
- **An `API` interface** exposing only the operations callers need, so
  callers run against a fake in tests — fakes in `pkg/<x>/fake.go`, httptest
  doubles in `pkg/<x>/<x>test/server.go`.
- **An explicit timeout on every exchange** — nothing upstream imposes a
  per-call deadline for you. (With `net/http`, set `http.Client.Timeout`;
  30s is the house default.)
- **One private `do()` chokepoint.** Build the request, set auth, perform the
  exchange, decode the response. Always bound a body read into a diagnostic
  with a limit (`io.LimitReader` or a fixed buffer); bounding the success
  decode too is recommended against a hostile or oversized response.
- **Failure mapping lives in `do()`.** This is the adapter boundary from
  [core.md](core.md): map status codes and transport failures to the shared
  `Cause` enum (429 → `CauseThrottled`, 401/403 → `CauseAuthDenied`, 404 →
  `CauseNotFound`, 5xx → `CauseServer`, …), record the diagnostic detail
  (status, a bounded body snippet, provider request ID) in fixed memory at
  the point of failure, and return the operation's result union. Callers
  branch on `Cause`, never on strings.
- **Auth caching.** Never re-authenticate per call when auth is expensive.
  For an **OAuth2 client-credentials** backend, use
  `golang.org/x/oauth2/clientcredentials`: its `.Client(ctx)` returns an
  `*http.Client` whose transport mints, caches, and refreshes the token for
  you — in the single-backend shape that one client caches for the whole
  process; don't hand-roll a token cache. Hand-roll one (cache + expiry skew
  + single-flight + invalidate-on-401) only for a non-OAuth scheme *and* the
  `Factory` shape, where it lives in the `Factory`. (An uncached login flood
  once DoS'd a backend; this rule is written in that incident's ink.)
- **Retry / rate limits.** A controller's resync loop spaces retries for
  free, so its clients don't retry. A **poller or MCP server has no such
  loop** — there, honor `Retry-After` on 429 with bounded exponential backoff
  and a small cap, respecting `ctx`. State the choice (retry or not) in a
  comment; `Cause.Retryable()` says which failures are worth it.
- **Pagination.** For offset/limit + `total` APIs, loop until the accumulated
  count reaches `total`. Don't rely on a single oversized page.
- **Credentials.** Load through a redaction-safe path and keep the secret a
  string only at the last boundary before the wire. In a plain service, read
  from env/flags at startup.
