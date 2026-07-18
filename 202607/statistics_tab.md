---
tier: epic
title: Replace the Admin Center Telemetry tab with a Statistics tab
goal: 'The Admin Center''s Telemetry tab becomes a Statistics tab that answers concrete
  questions about historical SASE agent activity (runs, outcomes, commits, providers/models/efforts,
  skills, memories, workspaces, plans, runtime by tribe/clan/family/agent, and questions)
  over preset and custom time ranges, across multiple numeric views; telemetry infrastructure
  that is not useful for later debugging is removed.

  '
phases:
- id: core-run-stats
  title: Rust core agent run statistics engine
  depends_on: []
  description: '''Rust core agent run statistics engine'' section: add an agent_stats
    module to sase_core that aggregates per-run records from the agent artifact index
    over a caller-supplied time range (outcomes, provider/model/effort, commits, plan/question
    counters from run metadata, runtime grouped by tribe/clan/family/agent/provider/model/workflow,
    time-bucketed run counts), with wire types, PyO3 bindings, and Rust tests.'
- id: core-activity-stats
  title: Rust core activity log statistics
  depends_on:
  - core-run-stats
  description: '''Rust core activity log statistics'' section: extend agent_stats
    with aggregators over the durable file logs — per-project skill_uses.jsonl and
    memory_reads.jsonl, user_question session request files, and plan files (tier
    + phase counts) — with wire types, bindings, and Rust tests.'
- id: stats-facade
  title: Python statistics facade and time ranges
  depends_on:
  - core-run-stats
  - core-activity-stats
  description: '''Python statistics facade and time ranges'' section: add a thin src/sase/stats/
    package that wraps the new Rust bindings, defines the preset time ranges and the
    custom range grammar, and builds presentation-ready view models, with pytest coverage.'
- id: statistics-pane
  title: Statistics tab in the Admin Center
  depends_on:
  - stats-facade
  description: '''Statistics tab in the Admin Center'' section: replace the Telemetry
    tab/pane with a Statistics tab hosting six numeric views behind a nested sub-tab
    strip, preset/custom range switching, a runtime group-by cycle, new keymaps, persisted-tab
    migration, behavior tests, and PNG visual snapshots.'
- id: telemetry-slimdown
  title: Telemetry infrastructure slim-down
  depends_on:
  - statistics-pane
  description: '''Telemetry infrastructure slim-down'' section: remove the chart-oriented
    telemetry CLI (dashboard, graph) and chart-only render modules, drop product-usage
    metrics that Statistics now answers while keeping the debugging/health surface,
    align the bare `sase telemetry` default with the list convention, and update tests
    and docs.'
create_time: 2026-07-18 18:31:59
status: wip
bead_id: sase-6y
---

# Plan: Replace the Admin Center Telemetry tab with a Statistics tab

## Context

The Admin Center (`src/sase/ace/tui/modals/config_center_modal.py`) currently has a **Telemetry** tab
(`telemetry_pane.py` / `telemetry_pane_data.py`): five stat tiles plus a 2×2 grid of braille line charts fed by the
Prometheus-style counter/gauge/histogram store (`~/.sase/telemetry/metrics.sqlite`, owned by the Rust core). The graphs
do not answer the questions the user actually has, and the underlying metric store _cannot_ answer most of them: it has
no per-agent records, no model/effort labels, no clan/family/tribe identity, no skills, plans, or questions tracking,
and raw samples roll up after 48h.

Meanwhile the data to answer every question already exists, durably:

- **Per-run records** — `~/.sase/agent_artifact_index.sqlite`, table `agent_artifacts` (a rebuildable index over the
  per-run artifact marker dirs; the markers are the source of truth). Rust core owner:
  `crates/sase_core/src/agent_scan/index.rs`. Columns include `project_name`, `agent_clan`, `agent_family`, `timestamp`,
  `status`, `model`, `llm_provider`, `started_at`, `finished_at`, `workflow_status`, retry-chain columns, and a full
  `record_json` blob per row. `record_json` verified to contain:
  - `agent_meta`: `reasoning_effort`, `clan_tribe`, `agent_clan`, `agent_family`, `name`, `run_started_at`,
    `stopped_at`, `workspace_num`, `workspace_dir`, `plan_submitted_at[]`, `plan_approved`, `plan_action`, `plan_path`,
    `sdd_plan_path`, `questions_submitted_at[]`, `question_session_id`, retry fields.
  - `done`: `outcome` (`completed` / `failed` / `plan_rejected` / `epic_approved` / `epic_launch_failed` / `stopped` /
    `noop`), `finished_at`, `workspace_num`, and `step_output.meta_commits[]` — one entry per commit with `sha`,
    `message`, `repo_name`, `cwd`, `diff_path`.
- **Skill usage** — `~/.sase/projects/<project>/skill_uses.jsonl` (`SkillUseEvent`, `src/sase/skills/use_log.py`):
  `timestamp`, `skill_name`, `agent_name`, `runtime`, `reason`.
- **Memory reads** — `~/.sase/projects/<project>/memory_reads.jsonl` (`MemoryReadEvent`, `src/sase/memory/read_log.py`):
  `timestamp`, `canonical_path`, `agent_name`.
- **Questions** — `~/.sase/user_question/<session_id>/question_request.json` (`questions[]` array; its length is
  "questions asked at once") and `question_response.json`.
- **Plans** — plan files in the SDD plans sidecar and mirrored under `~/.sase/plans/YYYYMM/`, with `tier: tale|epic`
  frontmatter and, for epics, a `phases:` list. The Rust core already parses tier frontmatter
  (`crates/sase_core/src/plan/read.rs`) and knows the effort vocabulary (`crates/sase_core/src/effort.rs`). NOTE:
  proposed-plan counts must come from `agent_meta.plan_submitted_at[]` (durable), not from the
  pending-notification-derived collectors in `src/sase/main/plan_inventory_collectors.py` (notifications are
  dismissible).

Per the Rust Core Backend Boundary rule, the aggregation engine belongs in `sase_core` (any frontend must get identical
numbers); Python stays a thin config/presentation adapter — the established model is `src/sase/telemetry/query.py` over
the `telemetry_*` bindings. The Rust repo is the `sase-core` linked repo; implementing agents MUST open it with the
`/sase_repo` skill and work in the checkout it prints. Bindings live in `crates/sase_core_py/src/lib.rs` (module
`sase_core_rs`); Python loads them via `require_rust_binding` (`src/sase/core/rust.py`).

## Product design

### The Statistics tab

Rename the Admin Center tab **Telemetry → Statistics** (tab key `statistics`). `_TAB_ORDER` stays alphabetical:
`config, logs, projects, statistics, tasks, updates, xprompts`. Keep the existing accent `#FF87D7`; description becomes
e.g. "Aggregate SASE agent activity over any time range". Persisted-tab state (`src/sase/ace/admin_center_tab.py`) maps
a stored legacy `"telemetry"` value to `"statistics"` on load.

Header line: `Statistics · <View> · <Range label> · updated HH:MM:SS`, followed by a nested `PanelTabStrip` of views
(precedent: the Projects pane's sub-tabs), the view body, and a hints line. Loading / empty / error states mirror the
current pane (empty state says no agent runs were recorded in the selected range).

### Views (nested sub-tabs, `[` / `]` to cycle)

Graphs policy, per the user: **numbers first; a graphic may appear only directly beside its exact numbers.** Allowed:
stat tiles (big number + delta + sparkline), labeled share/scale bars in table rows, and labeled distribution bars
(`«label» ███ 12`). The braille line charts are gone.

1. **Overview** — headline tiles: Agents Run (with ✓ completed / ✕ failed split and a delta vs the previous equal-length
   window), Success Rate %, Commits (total + committing-agent count), Plans Proposed (epic/tale split), Questions
   (sessions + total questions). Below: a runs-over-time strip of labeled bars (hour buckets for ranges ≤ 48h, day
   buckets otherwise, each bucket's count visible), and two mini top-5 tables (providers by runs, skills by uses).
2. **Runs** — outcome breakdown table (`outcome`, count, %, share bar) across all done-marker outcomes, plus
   in-progress/waiting counts; retry stats (chains, attempts, kills); commit attribution: total commits, agents with ≥1
   commit, average commits per committing agent, and a commits-per-agent distribution (0 / 1 / 2 / 3+ labeled bars); top
   target repos by commit count.
3. **Providers** — table grouped provider → model → effort (effort ordered by the core effort vocabulary; missing effort
   shown as `default`): runs, share %, success rate, average runtime.
4. **Runtime** — group-by dimension cycled with `g`: tribe → clan → family → agent → provider → model → workflow. Ranked
   table: group, runs, total wall time, mean, p50, p95, max, share bar. Runtime = `done.finished_at` (fallback
   `agent_meta.stopped_at`) − `agent_meta.run_started_at`; runs still in flight are excluded from duration math and
   shown as an "in progress: N" footnote.
5. **Activity** — three ranked top-N tables with exact counts: Skills (name, uses, distinct agents), Memories (canonical
   path, reads, distinct agents), Workspaces (project + workspace number, runs).
6. **Plans & Questions** — plans proposed (from `plan_submitted_at[]`), epic vs tale split, approved / rejected /
   pending counts, phases-per-epic distribution (labeled bars) and mean phase count; question sessions, distinct asking
   agents, questions-per-ask distribution (labeled bars) and mean questions per session.

### Time ranges

Presets cycled with `t`: **Today, 24h, 7d (default), 30d, 90d, All**. `c` opens an inline custom range input (precedent:
`xprompt_browser_filter_input.py`) accepting `Nd` / `Nw` / `Nh` relative forms, `YYYY-MM`, `YYYY-MM-DD..YYYY-MM-DD`, and
`YYYY-MM-DD..` (open end = now). Invalid input shows a toast and keeps the previous range. The header always displays
the resolved absolute range. Range and view selection are session state (not persisted).

### Keys and refresh

New keymap section `ace.keymaps.statistics` replacing `ace.keymaps.telemetry` in `src/sase/default_config.yml`:
`prev_view "["`, `next_view "]"`, `cycle_range "t"`, `custom_range "c"`, `cycle_group "g"`, `refresh "r"`. Auto-refresh
every 30s while the tab is active (statistics are historical; 10s is unnecessary). All loading follows the established
Telemetry-pane pattern and the TUI perf rules: thread workers (`run_worker(thread=True, exclusive=True)`),
`DetailPanelDebouncer` coalescing, no event-loop blocking, selection re-validated when worker results land, work gated
on tab visibility.

## Architecture

- **Rust owns aggregation.** New `agent_stats` module in `crates/sase_core/src/` reads the artifact index SQLite (path
  passed from Python), filters rows to the requested time window using the indexed `timestamp` / `started_at` /
  `finished_at` columns, deserializes each row's `record_json`, and folds the statistics in Rust. Current scale is a few
  thousand rows, so per-row deserialization is milliseconds; no schema migration of the index is required. (If profiling
  ever demands it, the index has a `schema_version` meta and a rebuild path, but that is explicitly not part of this
  epic.)
- **Composite responses, few round trips.** One run-stats query returns everything the Runs / Providers / Runtime /
  Overview views need for a window; one activity query covers skills/memories/questions/plans. The TUI issues at most
  two binding calls per refresh.
- **Python is presentation only.** `src/sase/stats/` mirrors `src/sase/telemetry/query.py`: binding wrappers, range
  grammar, and view-model builders. The TUI pane renders view models.
- **Rendering toolkit stays put.** `src/sase/telemetry/render/` is a deterministic Rich chart toolkit shared by the
  telemetry CLI and the new pane; Statistics reuses `tiles.py`, `bars.py`, `sparkline.py`, and the `axis.py` value
  formatters. Only the chart-only modules (`line.py`, `braille.py`) are removed once unreferenced (slim-down phase).

## Phase details

### Rust core agent run statistics engine

In the `sase-core` linked repo (open via `/sase_repo`):

- New `crates/sase_core/src/agent_stats/` module (pattern-match `telemetry/store.rs` for store-query structure and
  `agent_archive/mod.rs::agent_archive_facet_counts` for grouped counting). Request wire: index path implicit via
  binding arg; body holds `start_ts`, `end_ts`, `runtime_group_by` (one of
  `tribe|clan|family|agent|provider|model|workflow`), `bucket_seconds`, `top_n`. A run belongs to the window when its
  launch time (`started_at`, falling back to the `timestamp` dir name) falls inside it.
- Response wire (serde, mirrored into `wire.rs`-style types): totals (runs, completed, failed, other terminal outcomes,
  in-progress/waiting), outcome counts, retry/kill counts, provider→model→effort counts with success rate and mean
  duration, commit stats (total commits, committing agents, per-agent distribution 0/1/2/3+, top repos), plan counters
  from run metadata (runs with `plan_submitted_at`, `plan_approved`, `plan_action`), question counters from run metadata
  (`questions_submitted_at` lengths), workspace usage (project + workspace_num run counts), time-bucketed run counts,
  and runtime groups (count, total, mean, p50, p95, max per group value). Durations from `run_started_at` →
  `done.finished_at`/`stopped_at` as defined above; tolerate missing/partial `record_json` fields gracefully (skip,
  never error the batch).
- Effort values validate against `effort.rs`; unknown/absent map to a `default` bucket.
- One new `#[pyfunction]` in `crates/sase_core_py/src/lib.rs` (+ pymodule registration and the module doc-comment
  catalog), following the `py_telemetry_*` dict-in/dict-out pattern with `py.allow_threads`.
- Rust unit tests against a fixture index DB (build rows via the existing index insert paths); cover window filtering,
  outcome/provider/effort folds, duration fallbacks, distribution edges, and malformed `record_json` tolerance. Run this
  repo's standard checks.

### Rust core activity log statistics

Still in `sase-core`, extending `agent_stats`:

- **Skills / memories:** scan `<sase_home>/projects/*/skill_uses.jsonl` and `memory_reads.jsonl`, filter by event
  `timestamp` to the window, aggregate per `skill_name` / `canonical_path`: count + distinct `agent_name`, top-N.
  Tolerate missing files and malformed lines (skip).
- **Questions:** scan `<sase_home>/user_question/*/question_request.json`; per session record `len(questions)` and
  timestamp; aggregate sessions in window, questions-per-ask distribution. Run-level ask counts already come from the
  run-stats phase; this adds the per-session sizes.
- **Plans:** for runs in the window with plan submissions, resolve the plan file (prefer `sdd_plan_path`, fallback
  `plan_path`, fallback the `<sase_home>/plans/YYYYMM/` mirror), parse `tier:` via the existing `plan/read.rs`
  frontmatter handling, and count `phases:` entries for epics (extend plan parsing minimally if phase-list extraction is
  not yet exposed). Emit: tier split, approval split, phases-per-epic distribution. Missing/unresolvable plan files
  count under an `unknown` tier bucket rather than erroring.
- Wire types, one or two new pyfunctions (bindings doc-comment updated), and Rust tests with fixture JSONL/JSON/markdown
  files.

### Python statistics facade and time ranges

In this repo:

- `src/sase/stats/` package: `query.py` (thin wrappers over the new bindings via `require_rust_binding`, passing the
  artifact-index path and `sase_home`), `ranges.py` (preset table + custom grammar parser returning
  `(start_ts, end_ts, label)`; pure and unit-testable), and `views.py` (frozen view-model dataclasses per Statistics
  view, built off-thread from the binding responses — the same split as `telemetry_pane_data.py` today).
- Symvision: run the repo lint flow; consult `sase/memory/symvision.md` before adding pragmas.
- pytest: range grammar (all forms + invalid inputs + timezone handling pinned to the configured timezone), view-model
  builders against canned binding payloads, and a smoke test through the real bindings on a fixture index.

### Statistics tab in the Admin Center

In this repo (read `sase/memory/tui_perf.md` first — required):

- New `src/sase/ace/tui/modals/statistics_pane.py` + `statistics_pane_data.py`; delete `telemetry_pane.py` /
  `telemetry_pane_data.py` and their tests/goldens in this same phase.
- `config_center_modal.py`: `_TAB_ORDER` / `_TAB_LABELS` / `_TAB_COLORS` / `_TAB_DESCRIPTIONS` / `CenterTab` updates;
  lazy-import the pane in `compose` as today.
- `admin_center_tab.py`: map persisted `"telemetry"` → `"statistics"` in `load_admin_center_tab`.
- Keymaps: replace `TelemetryPaneKeymaps` with `StatisticsPaneKeymaps` across
  `src/sase/ace/tui/keymaps/{types,loader,__init__}.py`; update `src/sase/default_config.yml` (`ace.keymaps.statistics`)
  — the default-keymap gotcha applies.
- Command catalog (`src/sase/ace/tui/commands/catalog.py`): replace the `telemetry` command with `statistics` ("Open
  statistics"), aliases `statistics, stats, metrics, telemetry, admin center`; rename `action_open_telemetry_panel` →
  `action_open_statistics_panel` (`src/sase/ace/tui/actions/base.py`); update the three help-modal binding strings.
- Styles: replace the `#telemetry-*` block in `src/sase/ace/tui/styles.tcss` with statistics layout (title, tiles row,
  view strip, body, hints).
- Views/interactions exactly per Product design; custom-range input as an inline transient input widget; all data loads
  through `stats-facade` view models in thread workers with debounced, visibility-gated scheduling and 30s auto-refresh.
- Tests: port/replace `tests/ace/tui/test_telemetry_pane.py` behaviors (activation-gated load, coalescing, binding
  dispatch, hidden-tab inertness, soak) plus view cycling, group-by cycling, custom-range accept/reject; update
  `tests/ace/tui/test_config_center_tabs.py` and `tests/ace/test_admin_center_tab.py` (legacy value migration). PNG
  snapshots (`tests/ace/tui/visual/`): new deterministic fixture data and goldens for the Statistics tab (populated
  Overview, one non-default view, empty, loading); delete the three telemetry goldens and their test/helpers. Regenerate
  via `--sase-update-visual-snapshots` and `just test-visual`.

### Telemetry infrastructure slim-down

In this repo. Keep everything with debugging value; remove what Statistics supersedes:

- **CLI:** remove `sase telemetry dashboard` and `sase telemetry graph` (`src/sase/main/parser_telemetry.py`,
  `telemetry_handler.py`, `cli_dashboard.py`, `cli_graph.py`); keep `health`, `list`, `snapshot`, `status`. Make bare
  `sase telemetry` actually delegate to `list` through the central `_default_list_subcommands()` convention (today's
  help text claims it but the handler exits 1). Follow the CLI rules memory (help quality, alphabetical ordering, short
  aliases).
- **Render:** delete `render/line.py` and `render/braille.py` once unreferenced; keep `axis.py`, `palette.py`,
  `sparkline.py`, `bars.py`, `tiles.py` (used by Statistics and the remaining CLI).
- **Metrics catalog** (`src/sase/telemetry/metrics.py`, `catalog.py`) — remove these product-usage metrics and their
  emission sites; Statistics now answers them from durable records:
  - `sase_vcs_commits_total` (`vcs_provider/plugins/_git_core_ops.py`, `_git_commit_dispatch.py`)
  - `sase_bead_operations_total`, `sase_bead_status_transitions_total`, `sase_bead_active` (`src/sase/bead/project.py`,
    `src/sase/main/bead_fast_path.py`)
  - `sase_workspace_acquisitions_total`, `sase_workspace_releases_total` (`src/sase/running_field/_operations.py`) —
    keep the `sase_workspace_active` gauge (leak diagnostics)
  - `sase_notifications_sent_total` (`src/sase/notifications/senders.py`)
  - `sase_memory_proposals_{proposed,approved,rejected}_total` (`src/sase/memory/proposals/`)

  **Keep** all agent, LLM, axe, hook/mentor/workflow, and zombie metrics plus `sase_vcs_operations_total` (commit
  failures/conflicts are debugging signal) — these feed `sase telemetry health` and the doctor checks, which stay
  untouched (`src/sase/doctor/checks_telemetry.py` needs no change).

- **Out of scope / untouched:** the flusher/store/ingestion pipeline, retention config and health thresholds in
  `default_config.yml`, and the JSONL forensic logs (`src/sase/logs/tui_telemetry.py`) — those are exactly the debugging
  surface.
- Update `tests/telemetry/` (drop `test_cli_dashboard.py`, `test_cli_graph.py`, `render/test_line.py`; prune
  removed-metric instrumentation tests), parser/help tests, and any docs referencing the removed commands. Clean up
  symvision whitelists per `sase/memory/symvision.md`.

## Testing strategy

Every phase lands with its own tests (Rust: cargo test in `sase-core`; Python: pytest here) and must pass its repo's
full check command (`just check` in this repo) before completion. Visual changes go through the PNG snapshot suite with
pinned fonts/colors. The statistics-pane phase must keep j/k latency budgets intact (no event-loop work; verify with the
pane active).

## Risks and mitigations

- **Data quality drift** (missing `finished_at`, absent efforts, purged question sessions): every aggregator treats
  absent fields as skip-or-bucket-`unknown`, never as an error; tests pin these edges.
- **Index is a cache:** statistics read whatever the index currently holds; the index lifecycle (rebuild on stale
  schema) already runs in ACE. No new rebuild logic in this epic.
- **Cross-repo sequencing:** the two core phases must merge and publish `sase_core_rs` before the facade phase can
  exercise real bindings; the facade phase should degrade its smoke test to canned payloads if bindings are unavailable
  in CI.
- **Tab renumbering:** alphabetical `_TAB_ORDER` shifts digit jumps for Tasks/Updates/XPrompts; covered by updated tab
  tests — no persisted state breaks beyond the migrated tab name.
- **Memory files:** no `sase/memory/*.md`, `AGENTS.md`, or provider shim edits are needed or permitted in any phase.

## Non-goals

- No `sase stats` CLI command (possible follow-up).
- No per-project filtering inside the Statistics tab (per-project numbers appear only in the Activity workspace table);
  a project filter is a natural follow-up.
- No changes to telemetry storage/rollups/retention, doctor checks, or forensic JSONL logs.
- No artifact-index schema migration.
