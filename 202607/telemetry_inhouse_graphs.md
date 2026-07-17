---
tier: epic
title: In-house telemetry graphs for CLI and TUI
goal: 'SASE telemetry graphs work out of the box with no external services: metric
  history is persisted locally through a new Rust core store, rendered by an in-house
  terminal chart toolkit in both the reworked `sase telemetry` CLI and a new Admin
  Center Telemetry tab, and all Grafana/Prometheus/Pushgateway support is removed.

  '
phases:
- id: core-store
  title: Rust core metric store and query engine
  depends_on: []
  description: '''Rust core metric store and query engine'' section: add a SQLite-backed
    metric sample store with delta ingestion, instant/range queries, rollups, and
    retention to sase-core; expose it through sase_core_rs bindings and release a
    0.5.x wheel.'
- id: charts
  title: In-house terminal chart toolkit
  depends_on: []
  description: '''In-house terminal chart toolkit'' section: build deterministic Rich-based
    braille line charts, block bar charts, sparklines, and stat tiles with a fixed
    validated palette, shared by the CLI and the TUI.'
- id: ingest
  title: Local ingestion pipeline
  depends_on:
  - core-store
  description: '''Local ingestion pipeline'' section: replace prometheus_client accumulation
    and Pushgateway/exposition egress with in-house accumulators that flush batches
    to the core store; enable telemetry by default and drop the prometheus_client
    dependency.'
- id: cli
  title: Reworked sase telemetry CLI
  depends_on:
  - charts
  - ingest
  description: '''Reworked sase telemetry CLI'' section: rebuild dashboard, snapshot,
    health, and status on local store queries, add a graph subcommand, and remove
    the scrape/prom_query/plotext layers.'
- id: tui
  title: Admin Center Telemetry tab
  depends_on:
  - charts
  - ingest
  description: '''Admin Center Telemetry tab'' section: add a seventh Telemetry tab
    to the Admin Center with stat tiles, agent-focused charts, range and subsystem
    switching, worker-based loading, and PNG snapshot coverage.'
- id: teardown
  title: Grafana and monitoring stack removal
  depends_on:
  - cli
  - tui
  description: '''Grafana and monitoring stack removal'' section: delete the bundled
    Docker/Grafana/Prometheus stack, export-config, and the pushgateway chop script;
    rewrite docs/telemetry.md and sweep the repo for stale references.'
- id: smoke
  title: End-to-end exercises and perf validation
  depends_on:
  - teardown
  description: '''End-to-end exercises and perf validation'' section: exercise recording,
    CLI graphs, and the TUI tab against real activity; confirm visual snapshots, j/k
    latency, and startup timing are unaffected.'
create_time: 2026-07-17 11:24:59
status: wip
---

# Plan: In-house telemetry graphs for CLI and TUI

## Context and problem

SASE ships a Prometheus-based telemetry subsystem (40 metrics across 8 subsystems, defined once in
`src/sase/telemetry/metrics.py:METRIC_DEFS`) with a CLI
(`sase telemetry status|list|snapshot|dashboard|health|export-config`) and a bundled Docker Compose stack (Pushgateway +
Prometheus + Grafana). It has never worked right, for a structural reason: **there is no local storage**. Metrics live
only in-process; short-lived agent runners push to a Pushgateway on exit and the axe orchestrator exposes an HTTP
endpoint for Prometheus to scrape. Every graph therefore requires the external Docker stack to be running, with
telemetry explicitly enabled (`telemetry.enabled` defaults to `false`). In practice the stack is never up, so:

- `sase telemetry dashboard --charts` silently falls back to a summary view whose "sparklines" repeat the current value
  eight times (`cli_dashboard.py:178`) — flat fake graphs.
- `snapshot` and `health` fail unless a Pushgateway or exposition endpoint is reachable.
- The plotext chart layer wraps `plt.build()` in crash-guards for known plotext bugs (`charts.py:71-75, 104-108`) and
  silently renders empty panels.
- Grafana dashboards — the intended "great telemetry" experience — depend on all of the above working and are
  effectively dead weight.

This epic replaces the whole external pipeline — not just Grafana — with a local-first, in-house design: metric history
persisted through the Rust core, one in-house chart toolkit, graphs in the CLI and in a new Admin Center Telemetry tab,
telemetry enabled by default. Grafana, Prometheus, and the Pushgateway are removed entirely.

## Design overview

```
Instrumentation call sites (~60, unchanged API: .labels().inc()/.observe()/.set())
        │
        ▼
In-house metric accumulators (src/sase/telemetry/metrics.py singletons; stubs when disabled)
        │  periodic flush (long-lived procs, default 15s) + atexit flush (runners)
        ▼
sase_core_rs telemetry bindings  ──►  SQLite store  ~/.sase/telemetry/metrics.sqlite
        ▲                              (sase-core: WAL, bounded busy_timeout,
        │  instant/range queries,       raw samples + 5m/1h rollups, retention)
        │  quantiles, rollup aggregation in Rust
        │
   ┌────┴──────────────────────────────┐
   │                                   │
sase telemetry CLI                Admin Center ▸ Telemetry tab
(dashboard, graph, health,        (TelemetryPane: stat tiles + chart grid,
 list, snapshot, status)           worker-based loading, auto-refresh)
   │                                   │
   └──────── in-house chart toolkit ───┘
             (braille lines, block bars, sparklines, stat tiles — Rich renderables)
```

### Key design decisions

1. **Storage and queries live in the Rust core.** Both the CLI and the TUI (and any future frontend) need identical
   time-series behavior, which is the core-boundary litmus test. `sase-core` already has every idiom the store needs:
   rusqlite with WAL + bounded `busy_timeout` and schema-version migrations
   (`crates/sase_core/src/agent_scan/index.rs`), grouped aggregate queries (`agent_archive` facet counts), chrono time
   handling, and the `*Wire` + pyo3 binding registry pattern. Python keeps owning on-disk path resolution and passes the
   store path in, matching existing convention.
2. **Delta ingestion instead of cumulative scraping.** Processes flush _deltas_ since the last flush for counters and
   histogram aggregates (count/sum/per-bucket increments), and source-tagged _last values_ for gauges. Rate/total
   queries become simple SUMs over time buckets — no Prometheus counter-reset detection. Gauge instant values aggregate
   the most recent sample per (labels, source) within a staleness window (default 2× flush interval), so metrics from
   dead processes age out automatically. This eliminates the stale-pushgateway-group problem (and its cleanup script) by
   construction.
3. **Telemetry enabled by default.** With no network servers involved, recording costs a dict update per event plus one
   small batched SQLite write per flush interval per process. That is cheap enough to be on by default, which is what
   makes the graphs trustworthy: data is simply there. `telemetry.enabled: false` remains as an opt-out and keeps the
   existing no-op stub behavior.
4. **The instrumentation API is frozen.** All ~60 call sites keep `METRIC.labels(...).inc()/.observe()/.set()` and
   `METRIC_DEFS` stays the single source of truth (catalog, store schema hints, chart specs all derive from it). Only
   the backend under `metrics.py` changes; the ~10 egress call sites move from `push_metrics`/`register_push_on_exit` to
   `flush_metrics`/`register_flush_on_exit`.
5. **One in-house chart toolkit for both surfaces.** A new rendering package produces Rich renderables (braille-canvas
   line charts, eighth-block bar charts, sparklines, stat tiles, axes/legends) used by the CLI dashboard/graph commands
   and the TUI pane alike. plotext is removed. Output is deterministic (injected clock, no randomness) so both
   golden-text unit tests and PNG snapshots stay stable.
6. **Beauty is specified, not hoped for.** Chart design follows fixed rules: categorical series hues assigned in a fixed
   order keyed by entity (provider, workflow, subsystem) — never cycled or rank-dependent; status semantics
   (ok/warn/critical) use a reserved green/amber/red trio and are never reused for ordinary series; one y-axis per chart
   (never dual-axis); recessive grids/axes in muted grays; legends plus selective direct labels; the palette draws from
   the Admin Center accent family (`#00D7AF`, `#FFD700`, `#FFAF5F`, `#5FD75F`, `#AF87FF`, `#87D7FF`) so the tab feels
   native, with pairwise distinguishability checked for color-vision deficiency at design time.

## Phase details

### Rust core metric store and query engine

Work happens in the `sase-core` linked repo (open it with `sase repo open sase-core`), following its established module
recipe.

- New `crates/sase_core/src/telemetry/` module:
  - SQLite schema: `samples(ts, metric, kind, labels_json, source, value)` for raw deltas and gauge samples, plus
    `rollup_5m` and `rollup_1h` tables holding pre-aggregated (bucket_ts, metric, labels, count/sum/min/max, histogram
    buckets). `meta` table with `schema_version`, WAL mode, bounded `busy_timeout`, and corruption quarantine/recovery
    mirroring `agent_scan/index.rs`.
  - Ingestion: `record_batch` inserts a flush batch in one transaction and opportunistically folds expired raw rows into
    rollups.
  - Queries: `query_instant` (current values; gauge staleness-window aggregation) and `query_range` (start/end/step
    bucketing with sum/avg/rate aggregation, group-by label, and histogram quantile interpolation — the
    `histogram_quantile`/percentile math moves here from Python's `scrape.py`). Range queries transparently choose raw
    vs rollup resolution based on the requested window.
  - Retention/prune: raw 48h, 5m rollups 30d, 1h rollups 1y (all caller-configurable); `prune` is invoked
    opportunistically by long-lived writers, never by readers.
  - `store_stats` (db size, sample counts, last-write timestamps per subsystem) for the status command.
- `wire.rs` with serde `*Wire` models and a schema-version constant; pyo3 wrappers in `crates/sase_core_py`
  (`py_telemetry_record_batch`, `py_telemetry_query_instant`, `py_telemetry_query_range`, `py_telemetry_prune`,
  `py_telemetry_store_stats`) registered in the `#[pymodule]` block, all taking the store path and a bounded
  busy-timeout from Python.
- Rust unit tests: multi-writer concurrency, delta aggregation correctness, gauge staleness, rollup fold + retention,
  quantile math, corruption recovery.
- Release a new `sase-core-rs` 0.5.x wheel per that repo's release flow (additive API, so the existing `>=0.5.0,<0.6.0`
  pin range still matches; the ingest phase raises the floor). Downstream phases can develop against the workspace-local
  dev build, which editable installs already prefer.

### In-house terminal chart toolkit

New rendering package `src/sase/telemetry/render/` in this repo (pure presentation — no store access, so this phase runs
parallel to the core work; it consumes plain series/point data models).

- Components, all returning Rich renderables sized to a given width/height:
  - `braille.py` — a braille-dot canvas (2×4 dots per cell) used by `line.py` for smooth multi-series line charts with
    time x-axis, auto-scaled single y-axis, and legend.
  - `bars.py` — eighth-block horizontal/vertical bars for categorical comparisons.
  - `sparkline.py` — compact block-glyph sparklines fed by real range-query series (the fake repeated-value sparkline
    dies here).
  - `tiles.py` — stat tiles: big value, caption, delta vs previous window, optional embedded sparkline, optional status
    coloring.
  - `axis.py` / `palette.py` — shared time/value axis formatting (humanized durations, token counts, percentages) and
    the fixed categorical + reserved status palette from design decision 6.
- Deterministic rendering: the current time and terminal theme are inputs; identical data yields identical output,
  enabling golden-text unit tests in `tests/telemetry/render/` covering empty series, single point, many series,
  clipping, and narrow-width behavior.
- Graceful degradation: every component renders a labeled empty state ("no samples in range — telemetry began recording
  <time>") instead of blank panels or exceptions.

### Local ingestion pipeline

- Replace the real prometheus_client metric classes behind `metrics.py` with in-house `Counter`/`Gauge`/`Histogram`
  accumulators exposing the exact existing API (`.labels().inc()/.observe()/.set()`); `_stubs.py` no-op behavior and
  stub→real forwarding semantics are preserved for the disabled path.
- `_registry.py` rework: `init_telemetry()` keeps its name (drops `start_http_server`); a background flusher thread
  flushes deltas every `telemetry.flush_interval_seconds` (default 15) in long-lived processes (orchestrator,
  lumberjacks); short-lived runners flush once at exit. `push_metrics`/`register_push_on_exit` become
  `flush_metrics`/`register_flush_on_exit`; the ~10 egress call sites (agent runner, lumberjack,
  mentor/crs/fix/summarize-hook runners, orchestrator) are updated. Each process tags its samples with a `source` (job
  name + pid) for gauge aggregation.
- Flush failures are swallowed after bounded retries and never propagate into agent or daemon work; a debug-level log
  line records drops.
- Config: `telemetry.enabled` defaults to `true`; remove `telemetry.prometheus.*`; add `flush_interval_seconds` and
  `retention` knobs in `src/sase/default_config.yml` and `_config.py`; tolerate and ignore a legacy `prometheus:` block.
  Store path resolution (`~/.sase/telemetry/metrics.sqlite`) lives in `_config.py`.
- Raise the `sase-core-rs` floor in `pyproject.toml` to the wheel released by the core phase and drop the
  `prometheus_client` dependency.
- Rework `tests/telemetry/` accordingly: accumulator semantics, flush batching, delta correctness across simulated
  multi-process flushes (tmp store), stub forwarding, config defaults, and the existing instrumentation tests
  (`test_instr_*`) now asserting against the in-house registry.

### Reworked sase telemetry CLI

Follow the CLI conventions memory (alphabetical listings, short aliases for every public long option, colored output;
bare `sase telemetry` keeps delegating to `list` via the central mechanism).

- `dashboard` — the flagship live view, rebuilt on store queries + the render toolkit inside `rich.live.Live`: header
  stat-tile strip (Active Agents, Active Workspaces, Active Beads, Runs in range, Error rate with status color), then an
  agent-first chart grid (Agent Runs by status, Run Duration p50/p95, LLM Tokens by provider, Error Rate %), with
  `-s/--subsystem` switching chart sets and `-r/--range` (`15m|1h|6h|24h|7d`) selecting the window. `--source` and
  `--charts` flags are removed — charts are always available now. Falls back to a friendly empty state when the store
  has no samples.
- `graph` — new subcommand for one-shot, pipeable charts:
  `sase telemetry graph <metric> [-r range] [-g group-by-label] [-a agg] [-w width]`, printing a single rendered chart
  to stdout (with `--no-color` honoring standard conventions). This is the scriptable/low-level entry point.
- `snapshot` — instant store query instead of scraping; keeps `-f rich|json`; the raw `prometheus` output format is
  removed.
- `health` — same thresholds and exit codes (0/1/2), computed from store range queries over the last hour;
  `build_telemetry_health_payload`/`build_telemetry_status_payload` keep their signatures so `sase doctor` integration
  continues to work unchanged.
- `status` — shows enabled state, store path/size, last-write freshness per subsystem, and flusher health; reachability
  probes are gone.
- Delete `scrape.py`, `prom_query.py`, and the plotext `charts.py`; drop the `plotext` dependency. Update
  `parser_telemetry.py`/`telemetry_handler.py` and all affected tests (`test_cli_*`, `test_scrape`, `test_prom_query`,
  `test_charts` replaced by render-toolkit and new CLI tests).

### Admin Center Telemetry tab

Add the seventh tab to the Admin Center (`ConfigCenterModal`), following the established registration recipe and the TUI
performance rules.

- Registration: extend `CenterTab`, `_TAB_ORDER`, `_TAB_LABELS`, `_TAB_COLORS` (accent `#FF87D7`), `_TAB_DESCRIPTIONS`,
  and `compose()` in `src/sase/ace/tui/modals/config_center_modal.py`; digit/Tab cycling, numbering, and tab persistence
  all derive from `_TAB_ORDER` automatically. Update the three help-modal strings from "1-6" to "1-7"
  (`agents_bindings.py`, `axe_bindings.py`, `changespecs_bindings.py`) plus the xprompt filter-input docstring, per the
  help-popup maintenance convention. Add `TelemetryPane` to the shared sizing selector in `styles.tcss` and a pane style
  block. Add a command-palette entry alongside the existing Admin Center commands.
- `modals/telemetry_pane.py` — `TelemetryPane`, modeled on the `LogsPane` skeleton:
  - Layout: top stat-tile strip; a chart grid (default view: Agents — Runs by status, Run Duration p50/p95, LLM Tokens
    by provider, Error Rate %); a one-line health strip at the bottom reusing the health payload with status colors.
  - Keys (pane-local `BINDINGS`, documented in the help modal): cycle subsystem chart sets, cycle time range
    (`15m/1h/6h/24h/7d`), `r` refresh. Any new keymap entries also land in `src/sase/default_config.yml`.
  - Loading: `_loading` flag, `on_mount` → `_start_load`, `run_worker(thread=True, exclusive=True)` performing all store
    queries (bounded busy timeout) and pre-computing chart data models off-thread; `on_worker_state_changed` paints on
    the UI thread from the frozen result. Range and subsystem switches coalesce through `DetailPanelDebouncer`; a
    `set_interval` revalidation tick (thin, synchronous, ~10s, active only while the tab is visible) triggers the same
    coalesced worker path. Zero work is added to app startup — the pane loads only when opened.
- Visual snapshot coverage: `_open_telemetry_modal` helper, a deterministic `_patch_telemetry_*` data patcher injecting
  fixed series (no store/disk reads), a new `test_ace_png_snapshots_config_center_telemetry.py` with
  loading/empty/populated variants, and golden `config_center_telemetry_tab_120x40.png` snapshots.

### Grafana and monitoring stack removal

The final sweep, once nothing references the legacy pipeline:

- Delete `src/sase/telemetry/monitoring/` (Docker Compose, Prometheus config + alert rules, all Grafana provisioning and
  dashboard JSON), `cli_export_config.py`, its parser/ handler entries, and `tests/telemetry/test_cli_export_config.py`.
- Remove the `sase_chop_pushgateway_cleanup` script: module, `sase.scripts` export, and its `[project.scripts]` entry
  (stale-group cleanup is obsolete under delta ingestion).
- Rewrite `docs/telemetry.md` around the new architecture: local store, config reference, CLI commands (including
  `graph`), the Admin Center Telemetry tab, retention policy, and a short migration note for users of the old Docker
  stack.
- Repo-wide sweep asserting zero remaining references to grafana/prometheus/pushgateway/plotext (source, tests, docs,
  config, packaging), with Symvision whitelists updated for removed symbols as needed.

### End-to-end exercises and perf validation

- Functional: with telemetry on defaults, run real activity (a short agent run, bead operations, notifications), then
  verify `sase telemetry status/snapshot/health/dashboard/ graph` all show live data with no external services; verify
  the Telemetry tab renders populated charts for the same activity, and that range/subsystem switching and refresh
  behave; verify the empty-store and disabled states render their friendly fallbacks.
- Performance: confirm `SASE_TUI_PERF=1` j/k p95 stays under 16 ms on all tabs with the Telemetry tab present; confirm
  TUI startup timing is unchanged (no new startup imports or I/O); run the `just test-visual` suite and the relevant
  bench tables (`tests/ace/tui/bench_tui_jk.py`) comparing against pre-change baselines; sanity-check flusher overhead
  in a long-lived axe process (CPU/RSS steady, one small write txn per interval) and store growth over a simulated day
  against retention expectations.
- Gates: `just check` green; visual snapshot suite green; stall watchdog log clean during a soak of the Telemetry tab
  with auto-refresh active.

## Testing strategy (cross-phase)

- Rust: cargo unit tests for schema/migrations, delta aggregation, gauge staleness, quantiles, rollups/retention, and
  concurrent writers.
- Python unit: render-toolkit golden-text tests; accumulator/flush tests against tmp stores; CLI tests asserting
  rendered output and exit codes; config default/migration tests; doctor integration tests unchanged in behavior.
- TUI: pane unit tests (worker lifecycle, coalescing, selection/state across refresh) and PNG visual snapshots for the
  new tab in loading/empty/populated states.
- Every phase that changes files in this repo runs `just check` before completion; phases touching the visual surface
  also run `just test-visual`.

## Performance budget

- Instrumentation call overhead (enabled): in-process dict/int ops only — no I/O, no locks beyond a mutex around the
  accumulator; disabled path remains no-op stubs.
- Egress: one batched transaction per process per flush interval (default 15s) or at exit; bounded busy timeouts;
  failures never block or crash the instrumented process.
- TUI: all queries and chart-model computation off the event loop; paint from precomputed models; j/k p95 < 16 ms
  preserved; no startup cost; refresh ticks are thin, coalesced, and paused while the tab is hidden.
- Store: rollups + retention keep the database small (target well under ~50 MB steady state at default retention);
  readers never prune.

## Risks and mitigations

- **Braille/line rendering quality** is the main "beautiful" risk. Mitigation: golden tests reviewed via PNG snapshots
  at design time; bars/sparklines (simple glyph math) ship first within the toolkit phase; the line chart degrades to a
  filled block chart at very narrow widths.
- **Multi-process SQLite contention**: mitigated by WAL + bounded busy_timeout (the proven agent-index pattern), small
  batches, and low write rates; contention failures degrade to dropped flushes, never blocked agents.
- **Wheel release coordination**: downstream phases develop against the workspace-local sase-core dev build; the floor
  bump lands only after the 0.5.x release, with precedent in the sase-6b epic.
- **Enabled-by-default surprises**: overhead is measured in the smoke phase; the opt-out is documented; flush failures
  are silent-but-logged.
- **Gauge staleness semantics** (active counts fading after crashes) are a behavior change from push-forever semantics —
  an improvement, but documented explicitly in docs/telemetry.md.
- **Symvision lint on removals**: deleting public symbols across phases may require whitelist updates; each phase runs
  the lint gate and follows the symvision guidance.

## Out of scope / future work

- Alerting or notification delivery on health regressions (health thresholds and exit codes remain; wiring them to sase
  notifications is future work).
- JSON/export formats for `graph` output and external dashboard integrations.
- Backfilling or importing historical Prometheus data from existing stacks.
