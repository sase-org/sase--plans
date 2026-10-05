---
tier: tale
title: Services j/k stay inside the focused nav section
goal:
  On the Services tab, j and k cycle nav items inside the focused nav section only, the
  same way the Agents tab cycles inside the focused tribe panel. J and K stay the keys
  that move between nav sections.
size: medium
proposed_by: bbugyi200.athena.0wq
create_time: 2026-10-05 09:06:30
status: wip
---

# Plan: Services j/k stay inside the focused nav section

## Goal

On the Services tab, `j` and `k` (`next_patch` / `prev_patch`) must cycle the selectable
rows of the focused nav section and wrap inside that section. They must not step into
another section. `J` and `K` (`focus_next_service_panel` / `focus_prev_service_panel`)
already move between sections and must keep doing that.

A nav section here is one Services sidebar panel, in `SERVICES_PANEL_ORDER`: Service
Procs (`service_procs`), User Routines (`user_routines`), Plugin Routines
(`plugin_routines`), and Builtin Routines (`builtin_routines`). That is the same split
`J` / `K` already use. Do not collapse the three routine panels into one section. The
glossary's "Scheduled Routines" wording is the older name for those panels; the live
section boundary is the panel key.

## Current behavior

`action_next_patch` and `action_prev_patch` in
`src/sase/ace/tui/actions/navigation/_basic.py` branch three ways:

- Artifacts calls `_navigate_patch_panel`.
- Agents calls `_navigate_agents_panel`, which builds stops for the focused tribe panel
  only and wraps with `(pos + direction) % len(stops)`.
- Everything else is the Services path. It treats `_axe_items` as one flat list: `j`
  does `current_idx + 1`, and at the last row wraps to `0`. `k` does the reverse. A
  keystroke on the last row of Service Procs therefore lands on the first row of the
  next non-empty routine panel.

`J` / `K` are already correct. `AxePanelNavigationMixin._change_focused_service_panel`
in `src/sase/ace/tui/actions/axe_display/_panel_navigation.py` uses
`ServicesPanelIndex.adjacent_nonempty_panel`, lands on `first_global` going forward and
`last_global` going backward, skips empty panels, wraps the panel order, pushes an
entry-jump origin, and no-ops off the Services tab or when no other panel has rows.
Leave that method's behavior alone.

`_axe_items` is already the visible nav-item list. `AxeItemLoaderMixin._build_axe_items`
omits folded-away jobs and hidden oneshots, then rebuilds `_axe_panel_index` with
`build_services_panel_index`. The `── oneshots ──` divider is chrome drawn inside the
first oneshot option in `BgCmdList.update_list`; it is not its own index. Empty
placeholders are disabled options and are not in `_axe_items`. Cycling a panel slice's
`global_indices` is therefore exactly cycling that section's nav items. Do not add a
second fold or hidden-oneshot filter.

Arrow keys are not this bug. `up` / `down` are the focused `BgCmdList`'s OptionList
cursor and already stay inside that widget; they stop at the ends and do not wrap. Do
not bind them to `next_patch` / `prev_patch`, and do not make them wrap or cross panels.

Agents' extra `j` / `k` rule does not apply here. On Agents, a selected whole panel
(`❖`) or a panel with at most one stop makes `j` / `k` cycle whole panels
(`_change_whole_panel_focus`, `_escape_dead_end_panel_focus`). Services has no
whole-panel selection. A one-item Services section stays on that item. Crossing sections
is only `J` / `K`.

## Implementation

Replace only the Services arm of `action_next_patch` and `action_prev_patch`. Keep the
Artifacts and Agents arms as they are. Make the Services check explicit
(`current_tab == "services"`) instead of an `else` that means "axe". Both actions
already call `_jk_perf_begin("next"|"prev")` and `_record_jk_navigation` before the
branch; keep those names. Do not reuse the `next_service_panel` / `prev_service_panel`
perf names, which belong to `J` / `K`.

Add one helper, `_navigate_services_panel(self, direction: int) -> None`, next to the
Services arm (or on the Services panel-navigation mixin if that keeps `_basic.py` from
importing the panel index at runtime in a cycle — either place is fine if the call stays
inside the Services arm). Behavior:

1. If `_axe_items` is empty, return. Same no-op as today.
2. Read `_axe_panel_index`. It is initialized to an empty index in state init and
   rebuilt whenever items are built. If it is missing, return. Do not fall back to the
   old global walk.
3. Resolve the section with `panel_for_global(current_idx)`. An index that is not in any
   slice falls back to `service_procs`, which is the existing helper contract.
4. Take that slice's `global_indices` (visual order). If the slice is empty, return. Do
   not hop to another panel.
5. If `current_idx` is in that list, set `current_idx` to
   `indices[(pos + direction) % len(indices)]`. Direction `+1` is `j`, `-1` is `k`.
   Wrapping from the last row returns to the first row of the same section, and wrapping
   from the first row returns to the last row of the same section.
6. If `current_idx` is not in the slice, `j` selects `indices[0]` and `k` selects
   `indices[-1]`. That is a one-step correction for a stale cursor, not a cross-section
   move.
7. A one-item slice wraps onto itself. Leave `current_idx` unchanged in that case so the
   index watcher does not repaint.
8. Drive the move only by assigning `current_idx`. The existing watcher repaints both
   panels' highlights. Do not post `BgCmdList.SelectionChanged`, and do not call
   `_push_entry_jump_index_origin_if_changed`. Entry-jump origins stay a `J` / `K`
   concern.

Do not change keymap defaults. `j` / `k` remain `next_patch` / `prev_patch`. `J` / `K`
remain `focus_next_service_panel` / `focus_prev_service_panel`. Do not change
availability, the shared `next_patch` command-catalog label ("Next entry" is all-tabs),
or the Agents / Artifacts help copy.

## Copy

Update the Services-only sentences so they describe section cycling:

- `docs/ace.md`, Services "Navigation" table: `j` / `k` cycle the next / previous nav
  item in the focused panel (service proc, oneshot, routine, or job) and wrap inside
  that panel. `J` / `K` stay "jump into the first / last row of the next / previous
  panel".
- `src/sase/ace/tui/modals/help_modal/axe_bindings.py`: replace "Move to next / previous
  command" with a focused-panel wrap description. Leave the existing "Jump into next /
  prev panel" row for `J` / `K`.
- `src/sase/ace/tui/widgets/axe_onboarding.py`: the `next_patch` / `prev_patch` sentence
  currently says "move through the sidebar." Say that those keys cycle rows in the
  focused panel. The following `J` / `K` sentence already says they jump panels; keep
  it.

Do not edit glossary memory. `glossary:nav-item` and `glossary:nav-section` already
define `j` / `k` as movement among a section's nav items, and the oneshot divider as
chrome rather than a section.

## Tests

Add unit tests beside `tests/ace/tui/test_axe_service_panel_jumps.py`, using the same
minimal app surface (`current_tab`, `current_idx`, `_axe_items`, `_axe_panel_index` from
`build_services_panel_index`, plus the perf and navigation recorders `action_next_patch`
already calls). Cover:

- `j` on the last Service Procs row (including a trailing oneshot) wraps to the first
  Service Procs row and does not enter a routine panel.
- `k` on the first Service Procs row wraps to the last Service Procs row, not the last
  routine row.
- `j` and `k` inside a routine panel wrap inside that panel, including its job rows, and
  do not enter Service Procs or a sibling source panel (user vs plugin vs builtin).
- A one-item panel: both keys leave `current_idx` unchanged.
- An empty `_axe_items` list is a no-op.
- A cursor that is not in any slice snaps forward to the first Service Procs row and
  backward to the last, when that slice is non-empty.
- `j` / `k` do not push the Services entry-jump stack. `J` / `K` tests in
  `test_axe_service_panel_jumps.py` must still pass unchanged.
- Artifacts and Agents `action_next_patch` / `action_prev_patch` behavior is untouched.
  Do not rework those tests.

Then fix Services-tab tests that press `j` or `k` and expect the flat list to cross a
panel boundary. Intra-panel presses stay valid. The entry-editor modal test
`tests/ace/tui/test_axe_entry_editor_modal.py` uses `j` / `k` inside the property sheet;
leave it.

Known cross-panel cases to retarget, landing on the same row via `J` or `K` and then `j`
/ `k` only inside the destination panel:

- `tests/ace/tui/visual/test_ace_png_snapshots_services_panels.py::test_services_panels_routine_selected_png_snapshot`
  presses `j` seven times from Service Procs into User Routines and expects
  `current_idx == 7`.
- `tests/ace/tui/visual/test_ace_png_snapshots_axe_runs.py` (`_select_first_chop` and
  the five-`j` chop-run snapshot). Those fixtures start on a oneshot in Service Procs;
  the routine rows are the next panel because a missing routine origin falls back to
  User Routines.
- Any other `page.press("j")` / `page.press("k")` under
  `tests/ace/tui/visual/test_ace_png_snapshots_axe*.py` and
  `test_ace_png_snapshots_axe_descriptions.py` / `test_ace_png_snapshots_axe_layout.py`
  whose first rows are a different panel from the row the assertion selects.

Search those files before editing. If a press sequence already stays inside one panel,
leave it. Prefer keeping the resulting selection identical so existing PNGs still match.
Do not regenerate a snapshot unless the frame actually changes; if a PNG fails, fix the
key sequence first.

## Verification

Run the new unit tests, `tests/ace/tui/test_axe_service_panel_jumps.py`, and
`tests/ace/tui/test_service_panel_keymaps.py`. Run any non-visual Services navigation
test you had to edit. Visual PNG tests are optional unless a key-sequence edit is in a
file you can run; do not regenerate goldens to make a wrong landing pass.

## Out of scope

- Agents and Artifacts `j` / `k`, including Agents' whole-panel and dead-end cycling.
- `J` / `K` panel-jump landing, wrap, empty-panel skip, and entry-jump origins.
- Arrow keys, mouse selection, `g` / `G`, and output scrolling.
- Fold, hide-oneshot, or panel-membership rules.
- Glossary or other memory edits.
- Rebinding keys or renaming `next_patch` / `prev_patch`.
