# Metadata discovery and evidence currency

## Goal & Context
<!-- scope: business; source: user + inferred -->

Two gaps that make existing GNO capabilities less trustworthy or less usable for agents. All changes ship on CLI, MCP, REST and SDK per GNO's surface rule. [user]

1. **Metadata discovery.** GNO has typed custom metadata and shared predicates on every surface (2.4.0; stored as JSON in `documents.typed_metadata`, queried through `json_each`), but nothing tells a caller which keys exist, their types or common values. An agent cannot write a filter for metadata it cannot see. The only discovery today is `gno_list_tags`. [inferred]
2. **Superseded facts look current outside recall.** `recall` excludes superseded memory facts, but `docs/MEMORY.md` states they stay in ordinary search, and nothing marks them in search, query, ask or Capsule results, nor in the evidence the answer model reads. An agent that searches instead of recalling, or an ordinary `ask`, presents a replaced fact as current. [inferred]
3. **Verified answers can misquote.** In `src/pipeline/claim-verification.ts` (`verifyClaim`) a claim is supported when the judge model says so and its citations are valid; an explicit quotation in a claim is not checked against the cited text. [inferred]

## Acceptance Criteria
<!-- scope: both -->

- **R1:** [inferred] Collection-scoped metadata discovery on CLI, MCP (full profile), REST and SDK reports keys, observed types, document counts and a capped list of common values, read from the indexed metadata without reindexing. Documented next to the typed-filter examples and in command/tool help.
- **R2:** [inferred] Retrieval outputs (search, vsearch, query, get, multi-get, ask citations, Capsules) mark a superseded memory fact and name its current successor URI. Superseded Capsule evidence fails the existing freshness checks. Answer-generation evidence carries the same status, and the answer prompt tells the model not to present superseded facts as current.
- **R3:** [inferred] `ask --verify` rejects a claim whose explicit quotation does not match its cited evidence after whitespace and quote normalisation, records the mismatch, and keeps semantic verification for everything else.
- **R4:** [inferred] Affected contracts, JSON schemas, docs (`docs/TYPED-METADATA.md`, `docs/MEMORY.md`, `docs/CLI.md`, `docs/MCP.md`) and skill guidance match. The agents block changes only if its instructions change.

## Verification
<!-- scope: technical -->

- [inferred] Discovery parity test across the four surfaces on one fixture; schema contract test.
- [inferred] Supersession marking on each listed output and in the answer prompt; stale-Capsule check after a supersede.
- [inferred] Quote check regressions: verbatim, normalisation-only difference, altered quote.
- [inferred] `bun run eval:memory` stays at 100; `bun run lint:check && bun test`.
- [inferred] Live QA via `gno serve` and an MCP client: discovery output; a superseded fact found by search shows its successor.

## Boundaries
<!-- scope: business -->

- [inferred] No new predicate operators, case folding or numeric statistics in discovery; no metadata summaries in status/list.
- [inferred] No number/date presence checks in verification (valid computed and relative values would be falsely rejected).
- [inferred] No origin/trust labels, newer-mention lists, batch remember or `validFrom`.
- [inferred] Lands after the hybrid ranking spec, which also edits the answer prompt.
- [user] Never name other products in code, specs, commits, docs or the site.
