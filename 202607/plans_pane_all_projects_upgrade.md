---
tier: epic
title: 'Plans artifacts pane: all-projects scope, recency sort, and readable details'
goal: 'The ACE Artifacts → Plans sub-tab is useful the moment it opens: it aggregates
  plan proposals, epic beads, and archived plans across every enabled project by default,
  lists them newest-first on single non-wrapping rows, and renders a readable, beautiful
  detail pane that shows every property from the plan file''s frontmatter.

  '
phases:
- id: all-projects-data
  title: Multi-project plans snapshot and all-projects default scope
  depends_on: []
- id: list-redesign
  title: Recency-sorted, single-line plan pipeline list
  depends_on:
  - all-projects-data
- id: detail-redesign
  title: Readable detail pane with full frontmatter properties
  depends_on:
  - list-redesign
- id: polish-verify
  title: Visual snapshots, copy audit, and end-to-end verification
  depends_on:
  - detail-redesign
create_time: 2026-07-16 08:43:21
status: wip
bead_id: sase-6a
---

# Plan: Plans artifacts pane — all-projects scope, recency sort, and readable details

## Context and diagnosis

The "Plans" sub-tab of the ACE "Artifacts" tab (`src/sase/ace/tui/widgets/artifacts/plans_pane.py` plus its off-thread
loader `plans_data.py`) has five user-visible problems:

1. **Empty on first open (bug).** `AceApp` seeds `artifacts_project_scope` from `get_sole_project_filter(parsed_query)`
   (`src/sase/ace/tui/actions/_state_init.py`). When the query does not narrow to exactly one project, the scope stays
   `None`, and `_ensure_artifacts_project_choices` (`src/sase/ace/tui/actions/artifacts.py`) auto-picks a project only
   when exactly one project is enabled. With several enabled projects the scope remains `None` and
   `ArtifactsPlansPane._build_options` renders only "Pick a project to browse plans." Notably, the shared
   `InventoryProjectPicker` modal _already_ offers an "All projects" choice whose result is `project_key=None` — the
   Plans pane simply has no all-projects mode, so choosing it also produces the empty state. The expectation: on first
   open, plans from **all enabled SASE projects** are shown.

2. **Unreadable value highlighting in the Details pane.** The detail pane is a Textual `Markdown` widget, and
   `_bead_markdown`/`_archive_markdown`/`_proposal_markdown` wrap metadata values (Status, Type, Design path, Tier,
   Model, Path, …) in backticks. Textual's built-in `Markdown` component style for `.code_inline` is
   `background: $warning 10%; color: $text-warning 95%` in dark mode; under the app's `flexoki` theme this renders as
   orange-on-orange blocks in which the value text is effectively invisible.

3. **Frontmatter properties are not all shown.** Archived plan rows come from the Rust-backed plan search whose `Plan`
   wire record already carries a complete flattened `frontmatter: dict[str, str]` (`src/sase/plan_search/model.py`), but
   the detail rendering shows only Tier/Status/Created/Path. Proposal details show inventory metadata while the plan
   file's frontmatter YAML block is dumped verbatim into the rendered markdown body.

4. **No recency ordering.** Epics sort by status-then-id; proposals follow inventory order. The user wants the most
   recently proposed plans at the top.

5. **Wrapped rows in the left pane.** Epic rows render `▸ ● <id> <long title>  N/N phases LAUNCHED <changespec> #<bug>`
   inside an `OptionList`; long rows wrap across 2–3 lines, which destroys scanability.

## Design

### 1. All-projects scope (default) — fixes the initial-load bug

Treat `project_scope=None` as **"All projects"** (aggregate over enabled projects) instead of "no scope yet". This
aligns Plans with the picker's existing "All projects" choice (`InventoryProjectPickerResult.project_key=None`) and the
Commits pane's labeling of the `None` scope.

- **Loader.** Extend `load_plans_snapshot` (`plans_data.py`) so `project=None` resolves the enabled project set
  off-thread via `sase.core.project_lifecycle_facade.list_project_records` (same filtering as
  `_collect_artifacts_project_choices`: real, non-system-managed projects; enabled only), then aggregates per project:
  - **Proposals:** `build_plan_inventory` is already cross-project — instead of filtering rows to one project, keep rows
    whose `ProposedPlan.project` is in the enabled set (single-project scope keeps the current equality filter).
  - **Epics/phases:** iterate enabled projects, reusing the existing per-project `BeadProject` read. A project without a
    bead store is silently skipped in aggregate mode (today's "No bead store" error only applies to an explicitly scoped
    project). A project whose store fails to read must not blank the pane: record a per-project error note surfaced in
    the status line / detail empty-state, and keep the other projects' results.
  - **Archive:** run the existing `plan_search.facade.search(..., sort="recent", limit=50)` per project root and merge
    newest-first with a total cap (keep 50 per project, cap the merged section at ~100 with a "showing newest N" hint
    when truncated).
- **Snapshot shape.** `PlansSnapshot` becomes multi-project aware: `project: str | None` (`None` = all), the resolved
  project list, per-entry project attribution (each proposal already knows its project; each epic/phase/archive entry
  must carry the owning project), and per-project `workspace_dir`s. The mtime-based `source_key` must aggregate every
  scanned project **and** include the enabled-project identity set, so enabling/disabling a project invalidates the
  cached snapshot. Preserve the existing contract: unchanged key ⇒ reuse `previous` snapshot without re-reading stores.
- **Row-scoped actions.** `PlanRow` gains the owning project, and every action in
  `src/sase/ace/tui/actions/artifacts_plans.py` (`plans_cycle_status`, `plans_edit_bead`, `plans_launch_epic`,
  `plans_open_bug`, and the tracked-task helpers) resolves the bead store, workspace dir, and dedup keys from the
  selected row's project rather than the snapshot-global `snapshot.project`/`snapshot.workspace_dir`. Approve/reject
  already route through the notification and need no scope.
- **Scope UX.** The scope chip in `_scope_text()` shows "All projects" when scope is `None` (never "Pick a project");
  `_empty_detail()` and the disabled placeholder option copy change accordingly. Picking "All projects" in the picker
  now works for Plans by construction. The `p` keymap, picker modal, and the single-enabled-project auto-pick behavior
  stay as they are.
- **Perf guardrails** (per `memory/tui_perf.md` patterns already used by this pane): all disk access stays inside the
  existing `run_worker(thread=True)` task; loading/pending coalescing flags, the `DetailPanelDebouncer`, and the
  `_syncing_options` guard are unchanged. Aggregate mode must not add any event-loop work beyond rendering the merged
  snapshot.

### 2. Recency-sorted, single-line list

- **Ordering.** Keep the three sections (Proposals / Epics / Plan archive) with Proposals first — pending approvals are
  the actionable items. Within each section sort newest-first: proposals by notification timestamp descending; epics by
  `Issue.created_at` descending (missing timestamps sort last, tie-break by id); archive is already `sort="recent"`.
  Phases under an expanded epic keep hierarchical id order.
- **One line per row, always.** Build every option label as a Rich `Text` with `no_wrap=True, overflow="ellipsis"` so
  long titles ellipsize instead of wrapping.
- **Concise row anatomy.** Trim suffix noise so ellipsis is rarely needed:
  - Epic: `▸ ● sase-11 Title…  3/5  READY  2mo` — drop the word "phases" (keep `closed/total`), keep the state chip
    (`READY`/`BLOCKED`/`LAUNCHED`), add a compact relative age (from `created_at`) so the recency ordering is legible.
    Move `changespec_name` and `#bug` to the detail pane only.
  - Proposal: `◆ Title…  epic  2m` (agent/model live in the detail pane).
  - Archive: `▤ Title…  epic  done  06-12`.
  - Phase rows keep the `↳` + glyph + id + title shape with a short state suffix.
- **Project badges.** In all-projects mode only, append a dim `[project-display-name]` badge to proposal/epic/archive
  rows so provenance is obvious; omit it in single-project scope.
- **Wider left panel.** Bump `#plans-list-panel` (`src/sase/ace/tui/styles.tcss`) from `width: 48%; min-width: 46` to
  roughly `width: 55%; min-width: 56` so typical rows fit whole.
- **Status line.** Keep the counts line; in all-projects mode prefix it with the project count (e.g.
  `4 projects · 2 proposals · 231 epics · …`) and surface per-project load errors here.

### 3. Readable detail pane with full frontmatter

Restructure the detail panel from "one Markdown document with backticked values" into a two-part surface inside the
existing `#plans-detail-scroll`:

- **Properties header (Rich, not Markdown).** A `Static` rendering a title line plus an aligned key/value grid
  (borderless two-column Rich table: dim right-aligned keys, styled values). This gives exact styling control and
  matches the app's other detail panes (commits, ChangeSpec), and removes the unreadable inline-code blocks from
  metadata entirely:
  - **Proposal rows:** State (pending approval), Tier, Agent, Model, Proposed (timestamp + age), Project, Path —
    followed by **every** frontmatter property parsed from the plan file.
  - **Bead rows:** Status, Readiness, Type, Model, Assignee, ChangeSpec, External bug, Design path, Project,
    Created/Updated/Closed (+ close reason), dependencies with their states. Status/readiness values render as colored
    chips consistent with the list glyph colors.
  - **Archive rows:** **every** `plan.frontmatter` key, plus Source, Project, and Path. Render well-known keys first in
    a stable order (title, tier/kind, status, created, goal), then any remaining keys alphabetically; long values (e.g.
    `goal`) wrap inside the grid.
- **Body (Markdown widget).** Below the header, the existing `Markdown` widget renders only the document body:
  proposal/archive plan bodies with the leading frontmatter block stripped, bead description/notes. For proposals, parse
  the already-read file content's leading YAML block into a flat string map for display (same projection shape as the
  Rust wire's `frontmatter` map; `plans_data._plan_title` already does a display-only frontmatter read, and
  authoritative plan parsing/validation stays in the Rust core). Archive bodies from the Rust reader are already
  frontmatter-free.
- **Inline-code CSS fix.** Add an app-stylesheet override for the Markdown widget's `.code_inline` component style (e.g.
  `background: $foreground 10%; color: $text` on `MarkdownBlock`) so inline code inside any rendered plan body is
  readable under flexoki in the ace TUI. Keep the override app-wide in `styles.tcss` for consistency across current and
  future Markdown surfaces.
- The `Enter` full-screen preview (`PreviewPanelModal`) keeps its current behavior.

### 4. Explicitly out of scope

- No new keybindings, keymap changes, or `default_config.yml` edits.
- No changes to the Rust core (`sase-core`): archive frontmatter already crosses the wire, and proposal frontmatter
  display is a rendering-only projection of file content already read by the loader.
- No changes to plan approval flow, bead mutation semantics, or the other Artifacts sub-tabs (beyond the shared scope
  label copy that Plans owns).

## Phase details

### Phase `all-projects-data` — multi-project snapshot and default scope

- Rework `plans_data.py`: aggregate loader for `project=None` (enabled-project resolution, per-project
  proposals/beads/archive collection, per-entry project attribution, per-project error isolation, merged mtime
  `source_key` including the enabled-project set, archive caps).
- Update `PlansSnapshot`/`PlanRow` shapes; thread the row project through `actions/artifacts_plans.py` (bead updates,
  epic launch, open-bug, dedup keys, task subjects).
- Scope plumbing: `None` renders as "All projects" in `_scope_text()`; loading/empty copy in
  `_build_options()`/`_empty_detail()` updated; pane reloads on activation when scope is `None` and no snapshot exists
  (today's early-return when `project is None` in `_request_load` must go).
- Unit tests (`tests/ace/tui/test_artifacts_plans.py` + a data-layer test module): multi-project fixtures, default-scope
  load on first activation, per-project error isolation, source-key invalidation on enabled-set change, action routing
  to the row's project, picker "All projects" round-trip.

### Phase `list-redesign` — recency sort and single-line rows

- Sorting changes in `plans_data.py` (epics newest-first; proposals newest-first) and row construction in
  `plans_pane.py` (`_proposal_text`, `_epic_text`, `_phase_text`, `_archive_text`): `no_wrap`/ellipsis, compact
  suffixes, relative age, all-projects badges.
- `styles.tcss`: widen `#plans-list-panel`; status-line project count and error surfacing.
- Unit tests for ordering, single-line labels (no newline in rendered strips at narrow widths), badge presence only in
  all-projects mode.

### Phase `detail-redesign` — properties header and readable body

- Split the detail area into properties `Static` + body `Markdown` inside `#plans-detail-scroll`; implement the
  property-grid renderers for proposal/bead/archive rows; frontmatter display projection for proposals (flat string
  map + body stripping); remove the backtick-metadata markdown builders.
- Add the `.code_inline` override to `styles.tcss`.
- Unit tests: all frontmatter keys rendered (known-first ordering), proposal body shown without its frontmatter block,
  bead dependency states intact, detail debouncer path unchanged.

### Phase `polish-verify` — snapshots, copy, verification

- Update the two existing PNG snapshots and `_snapshot()`/`_choices()` fixtures
  (`tests/ace/tui/visual/test_ace_png_snapshots_artifacts_plans.py`), and add an all-projects populated snapshot
  exercising project badges and the properties grid (goldens via `--sase-update-visual-snapshots`; inspect
  `.pytest_cache/sase-visual/` artifacts).
- Audit stale copy: plans entries in the help modal, `_PLACEHOLDER_COPY["plans"]` in `widgets/artifacts/panes.py`, and
  any "Pick a project" strings the pane still shows.
- End-to-end verification in a real `sase ace` session: fresh open of Artifacts → Plans shows aggregated plans without
  picking a project; `p` scope narrowing/widening works; bead status cycling and proposal approve/reject work from
  all-projects mode; detail pane readable under flexoki. Run the full `just check` gate.

## Testing strategy

Each phase lands with its own unit coverage as described above; the final phase owns the visual snapshot refresh plus
manual end-to-end verification. Existing tests to keep green: `tests/ace/tui/test_artifacts_plans.py`,
`tests/ace/tui/test_artifacts_scaffold.py`, the plans PNG snapshots, and the `bench_artifacts_jk.py` navigation bench
(j/k latency must not regress; list rendering stays cached-snapshot-driven).

## Risks and mitigations

- **Aggregate load cost.** Reading bead stores for many enabled projects is more disk work per refresh. Mitigation: it
  all runs on the existing worker thread with the same coalescing flags, and the mtime source key means unchanged stores
  skip re-reads entirely on re-activation.
- **Cross-project id collisions.** Bead ids embed the project prefix by convention, but option ids and row attribution
  must not rely on that: namespace option ids (and `phases_by_epic` lookups) with the owning project in aggregate mode.
- **Stale action targeting.** Actions previously assumed one global project; missing one call-site would mutate the
  wrong store. Mitigation: actions consume only row-level project context, and tests cover each action in all-projects
  mode.
- **Visual churn.** The list and detail redesigns change golden PNGs; land goldens in the same phase as the rendering
  change to keep CI green, and rely on the deterministic visual fixtures (pinned fonts/colors).
