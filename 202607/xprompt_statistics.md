---
tier: epic
title: XPrompts sub-tab for the Admin Center Statistics panel
goal: 'The SASE Admin Center Statistics tab has an XPrompts sub-tab that reports which
  xprompts agent prompts actually used and how often, lets `g` regroup those counts
  by model, project, and co-usage, and lets the user zoom into one xprompt for a full
  breakdown of its models, projects, partner xprompts, and usage over time.

  '
phases:
- id: core-scan
  title: Project launch-boundary xprompt usage into the artifact scan record and index
  depends_on: []
  size: medium
  description: 'core-scan: in sase-core, add a UsedXPromptWire projection of each
    artifact directory''s xprompts.json to the agent-artifact scan record, sign that
    file so stale rows re-index, and bump the artifact index schema so existing indexes
    rebuild with the new projection.'
- id: core-stats
  title: XPrompt aggregation section in the run-statistics wire and query
  depends_on:
  - core-scan
  size: medium
  description: 'core-stats: in sase-core, fold the projected xprompt usage of every
    in-window run into a new optional xprompts section of the run-statistics response,
    with ranked rows, bounded model/project/partner cross-tabs, and an optional focused
    single-xprompt payload driven by new request knobs.'
- id: py-stats
  title: Python statistics models and builder for the XPrompts view
  depends_on: []
  size: medium
  description: 'py-stats: add the XPrompts view models, payload builder, and query
    request knobs to sase.stats so the view is built from the agreed payload contract,
    degrades to an explicit unavailable state when the section is missing, and is
    covered by fixture-driven tests.'
- id: tui-view
  title: XPrompts sub-tab with four grouping strategies
  depends_on:
  - py-stats
  size: medium
  description: 'tui-view: register the XPrompts view in the Statistics pane, render
    its four `g` grouping strategies with legends and honest empty states, and make
    the tab strip fit nine tabs at narrow widths.'
- id: tui-focus
  title: Zoom into one xprompt with a focus picker, scope chip, and keys
  depends_on:
  - tui-view
  size: medium
  description: 'tui-focus: add the xprompt focus picker modal, the XPrompt scope chip,
    the focus/clear keymaps and their configuration surfaces, the focused body rendering,
    and the contextual help updates.'
- id: land
  title: Land the cross-repo contract, snapshots, and documentation
  depends_on:
  - core-stats
  - tui-focus
  size: medium
  description: 'land: raise the sase-core-rs floor together with the Python artifact-index
    schema constant, verify the feature end to end against real recorded data, add
    the new PNG visual snapshots, and document the sub-tab and its keys.'
create_time: 2026-07-29 12:25:59
status: done
bead_id: sase-au
---

- **PROMPT:** [202607/prompts/xprompt_statistics.md](prompts/xprompt_statistics.md)
- **BEAD:** [sase-au](https://github.com/sase-org/sase--beads/blob/main/pages/sase-au/README.md)

# Plan: XPrompts sub-tab for the Admin Center Statistics panel

## Why

`sase ace` → `#` → **Statistics** currently answers "what did agents do" (runs, runners, projects, providers, runtime,
activity, plans & questions) but never answers "what did I _ask_ for". Every launch already records the xprompts its
prompt referenced, and that data is durable and complete — it is simply not aggregated anywhere. This epic surfaces it
as a ninth Statistics view.

## Ground truth: where xprompt usage already lives

At the launch boundary, `sase.axe.run_agent_runner_setup` calls
`sase.xprompt.used_xprompts.write_used_xprompts(artifacts_dir, prompt)` **before** xprompt expansion erases the
references, writing `<artifact_dir>/xprompts.json`:

```json
[
  { "name": "gh", "kind": "workflow", "positional": ["sase-org/sase"], "named": {}, "tags": ["rollover", "vcs"] },
  { "name": "split_file", "kind": "part", "positional": ["src/..."], "named": {}, "tags": [] }
]
```

Key properties established while researching this plan:

- One artifact directory is one indexed run (`agent_artifacts.artifact_dir` is the primary key). Workflow steps are
  `prompt_step_*.json` markers _inside_ that same directory, not separate rows.
- `collect_used_xprompts` deduplicates by `(name, kind, positional, named)`, so one name can appear more than once in
  the list when the same xprompt was referenced with different arguments.
- Workflow prompt-step execution writes `xprompts_<step>.json` with `step_only=True`, which leaves an existing shared
  `xprompts.json` untouched but **seeds** it when the launch boundary captured nothing. A row can therefore gain
  `xprompts.json` after it was first indexed — this is why the file needs its own index signature (see `core-scan`).
- The Rust run-statistics query already decodes every in-window row's `record_json` into `AgentArtifactRecordWire`.
  Projecting xprompt usage into that record makes the aggregation free of extra I/O.

**Counting scope decision:** the XPrompts view counts the **launch-boundary** `xprompts.json` only. That is literally
"which xprompts have been used in prompts". Workflow-internal step references (`xprompts_<step>.json`) are template
internals, not something the user typed, and are deliberately excluded. The view legend and the help modal must state
this so the numbers are never mistaken for total expansion counts.

## Architecture decisions

1. **Aggregate in Rust, not Python.** Shared backend behavior belongs in `sase-core` (`rust_core_backend_boundary`
   memory). A Python implementation would have to read thousands of `xprompts.json` files on every 30-second Statistics
   refresh.
2. **Extend the existing run-statistics response; do not add a new binding.** `query_run_stats` already scans and
   decodes exactly the rows we need, under exactly the window and project-filter semantics we want. A new `xprompts`
   section costs one more in-memory fold per row and zero extra I/O.
3. **Make the contract forward- and backward-compatible.** `AgentRunStatsRequestWire` is deserialized with
   `serde_json::from_value` and is not `deny_unknown_fields`, so an older installed `sase-core-rs` silently ignores the
   new request knobs. The response section is `Option<...>` with `#[serde(default)]`, mirroring how `runners` was added,
   so a Python client reading an older response sees `None` and renders an explicit "unavailable" state instead of a
   wrong number. This is what lets `py-stats`, `tui-view`, and `tui-focus` proceed in parallel with the two `sase-core`
   phases.
4. **Project usage into the scan record.** `AgentArtifactRecordWire` already carries `prompt_steps` and
   `raw_prompt_snippet`; `used_xprompts` joins them. The index caches it in `record_json`, so query time stays pure CPU.

## The wire contract (authoritative for every phase)

Both `sase-core` phases and every Python phase implement exactly this shape. Field names, types, and ordering are
normative.

### Scan record addition

```rust
/// Compact projection of one launch-boundary `xprompts.json` entry,
/// deduplicated by name.
pub struct UsedXPromptWire {
    pub name: String,          // e.g. "split_file" (no leading '#')
    pub kind: String,          // "workflow" | "part" | "unknown"
    pub tags: Vec<String>,     // sorted, may be empty
    pub references: u64,       // >= 1; entries sharing a name are collapsed
}
```

`AgentArtifactRecordWire` gains `#[serde(default)] pub used_xprompts: Vec<UsedXPromptWire>` as the last field before
`has_done_marker`, mirrored by the Python dataclass in `src/sase/core/agent_scan_wire_records.py` and its converter in
`src/sase/core/agent_scan_wire_conversion.py`.

`AGENT_SCAN_WIRE_SCHEMA_VERSION` stays **4**. It is asserted for equality across the boundary
(`agent_scan_wire_conversion.py`), so bumping it would break every mixed Rust/Python pair; the field is purely additive
and optional, which is the established pattern for this record.

### Request additions to `AgentRunStatsRequestWire`

```rust
#[serde(default = "default_xprompt_top_n")]        pub xprompt_top_n: u32,            // 40
#[serde(default = "default_xprompt_breakdown_n")]  pub xprompt_breakdown_top_n: u32,  // 5
#[serde(default)]                                   pub xprompt_focus: Option<String>, // exact xprompt name
```

### Response addition to `AgentRunStatsResponseWire`

```rust
#[serde(default)] pub xprompts: Option<AgentXPromptStatsWire>,
```

```rust
pub struct AgentXPromptStatsWire {
    pub runs_with_xprompts: u64,      // in-window runs with >= 1 launch xprompt
    pub runs_without_xprompts: u64,   // in-window runs with none
    pub distinct_xprompts: u64,       // before top-N truncation
    pub total_references: u64,        // sum of `references` across all runs
    pub rows: Vec<AgentXPromptStatsRowWire>,   // ranked, bounded by xprompt_top_n
    pub truncated_rows: u64,          // distinct_xprompts - rows.len()
    pub focus: Option<AgentXPromptFocusWire>,  // present iff xprompt_focus was set
}

pub struct AgentXPromptStatsRowWire {
    pub name: String,
    pub kind: String,
    pub tags: Vec<String>,
    pub runs: u64,
    pub references: u64,
    pub distinct_agents: u64,
    pub completed: u64,
    pub failed: u64,
    pub success_rate: f64,            // completed / runs, 0.0 when runs == 0
    pub total_runtime_seconds: f64,
    pub mean_runtime_seconds: Option<f64>,  // None when no run had a valid duration
    pub first_run_ts: f64,
    pub last_run_ts: f64,
    pub models: Vec<AgentStatsCountWire>,    // top xprompt_breakdown_top_n by runs
    pub projects: Vec<AgentStatsCountWire>,  // artifact-index project keys
    pub partners: Vec<AgentStatsCountWire>,  // co-referenced xprompt names
}

pub struct AgentXPromptFocusWire {
    pub name: String,
    pub found: bool,                  // false when the name had no in-window run
    pub kind: String,
    pub tags: Vec<String>,
    pub runs: u64,
    pub references: u64,
    pub distinct_agents: u64,
    pub completed: u64,
    pub failed: u64,
    pub success_rate: f64,
    pub total_runtime_seconds: f64,
    pub mean_runtime_seconds: Option<f64>,
    pub first_run_ts: f64,
    pub last_run_ts: f64,
    pub models: Vec<AgentStatsCountWire>,     // full ranked list, no breakdown cap
    pub providers: Vec<AgentStatsCountWire>,
    pub projects: Vec<AgentStatsCountWire>,
    pub partners: Vec<AgentStatsCountWire>,
    pub tribes: Vec<AgentStatsCountWire>,
    pub buckets: Vec<AgentRunBucketWire>,     // same bucket_seconds/edges as the response
}
```

`AgentStatsCountWire` is the existing `{name, count}` record. Shares are derived in Python (`count / row.runs`) rather
than duplicated on the wire.

**Ranking and determinism.** Every ranked list sorts by `count`/`runs` descending, then by name ascending, so output is
stable across runs and platforms. Missing `model`, `llm_provider`, project, or tribe values collapse to the existing
`"unknown"` sentinel used elsewhere in `agent_stats`.

---

## `core-scan`: project launch-boundary xprompt usage into the scan record and index

Repository: **`sase-core`**. Open it with the `/sase_repo` skill (`sase repo open sase-core -r "..."`) and use the
printed path as the only path for reads and writes. Commit there separately from the `sase` repo.

### Scanner

- `crates/sase_core/src/agent_scan/wire.rs`: add `UsedXPromptWire` (shape above) and the `used_xprompts` field on
  `AgentArtifactRecordWire`. Do **not** change `AGENT_SCAN_WIRE_SCHEMA_VERSION`.
- `crates/sase_core/src/agent_scan/scanner.rs`: read `<artifact_dir>/xprompts.json` next to the existing marker reads
  and project it. Rules:
  - A missing or unreadable file yields an empty vector — never a scan failure.
  - A malformed or non-array payload yields an empty vector and increments the existing soft-error scan stat used by the
    other marker readers, so diagnostics stay honest.
  - Entries missing a usable `name` (absent, non-string, or empty after trimming) are skipped.
  - `kind` accepts only `"workflow"` and `"part"`; anything else becomes `"unknown"`.
  - `tags` keeps only non-empty strings, deduplicated and sorted.
  - Entries are collapsed by `name`: `references` counts the collapsed entries; `kind` and `tags` come from the first
    entry with that name.
  - Output is sorted by `name` so `record_json` is byte-stable for unchanged input.
  - Unlike `raw_prompt_snippet`, this projection is not gated behind a scan option: it is small, and every index
    consumer benefits. If a scan option is added, it must default to on so the index keeps carrying it.

### Index

- `crates/sase_core/src/agent_scan/index.rs`:
  - Add `"xprompts.json"` to `MARKER_FILES` so `marker_signature` will sign it.
  - Add `xprompts: Option<String>` to `MarkerSignatures` and populate it from `dir.join("xprompts.json")`. This is what
    makes a late `step_only` seed re-index the row instead of caching an empty list forever.
  - Add an `xprompts_sig TEXT` column to the `agent_artifacts` DDL, to `ensure_agent_artifacts_column` under the new
    version gate, and to every statement that reads or writes the signature set — the `INSERT` column list and its
    `ON CONFLICT ... excluded` assignments in `upsert_record`, and the three `SELECT ... _sig` statements
    (terminalization candidates, the `where_sql` refresh selection, and the stale-row refresh) plus
    `pending_row_from_sql`.
  - Bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` 18 → **19** and add `migrate_record_json_refresh_v19` following the
    existing empty-body-with-doc-comment convention: the Python lifecycle detects the bump and rebuilds the index from
    source, which is what backfills `used_xprompts` for every historical run.

### Tests

Add Rust tests covering: a directory whose `xprompts.json` projects correctly (including duplicate names collapsing into
`references`); missing, empty, malformed, and non-array files projecting to an empty vector without failing the scan;
unknown `kind` normalization; deterministic ordering; a row whose `xprompts.json` appears _after_ first indexing being
re-indexed because `xprompts_sig` changed; and the new column surviving the `ensure_agent_artifacts_column` upgrade path
on an index created at the previous schema version.

Run `cargo fmt`, `cargo clippy --workspace --all-targets`, and `cargo test --workspace` before finishing.

### Done when

`scan_agent_artifacts` returns populated `used_xprompts` for artifact directories that have `xprompts.json`, an index
opened at schema 18 upgrades to 19 with the new column, and a late-written `xprompts.json` refreshes its cached row.

---

## `core-stats`: xprompt aggregation in the run-statistics query

Repository: **`sase-core`** (same access rules as `core-scan`).

### Wire

`crates/sase_core/src/agent_stats/wire.rs`: add the three request fields and the three response structs exactly as
specified in _The wire contract_, plus the two `default_*` functions. Bump `AGENT_STATS_WIRE_SCHEMA_VERSION` 3 → **4**
(the Python side reads the payload defensively and does not assert this value, so the bump is informational).

Add a serde round-trip test proving that a response payload with the `xprompts` key removed still deserializes with
`xprompts == None`, mirroring the existing `older_run_stats_payload_without_runners_deserializes` test.

### Query

`crates/sase_core/src/agent_stats/run.rs`, inside the existing single pass over in-window rows (after the
`launch_in_window` guard, alongside `fold_commits` / `fold_workspace`):

- Accumulate per xprompt name: run count, summed `references`, distinct agent names (`row.agent_name`, matching how
  `AgentChangeSpecWorkStatsWire::distinct_agents` is derived), completed/failed counts using the same `outcome` value
  the surrounding code already computed, runtime totals from the existing `run_duration_seconds` (a run without a valid
  duration contributes to `runs` but not to the mean), first/last launch timestamps, and per-name maps of model → runs,
  project → runs, and partner name → runs.
- Partners are every _other_ distinct xprompt name in the same run's `used_xprompts`; a run referencing one xprompt
  contributes no partner entries.
- `runs_with_xprompts` / `runs_without_xprompts` partition the in-window runs. Rows whose `record_json` failed to decode
  are already counted in `malformed_rows_skipped` and are excluded from both.
- Rank rows, truncate to `xprompt_top_n`, and set `truncated_rows` from the pre-truncation distinct count. Cross-tab
  lists inside each row truncate to `xprompt_breakdown_top_n`.
- When `xprompt_focus` is set, accumulate the deeper focus payload for that exact name in the same pass — including
  providers, tribes (reuse the existing tribe resolution behind
  `runtime_group_values(AgentStatsRuntimeGroupByWire::Tribe, ...)`) and launch buckets built with the same
  `build_empty_buckets` / `increment_bucket` helpers and `bucket_seconds` as the response. A focus name with no
  in-window run returns `found: false` with zeroed aggregates rather than `None`, so the UI can say so precisely.
- The `project` request filter already restricts the SQL selection, so focus and cross-tabs inherit it for free.
  `xprompt_focus` must **not** filter any other section of the response.

### Tests

Rust tests over a temporary index covering: counting by run rather than by reference; `references` exceeding `runs` when
a run referenced one name twice with different arguments; distinct-agent counting; the model/project/partner cross-tabs;
solo runs producing no partners; ranking ties broken by name; `xprompt_top_n` truncation reporting `truncated_rows`;
`xprompt_breakdown_top_n` truncation; focus payload correctness including buckets; `found: false` for an unknown focus
name; interaction with the `project` filter; and `runs_without_xprompts` counting runs with an empty projection.

Run `cargo fmt`, `cargo clippy --workspace --all-targets`, and `cargo test --workspace` before finishing.

### Done when

`agent_stats_query_runs` returns a populated `xprompts` section for a window containing runs with `used_xprompts`, and
returns focus detail when `xprompt_focus` is supplied.

---

## `py-stats`: Python statistics models and builder

Repository: **`sase`** (this repo). This phase is written against _The wire contract_ and is fully testable from fixture
payloads, so it does not wait on the `sase-core` phases.

### Query knobs

`src/sase/stats/query.py`: `query_run_stats` gains `xprompt_top_n: int = 40`, `xprompt_breakdown_top_n: int = 5`, and
`xprompt_focus: str | None = None`, forwarded into the request dict. Older installed extensions ignore unknown request
keys, so this is safe to send unconditionally.

### View models

`src/sase/stats/_view_models.py`:

```python
@dataclass(frozen=True, slots=True)
class XPromptCountRow:          # one cross-tab entry with its within-xprompt share
    key: str                    # raw key (project key, model name, partner name)
    label: str                  # display label (project display name; otherwise == key)
    count: int
    share: float                # count / owning row runs, 0.0 when runs == 0

@dataclass(frozen=True, slots=True)
class XPromptRow:
    name: str; kind: str; tags: tuple[str, ...]
    runs: int; references: int; distinct_agents: int
    completed: int; failed: int; success_rate: float
    total_runtime_seconds: float; mean_runtime_seconds: float | None
    first_run_ts: float; last_run_ts: float; share: float   # runs / runs_with_xprompts
    models: tuple[XPromptCountRow, ...]
    projects: tuple[XPromptCountRow, ...]
    partners: tuple[XPromptCountRow, ...]

@dataclass(frozen=True, slots=True)
class XPromptFocusView:
    # every XPromptRow field except `share`, plus:
    found: bool
    providers: tuple[XPromptCountRow, ...]
    tribes: tuple[XPromptCountRow, ...]
    buckets: tuple[RunBucket, ...]

@dataclass(frozen=True, slots=True)
class XPromptsView:
    available: bool             # False iff the payload had no `xprompts` section
    runs_with_xprompts: int
    runs_without_xprompts: int
    distinct_xprompts: int
    total_references: int
    truncated_rows: int
    rows: tuple[XPromptRow, ...]
    focus: XPromptFocusView | None
```

`available` mirrors the `RunnersView.available` precedent: it distinguishes "this core build cannot report xprompts"
from "this range genuinely has no xprompt usage" (`available=True`, `rows=()`).

### Builder

`src/sase/stats/_view_builders.py`: add `build_xprompts_view(run_payload, display_snapshot, *, timezone)` using the
existing `sase.stats._view_payload` helpers (`mapping`, `rows`, `integer`, `number`, `optional_number`, `text`, `ratio`,
`bucket_label`). Project keys become display labels through `project_display_for(...)`/`ProjectDisplaySnapshot` — the
`Show Project Names, Never ProjectSpec Keys` gotcha applies to every project cell in this feature. Model, partner,
provider, and tribe labels equal their keys.

`src/sase/stats/views.py`: add `xprompts: XPromptsView` to `StatisticsViews` and wire the builder into
`build_statistics_views` (it needs the resolved timezone for bucket labels).

`src/sase/ace/tui/modals/statistics_pane_data.py`: `load_statistics_view` gains an `xprompt_focus: str | None`
parameter, forwards it to `query_run_stats`, and records it on `StatisticsViewData` so the pane can compare it when a
worker result lands.

### Tests

Extend `tests/stats/test_views.py`: an absent `xprompts` key yields `available=False`; a present but empty section
yields `available=True` with zero rows; a populated section yields ranked rows with correct shares, cross-tab shares,
`mean_runtime_seconds=None` passthrough, and tag tuples; a focus payload builds `XPromptFocusView` with labelled
buckets; `found=False` is preserved. Add a `tests/stats/test_query.py` case asserting the three new request keys reach
the binding stub.

### Done when

`just check` passes and `build_statistics_views` produces a correct `XPromptsView` from fixture payloads with and
without the `xprompts` section.

---

## `tui-view`: the XPrompts sub-tab and its four grouping strategies

Repository: **`sase`**.

### Registration

`src/sase/ace/tui/modals/statistics_pane_data.py`:

- Add `"xprompts"` to `StatisticsView` and insert it into `VIEW_ORDER` **immediately after `"activity"`** (it answers
  the same "what did agents use" question) and before `"plans_questions"`.
- `VIEW_LABELS["xprompts"] = "XPrompts"`; keep the same compact label.
- `VIEW_DESCRIPTIONS["xprompts"] = "XPrompt usage across prompts, with model, project, and co-usage breakdowns."`
- `XPromptsGroupBy = Literal["usage", "model", "project", "pairing"]` and
  `XPROMPTS_GROUP_ORDER = ("usage", "model", "project", "pairing")`.
- `statistics_view_supports_grouping` returns `True` for `"xprompts"`.

### Tab strip capacity

Nine non-compact tabs render 118 columns wide, which still fits the 120-column layout, but the compact (sub-108-column)
line reaches exactly 90 columns and would clip the 90-column layout. Add an optional `compact_separator: str = " │ "`
parameter to `PanelTabStrip` (`src/sase/ace/tui/widgets/panel_tab_strip.py`) used only when `self._compact` is true, and
have `StatisticsPane` pass `"│"`. The default keeps `ConfigCenterModal`'s strip unchanged. Confirm
`_tab_ranges`/`_line_width` (and therefore click hit-testing) are still derived from the rendered text.

### Renderers

New module `src/sase/ace/tui/modals/statistics_pane_xprompts.py` with `StatisticsXPromptsRenderingMixin`, mixed into
`StatisticsViewsRenderingMixin` next to the existing Projects and Runners mixins, and dispatched from
`_view_renderable`. It must perform no I/O. Follow the visual language already established by
`statistics_pane_projects.py`: `box.SIMPLE` tables, `_share_bar`, `_percent`, `format_duration`, `_format_timestamp`,
`categorical_color`, and the module accent palette.

- **Summary line** (all modes): `N xprompts · R runs referenced · X references · U runs without xprompts (P%)`, plus
  `M more xprompts not shown.` when `truncated_rows` is non-zero — silent truncation is not acceptable.
- **`usage`** — one ranked table: XPrompt (a `#name` cell colored by `categorical_color(name)` with a dim `wf`/`part`
  kind badge and dim tags), Runs, Refs, Share + bar, Agents, Success, Wall, Last used.
- **`model`** — a drilldown table shaped like `_projects_drilldown_renderable`: a bold xprompt row, then its indented
  top models with count, within-xprompt share, and bar. A row with no model data gets a dim `(no model recorded)` child.
- **`project`** — the same drilldown over projects, rendering each project through the existing
  `_project_cell(project_key, project_label)` so the color square and display name match the Projects view.
- **`pairing`** — the same drilldown over partner xprompts, with a dim `(used alone)` child when a row has no partners.

### States

Dispatch order inside the XPrompts renderer:

1. `not views.xprompts.available` → an "XPrompt statistics unavailable" panel in the style of
   `_runners_unavailable_renderable`, explaining that the installed `sase-core-rs` build does not report xprompt usage
   yet and naming the effective refresh key.
2. `available` with `runs_with_xprompts == 0` → an honest empty panel: no prompt in this range referenced an xprompt,
   plus the effective range key (and the project-filter key when a filter is active, matching
   `_empty_state_renderable`).
3. Otherwise the grouped body.

The pane-level `views.empty` guard (no runs at all in range) still short-circuits first, as today.

### Grouping control

`StatisticsPane.action_cycle_group` gains an `"xprompts"` branch that advances `_xprompts_group_by` through
`XPROMPTS_GROUP_ORDER` and calls `self._selection_changed(reload=False)` — all four modes render from one payload, so
regrouping must never trigger a query.

`_group_scope_text` gains an `"xprompts"` branch rendering `XPrompts · By Usage | By Model | By Project | Used With`,
using a `_XPROMPTS_GROUP_LABELS` mapping next to the existing `_PROJECTS_GROUP_LABELS`.

### Legend

Add `VIEW_LEGENDS["xprompts"]` in `statistics_pane_legends.py`:

- `Runs` = `runs whose launch prompt referenced the xprompt`
- `Refs` = `references; the same name with different arguments counts twice`
- `Share` = `share of runs that referenced any xprompt`
- `Agents` = `distinct agent names`
- `Used with` = `xprompts referenced by the same run`
- `Scope` = `launch-boundary references only; workflow step templates excluded`

### Tests

Extend the existing Statistics pane suites (`tests/ace/tui/test_statistics_pane_rendering.py`,
`test_statistics_pane_interactions.py`, `test_statistics_legends_states.py`) plus the shared fixtures in
`tests/ace/tui/_statistics_pane_helpers.py` (add an `xprompts` section to `_run_payload`). Cover: the new tab appearing
in `VIEW_ORDER` and cycling with the prev/next keys; each grouping mode rendering its distinctive columns; `g` cycling
without issuing a reload; the unavailable, empty, and truncated states; project cells showing display names rather than
ProjectSpec keys; and the compact tab strip fitting at 90 columns.

### Done when

`just check` passes and the XPrompts sub-tab renders all four grouping strategies plus all three states from fixture
payloads.

---

## `tui-focus`: zoom into one xprompt

Repository: **`sase`**.

### Pane state

`StatisticsPane` gains `_xprompt_focus: str | None` and `_xprompt_focus_options: tuple[str, ...]` (cached from the most
recent unfocused result, mirroring `_project_filter_options`). `_start_load` passes the focus into
`load_statistics_view`, and `on_worker_state_changed` adds `result.xprompt_focus != self._xprompt_focus` to the
stale-result guard so a late worker never paints another selection's data.

### Picker modal

New `src/sase/ace/tui/modals/statistics_xprompt_picker_modal.py`, modelled on the existing picker modals
(`saved_query_picker.py`, `property_picker_modal.py`) for structure, styling, and dismissal idiom:

- `StatisticsXPromptPickerModal(ModalScreen[XPromptFocusChoice | None])`, where `XPromptFocusChoice` is a frozen
  dataclass with `name: str | None`; `None` inside the choice means **All xprompts**, and dismissing with `None` means
  cancelled. This keeps "clear the focus" and "changed my mind" distinguishable.
- Built entirely from already-loaded `XPromptsView` rows — the modal performs no I/O and issues no query.
- Rows: an `All xprompts` entry first, then the ranked xprompts with their runs count, kind badge, and tags, so the
  picker doubles as a discoverable index of what is available.
- A filter `Input` at the top narrows rows by case-insensitive substring; `j`/`k` and the arrow keys move; `enter`
  selects; `esc`/`q` cancels. The currently focused xprompt starts highlighted.
- Add the modal's styles next to the other Statistics rules in `src/sase/ace/tui/styles.tcss`.

### Keys and configuration

Two new focused-pane actions. Every one of these surfaces must be updated together or the keymap consistency guards
fail:

- `src/sase/ace/tui/keymaps/app_keymaps.py` — `StatisticsPaneKeymaps.focus_xprompt: str = "x"` and
  `clear_xprompt_focus: str = "X"`.
- `src/sase/ace/tui/keymaps/metadata.py` — `_STATISTICS_BINDING_META` entries `("focus_xprompt", "Focus XPrompt")` and
  `("clear_xprompt_focus", "Clear XPrompt Focus")`, placed after the project-filter entries so help and hint ordering
  stays logical.
- `src/sase/default_config.yml` — the `ace.keymaps.statistics` block (required by the `Default Keymap Config` gotcha).
- `src/sase/config/sase.schema.json` — both properties with descriptions.
- `docs/ace.md` — the `Remapping Statistics Pane Keys` YAML block and its explanatory paragraph.

`action_focus_xprompt` is a no-op unless the current view is `"xprompts"` and a result is loaded; it pushes the picker
and applies the choice with `_selection_changed(reload=True)`. `action_clear_xprompt_focus` clears the focus and reloads
only when a focus was set.

### Scope chip

Add a fourth `Static` `#statistics-scope-xprompt` with class `statistics-scope-part` to the `#statistics-scope` row,
using the existing accent-chip helper `_scope_text`. It is hidden (the `hidden` class, exactly like the group chip)
unless the current view is `"xprompts"`. It reads `x/X XPrompt  All xprompts` or `x/X XPrompt  ■ #name`, with the square
colored by `categorical_color(name)` so the color matches the tables. Because the scope parts are `1fr` (range is `2fr`)
and already `text-overflow: ellipsis`, no CSS width changes are needed; verify the 90-column layout.

While a focus is active, hide the **Group** chip and make `action_cycle_group` a no-op only if the focused body ignores
grouping — it does not (see below), so the Group chip stays visible and functional throughout.

### Focused body

When `_xprompt_focus` is set and the view is `xprompts`, the body renders the focus dashboard instead of the ranked
tables, keeping `g` meaningful:

1. **Focus header panel** (always): `#name` title with kind badge and tags, then Runs, Refs, Agents, Success (✓
   completed / × failed), Wall, Mean, First used, Last used.
2. **Group-dependent detail**, built from `XPromptFocusView`:
   - `usage` → a "Runs over time" bucket table (same shape as the Overview bucket table) plus a `Columns` of three
     panels: Top models, Top projects, Used with — each a count table with share bars. This is the at-a-glance
     dashboard.
   - `model` → the full ranked model table (count, share, bar), not truncated to the breakdown cap.
   - `project` → the full ranked project table, rendered with `_project_cell`.
   - `pairing` → the full ranked partner table. Providers and tribes render as a compact secondary `Columns` beneath the
     detail in every mode.
3. A dim footer naming the effective clear-focus key.

When `focus.found` is `False`, render a single honest panel: this xprompt has no runs in the selected range, plus the
effective range key and clear-focus key.

### Help modal

`statistics_help_modal.py` reads views and glossary from `VIEW_ORDER`/`VIEW_LEGENDS`, so those update themselves. Still
required:

- `_control_value` cases for `focus_xprompt` and `clear_xprompt_focus` reporting the current focus label, and a
  `cycle_group` case covering `xprompts`.
- `_controls_text` must hide the two focus controls when the current view is not `"xprompts"`, matching the existing
  `cycle_group` filter.
- A new always-present **XPrompt methodology** section next to the runner one, stating: counts come from the
  launch-boundary `xprompts.json` written before expansion; workflow step-template references are excluded; a run counts
  once per xprompt name while `Refs` counts argument variants separately; partners are other xprompts in the same run;
  the project filter applies before aggregation; and historical rows appear after the artifact index has been rebuilt at
  its current schema.
- `StatisticsPane.action_help` passes the focus label through to the modal.

The `Help Popup Maintenance` rule in `src/sase/ace/CLAUDE.md` makes this non-optional.

### Hints bar

`_hints_text` gains a focus entry (`x focus`) shown only on the XPrompts view, so the bottom hint line stays accurate
per view.

### Tests

Extend `tests/ace/tui/test_statistics_pane_interactions.py`, `test_statistics_pane_bindings.py`,
`test_statistics_scope_header.py`, and `test_statistics_help_modal.py`, and add a picker-modal test. Cover: `x` opening
the picker only on the XPrompts view; selecting a row setting the focus and triggering exactly one reload with the right
`xprompt_focus`; choosing **All xprompts** and pressing `X` both clearing it; cancelling leaving state untouched; the
scope chip text and its hidden class per view; a stale worker result for a different focus not painting; the focused
body per grouping mode; the `found=False` panel; and the new help controls and methodology section. Add the keymap
surface updates to `tests/test_keymaps_defaults.py`, `tests/test_keymaps_validation.py`,
`tests/test_keymaps_registry_loading.py`, and `tests/test_config_schema.py`.

### Done when

`just check` passes and the user can zoom into one xprompt, regroup its detail with `g`, and clear the focus.

---

## `land`: cross-repo contract, snapshots, and documentation

Repository: **`sase`**, after both `sase-core` phases are merged and released.

### Version floor and index schema constant — must land together

`src/sase/core/agent_scan_wire_records.py` holds `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION = 18`, and
`refresh_agent_artifact_index_if_schema_stale` rebuilds the index whenever the stored version is **less than** that
constant. Bumping the Python constant to 19 while an installed wheel still writes 18 makes every ACE startup rebuild the
whole index forever. Therefore:

- Bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` to `19` **only** in the same change that raises the `sase-core-rs` floor in
  `pyproject.toml` to the first published release containing both `sase-core` phases, and refresh `uv.lock`.
- Follow the precedent of commit `17fc09cdc` ("build(deps): require sase-core-rs>=0.12.9 for the chop report wire"): the
  commit message must name the wire the floor guarantees.
- If that release is not published yet, land everything else and keep the constant at 18 — the feature degrades to the
  "unavailable" panel for old builds and to newly indexed runs only for dev builds — then do the floor bump plus
  constant bump as one follow-up commit. Do not split them.

Note for local verification: `just install` builds `sase_core_rs` from the linked `sase/repos/linked/sase-core`
checkout, so a dev workspace can exercise the full feature before any release exists.

### End-to-end verification

With a dev build of both repos, run `sase ace`, open `#` → Statistics → XPrompts, and confirm against real recorded
history: totals look plausible against `~/.sase/projects/*/artifacts/**/xprompts.json`; `g` cycles all four groupings
without a visible reload; `x` opens the picker and focusing changes the body; `X` clears it; the range and project
filters compose with all of it. Confirm the first launch after the schema bump rebuilds the index once and not on every
subsequent start.

### Visual snapshots

Add PNG snapshots via `tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py` and its helpers
(`_ace_config_center_statistics_helpers.py`, `_ace_config_center_png_snapshot_helpers.py`), extending the populated
statistics fixture with an `xprompts` section: `config_center_statistics_xprompts_120x40` (usage),
`..._xprompts_model_120x40`, `..._xprompts_focus_120x40`, and `..._xprompts_narrow_90x30` (nine-tab compact strip).
Regenerate the existing statistics snapshots, which change because the tab strip gained a tab and a compact separator,
using `just test-visual --sase-update-visual-snapshots` and review every regenerated image before accepting it.

### Documentation

- `docs/ace.md`: document the XPrompts sub-tab — what it counts, the four `g` groupings, focus with `x`/`X`, and the
  launch-boundary scope caveat. Explicitly disambiguate it from the Admin Center's existing top-level **XPrompts** tab
  (the xprompt browser), which is a different surface. Update the `Remapping Statistics Pane Keys` block if `tui-focus`
  has not already.
- `docs/rust_backend.md`: extend the agent-artifact scan/index bullet to mention the launch-boundary xprompt projection
  and the run-statistics xprompt rollup.

### Done when

`just check` and `just test-visual` pass, the floor and constant move together, and the documented behavior matches what
the pane actually does against real data.

## Cross-cutting rules for every phase

- Run `just install` before `just check` — workspaces are ephemeral and dependencies drift.
- `just check` is mandatory for any phase that changes files in the `sase` repo.
- `sase-core` phases run `cargo fmt`, `cargo clippy --workspace --all-targets`, and `cargo test --workspace`, and reach
  that repo only through the `/sase_repo` skill.
- Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction shims: this feature needs no memory
  change, and no plan file can authorize one.
- Never render a ProjectSpec key (`gh_sase-org__sase`) in user-facing text; always resolve the display name.
- Never report a silently truncated list: `truncated_rows` has a visible footnote.
