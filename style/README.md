# Style guides — index

Load [core.md](core.md) for any Go work. Load the others only when their
condition matches your task; they extend the core — its rules still apply
underneath.

| Document | Load when |
|----------|-----------|
| [core.md](core.md) | **Always**, for any Go code. Naming, file ordering, enumerations, initialization and arguments, optionality, declaration grouping, failure handling (result unions and causes). |
| [testing.md](testing.md) | Writing or changing tests; adding code with contracts to hold (allocation-free, bounds-check-free); reimplementing a protocol or format a mature implementation already parses (differential fuzzing). |
| [tooling.md](tooling.md) | Setting up a module's toolchain, running the format/lint/check suite, or registering tool dependencies. Also the pre-handover checklist. |
| [project-structure.md](project-structure.md) | Creating a module or repository, or adding a binary or importable package to one. |
| [api-clients.md](api-clients.md) | Writing or changing a client for an external HTTP API. Skip otherwise. |
| [mcp-servers.md](mcp-servers.md) | Writing or changing an MCP server. Skip otherwise. |
