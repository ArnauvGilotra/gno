# Fast semantic retrieval and a retrieval ladder that uses it

## Goal & Context
<!-- scope: business; source: user + measured -->

Semantic retrieval is GNO's reason to exist beyond keyword search, and today it is too slow to use. On Gordon's default index (2,285 documents, 17.6k chunks, RTX 4090, Ryzen 7950X3D, GNO 2.10.1, 2026-10-11) end-to-end CLI timings were: `gno search` 0.2 s, `gno vsearch` 20.2 s, `gno query --fast` 19.9 s, `gno query` 78.0 s, `gno query --thorough` 78.9 s. The skill advertises ~0.5 s / ~0.7 s / ~2-3 s for the same commands. Agents follow the installed ladder, find semantic rungs slow, and fall back to BM25. If semantic search is fast, GNO stops being keyword search with extra steps. [user] [measured]

Root cause, measured on a copy of that index (`searchVectorVariants`, `src/store/vector/variant-search.ts`): every vector search on an activated partition runs [measured]
- an integrity probe that cross-checks the whole partition's vec0 mirror against `vector_variants` (line ~154): 8.3 s;
- a ranking query whose distances come from `vector_variants.embedding` but which also joins the vec0 table on `variant_id` and `embedding` purely as a consistency check: 9.0 s, versus 0.23 s for the same query without that join;
- a per-query re-proof of every eligible chunk's formatted-input hash in JS: ~0.25 s, linear in corpus size.

The vec0 KNN index itself is never used for ranking on an activated partition; `MATCH … k = 20` on the same partition takes 40 ms and a plain `vec_distance_cosine` scan over `vector_variants` 52 ms. `src/pipeline/hybrid.ts` (~line 730) then repeats the whole search once per query variant (original, expansions, HyDE), which is how balanced `query` reaches 78 s. The shape has existed since 2.0.0; its cost grows with chunk count, so a 100k-chunk index would take minutes per query. [measured] [inferred]

## Acceptance Criteria
<!-- scope: both -->

- **R1:** [user] Vector search no longer runs vec0 ↔ `vector_variants` consistency checks or per-chunk input-hash proofs on every query. The guarantees they provide stay: a stored vector is used for ranking only while it matches the current formatted input for its chunk under the active identity, and a partition whose vec0 mirror disagrees with its variants is never silently trusted. Where they are enforced moves to write/activation time, `gno embed`, `gno vec sync`, `gno doctor`, or a proof cached against an index change marker, invalidated by any write that could break it.
- **R2:** [user] Ranking stays exact and eligibility-first: collection, document, exclusion, typed-metadata and language eligibility apply before top-k, as today, so a selective filter never loses results to candidate truncation. No approximate index. For the same index and query, the returned chunks and order match the current implementation (ties broken deterministically), proven by a parity test over seeded corpora with unscoped, collection-scoped and metadata-filtered searches.
- **R3:** [user] Latency targets on the 2026-10-11 reference machine against Gordon's default index (17.6k chunks), recorded before and after in the PR:
  - vector leg of one search, warm (resident `serve`/`daemon`/`mcp`): ≤ 100 ms unscoped, scoped or filtered;
  - `gno vsearch` and `gno query --fast` CLI end to end, cold process: within 1 s of `gno search` plus model load and query embedding (report the split);
  - balanced `gno query` with expansions, warm: meets the ~2-3 s the skill already advertises, or the skill and docs state the measured number instead. Report the per-stage split (embed, vector, BM25, expansion, rerank). Running a query's vector variants in one pass is an allowed means, not a requirement.
- **R4:** [inferred] The before/after measurement is kept as an opt-in bench script (seeded deterministic vectors, no models; plus a mode that reads an existing index copy) so the numbers can be re-run after future vector-path changes. Not part of `bun test`, no CI gate.
- **R5:** [user] The retrieval ladder is updated to match the new speeds, in every place an agent reads it:
  - the `gno agents` block (`src/cli/commands/agents/block.ts`, bump `BLOCK_VERSION` so `gno agents update` / `verify` propagate it): natural-language questions go to `gno query --fast` first; `gno search` stays the rung for exact identifiers, quotes and error strings; balanced `gno query` is the retry before grep;
  - the shipped skill (`assets/skill/SKILL.md` and its reference files): the Search Modes table carries measured numbers with the hardware stated, the retry strategy matches the block, and MCP retrieval order matches;
  - MCP tool descriptions for `gno_query`, `gno_vsearch` and `gno_search` steer the same way;
  - the autoresearch skill eval (`~/repos/autoresearch-experiments/gno-skill`) is re-run and the skill iterated until it passes; the winning skill is copied back and reinstalled per the repo workflow.
- **R6:** [inferred] Docs match behaviour: `docs/CLI.md`, `docs/MCP.md`, `docs/ARCHITECTURE.md` (vector search path), any performance claims in README, and the gno.sh pages that state speeds or recommend a search mode. Website pages describe supported behaviour and numbers only, no internals.

## Verification
<!-- scope: technical -->

- [measured] Reproduce the before numbers with a read-only copy of the index outside the repo (bench script: direct `searchVectorVariants`, vec0 `MATCH`, plain scan; CLI timings for search/vsearch/query modes). Re-run after the change; both tables go in the PR.
- [inferred] Parity test (R2): seeded corpus, same top-k ids/order old vs new for unscoped, collection-scoped, metadata-filtered and exclusion-filtered queries, plus a test that a chunk whose formatted input changed is not ranked, and that a vec0 mirror disagreeing with its variants is detected by the enforcement point chosen in R1.
- [inferred] `bun run lint:check && bun test`; `bun run eval:hybrid` shows no regression; the memory and sessions gates stay at 100.
- [inferred] Live QA on `gno serve` + MCP: warm query latency from the Web UI and an MCP client, captured.

## Fallbacks if R3 is missed
<!-- scope: technical -->

Use in this order, stopping once the targets hold; record which were needed and the measured effect of each in the PR. [user] [inferred]

1. Compute distances for all of a query's vector variants in one pass instead of one search per variant.
2. Rank unscoped searches with the vec0 KNN index (`MATCH … k`), keeping the exact eligible-set scan for scoped or filtered searches; prove parity for both paths.
3. A vec0 partition key per collection, so a scoped search reads only that collection's rows.
4. Repack a fragmented vec0 table when well under 90% of its rows are live.
5. `ANALYZE` / `PRAGMA optimize` after large index runs, if a query plan shows missing statistics.
6. A resident in-memory exact scan for `serve`/`daemon`/`mcp` (shared buffer, worker threads, falls back to SQLite when stale); only if 1-5 cannot meet the warm target.
7. If the non-vector stages (expansion, rerank) keep balanced `query` above 2-3 s, state the measured number in the skill and docs rather than widening this spec.

## Boundaries
<!-- scope: business -->

- [user] No approximate nearest-neighbour index, no new runtime dependency, no change to embedding models or chunking.
- [inferred] No change to ranking quality logic here (blend, fusion, temporal inference live in their own spec); this spec changes speed only and must return the same results.
- [inferred] An in-memory resident vector copy (shared buffer + worker scan) is out of scope unless R3 cannot be met without it at Gordon's scale; record the decision either way.
- [inferred] Per-collection vec0 partitioning and vec0 repack of fragmented tables are implementation options, not requirements; adopt only if needed for R2/R3.
- [inferred] Planner statistics (`ANALYZE`/`PRAGMA optimize`) and a permanent scale gate are deferred until a measurement shows they matter.
- [inferred] Lands before the ranking changes in the correctness-fixes spec so its parity test compares like with like.
- [user] Never name other products in code, specs, commits, docs or the site.
