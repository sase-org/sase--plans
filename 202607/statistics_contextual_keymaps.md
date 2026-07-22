---
tier: tale
title: Context-aware Statistics controls and reverse project filtering
goal: 'The Admin Center Statistics tab advertises grouping only on views that can
  use it and lets users traverse the project filter in either direction with configurable
  p/P controls.

  '
create_time: 2026-07-22 07:44:53
status: wip
---

- **PROMPT:** [202607/prompts/statistics_contextual_keymaps.md](prompts/statistics_contextual_keymaps.md)

# Plan: Context-aware Statistics controls and reverse project filtering

## Context

The focused Statistics pane owns a configurable keymap scope spanning its eight views. Its range, grouping, and
project-filter controls are rendered in the scope row, and the same keymap metadata feeds contextual and global help.
Grouping is currently implemented only for Projects and Runtime, but the `g` control is still rendered as `Group —` on
every other view. Project filtering currently walks the cached project ranking only in the forward direction with `p`,
while retaining the selected filter across range/view changes and clearing an active filter directly when that filtered
result is empty.

Keep both changes on the existing fast path: key handlers should inspect only mounted/in-memory state, mutate the
selection synchronously, and reuse the existing debounced, worker-backed statistics reload. No project discovery, query,
file access, or other blocking work should be added to a keystroke or render path.

## User experience and behavior

- Treat Projects and Runtime as the authoritative set of grouping-capable Statistics views. Show the configured group
  control and current grouping value only on those two views; hide its scope-row widget on the other six views so the
  remaining range/project controls reclaim the space. Switching views must refresh that visibility immediately. Direct
  invocation of the group action on an unsupported view remains a harmless no-op.
- Add a configurable `cycle_project_filter_reverse` Statistics action with the default key `P`. Display the effective
  forward/reverse keys together anywhere the active project-filter control is advertised, matching the existing `t/T`
  range presentation and honoring user overrides.
- Preserve the current project cycle `(All projects, ranked projects...)`: `p` moves forward and wraps, while `P` moves
  backward and wraps. From All, `p` selects the first ranked project and `P` selects the last. Both directions use the
  project ordering cached from the latest unfiltered result, are no-ops when there are no choices, retain the filter
  across range/view changes, and schedule the same coalesced reload after a real selection change.
- Preserve the empty-result recovery rule in both directions: when the current loaded result is empty for the active
  project, either project-cycle key clears directly to All projects instead of stepping into another filtered result.
- Make contextual Statistics help view-aware: omit the group action while help is opened from a view that cannot group,
  show it with the current value on Projects/Runtime, and document both project-filter directions. Keep the
  application-wide help summary as a compact inventory of the complete Statistics key set, updated to include `p/P`.

## Implementation

1. Extend the focused Statistics keymap contract end to end. Add the reverse project action to `StatisticsPaneKeymaps`
   and the ordered Statistics binding metadata, give it a `P` default in `src/sase/default_config.yml`, and expose it in
   `src/sase/config/sase.schema.json`. This keeps binding construction, validation, duplicate detection, overrides, and
   help-key display consistent with the other scoped actions.
2. Refactor project traversal in `statistics_pane.py` behind one directional helper used by forward and reverse action
   methods. Reuse the existing cached option fallback, stale-selection handling, empty-filter escape hatch, and
   `_selection_changed(reload=True)` path so the new action cannot diverge from `p` or bypass worker/debouncer
   safeguards.
3. Define one reusable view-capability predicate/constant for grouping support and use it across dispatch and
   presentation. Update the scope row during initial composition and every view change so the group widget is mounted
   but participates in layout only for Projects/Runtime. Render the project scope and filtered empty/unavailable
   recovery guidance with both configured project-cycle keys.
4. Update `StatisticsHelpModal` to filter its control rows using the current view capability while still deriving
   keys/descriptions from the canonical metadata. Add the reverse-project current-value explanation, and update the
   three main help-panel summaries to advertise `p/P` without hard-coding a second behavior path.
5. Update the public keymap examples and behavior descriptions in `docs/configuration.md`, `docs/telemetry.md`, and
   `docs/ace.md`: list the new configurable action/default, explain forward/reverse ordering and wrapping, retain the
   empty-result clearing rule for either direction, and clarify that the visible group control is limited to Projects
   and Runtime.

## Compatibility and boundaries

- Existing user configurations remain valid: an omitted reverse action receives the bundled `P` default through the same
  scoped-keymap loader used today.
- Do not change project ranking, statistics query inputs, Rust bindings, or filter persistence. This is a pane-level
  traversal and presentation change, and `p` must retain its current forward behavior.
- Keep both project actions focused to the Statistics pane; neither key should become active on other Admin Center tabs
  or at application scope.

## Verification

- Extend keymap default, registry override/duplicate, schema, binding dispatch, help-summary, and effective-key
  rendering tests to cover `cycle_project_filter_reverse` and custom forward/reverse mappings.
- Add interaction coverage for forward and backward traversal, wraparound from both ends, cached ordering across
  reload/range changes, no-options no-ops, and the one-key empty-result escape in either direction. Assert every
  selection reaches the existing load call with the canonical project key.
- Add exhaustive view coverage proving the group control is visible and active only on Projects and Runtime, disappears
  immediately elsewhere, preserves each supported view's grouping state, and does not introduce unnecessary statistics
  reloads. Cover contextual help from both supported and unsupported views.
- Run the focused Statistics/keymap/config/help tests first. Then run the Statistics PNG snapshot suite, inspect
  actual/expected/diff artifacts, and update only goldens whose scope row or help copy intentionally changed.
- Because implementation changes files in this repository, run `just install` before verification and finish with
  `just check` as the required full repository validation.
