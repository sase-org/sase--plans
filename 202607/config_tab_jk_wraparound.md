---
tier: tale
title: Cycle Config tab selection with j and k
goal: 'The SASE Admin Center Config tab wraps j navigation from the last visible config
  row to the first and k navigation from the first visible row to the last, without
  changing arrow-key behavior or slowing the keystroke path.

  '
create_time: 2026-07-22 07:51:07
status: wip
---

- **PROMPT:** [202607/prompts/config_tab_jk_wraparound.md](prompts/config_tab_jk_wraparound.md)

# Plan: Cycle Config tab selection with j and k

## Context and scope

The Config tab is a presentation-only Textual tree implemented by `ConfigPane` in
`src/sase/ace/tui/modals/config_pane_widget.py`. Today, `j` and Down both dispatch to the same native Tree cursor-down
action, while `k` and Up share cursor-up, so every key clamps at the first or last rendered row. The requested behavior
belongs in this pane rather than the shared Rust core: only the Vim-style `j`/`k` bindings should cycle, and the arrow
keys should retain Textual's existing non-wrapping behavior.

Treat the edge as the first or last row actually visible in the Tree. That keeps wrapping consistent when section rows
are expanded or collapsed and when filtering or modified-only mode rebuilds the list. Empty trees remain no-ops, and a
single visible row remains selected in either direction. No configuration loading, filtering, editing, or tab-level
navigation behavior changes.

## Config tree navigation

Update `ConfigPane` so `j` and `k` use pane-local cycling actions while Down and Up continue using the current ordinary
cursor actions. The cycling path should inspect Textual's current cursor line and visible last line: delegate interior
moves to the existing Tree actions, but at the lower or upper boundary move directly to the opposite rendered endpoint.
Use the pane's established cursor-movement/update path so the Tree highlight, scrolling, `_selected_path`, and detail
panel stay synchronized immediately after a wrap.

Keep this keystroke handler synchronous, in-memory, and constant-time. It must not rebuild or traverse the configuration
model, read configuration, schedule background work, or add another refresh path. Reuse the currently rendered Tree as
the authority so collapsed and filtered views cannot disagree with wrap boundaries. The existing `j/k: move` hint
remains accurate, so no copy, styling, or visual-snapshot update is needed.

## Regression coverage

Extend `tests/ace/tui/test_config_pane_widget.py` with focused behavioral coverage through the real Admin Center Config
tab:

- Assert the binding split: `j` and `k` target the new cycling actions, while Down and Up retain the non-wrapping cursor
  actions.
- Put the cursor on the final visible row, press `j`, and verify that both the selected path and Tree cursor land on the
  first visible row; repeat from the first row with `k` and verify the last row.
- Exercise Down at the bottom and Up at the top to prove arrow navigation still clamps rather than wrapping.
- Cover a constrained rendered tree (for example, a one-result filter) so cycling is stable when the visible list has a
  single row and remains based on displayed rows rather than the full schema.

These tests should also protect the existing detail-selection synchronization and make failures deterministic using the
module's patched config fixture.

## Validation

Before running repository checks, run `just install` as required for an ephemeral SASE workspace. Then run the focused
widget suite with `.venv/bin/pytest -q tests/ace/tui/test_config_pane_widget.py`, followed by the mandatory full
`just check`. No PNG golden changes are expected because this change does not alter the rendered Config tab.
