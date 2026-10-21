# gopherstyle

TigerStyle for Go: all memory and goroutines preallocated at startup,
fixed-size types, assertions for invariants, and tagged result unions instead
of `error` values threaded through callers. We are building elegant, reliable
code. We value security, consistency, predictability, and reliability first
and foremost; efficient use of resources is a welcome benefit when it doesn't
require trading off our higher values.

This repository is the shareable home of that guidance. Point an agent (or a
person) at it from any Go project and they adopt these practices without
copying the documents into the project.

## How to use this repo from a project

Add a paragraph like this to the project's `AGENTS.md` (or
`CONTRIBUTING.md`):

> For all Go code, follow the practices in
> https://github.com/weberc2/gopherstyle. Start at `style/README.md` and load
> only the documents whose "load when" condition matches your task.

Agents can clone the repo, or fetch individual documents raw:

```
https://raw.githubusercontent.com/weberc2/gopherstyle/master/style/README.md
```

## Index

Each directory carries its own `README.md` index, so a reader loads one
index at a time and descends only where relevant. Every index entry says when
to load the document it names — the index exists as much to tell you what
*not* to read as what to read.

- [style/](style/README.md) — the style guides. Start here.
- [AGENTS.md](AGENTS.md) — how this repository itself is maintained. Read it
  before changing anything here; skip it if you're only consuming the guides.

## Standard library (planned)

The style depends on a small family of support packages — `assert`
(crash-through invariants), `enum` (text lookup behind int-backed enums, and
`enumtest.Complete` to hold the tables honest), `fail` (the closed set of
operational failure causes result unions carry), `static` (fixed-size
strings) — and reference implementations that demonstrate the style
end-to-end: a streaming allocation-free JSON tokenizer, a zero-allocation
HTTP/1.1 client, and an allocation-free SigV4 request signer. These are
slated to move into this module as its standard library, so a project
adopting the style can import `github.com/weberc2/gopherstyle/pkg/...`
instead of reimplementing the scaffolding. Until they land, the style docs
refer to them by package name.
