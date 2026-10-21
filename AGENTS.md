# AGENTS.md

Operating guide for agents maintaining this repository. For what the
repository is and how projects consume it, read [README.md](README.md)
first — this file only adds the rules you need when *changing* the repo.

## Document placement: AGENTS.md vs README.md

- **README.md** (at the root and in each directory) is for humans and agents
  both: purpose, consumption instructions, and the index of what lives where.
- **AGENTS.md** is for agents only: maintenance rules, curation policy, and
  operational detail a human browsing the guides doesn't need.
- Guidance content never lives in an index or in this file; it lives in the
  documents under `style/`. This file governs *how* guidance gets added,
  changed, or removed.

## Indexing discipline

Agents consuming this repo must be able to load only the parts relevant to
their task, one index at a time:

- Every directory holding more than one document carries a `README.md` index:
  one line per document — filename, one-line description, and a **load when**
  condition.
- The "load when" condition is the contract. An agent working on code with no
  external-API client must be able to tell from the index alone that it can
  skip `api-clients.md`. Write conditions in terms of the *task*, not the
  document's contents.
- Adding, renaming, or removing a document means updating its index entry in
  the same commit. A document absent from its directory's index is a bug.
- Keep the hierarchy shallow. Split a directory only when its index stops
  fitting on one screen; split a document only when a reader would routinely
  need one half without the other.

## Curation policy

- **TigerStyle wins conflicts.** When importing general Go guidance from
  elsewhere, omit anything that conflicts with the core style. Known
  conflicts already resolved this way: `%w` error wrapping, sentinel errors,
  and `errors.Is`/`As` (superseded by tagged result unions and `Cause`
  enums); pointer- or `Optional[T]`-based optionality (superseded by explicit
  presence fields); value-returning constructors for large fixed-size types
  (superseded by `Init` in place).
- **Keep each document something a person reads, not an encyclopedia.** Trim
  imported material to what the style actually practices; where a convention
  is inherited but has no example yet, the document says so rather than
  inventing one.
- **Topic-scoped guides state their own scope.** A guide like
  `api-clients.md` opens by saying when to read it and when to skip it, and
  notes that the core rules still apply underneath.
- **Contracts are tests, not prose.** When a guideline can be enforced
  mechanically (a tool, a test pattern like allocation or bounds-check
  contracts), the document names the enforcement; prose-only rules are a last
  resort.

## Provenance

The core guide was extracted from the `xfer` project's style guide and
trimmed in the local `gopherstyle` prototype
(`~/projects/temp/gopherstyle`); the general (non-Tiger) material — project
layout, API-client and MCP-server conventions — came from
`weberc2/mono:doc/style/go.md` and `doc/style/go/apiclient.md`, imported here
with conflicting error-handling guidance omitted per the curation policy.
The planned standard-library packages (see README) live in the prototype and
in `xfer`'s `internal/` tree pending import.
