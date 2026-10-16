---
title: Phone-width user sees the dashboard scroll sideways — Download missing models bu
date: "2026-10-07"
track: bug
category: ui
module: src/serve/public/components/BootstrapStatus.tsx
tags: [qa, fn-213-resident-crash-on-old-bun-and-mcp, web-ui]
problem_type: ui
symptoms: At 375px the Bootstrap & Storage 'Download missing models' button ends at 382px; the page scrolls sideways by 7px
root_cause: non-wrapping justify-between header row with an unshrinkable text column pushed the button past 375px
resolution_type: fix
last_audited: "2026-10-16"
related_to: [bug/ui/publisher-cannot-finish-encrypted-2026-09-06]
---

## Problem
A user viewing the GNO web dashboard on a phone-width screen sees the "Download missing models" button in the Bootstrap & Storage section run past the right edge, so the whole page scrolls sideways by a few pixels.

## Steps to reproduce (cold)
1. Start `gno serve` with no models cached (offline), open `http://127.0.0.1:<port>/` at a 375×812 viewport.
2. Wait for the dashboard to render.
3. Scroll to the "Bootstrap & Storage" heading.
4. Observe the "Download missing models" button beside the heading.

## Expected
The dashboard fits the viewport at phone width; nothing extends past the right edge.

## Actual
The button's right edge is at 382 px on a 375 px viewport (`document.documentElement.scrollWidth` 382), measured twice. The heading wraps to two lines and the description to a narrow column while the button keeps its full width.

## Evidence
- measurements: .flow/tmp/qa-fn-213-resident-crash-on-old-bun-and-mcp/P2-mobile-download-button.log
- screenshots: .flow/tmp/qa-fn-213-resident-crash-on-old-bun-and-mcp/P2-mobile-download-button-1.png, -2.png
- url: http://127.0.0.1:3997/

## Traceability
- R-IDs: none (pre-existing; the button dates from 872bbdd5, 2026-03-22)   scenario: S2   driver_rung: agent-browser   viewport: 375x812

## Resolution (fadedd0f, #298)

The header row was a non-wrapping flex row (`justify-between`) with an unshrinkable text column, so the button kept its full width past a 375 px viewport. Fix in `src/serve/public/components/BootstrapStatus.tsx`: stack the header on phones (`flex flex-col gap-4 sm:flex-row sm:items-start sm:justify-between`), add `min-w-0` on the text column and `self-start` on the button, and regenerate `globals.built.css` so the new utilities ship.
