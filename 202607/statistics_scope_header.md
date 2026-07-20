---
tier: tale
title: Statistics scope bar and view descriptions
goal: Make the Statistics tab explain the selected view and its active range, grouping,
  and project scope through a stable, compact header without changing data loading
  behavior.
create_time: 2026-07-20 13:50:09
status: wip
prompt: 202607/prompts/statistics_scope_header.md
---

# Plan: Statistics scope bar and view descriptions

## Context

The Statistics pane currently spreads scope across a centered title suffix, a separate range line, and a long hints
footer. That makes current grouping and project-filter state difficult to discover, while the seven view tabs offer no
explanation of what each view contains. This phase will implement the first section of the Statistics redesign epic: a
clearer reading order of title and refresh status, view catalog, view description, visible scope controls, data, and
only the remaining keyboard hints.

The work is presentation-only. Existing statistics queries, composite results, project display snapshots, worker-thread
loading, refresh cadence, debouncing, stale-result rescheduling, tab activity gates, and post-layout repaint behavior
remain unchanged.

## Header structure and copy

- Add a complete `VIEW_DESCRIPTIONS` mapping beside `VIEW_LABELS`, with one concise description for every entry in
  `VIEW_ORDER`, and export it for the pane presentation layer.
- Restructure the composed header so `Statistics · <view>` is the stable left-hand identity and loading, error, or
  last-updated status is independently aligned at the right. Grouping and project-filter values will move out of the
  title and into the scope row.
- Keep the view strip directly below the title, then add a single-line `› <description>` caption using the Statistics
  accent and dim treatment established by the Admin Center landing/header language.
- Replace the old range row with a stable three-part scope bar for Range, Group, and Project. Each part will render its
  configured key as a reverse-video cap, a bold semantic label, and the current value in an appropriate accent.

## Scope behavior and responsive presentation

- Range will show the preset display label plus its absolute span; custom input will explicitly identify the scope as
  Custom while retaining the parsed display/span information.
- Group will show the active Runtime or Projects grouping label. On views that do not support grouping, it will remain
  present with a dimmed `—` value so switching views does not reflow the row.
- Project will show `All projects` or the display label from the already-loaded project snapshot. A selected project
  will include its stable categorical swatch/color without replacing the textual label.
- Build these values with pure Rich renderable helpers from the pane's existing in-memory state. Add bounded resize
  handling that only toggles compact presentation when crossing the narrow threshold: below roughly 100 columns the
  range value omits the absolute span while all three scopes remain identifiable and ellipsize rather than wrap.
- Update Statistics-specific TCSS for the split title, description, scope row/chips, and single-line overflow behavior
  while preserving the body and custom-range layout.
- Slim the footer to configured previous/next view keys, custom-range, and refresh only. Range, group, and project
  actions will be discoverable in their chips. Do not add the future `?` help entry or keymap plumbing in this phase.

## State updates and compatibility

- Route selection, loading, completion, and error repainting through the existing selective `_update_static` path so
  descriptions, scope values, and status stay synchronized without rebuilding the pane.
- Preserve configurable key display names in both scope caps and the remaining footer hints.
- Keep custom-range focus and submission behavior, project canonical-key submission, project display labels, grouping
  semantics, and click/keyboard view switching unchanged.

## Validation

- Update focused Statistics pane tests to assert every ordered view has a description, the description follows view
  changes, and title status is separated from scope values.
- Cover range preset and custom values, responsive compact range rendering, Runtime/Projects group values, dimmed
  unsupported grouping, all-project versus selected-project display, categorical swatch styling, and configured keymaps
  in chips and footer.
- Retain and adapt the existing interaction tests for coalesced range/group loads, project filtering, custom ranges,
  inactive-tab bindings, and auto-refresh responsiveness, ensuring the new render helpers perform no I/O.
- Run focused unit tests while iterating, then install the current workspace dependencies and run the
  repository-mandated `just check`. If intentional Statistics PNG differences are exercised by that check, update only
  the affected Statistics goldens needed to keep the suite passing; the epic's later visual-suite phase remains
  responsible for the dedicated narrow capture, comprehensive snapshot refresh, and final polish pass.

## Boundaries

This phase does not add metric legends, actionable empty/error-state guidance, clickable overview tiles, the Statistics
help overlay, new keymap actions, or query/backend changes. Those remain owned by later phases of the parent epic.
