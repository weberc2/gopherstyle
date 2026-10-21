# MCP servers

Read this when writing or changing an MCP server; skip it otherwise. It
extends [api-clients.md](api-clients.md) — an MCP server is a thin wrapper
over a normal API client — and [core.md](core.md) underneath.

Build MCP servers on the **official Go SDK**,
`github.com/modelcontextprotocol/go-sdk` — Go-team-maintained and past v1
with a forward-compatibility guarantee. Don't introduce a bespoke framework
or a Node runtime.

- Register tools with typed Go input/output structs — the SDK derives the
  JSON Schema by reflection (`jsonschema:"…"` tags for descriptions) — via
  `mcp.AddTool(server, &mcp.Tool{…}, handler)`.
- **Prefer the streamable-HTTP transport** (`mcp.NewStreamableHTTPHandler`)
  for a server that runs as its own deployment or sidecar; offer
  `mcp.StdioTransport` too for local development.
- Keep the server a **thin wrapper over a normal API client**
  ([api-clients.md](api-clients.md)): tool handlers map tool args to client
  calls and hold no business logic. Build the one client at startup
  (single-backend shape) and share it across handlers.
- An MCP server has no resync loop spacing its retries — follow the
  poller/MCP retry rule in [api-clients.md](api-clients.md) (honor
  `Retry-After` on 429, bounded backoff, respect `ctx`).
- Pin the SDK version and verify handler signatures against that version's
  docs.
