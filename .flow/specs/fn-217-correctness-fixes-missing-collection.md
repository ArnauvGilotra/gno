# Correctness fixes: missing collection roots and unknown MCP arguments

## Goal & Context
<!-- scope: business; source: user + measured -->

Two defects found while comparing GNO 2.10.1 against how other local retrieval tools changed this autumn. Both make GNO quietly do the wrong thing. [user]

1. **A missing collection root empties the collection.** Reproduced 2026-10-11 in an isolated sandbox (own config/data/cache dirs): three indexed notes, root folder renamed away, `gno update` printed "3 marked inactive" and `gno search` went from 3 hits to 0; restoring the folder and updating brought them back. Cause: in the default `any` availability mode the walker returns an empty listing when `realpath(root)` fails (`src/ingestion/walker.ts` ~292) and sync deactivates every active path it did not see (`src/ingestion/sync.ts` ~1926). `local` mode already records an unresolvable root as an unproven prefix (`walker.ts` ~387-396), so the defect is the `any` path. An unmounted drive, an offline Google Drive/CloudStorage folder or a network share triggers it, and a resident `serve`/`daemon` watcher can do it during an outage. [measured]
2. **Unknown MCP arguments are silently dropped.** Most MCP input schemas are non-strict zod objects (`src/mcp/tools/index.ts`, `memory-remember.ts`, `memory-recall.ts`, `changes.ts`, `trace.ts`, `workspace-write.ts`, `request-status.ts`); egress, sessions, ask, audit and sections already reject unknown keys. A probe of `rememberInputSchema` with `derived_from` and `recall_receipt` (misspellings of `derivedFrom` and `receipt`) parsed as success with both keys discarded, so the remember fence never saw them. On reads, a misspelled scope key silently widens a search to every collection the caller can see. [measured] [inferred]

## Acceptance Criteria
<!-- scope: both -->

- **R1:** [measured] If a collection root cannot be resolved or enumerated, full and incremental syncs (CLI and resident watcher) leave that collection's documents as they are and report the collection, the path and the original error. A root that exists and enumerates as empty keeps deletion semantics.
- **R2:** [inferred] A `gno cleanup` run while a root is unavailable leaves the retained documents' content, chunks and vectors searchable.
- **R3:** [measured] MCP tool inputs reject unknown argument keys with an error naming the key; existing aliases keep working. Verified through the MCP transport, including a misspelled remember receipt that now fails instead of writing.

## Verification
<!-- scope: technical -->

- [measured] Regression test mirroring the sandbox repro: index a temp collection, remove the root, run update and cleanup, assert the documents stay active and searchable and the warning names the collection; restore the root and assert normal sync.
- [inferred] Shared-schema test that every registered MCP tool refuses an unknown key, plus one end-to-end MCP call.
- [inferred] `bun run lint:check && bun test`; memory and sessions gates stay at 100.

## Boundaries
<!-- scope: business -->

- [inferred] No new availability subsystem or status/doctor reporting matrix; the sync warning is the report.
- [inferred] REST and SDK input strictness are out of scope unless they share the MCP schemas.
- [inferred] Ranking changes live in the hybrid ranking spec.
- [user] Never name other products in code, specs, commits, docs or the site.
