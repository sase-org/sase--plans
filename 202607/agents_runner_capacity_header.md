---
tier: tale
title: Agents tab runner capacity header
goal: 'Make the Agents tab header clearly and accurately show global user-agent runner
  utilization, the configured limit, and the number of agents queued by that limit
  without adding TUI latency or conflating other wait states.

  '
create_time: 2026-07-20 08:30:12
status: wip
prompt: 202607/prompts/agents_runner_capacity_header.md
---

# Plan: Agents tab runner capacity header

## Product decision

Add a compact capacity chip immediately after the existing total-agent headline and before the status strip:

`12  [runners 8/10 · 2 queued]  [1 stopped · 8 running · 3 waiting]`

The chip is always present after the first Agents snapshot, including `0 queued`, so users can distinguish an empty
queue from unavailable data. Keep the current `Agents: …` loading treatment until that first coherent snapshot lands; do
not render provisional zeroes during startup.

The values have deliberately different scopes:

- The leading total and status strip retain their current visible/effective-agent semantics, including search, fold, and
  grouping behavior.
- `runners current/limit` describes the global user-agent admission pool across projects. The authoritative limit is the
  top-level `max_running_agents` configuration, not the separate `axe.max_agent_runners` limit used by Axe ChangeSpec
  work.
- `queued` counts live agents parked specifically at the implicit global-cap gate. It excludes dependency/time waits,
  question pauses, and explicit `%w(runners:N)` thresholds such as drain barriers, even though those rows may still
  contribute to the broader `waiting` status metric. Preserve the admission model's existing participant rules for roots
  and parallel family work; do not invent a second interpretation in the widget.

This makes an important case read correctly: one global-cap waiter plus one explicit drain-barrier waiter should show
`1 queued` and `2 waiting`.

## Runner-capacity snapshot

Introduce a small immutable runner-capacity summary carrying the configured limit, slots in use, and global-cap queue
count. Extend the existing runner-slot projection in `src/sase/ace/tui/models/agent_runner_slots.py` so it produces this
summary while it attaches per-row occupancy and FIFO context. Derive both outputs from the same PID-filtered, post-merge
Agents payload, before search/fold presentation filters, so the chip and row details cannot disagree because they
sampled different artifact states.

Use the existing runner-slot predicates and durable marker fields as the semantic foundation. An implicit cap waiter
must be live, be parked on a slot request, and have `wait_runners_explicit` false; an explicit runner threshold must not
inflate the global queue count. Do not clamp occupancy to the configured limit: if a live config reduction temporarily
produces `12/10`, showing that pressure is more reliable than hiding it.

Read `get_max_running_agents()` only in the established loader/worker flow. Carry the completed summary through the
prepared apply boundary and install it in app state atomically with the corresponding agent snapshot. UI render,
countdown, navigation, refilter, and fold paths must consume only this cached value—no config reads, stats, scans, or
subprocess work on the Textual event loop. In-memory refilters should preserve the global summary until the next loaded
snapshot. External config edits are picked up by the normal coalesced refresh; after an in-app Config Center write to
`max_running_agents`, request that same standard Agents refresh so the new limit appears promptly without adding a
special reload path.

Keep compatibility fallbacks deterministic for synchronous/replay/test callers: every apply path should receive a
defined summary, and older/prebuilt boundaries should degrade to an explicit neutral/default snapshot rather than
performing render-time I/O.

## Header presentation

Thread the cached summary through the Agents info-panel state update and include all three fields in its stable-state
comparison. Capacity changes should rebuild the one-line Rich text once; countdown-only ticks must continue down the
existing `layout=False` cheap path.

Render the new chip as restrained status chrome consistent with the rest of ACE:

- dim brackets, labels, slash, and separator;
- green slot occupancy below the limit, gold at the limit, and red above the limit;
- a crisp contrasting configured limit;
- a dim zero queue count, with the existing purple waiting accent when queued work is nonzero.

Keep the language fully readable rather than introducing unexplained glyphs. Preserve the current ordering of the
remaining status, neighbor, filter, view, grouping, and refresh segments, and retain the one-line/no-layout-update
contract so the added information does not cause vertical churn.

## Verification

Add focused model tests for the summary projection: zero and nonzero pools, implicit global-cap waiters versus explicit
runner thresholds and non-slot waits, yielded questions, parallel-family participation, deterministic live-only
counting, and an over-limit snapshot after a config reduction. Assert that queue totals and existing per-row queue
positions come from the same source data.

Extend prepared-boundary/apply and info-panel integration coverage to prove the summary survives fold/search/refilter
operations, updates atomically on refresh, observes a changed configured limit, and never reads configuration from a
render or countdown path. Update widget tests for exact copy, zero visibility, plural-neutral wording, color states,
loading behavior, and stable-state/countdown cache behavior.

Use the existing runner-slot-waits PNG scenario as the primary visual acceptance case: its explicit drain barrier and
implicit global-cap waiter should visibly produce `runners 0/10 · 1 queued` beside a `2 waiting` status strip. Update
the intentional golden and inspect the rendered image for hierarchy, spacing, and truncation pressure at the standard
120×40 viewport.

Before implementation checks, run `just install` as required for an ephemeral workspace. Then run the focused runner
slot, apply-boundary, info-panel, and PNG snapshot tests; run `just test-visual` for the complete visual suite; and
finish with the mandatory `just check`.

## Non-goals

Do not change runner admission limits, FIFO ordering, `%w(runners:N)` behavior, waiting-row/detail copy, or the Axe
tab's separate hook/agent runner dashboard. This tale only exposes the existing global user-agent capacity state
accurately and consistently on the Agents tab.
