---
title: gno serve misses changes made while it was not running
date: "2026-10-06"
track: bug
category: integration
module: src/serve/resident-runtime.ts
tags: [soak, fn-203, fn-205]
problem_type: integration
symptoms: files changed while serve was stopped stay unindexed after restart
root_cause: serve did not reconcile collections at startup; daemon did
resolution_type: fix
last_audited: "2026-10-16"
---

# gno serve misses changes made while it was not running

## Problem

`gno serve`, which the desktop app runs, does not reconcile the index with the disk at startup. Files added, edited or deleted while no resident was running, or in the moments before it was killed, stay missing from the index until something touches them or a manual sync runs. `gno daemon` does run an initial sync and catches up.

Evidence (fn-203 soak harness):
- Direct probe: 50 notes indexed; 20 notes added and 1 deleted while stopped; after starting `serve` and waiting 30 s, the index still had 50 active notes against 69 on disk. `gno daemon` reached 69 in the same time.
- `bun run soak --tier torture --classes signals --seed 3`: after SIGHUP or SIGKILL during a 300-note burst, the restarted `serve` never indexed the ~290 notes written just before the kill (I8/I9 convergence failures).

Desktop users therefore lose offline edits from search until the nightly or a manual sync. The battery-drain reporter's nightly script runs explicit scans, possibly for this reason.

## Acceptance Criteria

- **R1:** A resident `gno serve` reconciles each watched collection with the disk at startup (as `gno daemon` does), in the background, without blocking readiness or status.
- **R2:** Changes made while no resident ran are indexed within a bounded time after startup; a regression test covers adds, edits and deletes made while stopped.
- **R3:** The soak torture `signals` class converges after SIGHUP and SIGKILL restarts.
- **R4:** Startup reconcile honours the same lease, backoff and source-availability rules as watcher syncs (no new hot loop on a failing collection).

## Boundaries

- No change to `--no-sync-on-start` semantics for the daemon.

## Resolution (cccda101, #290, fn-205)

`WatchService` gained a `reconcileOnStart` option, enabled only for `serve` (`src/serve/resident-runtime.ts`, `reconcileOnStart: mode === "serve"`). Once the snapshot baseline is ready it queues one generation reconcile per watched collection through the normal flush path, so lease, backoff and source-availability rules apply. The daemon keeps its own initial sync.

Guidance: any resident mode that relies only on the watcher must reconcile at startup, or changes made while it was down are lost.
