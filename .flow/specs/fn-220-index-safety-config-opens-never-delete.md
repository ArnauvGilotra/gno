## Goal & Context

Three defects surfaced during the GNO 2.11.0 rollout (2026-10-12).

1. **Opening an index with a different config deletes collections.** `syncCollections` (src/store/sqlite/adapter.ts) deletes every `collections` row the opening config does not list, and `documents` cascades (`ON DELETE CASCADE`, `PRAGMA foreign_keys = ON`). It runs on every open: CLI `initStore` (src/cli/commands/shared.ts), SDK `createGnoClient` (src/sdk/client.ts), scoped index, resident runtime, sessions. So a read-only command such as `gno --config other.yml search x`, or an SDK client given a config but no `dbPath`, silently wipes the default index's other collections. This happened on Thor on 2026-10-11 23:41 UTC: a QA script opened the real index with a two-collection sandbox config and every real collection's documents were deleted (rebuilt by `gno update` + `gno embed`). The behavior has existed since the first store schema. Reproduced with two SDK clients on one temp index (1 doc → 0 docs).
2. **`gno agents update` installs where no block exists.** Its help says it refreshes an installed block, but `installAgents(opts, "update")` plans `install` for harness files without a block. On a machine whose harness files carry a user-managed GNO block (Gordon's canonical instructions), `update` would append a second, generic ladder.
3. **`gno skill install --scope user` ignores `CLAUDE_CONFIG_DIR` and `CODEX_HOME`.** src/cli/commands/skill/paths.ts joins `homedir()` with `.claude`/`.codex`; `gno agents` (harnesses.ts) honors both variables. Profile directories (e.g. `~/.claude-instances/*`, `~/.codex-instances/*`) never receive the skill.

## Acceptance Criteria

- **R1:** Opening an index never deletes a collection or its documents. Read and update paths only upsert the collections their config lists. Deleting a collection's index data happens only through the explicit removal paths (`gno collection remove`/`rename`, MCP/REST remove, `gno cleanup`), which keep their current behavior.
- **R2:** Collections present in the index but absent from the opening config are not searched, listed or returned by that client (no cross-config leakage), and `gno status` names them with the existing `gno cleanup` hint.
- **R3:** `gno agents update` only refreshes harness files that already contain the GNO block; files without one are reported as absent with a `gno agents install` hint and left untouched. `install` keeps appending.
- **R4:** `gno skill install`/`uninstall`/`paths` at user scope resolve the Claude target from `CLAUDE_CONFIG_DIR` and the Codex target from `CODEX_HOME` when set (trimmed, non-empty), and print the resolved directory.
- **R5:** Regression tests for each requirement (the two-client repro becomes a test); docs (CLI, SDK, CONFIGURATION, TROUBLESHOOTING, skill/agents sections), spec/cli.md and CHANGELOG updated.

## Verification

- Two-client test: index collection A, open with a config listing only B, reopen with A: A's documents intact; B's client cannot see A.
- CLI test: `gno --config other.yml search` against an index holding other collections leaves them intact.
- `gno agents update` on a file without a block: unchanged file, absent status.
- `gno skill install --scope user --target claude` with `CLAUDE_CONFIG_DIR` set writes there.
- `bun run lint:check && bun test`; live QA on a sandbox index.

## Boundaries

- No change to explicit removal semantics shipped in 2.10.1.
- No new config binding or index ownership model unless R1/R2 cannot be met without it.
