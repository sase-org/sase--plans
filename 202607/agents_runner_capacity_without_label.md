---
tier: tale
title: Label-free Agents runner-capacity chip
goal: 'Make the Agents header more compact by removing the redundant "runners" label
  before the current/maximum capacity values while preserving the chip''s meaning,
  styling, queue indicator, and performance behavior.

  '
create_time: 2026-07-20 10:27:06
status: done
prompt: 202607/prompts/agents_runner_capacity_without_label.md
---

# Plan: Label-free Agents runner-capacity chip

## Product outcome

Shorten the always-visible runner-capacity chip in the loaded Agents header from:

`12  [runners 8/10 · 2 queued]  [8 running · 2 waiting]`

to:

`12  [8/10 · 2 queued]  [8 running · 2 waiting]`

Remove only the dim `runners ` prefix. Retain the square brackets, two-space separation from the visible agent total and
following metric strip, slash-delimited `slots in use/configured limit` ordering, queue count and `queued` wording, and
all existing color semantics. The shorter chip should remain visible when the queue is zero and when the capacity
snapshot is the deterministic neutral `0/0` fallback. Preserve the startup `Agents: …` treatment until the first
coherent snapshot is installed.

## Implementation boundary

Make the copy-only presentation adjustment in the Agents info panel's runner-capacity text builder. Keep the immutable
runner-capacity snapshot, loader/worker projection, prepared apply boundary, app-state installation, Config Center
refresh wiring, and admission/queue predicates unchanged. Rendering, navigation, refiltering, folding, and countdown
ticks must continue to consume cached state without configuration reads or other event-loop work.

Do not rename internal runner-capacity fields or methods merely because the user-facing noun is removed. Retain the
existing full-rebuild behavior when capacity values change and the `layout=False` cheap countdown path. The numeric
styles remain green below the limit, gold at the limit, red above the limit, blue for the configured limit, and purple
for a nonzero queue; delimiters, zero queues, and the remaining copy stay dim.

## Tests and visual acceptance

Update focused info-panel assertions to expect `[current/limit · count queued]` everywhere, including the `0/0`
fallback, zero/nonzero queues, and the no-status-metrics case. Adapt the style test so it locates the occupancy number
without relying on the removed `runners ` substring, while continuing to assert the occupancy, slash, limit, and queue
styles. Keep coverage proving capacity changes trigger a full text rebuild and render/countdown paths do not read
configuration.

Update the runner-slot visual scenario's exact header assertion from `2  [runners 0/10 · 1 queued]  [2 waiting]` to
`2  [0/10 · 1 queued]  [2 waiting]`. Regenerate the affected Agents PNG goldens, inspect the primary 120×40 runner-slot
image, and confirm that removing the label improves horizontal room without changing header hierarchy or any row/detail
content. Because the capacity chip appears in every loaded Agents header, accept all deterministic Agents snapshot
changes caused solely by the shorter text, but do not update unrelated tabs or transient visual differences.

Before verification, run `just install` as required for an ephemeral workspace. Run the focused info-panel widget test
and the runner-slot PNG scenario, then run the complete visual suite and finish with the mandatory `just check`.

## Non-goals

Do not change runner counts, limits, queue membership, FIFO position, `%w(runners:N)` behavior, the broader `waiting`
metric, the Runners modal, or Axe's separate runner dashboard. Do not alter the historical approved plan for the
original capacity-header feature; this tale is a narrow follow-up to its rendered copy and corresponding expectations.
