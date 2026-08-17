---
tier: tale
title: Fix inaccurate data and metric definitions on the Statistics tab
goal:
  "Every number and metric definition rendered by the Admin Center Statistics tab
  matches what the code actually computes: the Projects Patch count reports truncated
  rows, the Perf Stalls tile stops summing overlapping stall and hitch records, and the
  Overview, Perf, Plans & Questions, and Activity views describe their own metrics
  correctly."
size: medium
proposed_by: bbugyi200.athena.04t
create_time: 2026-08-17 10:43:35
status: wip
---

# Fix Inaccurate Data On The Admin Center "Statistics" Tab

## Problem

An audit of every number and metric definition rendered by the Admin Center Statistics
tab found seven defects. One is a wrong number caused by a wire-key mismatch, one is a
double-counted metric, and five are metric definitions that contradict what the code
actually computes. Every finding below was verified either by running the real Rust
bindings from a workspace checkout or by reading the exact producer of the data.

Statistics code lives in `src/sase/stats/` (I/O-free view builders),
`src/sase/ace/tui/modals/statistics_pane*.py` (Textual pane and Rich renderables), and
the Rust producers in `../sase-core` (`crates/sase_core/src/agent_stats/`,
`crates/sase_core/src/perf_logs/`, `crates/sase_core/src/telemetry/store.rs`). Every fix
in this plan is Python-side in this repo; no `sase-core` change is required.

## Findings

### F1. Projects view under-reports the Patch count and never shows its truncation footnote

`build_projects_view` in `src/sase/stats/_view_builders.py` reads
`work["truncated_patch_rows"]`, but the Rust wire serializes that field as
`truncated_changespec_rows`. `AgentWorkStatsWire` in
`../sase-core/crates/sase_core/src/agent_stats/wire.rs` declares:

```rust
#[serde(rename = "truncated_changespec_rows", alias = "truncated_patch_rows")]
pub truncated_patch_rows: u64,
```

`serde`'s `rename` applies to serialization; `alias` only applies to deserialization.
The payload reaches Python through `serde_json::to_value` (`telemetry_result_to_py` in
`../sase-core/crates/sase_core_py/src/lib.rs`), so the emitted key is always
`truncated_changespec_rows` and the Python lookup always yields `0`.

Root cause: `sase-core` commit `3a5753f` ("refactor(core): use patch terminology across
core docs and internals") renamed the Rust field but deliberately pinned the wire key to
the legacy name. The two sibling fields were migrated correctly and carry explicit
comments (`rows(work, "changespecs")  # legacy wire key` and
`row.get("distinct_changespecs")  # legacy stats wire field`); this third field was
missed.

Verified against the installed binding with 60 distinct patches and the default
`work_top_n=50`:

```
work keys: ['changespecs', 'malformed_spec_files_skipped', 'projects',
            'truncated_changespec_rows', 'unattributed_runs']
patch rows returned: 50
truncated_changespec_rows: 10
truncated_patch_rows:      None
view.truncated_patch_rows: 0
view.patch_count:          50      (true distinct patches = 60)
```

User-visible impact, in all three Projects groupings:

- `_projects_summary` prints "50 Patches" when 60 exist, because
  `patch_count = len(patches) + truncated`.
- `_projects_footnotes` never emits "N additional Patch rows not shown.", so a silently
  truncated table reads as complete.

The existing tests never catch this: `tests/stats/test_views_projects.py`,
`tests/stats/_views_payloads.py`, and `tests/ace/tui/_statistics_pane_helpers.py` all
hand-write `truncated_patch_rows` fixtures, and `tests/stats/test_binding_smoke.py` (the
only real-binding test) never exceeds `work_top_n`.

### F2. The Perf "Stalls" hero tile double counts every freeze that crossed both thresholds

`build_stalls` in `src/sase/stats/_perf_view_sections.py` sets the tile value to
`stall_count + hitch_count`. Those two populations overlap: the watchdog's `_poll_once`
in `src/sase/ace/tui/util/_stall_watchdog_monitor.py` runs two independent threshold
checks with independent state, so one freeze past the stall threshold emits both a
`tui_hitch` record (at `DEFAULT_HITCH_THRESHOLD_SECONDS = 1.5`) and a `tui_stall` record
(at `DEFAULT_THRESHOLD_SECONDS = 5.0`). The same holds for the pump pair.

Verified on this host's real `tui_stalls.jsonl` over the last 7 days:

```
stall_count: 2   hitch_count: 50
tile value (stalls + hitches): 52.0   caption: Stalls   detail: worst 6.1s
per-event: [('tui_stall', 1, 6.096), ('tui_pump_stall', 1, 5.042),
            ('tui_hitch', 38, 6.096), ('tui_pump_hitch', 12, 2.464)]
top_contexts: [('agents', 49), ('artifacts', 3)]   sum: 52
```

There were 50 distinct freeze episodes (38 event-loop + 12 pump), two of which also
crossed the stall threshold. The tile reads 52. The identical `worst_seconds` of 6.096
on `tui_stall` and `tui_hitch` is direct proof that one freeze appears in both buckets.

The tile is also mislabeled: its caption is "Stalls" while its value is a
stall-plus-hitch total, so a session with zero stalls and fifty hitches renders as
"Stalls 50".

The same overlap inflates `stalls.top_contexts`, which the Rust `StallsAccumulator`
counts once per stall/hitch record (`agents 49` above, summing to the same 52).
Deduplicating contexts would need pairing information the JSONL records do not carry, so
this plan relabels that line rather than changing `sase-core`.

### F3. The Perf legend definition of "Hitch" is false

`VIEW_LEGENDS["perf"]` in `src/sase/ace/tui/modals/statistics_pane_legends.py` says
`Hitch = brief stall below the freeze threshold`. Hitch records are emitted for every
freeze at or above 1.5s, including freezes well above the 5s stall threshold — as the
6.096s `tui_hitch` above shows. Nothing filters hitches by an upper bound.

### F4. The Perf percentile methodology claim is false for the telemetry-backed rows and tiles

Both the legend and the help modal state one percentile method for the whole Perf view:

- `statistics_pane_legends.py`: `p50 = nearest-rank median of the window's samples` and
  `p95 = nearest-rank 95th percentile; same method as JKPerfTimer`.
- `statistics_help_modal.py`, "Perf methodology":
  `Percentiles — Nearest-rank on the sorted sample at index round(q * (n - 1)), clamped to [0, n - 1]; the same method JKPerfTimer.summary() uses.`

That is accurate only for the log-backed sections (Startup breakdown, Launch tile, stall
medians), which come from `percentile_sorted` in
`../sase-core/crates/sase_core/src/perf_logs/aggregate.rs`:

```rust
let index = (quantile.clamp(0.0, 1.0) * last as f64).round() as usize;
```

The "Latency & reliability" table and the "Agent p95" / "LLM p95" hero tiles come from a
different code path. `query_perf_telemetry` requests `aggregation="quantile"`, which the
telemetry store answers with `histogram_quantile` in
`../sase-core/crates/sase_core/src/telemetry/store.rs` — linear interpolation between
cumulative histogram bucket bounds, not a rank over retained samples:

```rust
let fraction = (target - previous_count) / (count - previous_count);
return Some(previous_bound + (bucket.le - previous_bound) * fraction);
```

Those values are therefore bucket-bounded estimates. For agent runs the buckets are
`[10, 30, 60, 120, 300, 600, 1800, 3600]` seconds (`METRIC_DEFS` in
`src/sase/telemetry/metrics.py`), so a returned p95 can never be more precise than the
bracketing bucket pair and can never exceed 3600s.

### F5. The Plans & Questions legend claim that skills and memories are global is false

`VIEW_LEGENDS["plans_questions"]` says
`Project scope = plans and questions honor the filter; skills and memories remain global`.
`scan_project_logs` in `../sase-core/crates/sase_core/src/agent_stats/activity.rs` skips
every project directory whose name does not match `request.project`, so skills and
memories are project-scoped exactly like plans and questions. `sase-core`'s own test
asserts it (`assert!(filtered.memories.is_empty());`).

Verified through the installed binding:

```
all skills   : [{'name': 'sase_plan', ...}, {'name': 'sase_repo', ...}]
alpha skills : [{'name': 'sase_plan', ...}]
```

The claim is also misplaced: skills and memories are not rendered on the Plans &
Questions view at all — they belong to the Activity view.

### F6. The Overview legend documents one Success denominator while the view renders two

`VIEW_LEGENDS["overview"]` says `Success = completed ÷ finished runs`. That is correct
for the "Success Rate" hero tile, which `build_overview_view` computes as
`ratio(completed, terminal_runs)` where `terminal_runs` is the sum of the `outcomes`
rows (only runs with a `done` record contribute, per `fold_lifecycle` in `run.rs`).

It is wrong for the "Top projects" table on the same view. That table's Success column
renders `ProjectWorkRow.success_rate`, which the Rust `finish_work` computes as
`ratio(value.completed, value.runs)` over _all_ runs, including `in_progress` and
`waiting` — which is why `VIEW_LEGENDS["projects"]` correctly documents
`Success = completed ÷ all runs`. Two columns labeled "Success" on one view use
different denominators and only one is documented.

### F7. The Activity view silently shows only the top five rows

`load_statistics_view` calls `query_activity_stats` and `query_run_stats` with the
default `top_n=5`. `finish_activity_counts` truncates skills and memories to `top_n`,
and `finish_workspaces` truncates workspaces to `top_n`. The Activity panels are titled
plain "Skills", "Memories", and "Workspaces" with no top-N label and no truncation
footnote, so a truncated list reads as the complete list. Overview is honest about the
same data ("Top providers", "Top skills", "Top projects"), and Projects and XPrompts
both report their truncation.

### F8. The Perf "By Provider" Count column mixes two units, and telemetry-scoped counts are unlabeled

Two smaller labeling gaps in the Perf view:

- `_provider_rows` in `src/sase/stats/_perf_view_latency.py` sets
  `counts = [int(invocations.get(name, runs.get(name, 0.0))) for name in names]`. A
  provider with LLM invocation samples reports invocations; a provider with only
  agent-run samples reports agent runs. One "Count" column therefore mixes two units
  across rows, and the Share bar is a share of that mixed total. The `Err%`/`Retry%`
  denominators use `invocations` only, so rows counted from agent runs render `—` for
  both.
- The telemetry-backed counts describe a different population from the index-backed run
  counts elsewhere on the tab. Over the same 7-day window on this host, the "Agent p95"
  tile detail reads `1143 runs` (`sase_agent_runs_total`, incremented per completed run
  of any workflow by `record_completion_metrics` in
  `src/sase/axe/run_agent_runner_finalize.py`) while Overview reads `Agents Run 692`
  (artifact-index rows whose launch timestamp falls in the window). Both numbers are
  correct for their own source, but nothing on the tab tells the reader they are not
  comparable. (Note: raw and rollup rows were checked for overlap in the live store —
  `samples` covered 1786804765–1786977509 and `rollup_5m` covered 1786372800–1786803900,
  so the telemetry total is not inflated by double counting.)

## Scope

In scope: `src/sase/stats/_view_builders.py`, `src/sase/stats/_perf_view_sections.py`,
`src/sase/ace/tui/modals/statistics_pane_legends.py`,
`src/sase/ace/tui/modals/statistics_pane_perf.py`,
`src/sase/ace/tui/modals/statistics_pane_views.py`,
`src/sase/ace/tui/modals/statistics_help_modal.py`, and their tests plus the Statistics
PNG snapshots.

Out of scope, and intentionally left alone:

- Any change to `../sase-core`. Every fix here is a Python-side read or a presentation
  change.
- Deduplicating `stalls.top_contexts` in the Rust `StallsAccumulator`, which would need
  hitch/stall pairing the JSONL records do not carry. F2 relabels the line instead. File
  a follow-up task bead if a true per-freeze context count is wanted.
- `RunsView` (`build_runs_view`) and `RuntimeView` (`build_runtime_view`). Both are
  built on every load and unit-tested but are not in `VIEW_ORDER` and are never rendered
  by `_view_renderable`, so no user-facing number depends on them. Do not delete them in
  this change; file a follow-up task bead if the dead views and the resulting unused
  `_FIXED_RUNTIME_GROUP_BY = "tribe"` query work should be removed.
- Reconciling the telemetry and artifact-index run populations. F8 only labels the
  difference; changing either counter's definition is separate work.

## Implementation

### Step 1 — F1: read the Patch truncation count from the key the wire actually emits

In `src/sase/stats/_view_builders.py`, `build_projects_view`:

```python
truncated = integer(
    work.get("truncated_changespec_rows"),  # legacy stats wire key
    integer(work.get("truncated_patch_rows")),
)
```

Read the legacy wire key first and fall back to the patch-named key, matching how
`changespecs` and `distinct_changespecs` are already handled, and keep an explanatory
comment in the same style. The fallback keeps the view correct if `sase-core` ever flips
the wire name. Note that `integer()` already takes a `default` second positional
argument, so no helper change is needed.

### Step 2 — F1 regression coverage against the real binding

Extend `tests/stats/test_binding_smoke.py` (or add a sibling test in `tests/stats/`)
with a case that builds more distinct patches than the default `work_top_n` and asserts,
through the real `sase_core_rs` binding, that `views.projects.truncated_patch_rows` is
non-zero and `views.projects.patch_count` equals the true distinct-patch total. Follow
the existing `pytest.importorskip("sase_core_rs")` and schema-guard skips in that file.
This is the test class that was missing; a hand-written payload fixture cannot catch a
wire-key rename.

Also add a `tests/stats/test_views_projects.py` case that feeds the legacy
`truncated_changespec_rows` key and asserts the footnote count survives, so both key
spellings stay covered without a binding.

### Step 3 — F2: stop summing stalls and hitches in the Perf hero tile

In `src/sase/stats/_perf_view_sections.py`, `build_stalls`, report the two populations
separately instead of `total = stall_count + hitch_count`:

- `value` becomes `float(stall_count)` — freezes at or above the stall threshold.
- `detail` becomes the hitch count plus the worst duration, e.g.
  `"50 hitches · worst 6.1s"`, dropping the hitch clause when `hitch_count` is 0 and the
  worst clause when `worst` is `None`.
- `sample_count` becomes `stall_count + hitch_count` only if that field is used as a raw
  record count; otherwise keep it consistent with `value`. Check its readers before
  deciding (`render_stat_tile` and `statistics_pane_perf._perf_stat_tile`).
- Leave `available` and `_stall_status` unchanged: a hitch-only window must still render
  (status `warning`, value `0`) rather than falling back to the no-samples empty tile.

`_perf_tile_value` in `statistics_pane_perf.py` already renders `str(int(tile.value))`
plus the detail line for `key == "stalls"`, so a tile reading "0 / 50 hitches · worst
6.1s" needs no renderer change — verify the width still fits the 5-tile strip at 120 and
90 columns.

Relabel the context line in `_perf_stalls_panel` from `Top context` to something that
names what is counted (freeze records, not distinct freezes), so the per-record double
count is no longer presented as a freeze count.

### Step 4 — F3 through F6: correct the metric definitions

In `src/sase/ace/tui/modals/statistics_pane_legends.py`:

- `perf` / `Hitch`: replace "brief stall below the freeze threshold" with a definition
  matching the watchdog — a freeze at or above the hitch threshold, noting that a freeze
  past the stall threshold is recorded as both.
- `perf` / `p50` and `p95`: keep the nearest-rank wording for the log-backed numbers and
  state that the telemetry-backed latency rows and the Agent/LLM tiles are interpolated
  histogram-bucket estimates. Keep each entry newline-free and short; `_legend_note`
  renders them on one dim line, so prefer adding one short entry (for example
  `Latency p50/p95`) over lengthening the existing two.
- `plans_questions` / `Project scope`: drop the false "skills and memories remain
  global" clause. State that plans and questions honor the project filter, and if a
  scope note about skills and memories is still wanted, say they honor it too and are
  shown on the Activity view.
- `overview`: disambiguate the two Success columns — the Success Rate tile is completed
  ÷ finished runs, and the Top projects column is completed ÷ all runs as the Projects
  view documents.

In `src/sase/ace/tui/modals/statistics_help_modal.py`, `_perf_methodology_text`, split
the single "Percentiles" row so the nearest-rank claim is scoped to the log-backed
sources and the telemetry latency rows and tiles are described as interpolated
histogram-bucket quantiles.

Update the assertions in `tests/ace/tui/test_statistics_legends_states.py` (lines around
54–64 pin the current overview, projects, and plans_questions strings) and
`tests/ace/tui/test_statistics_help_modal.py` to the new wording.

### Step 5 — F7: label the Activity view's top-N truncation

In `src/sase/ace/tui/modals/statistics_pane_views.py`, `_activity_renderable`, title the
three panels so the top-N bound is visible ("Top skills", "Top memories", "Top
workspaces"), matching the Overview wording for the same data. Keep the column headers
unchanged.

Do not raise `top_n` in this change. If a "N more not shown" footnote is wanted instead
of a title change, note that the activity wire carries no total or truncation count, so
it would need a `sase-core` change — file a follow-up task bead rather than widening
this plan.

### Step 6 — F8: label the Perf count semantics

Two presentation-only clarifications:

- In `src/sase/ace/tui/modals/statistics_pane_perf.py`, `_perf_latency_panel`, name the
  Count column so the By Provider mixed unit is not silent. The simplest honest option
  is a group-dependent header (LLM invocations under By Provider, agent runs under By
  Workflow) plus a short note that provider rows without LLM invocation samples fall
  back to agent runs, which is also why their Err% and Retry% read `—`.
- Add one Perf legend or help-modal line stating that Perf counts come from the
  telemetry store and the TUI logs, not the artifact index, so they are not comparable
  with the run counts on Overview, Projects, or XPrompts.

## Verification

1. `just install` first — this is an ephemeral `sase_<N>` workspace.
2. `just check` after each step's edits.
3. `sase monitor start --command 'just check-full' …` through `/sase_monitor` before
   landing, with a `--next` action. `just check-full` outruns a single turn; never run
   it inline.
4. `just test-visual` for the Statistics PNG snapshots. Steps 3–6 change rendered text,
   so expect `tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py`
   failures for the perf, perf-narrow, perf-degraded, overview, projects, and help
   snapshots. Inspect `.pytest_cache/sase-visual/` artifacts, confirm each diff is the
   intended wording or tile change, then accept with `--sase-update-visual-snapshots`.
5. Targeted suites: `tests/stats/test_views_projects.py`, `tests/stats/test_views.py`,
   `tests/stats/test_views_perf.py`, `tests/stats/test_binding_smoke.py`,
   `tests/ace/tui/test_statistics_perf.py`,
   `tests/ace/tui/test_statistics_legends_states.py`,
   `tests/ace/tui/test_statistics_help_modal.py`,
   `tests/ace/tui/test_statistics_pane_rendering.py`.
6. Manual confirmation in the real TUI (`/run` or `sase ace`): open Admin Center →
   Statistics and check that the Projects summary Patch count matches the footnote
   arithmetic, the Perf Stalls tile no longer sums the two event families, and the
   legend line under each view reads correctly at 120 and 90 columns.
