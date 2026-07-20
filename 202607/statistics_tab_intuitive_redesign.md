---
tier: epic
title: Intuitive Statistics tab redesign for the SASE Admin Center
goal: 'The Admin Center Statistics tab explains itself: every view, control, and metric
  is labeled, described, and discoverable through visible scope chips, per-view descriptions,
  truthful metric legends, a contextual help overlay, and actionable empty/error states
  — all with a polished visual language consistent with the rest of the Admin Center.

  '
phases:
- id: scope-header
  title: Scope bar and view descriptions
  depends_on: []
  size: medium
  description: '''Scope bar and view descriptions'' section: replace the heading/range/hints
    split with labeled scope chips (key cap + label + current value), add a per-view
    description line under the view strip, and slim the bottom hints line.'
- id: legends-states
  title: Metric legends and actionable states
  depends_on:
  - scope-header
  size: medium
  description: '''Metric legends and actionable states'' section: add verified one-line
    metric legends to every view, actionable empty/error-state guidance, and clickable
    Overview tiles that jump to their detail views.'
- id: help-overlay
  title: Statistics help overlay and keymap plumbing
  depends_on:
  - legends-states
  size: medium
  description: '''Statistics help overlay and keymap plumbing'' section: add the ?
    contextual help overlay (views, controls with current values, metric glossary,
    data freshness), wire the configurable help keymap, and sync the global help surfaces.'
- id: visual-suite
  title: Visual snapshot refresh and polish pass
  depends_on:
  - help-overlay
  size: small
  description: '''Visual snapshot refresh and polish pass'' section: regenerate and
    extend the PNG snapshot suite (including the help overlay and a narrow-width capture),
    verify compact-width degradation, and run a final consistency and beauty pass.'
create_time: 2026-07-20 13:46:08
status: done
bead_id: sase-8a
---

# Plan: Intuitive Statistics tab redesign for the SASE Admin Center

## Context and problem

The Admin Center Statistics tab (`src/sase/ace/tui/modals/statistics_pane*.py`) is functionally rich — seven views
(Overview, Runs, Projects, Providers, Runtime, Activity, Plans & Questions), six range presets plus custom ranges,
per-view grouping, and a project filter — but almost none of that capability is discoverable or explained:

- **Views are bare names.** Nothing tells the user what "Activity" or "Plans & Questions" contains without visiting each
  one. The Admin Center landing solved this with per-tab description lines; the Statistics sub-views have no equivalent.
- **Metrics are undefined.** "Success", "Share", "Wall", "Specs", "p50/p95", "unattributed runs", "retry chains /
  attempts / kills", "committing agents", and the "↑ 33.3% vs previous" delta all appear with no definition anywhere in
  the UI. The global help modal's entire coverage is one line: "Statistics [/] t/c/g/p/r".
- **State is invisible until you press keys.** `t` cycles six presets, `g` only works on two of seven views (but is
  always hinted), and `p` cycles projects blindly. The current group/filter values appear only as suffixes crammed into
  the title line.
- **Dead-end states.** The empty state ("No agent runs were recorded…") and error state offer no next action, even
  though widening the range (`t`), clearing the filter (`p`), or retrying (`r`) is always one key away.
- **Weak affordances.** Overview tiles summarize Runs, Plans, and Questions data but are inert; the deeper views they
  summarize must be found by cycling.

The redesign keeps the pane's data model, worker/debounce load path, and query layer **unchanged** — this is a
presentation and guidance overhaul, not a data change.

## Design principles

1. **Show state where it is controlled.** Every scope dimension (range, grouping, project filter) is rendered as a chip
   that shows its key, its name, and its current value. The user never has to press a key to learn what a key does.
2. **Describe before you drill.** Each view gets a one-line description in the header, mirroring the Admin Center's
   `› description` pattern, so the seven views read as a labeled catalog rather than a mystery carousel.
3. **Every metric is defined in-place or one key away.** Non-obvious columns get a one-line dim legend under their view;
   the full glossary lives in a `?` overlay.
4. **Dead ends become doorways.** Empty and error panels name the keys that resolve them; Overview tiles link to the
   views that explain them.
5. **Truthful labels only.** All legend and glossary copy must be verified against the actual semantics in
   `src/sase/stats/views.py` and the Rust `agent_stats_query_*` bindings — never guessed. Verified anchors: success rate
   = completed ÷ terminal (finished) runs; the Overview delta compares the preceding equal-length window; runtime stats
   exclude in-progress runs.
6. **One visual system.** Reuse the existing telemetry palette (`src/sase/telemetry/render/palette.py`) and the
   Statistics accent (`#FF87D7`). Text wears text tokens (default/dim); color marks values and identities, never prose.
   Meaning is never carried by color alone — glyphs and words accompany it.

## Target layout

```
Statistics · Overview                                      updated 12:20:05
 Overview  Runs  Projects  Providers  Runtime  Activity  Plans & Questions
› Headline totals for this range, with trend vs the previous period.
 [t] Range  Last 7 days · Jul 01 → Jul 08   [g] Group  —   [p] Project  All
 ┌ tiles (Overview only) ─────────────────────────────────────────────────┐
 ┌ view body ┄ tables/panels with a dim legend line per view ┄────────────┐
       [ / ] switch view · c custom range · r refresh · ? help
```

Top-down reading order: what am I looking at → which views exist → what this view shows → what scope applies → the data
→ the remaining keys.

## Phase designs

### Scope bar and view descriptions

Rework the pane header (`statistics_pane.py` compose + `statistics_pane_rendering.py`) into the layout above:

- **Title line** keeps `Statistics · <view>` at left and moves the status text (`updated HH:MM:SS` / `refreshing…` /
  `load failed`) to the right edge.
- **View description line** directly under the `PanelTabStrip`, rendered as `› <description>` in the Statistics accent
  (dimmed), exactly like the Admin Center header caption. Add a `VIEW_DESCRIPTIONS: dict[StatisticsView, str]` alongside
  `VIEW_LABELS` in `statistics_pane_data.py` with copy along these lines (final copy may be tuned during implementation,
  but every view must have one line ≤ ~78 chars):
  - overview: "Headline totals for this range, with trend vs the previous period."
  - runs: "Run outcomes, lifecycle states, retries, and commit attribution."
  - projects: "Work per project and ChangeSpec: runs, success, commits, wall time."
  - providers: "Runs by provider, model, and effort, with success and mean runtime."
  - runtime: "How long runs take — mean, p50, p95, max — grouped by g."
  - activity: "Which skills, memories, and workspaces agents actually used."
  - plans_questions: "Plan proposals and question sessions raised by agents."
- **Scope chips row** replaces the old `#statistics-range` line. Three chips, each: reverse-video key cap (reuse the
  landing-row key-chip treatment from `config_center_modal.py` for visual kinship), bold label, value in the chip's
  accent: `[t] Range` (preset label plus a short absolute span; a custom range shows `Custom · <value>`), `[g] Group`
  (current grouping on Runtime/Projects; dimmed with value `—` on views without grouping so the row never reflows),
  `[p] Project` (`All projects` or the selected project with its categorical color swatch). Chips ellipsize (`no_wrap`,
  `overflow="ellipsis"`); below ~100 columns the range chip drops the absolute span, keeping the preset label.
- **Hints line** slims to only keys that have no chip: `[ / ] switch view · c custom range · r refresh · ? help` (the
  `?` entry lands in the help-overlay phase; this phase leaves a placeholder-free line without it).
- All new content is built by pure renderable helpers fed from existing pane state — no I/O, no new refresh paths, no
  changes to the worker/debounce/coalescing logic. Update `styles.tcss` for the new header rows.

### Metric legends and actionable states

- **Legend lines.** Add a shared `_legend_note(...)` helper producing a single dim line of `term = meaning` pairs
  separated by `·`, appended inside each view's renderable (`statistics_pane_views.py`, `statistics_pane_projects.py`).
  Coverage (wording verified against `src/sase/stats/views.py` and, where payload fields are opaque, the Rust
  `agent_stats_query_runs`/`agent_stats_query_activity` bindings in the sibling `sase-core` repo — consult it read-only
  via the `/sase_repo` skill):
  - Overview: delta compares the preceding equal-length window; tiles open their detail view.
  - Runs: definitions for retry chains / attempts / kills, waiting, and "committing agents"; outcomes share is of
    finished runs.
  - Projects: Share = share of runs in range · Wall = summed agent runtime · Specs = distinct ChangeSpecs · ✓/× =
    completed/failed · meaning of "unattributed runs".
  - Providers: Success = completed ÷ finished runs · Avg runtime = mean of finished runs.
  - Runtime: p50/p95 = median / 95th-percentile runtime · Share = share of total runtime; fold the existing "In progress
    excluded" footnote into this legend.
  - Activity: Agents = distinct agents that used the item.
  - Plans & Questions: what "(all projects)" scope markers mean under a filter.
- **Actionable empty state.** "No agent runs recorded in <range label>." plus a guidance line: `t` widen the range, and
  — only when a project filter is active — `p` to clear the filter (naming the filtered project). Error panel gains
  "press r to retry".
- **Clickable tiles.** Wrap the five Overview tiles in a click-aware Static subclass that switches to the view
  explaining the tile (Agents Run / Success Rate / Commits → Runs; Plans Proposed / Questions → Plans & Questions)
  through the existing `_set_view` path. Purely presentational; keyboard flow is unchanged and the mapping is documented
  in the Overview legend and help overlay.
- Completeness is enforced by tests: every `VIEW_ORDER` entry must have a description and a legend.

### Statistics help overlay and keymap plumbing

- **New modal** (`statistics_help_modal.py`) opened by a new pane-local `help` action, default key `?` (verified free
  inside the Admin Center: app-level `?` is reverse-search on the main screen and does not reach this modal). Dismiss
  with `?`, `escape`, or `q`. Content is built entirely from already-loaded state and static copy — no I/O:
  1. **Views** — the seven labels with their `VIEW_DESCRIPTIONS` lines.
  2. **Controls** — every Statistics binding with its configured key and current value (range, group, filter), plus
     which views support grouping.
  3. **Metric glossary** — the union of the per-view legend definitions, organized by view.
  4. **Data & freshness** — statistics come from durable agent-run records via the Rust stats bindings; auto-refresh
     every 30 s while the tab is visible; exact range timestamps shown here. Match the established help-modal visual
     conventions (fixed-width bordered sections, Statistics accent border).
- **Keymap plumbing.** Add `help` to `ace.keymaps.statistics` in `src/sase/default_config.yml`, a field on
  `StatisticsPaneKeymaps`, and an entry in `_STATISTICS_BINDING_META` (`keymaps/types.py`) so
  `build_statistics_bindings`, `statistics_help_bindings`, config validation, and the hints line all pick it up
  automatically. Update the keymap default/validation tests.
- **Global help sync (required by ace conventions).** Update the "Admin Center: 1-7 jumps; Statistics [/] t/c/g/p/r"
  line in `help_modal/axe_bindings.py`, `help_modal/changespecs_bindings.py`, and `help_modal/agents_bindings.py` to
  include `?`. Statistics keys stay pane-local and out of the keybinding footer, per the footer convention.
- Add the `? help` entry to the pane hints line.

### Visual snapshot refresh and polish pass

- Regenerate the existing Statistics PNG goldens (overview, loading, empty, projects, projects_drilldown, runtime at
  120x40) via `just test-visual` with `--sase-update-visual-snapshots`, inspecting each actual/expected artifact —
  snapshot updates are accepted deliberately, not blindly.
- Add new goldens: the help overlay; one legend-bearing detail view (runs); one narrow capture (~90x30) proving compact
  degradation of chips and description.
- Final polish pass across all seven views: alignment, spacing, consistent `·` separators, chip/cap styling matching the
  Admin Center landing, no color-only meaning, no truncated legend text at 120 columns.
- Perf sanity: confirm no I/O crept into render paths and the auto-refresh soak test still passes; spot-check with
  `SASE_TUI_PERF=1` that pane interactions stay responsive.

## Testing

- **Unit** (`tests/ace/tui/test_statistics_pane.py` and new sibling modules where the file would grow unwieldy): chip
  builders render key/label/value per view (group chip dims on non-grouping views; filter chip reflects selection);
  every view has a description and a legend; empty-state guidance includes the filter hint only when a filter is active;
  tile→view click dispatch; hints line content.
- **Help overlay completeness guard:** a test asserting every `VIEW_ORDER` entry and every `_STATISTICS_BINDING_META`
  action appears in the overlay content, so future keys/views cannot silently go undocumented.
- **Pilot tests:** `?` opens and closes the overlay; overlay is inert on other Admin Center tabs; existing
  coalescing/selection tests keep passing unchanged.
- **Keymap tests:** defaults, validation, and duplicate-detection cover the new `help` action.
- **Visual:** snapshot suite per the visual-suite phase.

## Performance and architecture constraints

- Presentation-only: no changes to `sase/stats/*` queries or the Rust core; the sibling `sase-core` repo is consulted
  read-only (via `/sase_repo`) to verify metric wording. All new renderables are pure functions of already-loaded state.
- Preserve every existing guardrail: worker-thread loads, debounced scheduling, stale-result rescheduling, active-tab
  gating, thin timer callbacks, and the post-layout repaint. No new refresh code paths; header updates go through the
  existing `_update_static` path.
- The help overlay performs no disk or subprocess work on open.

## Risks and mitigations

- **Copy drifts from real semantics.** Mitigated by the verification requirement (principle 5) and the overlay
  completeness test; legends state definitions, not interpretations.
- **Header growth costs vertical space.** The redesign adds one net line (title + tabs + description + chips replaces
  title + range + tabs + always-on hints duplication); the narrow-width snapshot proves small terminals stay usable.
- **Snapshot churn.** All six existing Statistics goldens change; the visual-suite phase reviews each diff artifact
  explicitly.
- **Key conflicts.** `?` verified unbound within the Admin Center; the configurable keymap path lets users remap it, and
  duplicate-key validation catches clashes.
