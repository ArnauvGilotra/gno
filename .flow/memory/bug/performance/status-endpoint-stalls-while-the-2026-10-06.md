---
title: Status endpoint stalls while the resident is busy
date: "2026-10-06"
track: bug
category: performance
module: src/serve/resident-runtime.ts
tags: [soak, fn-203, fn-208]
problem_type: performance
symptoms: "/api/resident/status p99 0.3-5.8 s under load, 1 ms idle"
root_cause: "per-status MIN/MAX vector scan, config re-parse on every read, and unsliced graph projection blocked the event loop"
resolution_type: fix
last_audited: "2026-10-16"
---

# Status endpoint stalls while the resident is busy

## Problem

`/api/resident/status` answers in about 1 ms when the resident is idle but stalls for hundreds of milliseconds to seconds under load. The handler only reads in-memory state, so this is main-thread blocking: synchronous SQLite work or long synchronous loops in the resident starve the event loop. The battery-drain reporter saw status requests time out at 8 s for the same reason.

fn-203 soak harness measurements (probe every 2 s; provisional limit p99 250 ms):

| Scenario | p99 | max |
| --- | --- | --- |
| smoke churn (edits, readers, MCP, CLI writers) | 465 ms | 695 ms |
| embedder failing (500s, malformed, hang, down) | 292 ms | 1.3 s |
| config rewritten every 15 s under churn | 1.1 s | 1.1 s |
| 300 event-stream subscribers + churn | 1.7 s | 1.7 s |
| after waking from 90 s SIGSTOP (laptop sleep) | 4.8 s | 5.8 s |

Repro: `bun run soak --tier torture --classes sleep-wake,sse-flood,config-edit --contain`.

## Acceptance Criteria

- **R1:** Profile the resident in each scenario above (`bun --cpu-prof`) and name the synchronous work that holds the event loop.
- **R2:** Bound that work: yield between batches, move it off the main thread, or split long transactions, so status p99 stays within the I4 limit in all five scenarios.
- **R3:** Set the I4 limit in `scripts/soak/config.ts` from a measured healthy baseline (the open fn-203 decision) and record it.

## Resolution (52022274, #292, fn-208)

Profiling named three kinds of synchronous work holding the event loop:

- Every status build checked stored embedding dimensions with a MIN/MAX scan over every vector. Migration `035-vector-length-index.ts` indexes `(model, length(embedding))`, so each bound is one seek (`src/store/vector/freshness.ts`).
- Every resident read re-read and re-parsed the config file. A stat stamp (inode, size, mtime, ctime) in `src/serve/resident-runtime.ts` skips the reload while the file is unchanged.
- Graph projection after a burst ran one long incoming-link statement. It now resolves links in chunks and yields through `createTimeSlicer` (`src/core/time-slice.ts`) whenever a slice passes 20 ms (`src/ingestion/graph-reconciliation.ts`).

The soak I4 limit in `scripts/soak/config.ts` is `statusP99Ms: 150` / `statusMaxMs: 1_000`, set from a healthy baseline of worst p99 65 ms. `GNO_SOAK_CPU_PROF=<dir>` profiles each soak resident.

Guidance: keep status-path work O(1) through indexes, cache config reads behind a stat stamp, and time-slice long synchronous loops.
