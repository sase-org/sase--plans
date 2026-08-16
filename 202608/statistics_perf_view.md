---
tier: epic
title: Admin Center Statistics Perf view
goal: 'The Admin Center Statistics tab gains an eighth "Perf" view that answers "is SASE
  fast right now, and where is it slow?" from durable data SASE already records — TUI
  startup and stall behavior, agent-launch latency, and agent/LLM/hook latency and
  reliability — with honest coverage reporting and no new event-loop work.

  '
phases:
  - id: core_perf_logs
    title: Rust perf-log aggregation and binding
    depends_on: []
    size: medium
    description: "core_perf_logs: add a `perf_logs` module to the sibling Rust core that
      reads the six durable TUI/launch perf JSONL logs with bounded tail reads,
      aggregates them over a caller-supplied window into one composite wire payload, and
      exposes it as the `perf_logs_query` Python binding.

      "
  - id: stats_facade
    title: Python perf facade and view model
    depends_on:
      - core_perf_logs
    size: medium
    description: "stats_facade: add the thin `sase.stats` adapter that calls the new
      Rust binding, fans out the fixed telemetry-store query set, and builds one
      immutable presentation-ready PerfView with status grading and coverage notes.

      "
  - id: perf_view
    title: Perf view registration and interaction
    depends_on:
      - stats_facade
    size: medium
    description: "perf_view: register Perf as the eighth Statistics view — tab strip,
      numbered selection, description, legends, grouping modes, non-project scope, lazy
      loading, and the contextual help methodology section.

      "
  - id: perf_render
    title: Perf view rendering
    depends_on:
      - perf_view
    size: medium
    description: "perf_render: build the Perf renderable — the five hero tiles, the TUI
      startup and stall panels, the grouped latency table, the data-and-instrumentation
      strip, and every degraded, partial, and empty state.

      "
  - id: perf_visuals
    title: Visual snapshots and documentation
    depends_on:
      - perf_render
    size: small
    description:
      "perf_visuals: add the PNG visual snapshots and their fixtures for the populated,
      narrow, and telemetry-disabled Perf states, and document the view in the ACE
      performance runbook."
proposed_by: bbugyi200.athena.032
create_time: 2026-08-15 20:25:25
status: wip
---

- **PROMPT:**
  [prompts/202608/statistics_perf_view.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/statistics_perf_view.md)

# Plan: Admin Center Statistics Perf view

## Why

SASE records a great deal of performance data and surfaces almost none of it. The
`sase telemetry health` CLI renders a one-hour traffic light; the Logs tab shows raw
JSONL tails; `docs/perf_runbook.md` documents a capture-and-compare workflow that
requires leaving the TUI. There is no place where a user can answer the question they
actually have — _is this thing fast right now, and what is slowing it down?_

Everything needed to answer that already exists on disk:

- **Telemetry store** (`~/.sase/telemetry/metrics.sqlite`, enabled by default,
  Rust-owned, 48h raw / 30d 5-minute / 365d hourly rollups). Duration histograms for
  agent runs, LLM invocations, hooks, workflows, and axe cycles, plus counters for
  errors, retries, and tokens. Already queryable from Python through
  `sase.telemetry.query.query_range` / `store_stats`.
- **Durable TUI perf logs** under `~/.sase/logs` (`sase/logs/tui_telemetry.py`), all
  always-on and byte-bounded: `tui_startup.jsonl` (one record per ACE session),
  `tui_stalls.jsonl` (event-loop and message-pump stalls, hitches, and recoveries),
  `tui_launch_timing.jsonl`, `tui_agent_loads.jsonl`, `tui_git_ops.jsonl`, and
  `tui_external_tools.jsonl`.

The Perf view stitches those into one screen, ordered the way a user actually
experiences SASE: **open the TUI → it stays responsive → launch an agent → the agent
runs → the model answers.**

## Design

### Placement and identity

`Perf` becomes the **eighth and last** entry in `VIEW_ORDER`
(`src/sase/ace/tui/modals/statistics_pane_data.py`). Appending rather than inserting
keeps the existing 1–7 numbered selection stable; renumbering established views to make
room for a diagnostic view would be a poor trade. The `StatisticsPane` docstring ("Seven
numeric Statistics views…") becomes eight.

Labels: `Perf` full, `Perf` compact, `Prf` micro. Description: _"TUI responsiveness,
launch and agent latency, and the health of the data behind these numbers."_

### Scope semantics — the honest part

Perf is **global**. It is not project-scoped, because none of its sources can be:
telemetry duration histograms are labeled by provider/workflow/hook type, never by
project, and the TUI perf logs carry no project at all. Silently ignoring an active
project filter would make the view lie.

So while Perf is active, the project scope chip stays visible (so the user does not lose
track of a filter that will re-apply when they leave) but renders dimmed with a
`· not applied` suffix, and the legend states that Perf is global. `p`/`P` continue to
cycle the underlying filter; they simply have no effect on what Perf shows.

The range controls (`t`/`T`/`c`) apply normally, with a coverage caveat rendered in the
view: the telemetry store rolls raw samples up after 48 hours and the JSONL logs are
byte-bounded, so "All time" means "as far back as the retained data goes." This mirrors
the existing `plans_questions` coverage-start note.

### Grouping (`g`)

Perf joins `statistics_view_supports_grouping`. `PERF_GROUP_ORDER` is
`("subsystem", "provider", "workflow")`, rendered as _By Subsystem_ / _By Provider_ /
_By Workflow_. The mode selects how the latency-and-reliability table is grouped, and
each mode maps onto label sets that actually exist in `METRIC_DEFS`:

| Mode         | Rows                                                | Source metrics                                                                                                                                                                                             |
| ------------ | --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| By Subsystem | Agent runs, LLM calls, Hooks, Workflows, Axe cycles | each `*_duration_seconds` histogram, ungrouped                                                                                                                                                             |
| By Provider  | one row per provider                                | `sase_agent_run_duration_seconds` grouped by `llm_provider`; `sase_llm_invocation_duration_seconds`, `sase_llm_errors_total`, `sase_llm_retries_total`, and the three token counters grouped by `provider` |
| By Workflow  | one row per workflow                                | `sase_agent_run_duration_seconds` and `sase_workflow_duration_seconds` grouped by `workflow`                                                                                                               |

Because the grouped modes need _different_ telemetry queries, a group change on Perf
must reload (`_selection_changed(reload=True)`), unlike Projects and XPrompts which
regroup a cached payload. Only the active mode's queries run, keeping the steady-state
30-second refresh cost flat.

### Layout

```
┌ Startup ─┐┌ Stalls ──┐┌ Launch ──┐┌ Agent p95┐┌ LLM p95 ─┐   ← 5 hero tiles
└──────────┘└──────────┘└──────────┘└──────────┘└──────────┘     (same geometry
┌ Startup breakdown ────┐┌ Stalls & hitches ───┐                  as Overview)
└───────────────────────┘└─────────────────────┘
┌ Latency & reliability · By Subsystem ───────────────────────┐
└─────────────────────────────────────────────────────────────┘
┌ Data & instrumentation ─────────────────────────────────────┐
└─────────────────────────────────────────────────────────────┘
```

**Hero tiles** reuse `render_stat_tile` from `sase.telemetry.render`, so Perf inherits
the Overview tiles' exact visual language — same fixed geometry, same delta arrow, same
sparkline glyphs, same CVD-validated palette. Five tiles keep the existing
`#statistics-tiles` container and `_tile_width()` math unchanged.

1. **Startup** — median `visible_ready_seconds`, sparkline over the window's sessions,
   delta versus the preceding equal-length window, `lower_is_better=True`.
2. **Stalls** — stall + hitch count, with `worst 3.2s` as the second line.
3. **Launch** — p95 agent-launch `total_ms`, with `N launches · M slow stages`.
4. **Agent p95** — p95 `sase_agent_run_duration_seconds`, with the run count and delta.
5. **LLM p95** — p95 `sase_llm_invocation_duration_seconds`, with `err X% · retry Y%`.

Tiles 4 and 5 grade status through the **existing**
`sase.telemetry._config.HealthThresholds` (`p95_latency_warn/critical`,
`error_rate_warn/critical`), so the Perf view and `sase telemetry health` can never
disagree and the user's existing `telemetry.health_thresholds` config already tunes
them. Tiles 1–3 have no configured thresholds; they use module constants documented in
the help modal (startup ok < 2s / warn < 5s; stalls ok at zero, warn on any hitch,
critical on any stall). Making those configurable is a deliberate non-goal — see below.

**Startup breakdown** table: `Stage | p50 | p95 | max` over `process→mount`,
`mount→first paint`, `visible ready`, `all surfaces ready`, footed with the session
count and the slowest session's timestamp and initial tab.

**Stalls & hitches** table: `Event | Count | Worst | Median | Last seen` for
`tui_stall`, `tui_pump_stall`, `tui_hitch`, `tui_pump_hitch`, plus a _Top context_ row
ranking the `current_tab` values carried on those records — the single most actionable
thing in the view, because it names _where_ the TUI froze. Rate-limited hitches report
their suppressed count rather than silently undercounting.

**Latency & reliability** table: `Row | p50 | p95 | max | Count | Err% | Retry%` plus a
share bar, grouped per the active mode. In _By Provider_ mode it also carries tokens
in/out and cache-read share, which is the one genuinely useful cost-shaped signal
available.

**Data & instrumentation** strip — the reliability contract made visible:

- Telemetry: enabled/disabled, store size, raw/5m/1h row counts, the **resolution the
  store actually served for this range** (`raw`/`5m`/`1h`/`mixed`, straight off
  `TelemetryRangeQueryResultWire.resolution`), and per-subsystem write freshness.
- Logs: per source, records in window, earliest retained record, and whether the bounded
  writer truncated older records.
- Probes: whether `SASE_TUI_PERF` and `SASE_TUI_TRACE` are set, with the exact env var
  to enable each. The opt-in probes are _not_ read by this view; the strip only tells
  the user they exist and are off, so a curious user can enrich their own data.

### Reliability

Every failure mode degrades to a legible panel, never a blank view or a traceback:

- **Telemetry disabled** (`telemetry.enabled: false`) — the telemetry-fed tiles and
  table render an explanatory empty state naming the config key; the JSONL-fed TUI
  sections still work. Perf is never all-or-nothing.
- **No samples in range** — `sase.telemetry.render.empty_state` with the recording-start
  hint, the same treatment the shared render toolkit already gives.
- **Missing or empty log file** — that source reports `present: false` in the coverage
  strip; its section renders an empty state.
- **Malformed JSONL lines** — counted and reported as _"N unreadable records skipped"_,
  never fatal. This mirrors the Runners view's established "Partial valid snapshot"
  convention.
- **Missing Rust binding** — `require_rust_binding` raises and the pane's existing error
  state renders with the refresh hint.
- Perf joins `{"runners", "xprompts"}` in the `views.empty` bypass in
  `_paint_current_view`, because Perf has its own data and must render even when zero
  agent runs were recorded in the range.

### Performance discipline

The view must obey `sase/memory/tui_perf.md`. All loading already happens inside
`StatisticsPane`'s existing `run_worker(thread=True, exclusive=True)`; Perf adds no new
code path and touches neither the event loop nor a Textual pump callback. Bounded work
is enforced at both ends: the Rust reader tail-caps bytes and records per source, and
the telemetry fan-out is a **fixed, enumerated** query set (five histograms plus their
companion counters; at most one grouped variant active at a time) specified in this plan
so it cannot grow unnoticed. `_start_load` only requests the perf payload when
`view == "perf"`, and skips the previous-window comparison query for every other view's
sake.

### Non-goals

- No `sase stats perf` CLI. `sase telemetry health` already covers the terminal case.
- No new config keys. TUI status thresholds stay as documented module constants; making
  them configurable is a clean follow-up if the defaults prove wrong.
- The opt-in `~/.sase/perf/tui_jk.jsonl` and `tui_trace.jsonl` probes are reported as
  on/off, not parsed. They are debugging instruments with unbounded volume.
- No changes to keymaps or `src/sase/default_config.yml`: Perf introduces no new key,
  reusing `[`/`]`, `0`+`8`, `g`, `t`/`T`/`c`, `r`, and `?`.

## Rust perf-log aggregation and binding

Worked in the sibling **sase-core** repo (open it with the `/sase_repo` skill; never
edit it through any other path). Per `sase/memory/rust_core_backend_boundary.md`,
aggregation over a durable store is core backend logic: any other frontend would need
these numbers to match the TUI's.

Add `crates/sase_core/src/perf_logs/` with `mod.rs`, `sources.rs`, `aggregate.rs`, and
`wire.rs`, registered in `crates/sase_core/src/lib.rs`.

**Wire request** (`PerfLogsQueryWire`, schema version 1):

```
{
  "start_ts": i64, "end_ts": i64,
  "max_records_per_source": u64,   // default 20000
  "max_bytes_per_source": u64,     // default 8 MiB
  "sources": [{"id": "startup"|"stalls"|"agent_loads"|"launch_timing"
                     |"git_ops"|"external_tools",
               "path": "<absolute path>"}]
}
```

Paths are supplied by the caller rather than derived from a logs directory, because
every path in `sase/logs/tui_telemetry.py` is independently env-overridable
(`SASE_TUI_STALL_PATH` and friends) and tests rely on that.

**Wire response** (`PerfLogsSnapshotWire`) carries `startup` (session count, per-stage
`p50`/`p95`/`max`/`samples`, a capped per-session `visible_ready` series, and the
slowest session), `stalls` (per-kind count/worst/median/last-seen/suppressed, ranked
`current_tab` contexts, recovery count), `launches` (count, p50/p95/max ms, slow-stage
count, worst stages), `agent_loads`, `git_ops` (including timeout count),
`external_tool_waits`, and a `coverage` array with one entry per source recording
`present`, `records_scanned`, `records_in_window`, `earliest_ts`, `latest_ts`,
`truncated`, and `malformed_skipped`.

**Three details that will silently break this if missed:**

1. **The timestamp field is not uniform.** `tui_startup.jsonl` and
   `tui_agent_loads.jsonl` write `"timestamp"` as an ISO-8601 UTC string;
   `tui_stalls.jsonl`, `tui_launch_timing.jsonl`, `tui_git_ops.jsonl`, and
   `tui_external_tools.jsonl` write `"ts"` as a float epoch. The reader must accept both
   per source and count a record whose timestamp is unparseable as malformed rather than
   as in-window.
2. **Reads must be tail-bounded**, seeking from the end like `read_tail_seek` does for
   the Logs tab, so an All-time range never turns into an unbounded scan. When the cap
   truncates, set `truncated: true` for that source — the UI reports it rather than
   implying full coverage.
3. **Percentiles use nearest-rank on the sorted sample**, index `round(q * (n - 1))`
   clamped to `[0, n-1]` — identical to `JKPerfTimer.summary()` in
   `src/sase/ace/tui/util/perf.py`, so the always-on view and the opt-in probe never
   report different p95s for the same data.

Recoveries (`tui_stall_recovered`, `tui_pump_stall_recovered`, and the hitch variants)
carry `duration_seconds`, not `stall_seconds`; count them separately rather than folding
them into the stall rows.

Register `perf_logs_query(request: dict) -> dict` in `crates/sase_core_py/src/lib.rs`
following the `agent_stats_query_runs` pattern exactly (`py.allow_threads`, wire
deserialize from `PyDict`, `PyRuntimeError` on failure), and document it in that file's
module-level binding list.

Rust tests cover: mixed timestamp formats; a window that excludes older records; a
truncating cap; malformed and partially-written trailing lines; an absent file; an empty
file; percentile edges at n=1 and n=2; and hitch suppression counts. Land with the
sase-core repo's own `just` gates green.

Do **not** bump `sase-core-rs` in this repo's `pyproject.toml`. The release-branch
ratchet (`tools/ratchet_core_window`) owns that window; downstream phases build against
the local checkout with `just rust-install`.

## Python perf facade and view model

In this repo, alongside the existing statistics facade — `sase/stats/` is already the
"Python presentation facade for historical SASE statistics", and Runners set the
precedent of a view getting its own module there.

**`src/sase/stats/perf_query.py`** — thin adapters, no aggregation:

- `query_perf_logs(*, start_ts, end_ts, paths=None)` resolves the six canonical paths
  through `sase.logs` (honoring env overrides), calls
  `require_rust_binding("perf_logs_query")`, and returns the payload.
- `query_perf_telemetry(*, start_ts, end_ts, group_by)` runs the enumerated telemetry
  fan-out through `sase.telemetry.query.query_range` and `store_stats`, following the
  shape `sase/telemetry/cli_health.py::_collect_health_window` already uses. Returns
  `{"enabled": bool, ...}` and never raises for a disabled or empty store.

The enumerated query set, fixed by this plan:

- Quantile (`aggregation="quantile"`, `quantile=0.5` and `0.95`) and `max` over
  `sase_agent_run_duration_seconds`, `sase_llm_invocation_duration_seconds`,
  `sase_hook_duration_seconds`, `sase_workflow_duration_seconds`,
  `sase_axe_cycle_duration_seconds`.
- Sums over `sase_agent_runs_total` (total and `status=error`),
  `sase_llm_invocations_total`, `sase_llm_errors_total`, `sase_llm_retries_total`,
  `sase_hook_executions_total`, `sase_hook_retries_total`, and the three
  `sase_llm_*_tokens_total` counters.
- In a grouped mode, the grouped variants named in the Design table replace their
  ungrouped counterparts — they are not additive.
- One `store_stats()` call.

**`src/sase/stats/_perf_view.py`** — immutable, I/O-free view models
(`frozen=True, slots=True`) and
`build_perf_view(perf_payload, telemetry_payload, *, selected_range, previous_perf_payload=None, group_by="subsystem", now=None)`.
It owns: status grading (reusing `HealthThresholds` for latency and error rate, module
constants for startup and stalls), delta computation against the preceding equal-length
window, coverage/truncation/malformed notes, and an `available` flag per section so the
renderer never has to guess. Export `PerfView` and `build_perf_view` from
`src/sase/stats/__init__.py`.

Wire it into `src/sase/ace/tui/modals/statistics_pane_data.py`: add `perf` to the
`StatisticsView` literal and a `perf: PerfView | None = None` field to
`StatisticsViewData` (top level, not inside `StatisticsViews`, so lazy loading stays
obvious and `build_statistics_views` and its tests are untouched).
`load_statistics_view` gains a `perf_group_by` parameter and only builds the perf
payload when `view == "perf"`.

Tests in `tests/stats/test_perf_query.py` and `tests/stats/test_views_perf.py` cover
grading at each threshold boundary, delta math, disabled telemetry, empty payloads,
partial coverage, and the group-mode query selection — with the binding and telemetry
adapters faked, following `tests/stats/_views_payloads.py`.

## Perf view registration and interaction

All in `src/sase/ace/tui/modals/`.

`statistics_pane_data.py`: append `"perf"` to `VIEW_ORDER`; add `VIEW_LABELS`,
`VIEW_COMPACT_LABELS`, `VIEW_MICRO_LABELS` (`Prf`), and `VIEW_DESCRIPTIONS` entries; add
`PerfGroupBy` and `PERF_GROUP_ORDER`; return `True` from
`statistics_view_supports_grouping("perf")`.

`statistics_pane.py`: add `_perf_group_by` state; extend `action_cycle_group` with the
perf branch that reloads rather than repaints; thread `perf_group_by` through
`_start_load`, the worker `task()`, and the `on_worker_state_changed` staleness guard (a
perf result for a stale group must trigger a reload, exactly like a stale range does);
add `"perf"` to the `views.empty` bypass; update the class docstring to eight views.

`statistics_pane_rendering.py`: `_PERF_GROUP_LABELS`; the grouped scope chip; the dimmed
`· not applied` project chip while Perf is active; a `_PERF_STACK_BELOW_WIDTH` threshold
(108) plus the matching `on_resize` repaint branch, mirroring the runners stacking logic
exactly.

`statistics_pane_legends.py`: a `perf` entry defining p50/p95, Startup, Stall, Hitch,
Global, Coverage, and Resolution.

`statistics_help_modal.py`: a _Perf methodology_ section covering percentile method, the
two threshold sources (configured `telemetry.health_thresholds` versus the documented
startup/stall constants), why Perf is global, the retention and rollup caveat, and what
each probe env var would add.

Check the tab-strip tiering: an eighth tab widens the full tier, so re-measure and, if
needed, raise `_VIEWS_COMPACT_BELOW_WIDTH` (111) and `_VIEWS_MICRO_BELOW_WIDTH` (75) so
the strip never overflows at 120 or 90 columns, and extend
`tests/ace/tui/test_panel_tab_strip_compact.py`.

Tests: extend `tests/ace/tui/_statistics_pane_helpers.py` with a perf payload fixture,
and `test_statistics_view_number_select.py` (1–8),
`test_statistics_pane_interactions.py` (group cycling reloads on Perf),
`test_statistics_scope_header.py` (project chip suffix),
`test_statistics_legends_states.py`, `test_statistics_pane_bindings.py`, and
`test_statistics_help_modal.py`.

## Perf view rendering

New `src/sase/ace/tui/modals/statistics_pane_perf.py` holding
`StatisticsPerfRenderingMixin`, mixed into `StatisticsViewsRenderingMixin` beside the
runners, projects, and xprompts mixins, and dispatched from `_view_renderable`. Follow
the house rules those modules establish: pure rendering, zero I/O, everything read off
the already-loaded immutable snapshot, `Any`-typed view arguments, module-level color
constants.

Reuse `sase.telemetry.render` rather than hand-rolling: `render_stat_tile` for the hero
row, `render_sparkline` for the startup trend, `format_duration` / `format_percentage` /
`format_tokens` for values, `status_color` for grading, `categorical_color` for
per-provider and per-workflow series, and `empty_state` for absent data. This is what
keeps Perf visually of a piece with Overview instead of looking bolted on.

`_paint_overview_tiles` currently hard-codes the Overview tiles; add a sibling
`_paint_perf_tiles` and select between them in `_paint_current_view`, keeping
`_set_tiles_visible(self._view in {"overview", "perf"})`. No `styles.tcss` changes are
needed — Perf reuses `#statistics-tiles`, `.statistics-tile`, and
`#statistics-body-scroll` as they stand. The `_OverviewTile` click-to-navigate message
is Overview-specific; Perf tiles are non-interactive, so mount them as plain tiles
rather than giving them a misleading tooltip.

Render every state named in Design → Reliability, including the two-column startup /
stalls section collapsing to stacked panels below `_PERF_STACK_BELOW_WIDTH`.

Tests in a new `tests/ace/tui/test_statistics_perf.py`, following
`tests/ace/tui/test_statistics_runners.py`: assert on the rendered text of each panel
for a populated snapshot, a telemetry-disabled snapshot, a no-logs snapshot, a
partial-coverage snapshot with malformed and truncated sources, and each group mode;
plus the narrow-width stacking flip.

## Visual snapshots and documentation

Extend `tests/ace/tui/visual/_ace_config_center_statistics_helpers.py` with a perf
payload for `_patch_statistics_populated` and a new `_patch_statistics_perf_degraded`
that returns a telemetry-disabled, partial-coverage snapshot — the degraded state is
where a dashboard usually turns ugly, so it gets a golden of its own.

Add to `tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py`:
`config_center_statistics_perf_120x40`, `config_center_statistics_perf_90x30` (asserting
the stacked layout), and `config_center_statistics_perf_degraded_120x40`. Generate the
goldens with `just test-visual --sase-update-visual-snapshots` and inspect
`.pytest_cache/sase-visual/` artifacts before accepting them. The help-modal golden also
moves, since the Perf methodology section is new content.

Add a _Reading the Admin Center Perf view_ section to `docs/perf_runbook.md` that maps
each panel to its source file or metric, states the retention and rollup caveat, and
points at `SASE_TUI_PERF` / `SASE_TUI_TRACE` for the deeper capture workflow the runbook
already documents.

## Verification

Every phase runs `just install` first (workspaces are ephemeral), then `just check`
before handing off. The phases touching the Statistics pane also touch the broadening
set, so land the combined tree with `just check-full` — run through `/sase_monitor` with
a `--next` action, never inline. `just test-visual` is required by the final phase and
is worth a spot-check in `perf_render` too.
