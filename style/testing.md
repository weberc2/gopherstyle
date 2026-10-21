# Testing

Load this when writing or changing tests, adding code that carries a
mechanical contract (allocation-free, bounds-check-free), or reimplementing a
protocol or format that a mature implementation already parses. The
[core](core.md) rules still apply.

## Tiers: what a test runs against

Every change ships tests that prove it is actually delivered. Tests are
layered by *what they run against*:

> logic → unit; real backend semantics → integration.

The unit tier is the bulk: fakes (an in-memory `net.Conn`, an `httptest`
server) exercise branch logic exhaustively — state transitions, framing,
failure classification — and run under a bare `go test ./...`. When a package
needs a real backing service, its tests take a build tag
(`//go:build integration`), run under `go test -tags integration ./...`, and
**skip (don't fail)** when their endpoint's environment variable is unset, so
the default tier stays fast and dependency-free.

Prefer **table-driven tests**: one `[]struct{ name string; ... }` slice run
under `t.Run(tc.name, ...)`, so a new case is one row and failures name
themselves. This suits the unit tier's exhaustive branch coverage especially.

## Contracts are tests, not comments

When the design promises a mechanical property, a test holds it:

- **Allocation contracts** are asserted with `testing.AllocsPerRun` in a unit
  test and again in benchmarks (`TestDoAllocs`, `TestTokenizerAllocs`). Keep
  the peers used in those measurements allocation-free themselves, since the
  count is process-wide.
- **Bounds-check contracts**: hot loops meant to be free of bounds checks are
  held to it by a test that reads the compiler's report
  (`go build -gcflags=-d=ssa/check_bce` output — see `TestNoBoundsChecks` in
  the reference packages).

## Fuzzing: differential, seeded from other implementations

When we reimplement a protocol or format that a mature implementation already
parses (the JSON tokenizer against `encoding/json`; the HTTP/1.1 client
against `net/http`), fuzz it **differentially**: feed the same bytes to both,
require identical results, and enumerate every deliberate divergence as a
named reason that the oracle verifies against the input bytes — never a
blanket "we may be stricter". Seed the corpus from other projects' test
suites with a `//go:build ignore` generator (`corpusgen.go`) that writes
`testdata/corpus/*.seeds`, so every edge case those projects found runs on
every `go test`. Fuzz targets, seed loaders, and oracles live in `*_test.go`
next to the code; run them with
`go test -fuzz=FuzzName -fuzztime=5m ./pkg/<x>/`.
