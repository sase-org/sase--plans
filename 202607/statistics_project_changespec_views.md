---
tier: epic
title: Track and display agent work by project and ChangeSpec in Statistics
goal: 'The Admin Center Statistics tab shows which projects and ChangeSpecs SASE agents
  ran on, through a dedicated Projects view with three grouping strategies (by project,
  by ChangeSpec, and a project-to-ChangeSpec drill-down), new project/changespec runtime
  group-by dimensions, a Top Projects overview table, and a global project filter;
  ChangeSpec attribution becomes durable and reliable by propagating commit-time spec
  names into the indexed run records and normalizing the historical launch-name fallback.

  '
phases:
- id: commit-attribution
  title: Commit-time ChangeSpec attribution
  depends_on: []
  description: '''Commit-time ChangeSpec attribution'' section: propagate per-commit
    ChangeSpec names from commit_results.json into done.step_output.meta_commits[]
    and agent_meta.json''s commit_changespec_name so which-spec-an-agent-committed-to
    reaches the durable artifact index, with regression tests.'
- id: core-work-stats
  title: Rust core work statistics engine
  depends_on: []
  description: '''Rust core work statistics engine'' section: extend sase_core''s
    agent_stats with per-project and per-ChangeSpec rollups (attribution normalization,
    spec-status join from .sase files, project filter, project/changespec runtime
    group-by variants), wire types, PyO3 bindings, and Rust tests.'
- id: stats-facade
  title: Python stats facade work views and filters
  depends_on:
  - core-work-stats
  description: '''Python stats facade work views and filters'' section: extend src/sase/stats/
    with the work-stats view models, project/changespec group-by literals, project-filter
    plumbing, Top Projects overview data, and the sase-core-rs dependency floor bump,
    with pytest coverage.'
- id: projects-view
  title: Projects view in the Statistics tab
  depends_on:
  - stats-facade
  description: '''Projects view in the Statistics tab'' section: add the Projects
    sub-view with three g-cycled grouping strategies, project color chips and ChangeSpec
    status glyphs, the Top Projects overview table, runtime group-by extension, the
    p project-filter key, keymap config, styles, behavior tests, and PNG visual snapshots.'
create_time: 2026-07-18 22:19:28
status: wip
bead_id: sase-70
---

# Plan: Track and display agent work by project and ChangeSpec in Statistics

## Context

The Admin Center Statistics tab (built by the sase-6y epic) computes all of its numbers on demand from the durable agent
artifact index (`~/.sase/agent_artifact_index.sqlite`, table `agent_artifacts`), via the Rust `agent_stats` module in
the `sase-core` linked repo and the thin Python facade in `src/sase/stats/`. The sase-6y plan explicitly deferred
per-project statistics as "a natural follow-up"; this epic is that follow-up, extended to ChangeSpecs.

What the durable data already records, verified against the live index (3,291 rows):

- **Project attribution is complete.** `project_name` is an indexed column populated on every row.
- **ChangeSpec attribution exists but is conflated.** `cl_name` is an indexed column (index
  `idx_agent_artifacts_cl_name`) whose value is the launch-time ChangeSpec name — but when an agent is launched without
  a spec, the launch path fills it with a project-identity placeholder instead of leaving it empty. In the live index
  ~97% of rows carry a fallback (`gh_sase-org__sase`, `sase-org/sase`, or `unknown`); only runs launched against a real
  spec carry a real name (which then matches a `NAME:` entry in the project's `.sase` / `-archive.sase` file).
  `agent_meta.changespec_name` always equals `cl_name`. The Rust indexer already has an "unknownish" notion:
  `cl_name_is_unknownish` in `crates/sase_core/src/agent_scan/index.rs` (used to backfill `done.cl_name` from
  `agent_meta`).
- **Commit-time spec attribution is captured but dropped before it reaches the index.** When `sase commit` routes a
  commit into a ChangeSpec, `write_result_marker` (`src/sase/workflows/commit/commit_tracking.py:486`) records
  `changespec_name` / `commit_changespec_name` in the per-run `commit_results.json` marker. At run finalize,
  `extract_step_output_and_diff_path` (`src/sase/axe/run_agent_helpers_state.py:105`) projects those markers into
  `done.step_output.meta_commits[]` — but the projection `_commit_result_list_record`
  (`src/sase/axe/run_agent_helpers_state.py:63`) keeps only `message`, `sha`, `cwd`, `repo_name`, `diff_path` and drops
  `changespec_name`. Consequently `commit_changespec_name` is empty in **all** 3,291 indexed records, and an agent that
  creates its ChangeSpec mid-run (the common PR flow) is never attributed to it. The `AgentMetaWire` schema already
  declares `commit_changespec_name` (`src/sase/core/agent_scan_wire_markers.py:118`; Rust
  `crates/sase_core/src/agent_scan/wire.rs`), so no wire-schema change is needed — only writers.
- **Spec lifecycle is durable.** Active specs live in `~/.sase/projects/<key>/<key>.sase` and terminal ones (Submitted /
  Archived / Reverted) in the sibling `<key>-archive.sase`; the Rust core already parses both shapes
  (`parse_project_file`, `crates/sase_core/src/parser.rs:36`, producing `ChangeSpecWire` with `name`, `status`,
  `pr_url`, …).

What is missing is purely the aggregation and display layer: no stats fold groups by project or ChangeSpec, the
`AgentStatsRuntimeGroupByWire` enum has no Project/Changespec variants, and the run-stats SQL never selects `cl_name`.
Project appears only bundled with `workspace_num` in the Activity view's workspace rows.

Per the Rust Core Backend Boundary rule, all new aggregation logic belongs in `sase_core`; Python stays a thin
presentation adapter. The Rust repo is the `sase-core` linked repo: implementing agents MUST open it with the
`/sase_repo` skill and work only in the checkout path it prints. Bindings live in `crates/sase_core_py/src/lib.rs`
(module `sase_core_rs`); Python loads them via `require_rust_binding` (`src/sase/core/rust.py`).

## Attribution model

One rule, implemented once in Rust, used by every view. For each run in the window:

1. **Commit attribution (strongest).** Every distinct real ChangeSpec name appearing in
   `done.step_output.meta_commits[].changespec_name` (written from `commit-attribution` onward), falling back to
   `agent_meta.commit_changespec_name`. A run that committed into a spec ran on that spec, regardless of what it was
   launched with.
2. **Launch attribution.** Otherwise, `cl_name` (column) when it is a _real_ spec name — i.e. not unknownish (reuse
   `cl_name_is_unknownish`) and not a project-identity placeholder. Placeholder detection must cover the directory key
   (`gh_sase-org__sase`), the owner/repo display form (`sase-org/sase`), the bare project name, and the project file
   basename; both live fallback shapes were observed in the index.
3. **Unattributed.** Otherwise the run belongs to the per-project "(no ChangeSpec)" bucket. This bucket is honest and
   visible — most historical runs land here by design, and hiding it would misrepresent the data.

Counting semantics: project-level metrics count each run exactly once (by `project_name`). ChangeSpec-level metrics
count a run once per attributed spec (a run committing into two specs appears in both rows), so per-spec run counts may
legitimately sum to more than the project total; the drill-down view makes this visible rather than hiding it.

**Spec-status join.** For every attributed spec name, the engine parses the project's active and archive `.sase` files
(existing parser) and attaches the spec's `STATUS` (WIP / Draft / Ready / Mailed / Submitted / Archived / Reverted) plus
whether a PR URL is present. Names found in neither file get status `unknown` (spec deleted or renamed). Parse failures
and missing files are tolerated per file (skip, count in a `malformed` counter), never fatal.

## Product design

### The Projects view

A seventh Statistics sub-view, **Projects**, inserted after Runs (`VIEW_ORDER`: overview, runs, projects, providers,
runtime, activity, plans_questions). Cycled into with the existing `[` / `]` keys. The `g` key — which already cycles
the Runtime view's group-by — cycles the Projects view's **grouping strategy** (same key, view-local semantics, matching
the established `cycle_group` action):

1. **By Project.** A summary line (`N projects · M ChangeSpecs · K unattributed runs`) above a ranked table: color
   chip + project, runs (with ✓ completed / ✕ failed split), success %, commits, ChangeSpecs touched, total wall time,
   share bar (share of runs), last run. Sorted by runs descending.
2. **By ChangeSpec.** Ranked table: status glyph + spec name, project chip, runs, distinct agents, commits, total wall
   time, last run. A single muted "(no ChangeSpec)" aggregate row is pinned at the bottom with the unattributed run
   count. If the table is capped, a footer states exactly how many rows were dropped (no silent truncation).
3. **Project ▸ ChangeSpec drill-down.** Nested table: one header row per project (chip + project rollup numbers), its
   attributed specs indented beneath (glyph, runs, agents, commits, wall time), and a per-project muted "(no
   ChangeSpec)" child row. Projects ordered by runs descending, specs within likewise.

The current grouping strategy is named in the view header area and the hints line, exactly as the Runtime view names its
group-by dimension.

### Beauty rules

- **Deterministic project colors.** Each project gets a stable color chip (`■`) from the existing `categorical_color()`
  hashing in `src/sase/telemetry/render/palette.py` — the same project is always the same color across tables, the
  Overview top-projects table, and drill-down headers.
- **ChangeSpec status glyphs reuse the ChangeSpecs-tab conventions.** Do not invent a new status vocabulary: reuse the
  status → glyph/color mapping the ChangeSpecs tab already renders so a Submitted spec looks the same everywhere in ACE.
  `unknown` status renders as a muted `?`.
- Share bars via `src/sase/telemetry/render/bars.py`; durations and counts via the `axis.py` formatters; tables follow
  the existing pane's Rich table idiom (accent `#FF87D7`, `_CYAN`/`_GOLD`/`_GREEN`/`_RED` locals in
  `statistics_pane.py`).
- Empty state: the existing "no agent runs in range" treatment; a range with runs but zero attributed specs still
  renders By Project normally and shows an explanatory muted line in By ChangeSpec.

### Overview and Runtime integration

- **Overview** gains a third mini table, **Top projects** (chip + project, runs, success %), alongside the existing Top
  providers and Top skills tables.
- **Runtime** group-by gains `project` and `changespec` dimensions (`RUNTIME_GROUP_ORDER` grows 7 → 9; the changespec
  group value is the attributed spec name, else "(no changespec)"), giving runtime p50/p95/max per project and per spec
  for free.

### Global project filter

`p` cycles the project filter: All → each project with runs in the current range (ordered by run count descending) →
back to All. The header line appends the active project (e.g.
`Statistics · Projects · 7d · sase-org/sase · updated HH:MM:SS`).

- The filter applies to everything computed from run records (Overview tiles and buckets, Runs, Providers, Runtime,
  Projects, workspace rows) via a `project` field on the run-stats request, and to skills/memories aggregates (their
  JSONL logs are per-project directories) via the same field on the activity request.
- Question sessions and plan-file extras (phases-per-epic, tier splits from mirrored plan files) are not project-scoped
  on disk; while a filter is active those tables show a muted "(all projects)" tag instead of silently pretending to be
  filtered. Run-derived plan/question counters do filter.
- Filter, view, grouping, and range remain session-only state (matching the sase-6y decision); switching the filter
  reuses the existing debounced reload path and stale-result discard (worker results whose range/group/filter no longer
  match are dropped and rescheduled).

## Architecture

- **Rust owns attribution and aggregation.** The attribution model, placeholder normalization, spec-status join,
  project/changespec folds, and the project filter all live in `crates/sase_core/src/agent_stats/` so any future
  frontend gets identical numbers. Python receives presentation-ready aggregates.
- **Single pass, few round trips.** The work rollups ride the existing one-pass fold in `agent_stats/run.rs` and the
  existing two binding calls per refresh (`agent_stats_query_runs`, `agent_stats_query_activity`); no new binding round
  trip. `cl_name` joins the existing SELECT; everything else comes from the already-deserialized `record_json`. No
  artifact-index schema migration (current scale: a few thousand rows per query).
- **Python is presentation only.** `src/sase/stats/` gains view models and literals; the pane renders them. All TUI
  loading keeps the established pattern: thread workers (`run_worker(thread=True, exclusive=True)`), debounced
  coalescing, visibility gating, 30s auto-refresh, no event-loop blocking (per `sase/memory/tui_perf.md`).

## Phase details

### Commit-time ChangeSpec attribution

In this repo. Two small, surgical writers so commit-level spec attribution reaches the durable index:

- `_commit_result_list_record` (`src/sase/axe/run_agent_helpers_state.py:63`): include `changespec_name` (from the
  marker's `changespec_name` / `commit_changespec_name`) in the projected record when present and non-empty, so it flows
  into `done.step_output.meta_commits[]` and therefore `record_json`.
- At commit time, when a primary-workspace commit lands in a real ChangeSpec, persist
  `agent_meta.json["commit_changespec_name"]` following the `_persist_commit_diff_path` pattern
  (`src/sase/workflows/commit/commit_tracking.py:284`): guard against linked/external-repo commits overwriting the
  primary contract, skip no-op rewrites, call `update_agent_artifact_index_for_marker_mutation` after writing. Do not
  modify the launch-time `cl_name` fallback behavior — other subsystems (chat filenames, TUI labels, done backfill) rely
  on it; normalization of placeholders is the Rust engine's job.
- Tests: extend the existing commit-tracking and marker-projection tests — a commit with a spec name reaches
  `meta_commits[].changespec_name` and `agent_meta.commit_changespec_name`; a linked-repo commit does not overwrite the
  primary value; records without spec names project unchanged. Run `just check`.

### Rust core work statistics engine

In the `sase-core` linked repo (open via `/sase_repo`; run that repo's standard checks):

- **Attribution helpers** in `crates/sase_core/src/agent_stats/`: a placeholder detector (reusing
  `cl_name_is_unknownish` plus project-identity comparison against `project_name`, the owner/repo display form, and the
  project-file basename) and a per-run attribution resolver implementing the Attribution model section (commit names
  from `meta_commits[].changespec_name` / `agent_meta.commit_changespec_name`, else real launch `cl_name`, else
  unattributed).
- **Request wire** (`agent_stats/wire.rs`): `AgentRunStatsRequestWire` gains `project: Option<String>` (exact
  `project_name` match applied in the SQL WHERE) and `work_top_n: Option<u32>` (cap for per-spec rows, default generous,
  e.g. 50). `AgentActivityStatsRequestWire` gains the same `project` field, scoping the skills/memories directory scans;
  question/plan-document aggregates ignore it by design (documented on the field).
- **Response wire**: a `work` section — per-project rollups (project, runs, completed, failed, other terminal,
  in-progress/waiting, success rate, commits, distinct ChangeSpecs touched, unattributed run count, total runtime
  seconds, last run timestamp), per-ChangeSpec rollups (project, name, status, has PR, runs, distinct agents, commits,
  total runtime seconds, first/last run timestamps), a total unattributed count, and a truncated-changespec-rows count
  when `work_top_n` capped the list. Add `Project` and `Changespec` variants to `AgentStatsRuntimeGroupByWire` and map
  them in `runtime_group_value` (`agent_stats/run.rs`).
- **Folds**: add `cl_name` to the run-stats SELECT (`run.rs:85`) and implement the work folds alongside
  `fold_workspace`/`finish_workspaces`; durations, window filtering, and missing-field tolerance follow the existing
  conventions (skip, never error the batch; count malformed rows).
- **Spec-status join**: after the fold, parse each involved project's `project_file` and its sibling `-archive.sase`
  with the existing `parse_project_file`, mapping spec name → status / PR presence; tolerate missing/corrupt files per
  project.
- **Bindings** (`crates/sase_core_py/src/lib.rs`): the new request fields and response section flow through the existing
  dict-in/dict-out `agent_stats_query_runs` / `agent_stats_query_activity` functions under `py.allow_threads`; update
  the module doc-comment catalog. Follow the crate's response schema-version convention for additive fields.
- **Tests** (inline `mod tests`, fixture index built via the real rebuild path, per the existing agent_stats tests):
  attribution precedence (commit beats launch), placeholder normalization for both observed fallback shapes and
  `unknown`, multi-spec runs counted once per spec but once per project, spec-status join including archive statuses and
  `unknown` names, project filter on both queries, `work_top_n` truncation accounting, runtime group-by variants,
  malformed `record_json` tolerance. Round-trip pyo3 tests extended.

### Python stats facade work views and filters

In this repo:

- `src/sase/stats/query.py`: extend `RuntimeGroupBy` with `"project"` / `"changespec"`; pass `project` and `work_top_n`
  through both query wrappers.
- `src/sase/stats/views.py`: frozen view-model rows for project and ChangeSpec work stats; a `_ProjectsView` carrying
  both row sets plus the summary counts (projects, specs, unattributed, truncated); `_OverviewView` gains
  `top_projects`; `build_statistics_views` assembles them from the binding payloads (drill-down nesting is derived in
  the view builder from the two row sets — no extra binding data needed).
- Dependency floor: bump the `sase-core-rs` requirement in `pyproject.toml` to the first version publishing the new wire
  fields (established pattern: floor bump + the core validation probe so stale wheels rebuild during setup).
- pytest: view-model builders against canned payloads covering populated / empty / truncated / unknown-status cases;
  group-by literal round-trips; the binding smoke test (`tests/stats/test_binding_smoke.py`) extended to assert the work
  section exists, degrading gracefully (skip) if the installed binding predates it.
- Symvision: run the repo lint flow; consult `sase/memory/symvision.md` before adding pragmas. Run `just check`.

### Projects view in the Statistics tab

In this repo (implementing agent must read `sase/memory/tui_perf.md` first — required):

- `src/sase/ace/tui/modals/statistics_pane_data.py`: add `projects` to `VIEW_ORDER` / `VIEW_LABELS` (after `runs`); add
  `PROJECTS_GROUP_ORDER = ("project", "changespec", "drilldown")`; extend `RUNTIME_GROUP_ORDER` with `project` /
  `changespec`; thread the project filter and work view models through `load_statistics_view` and `StatisticsViewData`.
- `src/sase/ace/tui/modals/statistics_pane.py`: render the three grouping strategies per Product design (project chips
  via `categorical_color`, spec glyphs reusing the ChangeSpecs-tab status conventions, share bars, pinned "(no
  ChangeSpec)" rows, truncation footers); make `action_cycle_group` view-sensitive (Runtime → runtime dimension,
  Projects → grouping strategy; a toast/no-op on other views, matching current behavior); add
  `action_cycle_project_filter` on `p` (cycle order: All, then projects by run count in the current result); header and
  hints updates; "(all projects)" tags on unscoped tables when a filter is active; stale-result discard extended to
  compare the filter.
- Keymaps: add `cycle_project_filter "p"` to the statistics keymap section across
  `src/sase/ace/tui/keymaps/{types,loader,__init__}.py` and `src/sase/default_config.yml` (default-keymap gotcha
  applies; `g` reuses the existing `cycle_group` binding).
- Styles: extend the `#statistics-*` block in `src/sase/ace/tui/styles.tcss` only as needed (drill-down indentation,
  chip/glyph spacing).
- Overview: render the Top projects mini table beside Top providers / Top skills.
- Tests: extend `tests/ace/tui/test_statistics_pane.py` — grouping cycle on the Projects view, view-sensitive `g`,
  project-filter cycle (including filter + range interplay and stale-result discard), unscoped-table tagging, hidden-tab
  inertness unchanged; update the deterministic fixture payloads in
  `tests/ace/tui/visual/_ace_config_center_statistics_helpers.py` with work-stats data.
- PNG snapshots (`tests/ace/tui/visual/`): new goldens for the Projects view (By Project and drill-down groupings); the
  existing four statistics goldens must be regenerated because the sub-tab strip gains a label. Regenerate via
  `--sase-update-visual-snapshots` and verify with `just test-visual`.
- Run `just check`; j/k latency budgets stay intact (no event-loop work; all loads through the existing worker path).

## Testing strategy

Every phase lands with its own tests and must pass its repo's full check command (`just check` here; the standard cargo
checks in `sase-core`) before completion. The attribution model is pinned by Rust unit tests (precedence, normalization,
counting semantics) so the TUI never re-derives it. Visual changes go through the PNG snapshot suite with pinned
fonts/colors. Pane tests keep the activation-gating, coalescing, and soak coverage intact.

## Risks and mitigations

- **Placeholder detection misses a fallback shape** → a project key masquerades as a ChangeSpec row. Mitigation:
  detection compares against every project-identity form present on the record (key, display, basename) plus unknownish;
  Rust tests enumerate the observed live shapes; the spec-status join independently marks unmatched names `unknown`,
  which visibly flags any leak.
- **Cross-repo sequencing**: the core phase must merge and publish `sase_core_rs` before the facade phase can exercise
  real bindings; the facade smoke test degrades to canned payloads when the installed binding predates the work section
  (same mitigation as sase-6y).
- **Per-spec double counting misread as a bug**: counting semantics are documented in the Attribution model, pinned by
  tests, and the drill-down view shows per-spec rows under their project rollup so the relationship is visible.
- **PNG golden churn**: adding a sub-tab label changes all statistics goldens; the pane phase regenerates them
  deliberately in one commit with `--sase-update-visual-snapshots`.
- **Data quality drift** (missing spec files, deleted specs, malformed markers): every aggregator skips or buckets as
  `unknown`, never errors; counters expose skipped rows.
- **Memory files**: no `sase/memory/*.md`, `AGENTS.md`, or provider shim edits are needed or permitted in any phase.

## Non-goals

- No `sase stats` CLI surface for work stats (possible follow-up).
- No artifact-index schema migration and no new denormalized columns (`cl_name` already exists; the rest rides
  `record_json`).
- No backfill or rewriting of historical marker files; historical runs are normalized at query time only.
- No persisted filter/grouping/range state (session-only, per the sase-6y decision).
- No per-bead or per-xprompt statistics dimensions (future work).
- No changes to the ChangeSpecs tab or spec files themselves; statistics only read them.
