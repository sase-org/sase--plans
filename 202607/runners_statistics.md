---
tier: epic
title: Runner concurrency statistics
goal: 'The SASE Admin Center Statistics tab gains a responsive Runners view that accurately
  and beautifully explains runner-slot occupancy throughout the selected time range
  with exact totals, time shares, and bounded trend slices.

  '
phases:
- id: core_runner_analytics
  title: Core runner-occupancy analytics
  depends_on: []
  size: medium
  description: '''Core runner-occupancy analytics'' section: extend the shared Rust
    statistics contract with overlap-aware runner intervals, occupancy distributions,
    and bounded time buckets.'
- id: python_statistics_contract
  title: Python statistics model contract
  depends_on:
  - core_runner_analytics
  size: small
  description: '''Python statistics model contract'' section: adapt the Rust payload
    into immutable, timezone-aware Runners view models and current-capacity context.'
- id: runners_statistics_experience
  title: Runners Statistics experience
  depends_on:
  - python_statistics_contract
  size: medium
  description: '''Runners Statistics experience'' section: add the responsive Runners
    sub-tab, summary cards, timeline, occupancy distribution, help, and visual coverage.'
- id: end_to_end_verification
  title: End-to-end verification and visual acceptance
  depends_on:
  - runners_statistics_experience
  size: small
  description: '''End-to-end verification and visual acceptance'' section: exercise
    boundary cases, project and range scopes, responsive snapshots, performance constraints,
    and both repositories'' required checks.'
create_time: 2026-07-21 16:28:12
status: wip
---

# Plan: Runner concurrency statistics

## Product decision

Add **Runners** as the eighth internal view of the existing Admin Center **Statistics** tab, immediately after **Runs**.
The Statistics pane already owns historical ranges, project filtering, refresh behavior, contextual help, and keyboard /
mouse view navigation, so a new top-level Admin Center tab would duplicate controls and separate concurrency from the
run data that explains it.

The view should answer three questions without requiring the user to interpret a generic run chart:

1. How much runner capacity was used across the selected interval?
2. Exactly how long, and for what share of the interval, were 0, 1, 2, ... agents running?
3. When did the quiet and busy slices occur, and how high did concurrency peak?

## Metric contract and reliability rules

“Runner” means an agent execution that participates in `max_running_agents` admission: a root user agent or an eligible
parallel family member represented by `ace-run`, following the same visibility and `appears_as_agent` rules as live
runner-slot counting. Sequential family bookkeeping and non-agent workflow steps must not become extra runners.

The shared backend will derive half-open active segments from durable artifact records, using the established runtime
timestamps and human-wait exclusions rather than launch counts or summed runtimes. It must:

- start at `run_started_at`, end at the earliest valid stop/finish marker, and cap a live segment at the query's
  exclusive end;
- include agents that started before the selected range but overlap it, and clip every segment to the selected bounds;
- exclude plan/question windows in which the agent yielded execution, using the shared runtime interval semantics;
- honor the project filter before concurrency is combined;
- skip malformed or impossible intervals without failing the remaining snapshot, and report skipped data;
- combine equal-timestamp transitions deterministically so adjacent or simultaneous starts/stops cannot create phantom
  duration; and
- retain zero-runner time, because idle capacity is part of the requested distribution.

All duration-based results use one explicit analysis window. Normal presets and custom ranges use the complete selected
window. The `All time` sentinel begins at the earliest valid recorded runner segment (then includes later idle gaps) so
the Unix-epoch implementation detail cannot turn decades before SASE existed into a misleading 100% idle result. The
response exposes this effective coverage boundary for labels and help text. A fixed range with no runners remains a
valid 0-runner result; an all-time query with no recorded runner boundary uses the existing no-data guidance.

The backend owns the exact cross-frontend definitions:

- **Peak runners:** maximum simultaneous active segments, with exact time spent at that peak.
- **Average runners:** runner-seconds divided by analyzed wall-clock seconds (time-weighted, never an average of bucket
  averages).
- **Busy time:** wall-clock time with at least one runner, plus its share of analyzed time.
- **Runner time:** the integral of concurrency over time, equivalent to the sum of clipped active-segment durations.
- **Occupancy distribution:** one ordered row per observed count from zero through the peak, with exact seconds and
  share; the rows' durations sum to the analysis window and `sum(count * seconds)` equals runner time.
- **Trend buckets:** bounded, contiguous slices carrying exact start/end, time-weighted average, peak, busy seconds, and
  runner-seconds. The response stays bounded for 90-day and all-time ranges rather than shipping one row per day
  since 1970.

`max_running_agents` is not historically versioned. The UI therefore displays the **current global limit** as a useful
reference and may show time at/above that value, but must not call it historical saturation. With a project filter, it
is labeled as a global reference rather than a project-specific capacity limit. If observed peak exceeds the current
value, the visualization expands and calls out the oversubscription instead of clipping it.

## Core runner-occupancy analytics

Open the linked `sase-core` repository through `sase repo open` and extend the shared Rust agent-statistics domain and
wire API. Refactor the existing agent runtime interval code as needed so runner statistics and clan/family runtime use
one implementation for valid terminal timestamps and human-wait subtraction. Add a runner-eligibility predicate that
matches live slot admission semantics, then perform an event sweep over clipped active segments to build summary values,
the exact discrete occupancy distribution, and a caller-bounded or backend-bounded trend series.

Extend `AgentRunStatsResponseWire` with a nested runner-statistics record (and bump the schema version), keeping
sensible serde defaults for compatibility. Reuse the existing project-scoped, read-only artifact-index query and avoid
decoding obviously irrelevant old terminal rows: a record can be rejected from overlap using cached launch/finish
columns before its JSON is parsed, while open or boundary-overlapping candidates still receive full validation. Keep all
new work in Rust because another frontend should receive the same concurrency answer as the TUI. Export the wire types
through `sase_core`, preserve the PyO3 dictionary binding, and update its contract/smoke coverage.

Rust tests should cover overlapping and disjoint agents, simultaneous boundaries, carry-in agents, exact start/end
clipping, live agents, stop-vs-finish precedence, yielded human-wait segments, root/serial/parallel family eligibility,
`appears_as_agent`, hidden records, project filtering, malformed rows/timestamps, an observed peak above ten, a wholly
idle fixed range, all-time effective coverage, partial final buckets, bounded long-range output, and the distribution /
runner-seconds conservation identities.

## Python statistics model contract

Extend the thin Python query and immutable view-model layers under `src/sase/stats/` to consume the new nested wire
record without reimplementing interval math. Add frozen models for summary values, occupancy rows, and trend slices;
format slice labels in the configured SASE timezone and tolerate a missing runner object as an older/partial payload.
Capture `get_max_running_agents()` inside the existing off-thread Statistics load and carry the current limit alongside
the immutable result, so rendering performs no configuration I/O.

Keep Runners emptiness separate from the current launch-count-based `StatisticsViews.empty`: fixed windows with no
launches still render an honest 0-runner distribution, and a carry-in agent still renders even when the Runs view has
zero launches. Add focused unit tests for mapping/default behavior, time-weighted and percentage fields, timezone
labels, project-filter propagation, effective all-time coverage, current-limit context, and frozen-model guarantees.
Update the shared deterministic fixtures once so every existing Statistics test continues to describe a complete
payload.

## Runners Statistics experience

Add `runners` to `StatisticsView`, place it after `runs` in view order, and preserve the existing `[` / `]`, click,
range, project, refresh, and help interactions. Eight full labels are too wide for the supported narrow layout, so give
the reusable panel tab strip optional compact labels (for example `Plans/Q`) and opt Statistics into compact reflow at a
measured width; hit targets and the active-view treatment must remain correct.

The wide Runners composition should present, in this order:

1. A concise row of consistent statistic cards for **Peak**, **Average**, **Busy**, **Runner time**, and **Current
   limit**. Secondary copy supplies exact peak duration, busy duration/share, and time at/above the current limit where
   that comparison is meaningful.
2. A fixed-baseline **Concurrency over time** chart. Cyan fill represents each slice's time-weighted average, a magenta
   tip/marker preserves its peak, and a gold reference marks the current global limit. The vertical scale is zero to
   `max(observed peak, current limit)`, never a silently autoscaled sparkline; start/end labels and a small legend make
   compression explicit.
3. An **Occupancy by runner count** distribution with exact runner count, human duration, percentage, and a proportional
   bar for every observed level including zero. Idle uses a quiet neutral treatment; increasing occupancy uses the
   established cyan/magenta palette; at/above-current-limit comparisons use gold/red without implying that today's limit
   was historically configured.
4. A compact **Busiest slices** table ranked deterministically by peak, average, then time, showing the exact slice
   label, peak, average, busy duration, and runner time. This gives precise drill-down numbers to complement the
   chronological chart.

At wide sizes, the timeline and distribution may share a balanced row when their measured content remains legible; at
90-column and shorter viewports they stack and remain reachable through the existing vertical scroll. Reuse the
telemetry palette, duration formatting, Rich panels, and partial-block glyph language already used elsewhere, but add a
runner-specific renderer where necessary so average and peak are not conflated. Rendering must be pure and bounded by
the returned slice count.

Update the view description, inline legend, contextual help, loading/error/empty behavior, headings, and docstrings.
Help must plainly define runner eligibility, carry-in/live clipping, zero occupancy, time-weighted average, all-time
coverage, project scoping, skipped malformed data, and the current-limit caveat. Add interaction/render tests plus
deterministic populated, idle, carry-in, over-limit, project-filtered, and partial-payload fixtures. Add dedicated wide
and narrow PNG snapshots for Runners and deliberately rebaseline existing Statistics snapshots only where the new tab
strip or compact labels change them.

## End-to-end verification and visual acceptance

Exercise an acceptance matrix covering Today, 24h, 7d, 90d, All time, a calendar custom range, all projects, one
project, 0 runners, a single runner, overlapping runners, a live carry-in runner, yielded waits, and a peak above the
current limit. Confirm that occupancy durations sum to the displayed analysis window, shares sum to 100% within
floating-point tolerance, runner time matches the occupancy integral, and the chart/table show identical peak and
average values.

Verify responsiveness at the existing 120x40 and 90x30 snapshot sizes: every view remains discoverable, compact labels
do not collide, charts do not crop numeric columns, keyboard and mouse navigation agree, and scrolling exposes the lower
details. Inspect generated visual diffs rather than accepting snapshots blindly.

Performance verification must confirm that the trend payload has a strict size bound, irrelevant historical terminal
rows take the cheap path, no new configuration/index I/O reaches a render or UI-message path, the existing threaded and
coalesced Statistics loader remains the only refresh route, and auto-refresh continues only while Statistics is active.

Run the linked Rust repository's full gates (`cargo fmt --all -- --check`,
`cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`). In the main SASE workspace, run
`just install` before other checks as required, then targeted Statistics/core-binding tests, `just test-visual`, and
finally `just check`. Resolve all failures and review the final diff for accidental schema, snapshot, generated-file, or
unrelated changes before handoff.

## Scope boundaries

- Do not create a new top-level Admin Center tab or a separate historical telemetry store.
- Do not infer a historical `max_running_agents` timeline from today's configuration.
- Do not count Axe ChangeSpec runners, hooks, or workflow processes governed by separate limits in this user-agent view.
- Do not move concurrency math into Textual/Rich rendering or perform artifact/config reads on the UI thread.
- Do not edit SASE memory or generated instruction files for this feature.
