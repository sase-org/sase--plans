---
tier: tale
title: Focus-independent custom gate command keymaps
goal: 'Custom notification gate command lists respond immediately to j, k, and Space
  without a preliminary Tab, while preserving outcome selection, feedback entry, and
  the submitted command set.

  '
create_time: 2026-07-17 10:21:14
status: wip
prompt: 202607/prompts/custom_gate_command_keymaps.md
---

# Plan: Focus-Independent Custom Gate Command Keymaps

## Context

`CustomGateModal` renders one choice panel per terminal outcome. A panel may contain a `GateExtrasSelectionList` whose
rows represent optional add-on commands. The modal deliberately focuses the first outcome button on mount, and moving
between outcomes returns focus to the newly active outcome button. Textual's selection list only handles its toggle key
while that list owns focus, while the modal currently has no `j` / `k` forwarding actions. As a result, the advertised
command-selection workflow is inert until the user tabs into the list (and Vim-style row movement is not consistently
available through the modal at all).

This is presentation-only Textual behavior in `src/sase/ace/tui/modals/custom_gate_modal.py`; command execution,
notification-gate envelopes, the Rust core, global/default keymap configuration, and tracked-task submission should not
change. The handlers must remain synchronous and in-memory so the keystroke path adds no I/O, refresh, timer, or event
pump work.

## Product Behavior

- From the initially focused outcome button, `j` moves the highlight to the next add-on command in the active choice
  panel, `k` moves it to the previous command, and `space` toggles the highlighted command.
- The first command keypress also transfers focus to the active command list, giving the user the normal focused-list
  highlight without requiring a separate Tab. The keypress itself must still perform the requested move or toggle.
- The actions always target the currently selected choice's visible command list. Switching outcomes with `left` /
  `right` and then pressing a command key must never mutate a hidden previous panel.
- A choice with no add-on commands treats these keys as safe no-ops and retains all existing outcome, submit, cancel,
  scroll, and debug behavior.
- When a feedback `Input` owns focus, printable `j`, `k`, and spaces remain feedback text and do not navigate or toggle
  commands. The modal bindings should rely on normal Textual focus/event precedence rather than priority bindings or a
  global key handler that would steal input.
- Default-selected command IDs and stable envelope order remain unchanged, and submission continues to return exactly
  the selected IDs for the active choice.

## Implementation Outline

1. Add modal-local bindings on `CustomGateModal` for next command, previous command, and toggle command.
   - Use ordinary non-priority screen bindings so focused text inputs keep ownership of printable characters.
   - Keep the shortcuts local to this modal; do not add app command-catalog entries or fields in
     `src/sase/default_config.yml`.

2. Centralize lookup/focus of the active choice's `GateExtrasSelectionList`.
   - Resolve the list by the current choice index only when that choice actually declares extras.
   - Return a handled no-op when no list exists.
   - Focus the resolved list without forcing the outer review pane to jump, then delegate to Textual's existing
     `action_cursor_down()`, `action_cursor_up()`, and `action_select()` behavior. Reuse the widget's current highlight
     and selection state rather than introducing parallel cursor state.

3. Refresh the custom-gate footer hint so it advertises `j` / `k` command navigation and no longer implies that Tab is a
   prerequisite. Keep the wording compact enough for the existing one-line footer and avoid layout/style changes.

4. Extend `tests/ace/tui/test_custom_gate_modal.py` with pilot-driven regression coverage.
   - Start from the modal's real initial button focus, press `j` / `k` / `space`, and assert focus, highlight movement,
     toggled IDs, and the eventual `CustomGateModalResult` without pressing Tab.
   - Exercise multiple outcomes so the shortcuts operate only on the active panel, including a choice without extras.
   - Focus a feedback input and type `j`, `k`, and a space to prove those characters are inserted and the command
     selection is unchanged.
   - Preserve the existing required-feedback, default-selection, choice-navigation, and submission assertions.

5. Update only the intentional custom-gate PNG goldens affected by the footer copy, inspect their diffs, and rerun the
   focused visual suite without update mode. No visual change to choice/list focus styling is expected in the initial
   snapshot because the modal should retain its existing mount focus until a command key is pressed.

## Validation

1. Run the focused interaction tests:

   ```bash
   just test -- tests/ace/tui/test_custom_gate_modal.py tests/ace/tui/test_gate_debug_modal.py
   ```

2. Regenerate and review only the custom-gate visual snapshots if the footer assertion fails, then revalidate them in
   normal comparison mode:

   ```bash
   just test-visual -- --sase-update-visual-snapshots tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py
   just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py
   ```

3. Per repository policy, run `just install` before the final repository-wide check, then run:

   ```bash
   just check
   ```

## Risks and Guardrails

- Textual resolves focused-widget bindings before ordinary screen bindings. Pilot tests must prove that the modal owns
  the keys from buttons while `Input` continues to consume printable text; do not make the new bindings priority
  bindings to force the result.
- Each choice panel remains mounted while inactive. Always derive the target from `_choice_index` instead of querying
  the first `GateExtrasSelectionList`, or command toggles can silently affect a hidden outcome.
- Delegate to the existing selection-list actions so disabled-row handling, scrolling, highlight painting, and toggle
  messages stay consistent with Textual and do not acquire a second state machine.
- Footer changes affect deterministic PNG fixtures. Accept only the expected text-region diffs and do not update
  unrelated goldens.
