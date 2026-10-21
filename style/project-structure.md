# Project structure

Load this when creating a module or repository, or adding a binary or
importable package to one.

## Layout: `cmd/<program>` and `pkg/<package>`

Main programs live under `cmd/<program>/` (one directory per binary) and
importable packages under `pkg/<package>/`. Nothing importable lives at the
module root.

```
foo/
  cmd/
    foo/
      main.go
  pkg/
    json/
      tokenizer.go
    http1/
      conn.go
  doc/
    decision/
      0001-<slug>.md
    style/            # or a pointer to this repository
```

- Package names are short, all-lowercase, no separators: `namedreturn`, not
  `named_return` — and the same for `.go` file names. More than about two
  mashed-together words is a signal to split the package.
- Keep `main.go` thin: parse flags/env, construct the top-level values,
  delegate to a package. Logic in `main` can't be imported or tested.

## Separate modules for benchmark/comparison code

Code that exists to compare against third-party implementations (a `bench/`
directory pitting a package against popular libraries) is its own module, so
those dependencies stay out of the main module's `go.mod`. The main module
stays hostile to new dependencies; a comparison suite is where dependency
sprawl is allowed to live.

## Decision records

Significant design choices get a numbered decision doc under `doc/decision/`
(`0003-zero-allocation-http1-client.md`) stating the problem, the choice, and
the verification strategy. The style guides say *how* code looks; decision
docs say *why* a particular design exists.
