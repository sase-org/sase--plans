---
tier: tale
title: Statistics sub-tab description rail
goal:
  Give every SASE Admin Center Statistics sub-tab an accurate, responsive, and visually
  polished orientation caption without reducing chart space or changing statistics
  behavior.
size: medium
proposed_by: bbugyi200.athena.0a2.f1
create_time: 2026-08-21 20:07:23
status: wip
---

# Plan: Statistics sub-tab description rail

## Outcome

Turn the existing one-row Statistics description into the same polished orientation rail
used by the Config catalog. The rail will sit directly beneath the eight-view tab strip,
explain the selected view in concise action-oriented language, and switch to a
hand-authored compact sentence whenever the full copy does not fit. It will be centered,
accent-colored, non-focusable, and visually associated with the nested Statistics strip
while remaining distinct from the top-level Admin Center caption.

The Statistics pane already reserves this row as `#statistics-description`, so this work
must not add height or take space from tiles, tables, legends, scrolling content, scope
controls, or hints. It is a presentation-only Python/Textual change: do not alter query
semantics, range/project/XPrompt/group filters, lazy loading, refresh cadence,
worker/debouncer behavior, navigation bindings, session state, or the Rust core
boundary. No feature flag is needed because the completed treatment is intended to ship
as the normal Statistics presentation rather than as an early or compatibility path.

## Copy and visual language

Make the ordered Statistics view catalog the source of truth for full labels, compact
and micro labels, and both description variants. Use this reviewed copy:

| Sub-tab           | Full description                                                                       | Compact description                                 |
| ----------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------- |
| Overview          | Scan run volume and outcomes, commits, plans, questions, and trends at a glance.       | Scan run outcomes, work totals, and trends.         |
| Runners           | Track runner concurrency, occupancy, idle time, peaks, and today's global limit.       | Track occupancy, peaks, idle time, and limits.      |
| Projects          | Compare run outcomes, commits, Patches, and wall time across projects.                 | Compare outcomes, Patches, and wall time.           |
| Providers         | Compare provider, model, and effort usage, success rates, and average runtime.         | Compare model usage, success, and runtime.          |
| Activity          | See which skills, memories, and workspaces agents use most.                            | See top skills, memories, and workspaces.           |
| XPrompts          | Explore XPrompt adoption, model and project breakdowns, pairings, and focused details. | Explore XPrompt usage, pairings, and focus.         |
| Plans & Questions | Review plan decisions, epic structure, and how agents ask for clarification.           | Review plan outcomes, epic shape, and questions.    |
| Perf              | Assess TUI responsiveness, launch and agent latency, stalls, and data health.          | Assess responsiveness, latency, stalls, and health. |

Render the chosen sentence as `› <description>` in the Statistics magenta accent. Match
the Config rail's clean one-line treatment: centered, without a border or extra icon,
and no dimming that makes the guidance look disabled. Select the full sentence whenever
its Rich terminal-cell width fits the rail; otherwise select the compact sentence.
Retain single-line ellipsis only as a last-resort fallback below the established narrow
viewport. Never wrap the rail, because wrapping would shift the scope and data surfaces
as the user moves between views.

The Help modal should continue to show the full descriptions, not the width-dependent
compact variants. Copy must describe what the view actually measures: Runners compares
historical occupancy with today's effective global limit; Projects includes attributed
Patches and summed wall time; Activity covers skills, memories, and workspaces; XPrompts
covers usage, breakdowns, pairings, and focused detail; and Perf is global rather than
project-scoped.

## Implementation

1. In `src/sase/ace/tui/modals/statistics_pane_data.py`, replace the parallel hand-kept
   view metadata dictionaries with one immutable ordered `StatisticsViewSpec` catalog
   containing the view ID, full/compact/micro labels, and full/compact descriptions.
   Derive `VIEW_ORDER` and any compatibility mappings still consumed by renderers,
   tests, and `StatisticsHelpModal` from that catalog so there is one authoritative row
   per view. Preserve the current eight-view order and all existing labels/shortcuts.

2. Add a small presentation helper/widget for the Statistics rail and use it from
   `statistics_pane_layout.py` in the existing `#statistics-description` position. The
   widget should accept the active immutable spec, render Rich `Text` with the existing
   Statistics accent, and choose full versus compact copy using `Text.cell_len` and its
   actual laid-out width. It must set `can_focus = False`, keep markup disabled, and
   repaint on resize only when the selected variant changes. Keep all measurement and
   rendering in memory: no file access, subprocesses, data reloads, or focus mutations
   are allowed on resize.

3. Update the existing view-chrome path so every successful view selection changes the
   heading, active tab, rail, scope controls, hints, and body from the same `_view`
   source in one event-loop turn. This must cover bracket cycling, numbered selection,
   tab clicks, and Overview tile click-through. Preserve the current cached composite
   result behavior and the special Perf reload: changing a description must never start
   a worker, while an old or failed worker result must never restore stale copy.
   Returning to Statistics after another Admin Center tab must retain the mounted view
   and its matching rail.

4. Refine `src/sase/ace/tui/styles.tcss` for the existing rail: keep its one-row height
   and full width, center the caption, prohibit wrapping, and bound overflow. Do not add
   margins or move the scope row. Verify the rail remains legible against both the
   nested active-tab magenta and the cyan/purple top-level Admin Center chrome, and that
   the body scroll region retains its current usable height at 120x40 and 90x30.

## Tests and acceptance criteria

Extend the focused Statistics presentation and interaction tests (primarily
`tests/ace/tui/test_statistics_scope_header.py`,
`tests/ace/tui/test_statistics_pane_interactions.py`, and
`tests/ace/tui/test_statistics_help_modal.py`) to prove:

- the catalog contains exactly the eight ordered views, every view has the exact
  reviewed full/compact copy, and all exported order/label/description helpers are
  derived from that catalog;
- full copy is chosen at a fitting width, compact copy is chosen below the measured
  boundary, and Unicode terminal-cell width rather than Python character count governs
  the decision;
- the initial Overview view and every bracket, numbered, mouse-tab, and Overview-tile
  navigation path leave the heading, active tab, rail, and `_view` in agreement;
- fast navigation across a pending Perf load, load success, load failure, periodic
  refresh, and hiding/showing the parent Admin Center tab cannot replace the active
  view's rail with stale text or trigger work solely for the description;
- repeated resize events inside the same full/compact tier do not repaint the rail,
  crossing the threshold repaints it exactly once, and resizing does not repaint the
  Statistics body unless one of its existing runners/perf/scope breakpoints also
  changes;
- the rail cannot receive focus and does not interfere with the custom-range input,
  filters, scrolling, numbered selection, help, or any configured Statistics binding;
  and
- `StatisticsHelpModal` lists every view with the canonical full description regardless
  of the pane's current width.

Regenerate affected PNG goldens only through the repository's
`--sase-update-visual-snapshots` flow; never hand-edit PNGs. Existing Overview, Runners,
Projects, XPrompts, and Perf wide snapshots should show the full centered rail. Existing
Overview, Runners, XPrompts, and Perf 90-column snapshots should show compact copy with
no awkward truncation. Add 120x40 visual cases for Providers, Activity, and Plans &
Questions so all eight descriptions receive direct visual coverage, and rerun the Help,
loading, empty, focused/grouped, and degraded-state goldens whose underlying Statistics
surface or full Help copy changes.

Accept the visual result only when both Admin Center hierarchy levels remain obvious,
the selected nested tab and rail read as one unit, every standard-width sentence is
complete, every established narrow-width sentence is complete, the scope row stays
immediately below the rail, and no tile, table, legend, hint, or scrollable body loses a
row.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run the focused catalog/description, interaction, Help, resize, loading, and binding
   tests while iterating.
3. Regenerate the intentional Statistics PNG changes with
   `--sase-update-visual-snapshots`, rerun the Statistics visual module normally, and
   inspect the resulting wide and narrow images/diffs.
4. Run `just check`. If selection broadens/escalates or reports unusual coverage, run
   `just check-full` only through `/sase_monitor`, following repository policy.
