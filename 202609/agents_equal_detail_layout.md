---
tier: tale
title: Add an equal-height Agents detail layout
goal:
  The Agents view picker supports a reliable three-layout cycle with direct equal-height
  selection, reverse cycling, and a 0 metadata-only shortcut.
size: medium
proposed_by: bbugyi200.apollo.0e.f0
create_time: 2026-09-18 05:46:51
status: wip
---

# Plan: Add an equal-height Agents detail layout

## Goal

Extend the Agents tab's `p` view/layout picker from two saved vertical layouts to three.
The metadata and File/LLM Calls panels must support metadata-larger (70/30),
equal-height (50/50), and secondary-larger (30/70) layouts. Users must be able to choose
each layout directly or cycle in either direction, while the metadata-only view moves
from `n` to `0`.

This is presentation-only TUI behavior. Keep it in the Python/Textual frontend; do not
add Rust-core state or persistence. The chosen layout remains session-local and shared
between the File and LLM Calls views, as it is today.

## Interaction contract

- Keep the configurable app-level `choose_agent_view` opener unchanged (default `p`).
- Inside `AgentViewModal`, use these literal, picker-local keys:
  - `f`: show File.
  - `t`: show LLM Calls.
  - `0`: hide the secondary panel and show metadata only. Remove `n` as a picker
    shortcut; it must be contained by the modal without applying a choice.
  - `1`: metadata larger, at 70% metadata / 30% File or LLM Calls.
  - `=`: equal height, at 50% / 50%.
  - `2`: File or LLM Calls larger, at 30% metadata / 70% secondary.
  - lowercase `p`: apply the next layout.
  - uppercase `P`: apply the previous layout.
- Define the circular order once as metadata-larger -> equal -> secondary-larger ->
  metadata-larger. Reverse cycling must be the exact inverse. With the existing
  secondary-larger default, default `pp` therefore still moves to metadata-larger;
  default `pP` moves to equal.
- Render separate "Next layout" and "Previous layout" rows with subtitles that name the
  concrete transition from the saved layout. Preserve the existing Current/Saved badges
  and disabled explanations.
- Direct layout selection and both cycle directions are disabled whenever there is no
  visible File/LLM Calls panel to resize. They must retain the current warning/no-op
  behavior for metadata-only, missing-file, summary/container, and historical-attempt
  states.
- Treat `p` and `P` as distinct modal input. Do not lowercase the event before routing
  these two choices, and keep explicit modal bindings so the app-level opener cannot
  steal the second chord key.
- Keep the initial saved layout as secondary-larger. Switching among File, LLM Calls,
  and metadata-only views must preserve whichever of the three layouts was last saved.
- When `Z` is pressed from an equal layout, open the currently selected secondary view
  (File or LLM Calls), because neither pane is dominant and that view is the user's
  explicit selection. Continue opening metadata for metadata-larger and metadata-only
  layouts.

## Implementation

### 1. Represent and render three layout states

- In `src/sase/ace/tui/widgets/_agent_detail_panels.py`, add an equal member to
  `DetailLayoutMode` and define the forward/reverse order in one reusable place.
- Replace the two-state `_layout_swapped` storage in `AgentDetail` with an explicit
  `DetailLayoutMode` value. Make `detail_layout_mode`, direct setting, and directional
  cycling operate on that value; avoid deriving a three-state model from booleans.
- Update all layout application and panel-mode transitions to remove stale
  metadata-larger/equal/secondary sizing classes before applying the saved layout.
  Metadata-only and missing-secondary paths must still expand metadata to 100%.
- Add Textual CSS for the equal state so both visible scroll containers receive equal
  flexible heights. Apply the same rule to the normal metadata scroll and the search
  metadata scroll, and to both File and LLM Calls secondary scrolls.
- Replace or narrow `is_layout_swapped()` and its callers so zoom targeting asks the
  explicit layout mode. Preserve existing metadata-larger and secondary-larger zoom
  behavior, and implement the equal-layout rule from the interaction contract.
- Update test doubles and focused panel-mode tests that currently seed or expose
  `_layout_swapped`.

### 2. Extend the picker model and case-sensitive modal routing

- In `src/sase/ace/tui/actions/agents/_agent_view_picker.py`, build the direct choices
  for `1`, `=`, and `2`, including the equal split's labels, percentages, badges, and
  the active File/LLM Calls name.
- Replace the binary "swap" result/application path with a directional layout-cycle
  result. Generate next/previous subtitles from the same canonical ordering used by the
  detail widget so the UI copy cannot drift from behavior.
- Change the metadata-only choice and selected-key mapping from `n` to `0`.
- In `src/sase/ace/tui/modals/agent_view_modal.py`, add explicit direct bindings for
  `0`, `=`, lowercase `p`, and uppercase `P`; update the result model and route exact
  case before any case-insensitive handling. Keep keyboard navigation, mouse selection,
  printable-key containment, one-shot dismissal, and disabled-row feedback intact.
- Adjust the modal's list height/style only if the two added layout rows would clip at
  supported terminal sizes; preserve the established visual grammar rather than
  redesigning the modal.

### 3. Align discovery surfaces and documentation

- Update the Agents help modal to describe the picker keys and the default opener chords
  (`pp` next and `pP` previous), while making clear that the second key is local to the
  picker.
- Update the `choose_agent_view` command metadata/keywords where useful so Equal, next,
  and previous layout behavior is discoverable without creating new app-level keymap
  settings.
- Update `docs/ace.md` and `docs/configuration.md` everywhere they currently advertise
  `n`, two layouts, or "swap sizes." Document `0`, `1`/`=`/`2`, the circular order, the
  two cycle directions, the unchanged session-local/default behavior, and the
  disabled-layout cases.
- Sweep Agents-specific tests and visual workflows for old picker interactions. Change
  `p n` to `p 0`, and change any remaining bare `p` that intended the former layout
  toggle to an explicit picker chord or direct layout choice. Do not alter unrelated
  uses of `n`, `p`, `P`, or `=` on other tabs/modals.

## Tests and visual verification

- Extend `tests/ace/tui/modals/test_agent_view_modal.py` to cover `0`, `=`, distinct
  lowercase/uppercase `p` routing, disabled next/previous rows, navigation, click, and
  confirmation that `n` no longer selects metadata-only.
- Extend `tests/ace/tui/test_agent_view_picker.py` (and add focused widget tests if that
  keeps responsibilities clearer) to cover:
  - direct selection of all three layouts;
  - forward and reverse cycles through all states, including wraparound;
  - default `pp` and `pP` behavior with a real visible File panel;
  - saved equal layout surviving File/LLM Calls/metadata-only view changes;
  - clean class transitions with no stale equal or priority sizing classes;
  - unchanged rejection behavior when layout controls are unavailable;
  - equal-layout `Z` targeting the active File or LLM Calls view.
- Add or update deterministic Agents PNG coverage for both the expanded picker (showing
  the complete revised key grammar) and an applied 50/50 split with representative
  metadata plus File or LLM Calls content. Inspect generated image diffs before
  accepting golden updates.
- After focused automated coverage, capture the live Agents UI with `sase screenshot` at
  a practical terminal size and inspect the PNG to confirm the two panes are visually
  equal, picker rows are not clipped, and Current/Saved/disabled treatments remain
  legible.

## Validation

1. Run the formatter for touched Python, Markdown, YAML, and TCSS files using the
   repository's standard formatting command.
2. Run the focused modal, picker, panel-layout, zoom-target, help, and affected Agents
   visual tests.
3. Run `git diff --check` and a targeted search for stale Agents picker copy such as
   `p n`, `` `n` none ``, and "swap sizes"; review hits in context so unrelated uses are
   left alone.
4. Run `just check` as the repository-wide required gate. If it escalates because of
   visual assets or other touched paths, let the escalated suite complete.

## Non-goals

- Do not make the detail layout persistent across TUI sessions.
- Do not add configurable app-level actions for next/previous layout; `p`, `P`, and `=`
  are literal keys owned by the open picker.
- Do not change the view picker opener, the Agents grouping picker, bracket behavior, or
  layout controls on other tabs and modals.
- Do not change the 70/30 proportions of the two existing layouts.
