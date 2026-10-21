# Go style — core

The core conventions for gopherstyle code. They began as a copy of `xfer`'s
guide and have been trimmed to what the style practices; where a convention
is inherited and has no example here yet, it says so. Extend this guide as
new conventions are established — but keep it a doc people read, not an
encyclopedia.

## Line length: 80 columns

Source lines stay under 80 characters, enforced by `golines` (which runs
`gofmt` as part of its pass, so you don't run both). Commands and tool
registration: [tooling.md](tooling.md).

## Naming

- **Default to UpperCamelCase for string names.** When you choose the textual
  form of a name (enum value, log key, field label, condition reason),
  default to `UpperCamelCase` ("User", "InputTokens"). Exception: when the
  name must match an external API or wire protocol, use whatever that API
  requires. The constraint is on *our* naming choices, not bytes we must send
  to someone else.
- **File names match the principal type**, lowercased: a file defining
  `Session` is `session.go`, not `chat.go` (even in package `chat`). A small
  cohesive cluster (`Store` interface + `MemStore` impl) is named after the
  concept (`store.go`). A reader guessing the file name should be right most
  of the time; package name and file name carry different information —
  don't duplicate.
- **Prefer `Noun` or `FetchNoun` over `GetNoun`.** A read/lookup reads better
  as the thing it returns (`conn.Response()`), or as `Fetch<Noun>` when it
  does I/O or can fail (`FetchItem`). The `Get` prefix is noise on our own
  names; reserve it for a form an external API forces on us.

## File ordering: BLUF, type-cohesive

Order declarations "bottom-line-up-front" — the highest-level declaration at
the top, its leaf dependencies below. A reader grasps the file's purpose from
the first declaration and descends only as they need detail. Keep a single
type's neighborhood contiguous (type, constructor, methods); don't interleave
another type's group between them.

```go
// Good — Conn's group is intact, then Option's, then leaf helpers.
type Conn struct { ... }
func (c *Conn) Init(host string, port uint16, scheme Scheme, opts ...Option)
func (c *Conn) Do(req *Request) DoResult { ... }

type Option func(*Conn)
func WithDialTimeout(d time.Duration) Option { ... }

func writeHostHeader(dst []byte, host string, port, def uint16) int  // leaf
```

## Enumerations: int-backed with text marshaling

Default to integer-backed enums with `iota` constants closed by a count
sentinel, a string-lookup table sized by that sentinel, and
`encoding.TextMarshaler` / `TextUnmarshaler` for text round-trip (JSON, YAML,
logs, `%s`/`%q`). The integer type is cheap to compare and switch on; the
text methods give a stable display form without committing the wire layout to
the type. `pkg/enum` holds the reverse lookup, so the whole apparatus is
three one-line methods:

```go
type Role uint8

func (r Role) String() string { return roleStrings[r] }

func (r Role) MarshalText() ([]byte, error) { return []byte(r.String()), nil }

func (r *Role) UnmarshalText(text []byte) error {
    return enum.Parse(r, roleStrings[:], text)
}

const (
    RoleUser Role = iota
    RoleAssistant

    roleCount
)

var roleStrings = [roleCount]string{
    RoleUser:      "User",
    RoleAssistant: "Assistant",
}
```

The sentinel is what keeps the table honest. Every enum ships a test that
calls `enumtest.Complete(t, roleCount)` (`pkg/enum/enumtest`), which walks
every value below the sentinel and fails on an empty or duplicated display
form, or a text form that does not read back. A constant added without a
table entry fails that test instead of printing as an empty string.

**Switches over an enum name every member.** `go tool exhaustive` enforces
it, with the sentinels excluded (command in [tooling.md](tooling.md)). A
`default` case does not count as covering the members it absorbs. List them,
so that a new member fails the check at every switch that must consider it
instead of falling silently into whatever the default does. A case that no
member can reach — the `default` after a full listing, or an arm the
surrounding code has already ruled out — says so with
`panic("pkg: what cannot happen")`. That is the one crash not written through
`pkg/assert`: the compiler wants a terminating statement there, and a
function call is not one.

`type Role string` makes typos silently valid (any string is a `Role`) and
fixes the display form to whatever the wire chose; the iota form keeps the
set closed and display independent of wire.

A wire form that is itself a string is **not** a reason to reach for
`type X string`. Keep the int-backed enum and emit the wire token only at the
(un)marshal or request boundary — a small `[count]string` lookup maps the
value to the bytes the API wants, exactly as it maps to the display form
(`http1.Method` displays as its wire token because they coincide). `type X
string` is for genuinely *open* string sets (arbitrary caller-supplied
labels), never a closed enumeration whose members you can list.

## Initialization and arguments

- **Fixed-size types initialize in place.** Everything meant to be
  preallocated — a connection, a request arena, a fixed-size string — has an
  `Init` method (`func (c *Conn) Init(...)`) rather than a `New` that returns
  the value, and callers fill an existing destination. The by-value form
  zeroes and copies the whole struct on every assignment into an existing
  destination; benchmarked (`static`'s string benchmarks), it is 54× slower
  for short inputs and 3–4× at near-max, because inlining does not elide the
  return temporary. `Init` does not zero the tail past `len`; code must never
  read past `len`.
- **Small values may have value constructors.** `func New(...) T`, never
  `*T`: methods may still take pointer receivers, and a caller who needs a
  pointer takes one. `new(expr)` (Go 1.26+) does that without a named local:
  `c := new(http1.Conn)`.
- **Required arguments positional; optional via functional options.**
  `Conn.Init(host, port, scheme, opts...)` takes what it cannot default
  positionally, and `WithDialTimeout` and friends as `type Option
  func(*Conn)`. Don't smuggle required arguments into options (no
  half-initialized values by omission); don't overuse options either — if
  everything is optional and zero defaults aren't meaningful, prefer a config
  struct.

## Optionality: explicit presence fields, no pointers, no wrappers

Express "maybe a T" with an explicit presence field next to the value in a
fixed-size struct — a bool, a count, a sentinel, or a result-type arm that
makes the value valid: `Response.HasBody` beside `ContentLength`, which is
`-1` when the length is not known; `ReadResult.N` valid under `ReadOK`;
`Request.Header` returning `([]byte, bool)`. Never a `*T` (heap allocation,
nil-deref invitation) and not a generic `Optional[T]` wrapper either: in this
style the enclosing struct is preallocated and fixed-size, so presence is
just another field, and the result-union convention below already gives most
operations a place to say "absent."

## Group related declarations in a `var(...)` block

Declare several variables together at the top of a scope with one `var(...)`
block, not multiple `var`/`:=` statements. Within the block, keep all
assignments together and all naked (zero-value) declarations together —
don't intermix them, or alignment goes noisy.

```go
var (
    reader = strings.NewReader(data)
    tk     json.Tokenizer
    dec    decoder
    buf    [bufferSize]byte
    tokens [10]json.Token
    r      json.Result
)
```

## Failures: tagged result unions, causes, edge emission

This style does not propagate `error` values. Failure handling separates the
**control plane** (what the program does next — travels up the stack) from
the **observability plane** (what the operator needs — recorded where the
failure happened, never threaded through callers).

- **Fallible operations return per-op tagged result unions.** A result struct
  whose `Type` field is an int-backed enum of that operation's outcomes
  (`DoOK`, `DoErr`; `ReadOK`, `ReadEOF`, `ReadErr`), with payload fields
  valid only under the arms that define them. Per-op enums keep `switch`
  statements exhaustive and meaningful — `Do` cannot return `EOF`.
- **The zero value of a result type is its OK arm; the zero value of a
  diagnostic enum is None.** `DoResult{}` is a success, and `CauseNone` and
  `ReasonNone` say no failure was recorded. The two agree, since absence of
  failure is success, but the consequence is that a result nobody filled in
  reads as success: never use a zero result to mean "not yet known" — give
  that its own field.
- **Internal steps of an operation return the operation's own union.** A
  helper that writes the head or refills a window returns `DoResult` or
  `ReadResult`, and its OK arm means the step completed and the caller
  carries on: `if res := c.writeAll(b); res.Type != DoOK { return res }`.
  No second `bool` or `error` channel beside the union; the arm already
  says it.
- **A resumable operation threads a cursor result through its calls.**
  `json.Tokenizer.Push(input, tokens, r Result) Result` takes the previous
  result and returns it advanced: `BytesRead` and `Tokens` are where the
  next call continues, `Type` is why this one stopped, and on failure
  `BytesRead` is the byte at fault. The zero `Result` is the starting cursor,
  which is the zero-is-OK rule again.
- **The `Err` arm carries a shared `Cause` enum**, one project-wide closed
  set of operational failure classes in `pkg/fail` (`CauseThrottled`,
  `CauseAuthDenied`, `CauseTimeout`, `CausePreconditionFailed`,
  `CauseNotFound`, `CauseServer`, `CauseNetwork`, `CauseProtocol`,
  `CauseTLS`, `CauseCanceled`, `CauseIO`). Policy derives from the cause
  (`func (c Cause) Retryable() bool`), never from strings. One shared enum
  bounds metrics cardinality and keeps the mapping tables in the adapters.
- **Beside the `Cause`, a package-local `Reason` names the exact rule or
  limit.** `http1.Reason` says which HTTP rule the peer broke or which fixed
  limit was exceeded. It exists for failure records and tests; policy keys
  off `Cause`, never `Reason`.
- **`error` stops at the adapter boundary.** The adapter that dialed the
  socket or called the SDK maps the failure to a `Cause` and returns only
  the result union; callers upstream see codes, not strings. `errcheck`
  (command in [tooling.md](tooling.md)) still enforces that no `error` is
  dropped by accident inside adapters; a bare `_ = c.Close()` is reserved
  for a path an earlier failure already dominates. Tests are exempt: their
  own assertions catch a test server that failed to write.
- **Diagnostics are recorded in fixed memory at the point of failure.** In
  `http1` that is the `Reason` in the result and the `Conn`'s
  `FailureDetail`: a `static.String256` holding the transport error's text,
  or the bytes at which the peer's violation was found, from the line at
  fault. (In `xfer`, the layer above also increments a preallocated
  `[Op][Cause]` counter matrix and writes a fixed-size failure record — op,
  cause, HTTP status, provider request ID, attempt, timestamp — into a
  per-worker ring buffer; no example here yet.)
- **Programmer errors are assertions, not causes.** Invariant violations
  crash via `pkg/assert`; the `Cause` set is reserved for the environment
  misbehaving. If a `Cause` value could only arise from a bug, it should
  have been an assert. `assert.True` takes a constant message and costs one
  branch, for checks made per byte; `assert.Equal`, `assert.Less`, and
  `assert.LessEq` name the values in the crash and format them only on the
  failing path, for checks made per operation.
- Causes and result-type names are **API**: `UpperCamelCase` per the naming
  rule, held complete by `enumtest.Complete`, stable once shipped.

## Prefer `go doc` over grepping source for API lookup

When investigating an unfamiliar package, reach for `go doc` first — it
parses the package, shows full signatures with docstrings, and skips test
files:

```sh
go doc gopherstyle/pkg/http1.Conn.Do
go doc -all gopherstyle/pkg/json | less
```

Fall back to grep only for usage examples, or when the symbol name itself is
the unknown.
