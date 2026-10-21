# Tooling

Load this when setting up a module's toolchain, running the
format/lint/check suite, or registering tool dependencies.

## Tools are Go tool dependencies — no global installs

Every tool is registered via Go 1.24+'s `go tool` mechanism (`go.mod`'s
`tool` block), so a fresh checkout has the whole toolchain after `go mod
download`. Register one with:

```sh
go get -tool <import path>@latest
```

## The suite

Run before handing code over, along with the tests:

```sh
go tool github.com/golangci/golines -m 80 -w --shorten-comments .
go tool exhaustive -ignore-enum-members 'Count$' ./...
go tool errcheck -ignoretests ./...
go test ./...
```

- **`golines`** enforces the 80-column rule and runs `gofmt` as part of its
  pass, so you don't run both. Use the maintained fork
  (`github.com/golangci/golines`, not the archived `segmentio/golines`).
- **`exhaustive`** enforces that switches over an enum name every member,
  with the count sentinels excluded (see [core.md](core.md),
  *Enumerations*).
- **`errcheck`** enforces that no `error` is dropped by accident inside
  adapter code (see [core.md](core.md), *Failures*). Tests are exempt
  (`-ignoretests`): their own assertions catch a test server that failed to
  write.
