---
tier: tale
title: Statistics metric legends and actionable states
goal: 'Every Statistics view explains its displayed metrics, empty and failed loads
  offer context-aware recovery actions, and Overview tiles navigate directly to the
  detail views they summarize without changing the existing data path.

  '
create_time: 2026-07-20 14:22:29
status: done
prompt: 202607/prompts/statistics_metric_legends_states.md
---

# Plan: Statistics metric legends and actionable states

## Context

The first Statistics redesign phase has already established the title, view descriptions, scope chips, and focused hint
row. This phase builds on that presentation structure without changing the statistics queries, immutable view models,
worker/debounce lifecycle, refresh cadence, or keyboard navigation.

Metric copy must reflect the implementation rather than inferred meaning. The Python view builders and the read-only
Rust statistics aggregators establish the relevant semantics: Overview success and outcome shares use finished runs;
provider and project success values currently divide completed runs by all runs; runtime rows contain only records with
a valid finished/stopped duration; committing-agent counts are run records with at least one recorded commit; activity
agents are distinct agent names; and plan/question activity marked `(all projects)` comes from durable global records
that a project filter cannot scope.

## Shared metric legends

Introduce one structured, exhaustive legend definition keyed by every entry in `VIEW_ORDER`, plus a shared
`_legend_note(...)` renderer that produces a single dim `term = meaning` line. Keep the definitions in
presentation-oriented code that both the general and Projects renderers can reuse and that the later help overlay phase
can consume without duplicating copy.

Append a legend to all seven view renderables, adapting the renderable wrappers where a view currently returns only a
`Panel` or `Columns`. Cover the definitions called out by the epic design: previous equal-window deltas and tile
navigation; finished-run outcome shares, waiting/retry/commit terminology; Project Share, Wall, Specs, outcome glyphs,
and unattributed runs; provider Share, Success, and average runtime using the verified aggregator semantics; runtime
percentiles, runtime share, and in-progress exclusion; distinct Activity agents; and the global scope of
`(all projects)` plan/question values.

## Actionable empty and error states

Replace the generic empty message with a pure renderable that names the selected range and shows the effective
configured time-range key as the way to widen it. When a project filter is active, add the effective project-filter key
and the project's display label as a second recovery action; omit that guidance when all projects are selected.

Add a retry line to the first-load error panel using the effective configured refresh key. Preserve the existing
behavior that keeps successfully loaded data visible when a later refresh fails, while continuing to expose the failed
status in the header and the refresh binding in the hint row. These render paths remain pure and perform no disk,
subprocess, or query work.

## Clickable Overview tiles

Replace the five inert tile `Static` widgets with a small click-aware widget that emits a typed navigation message.
Handle that message in `StatisticsPane` and route it through the existing `_set_view` method so tab-strip state,
descriptions, scope, and cached composite data all update through the same path as keyboard or tab clicks. Map Agents
Run, Success Rate, and Commits to Runs; map Plans Proposed and Questions to Plans & Questions. Give the tiles an
appropriate pointer/hover affordance without adding focus behavior or a new refresh path.

## Tests and validation

Add focused presentation coverage in a sibling Statistics test module rather than further expanding the already-large
interaction test file. Assert that the legend map exactly covers `VIEW_ORDER`, every legend is non-empty and
single-line, and representative rendered output carries the verified definitions. Cover the empty state with and without
a project filter, the error retry guidance, the complete tile-to-view mapping, and click dispatch through `_set_view`
without an extra data load.

Run `just install` before repository checks, then run the focused Statistics unit and pilot tests. Exercise the existing
Statistics visual cases; update only the goldens directly changed by these legends/states after inspecting their diffs,
leaving new overlay/narrow captures and the final polish pass to their dedicated epic phases. Finish with `just check`
so formatting, lint, typing, unit, and visual snapshot coverage all pass while the auto-refresh responsiveness test
remains green.
