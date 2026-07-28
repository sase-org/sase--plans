---
tier: tale
title: Add q quit to the AXE entry editor
goal: The AXE lumberjack/chop editor owns a safe, discoverable q shortcut that closes
  only the panel without breaking text entry or Esc navigation.
create_time: 2026-07-24 18:34:45
status: done
---

- **PROMPT:** [202607/prompts/axe_editor_q_quit.md](prompts/axe_editor_q_quit.md)
- **AGENTS:**
  - [bbugyi200.athena.jt](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.jt/README.md)
  - [bbugyi200.athena.jt--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.jt.md#member-code)
- **COMMITS:**
  - [c1fc89c](https://github.com/sase-org/sase/commit/c1fc89c571ff2607365caef4bcfe527073eb7d9e) — feat(ace): let q close the AXE entry editor

# Add a screen-local `q` quit keymap to the AXE entry editor

## Context

The AXE tab opens `AxeEntryEditorModal` for both lumberjacks and chops. The modal currently owns `Esc`, which walks
backward through its state machine (INSERT → NORMAL → browse, preview → edit, browse → close), but it does not own `q`.
Because `q` is ACE's app-level quit binding, an unconsumed `q` from this panel can escape to the application instead of
closing only the editor.

The modal also embeds `VimTextArea` controls. In INSERT mode, printable `q` must remain ordinary field input. In NORMAL
mode, `q` is not consumed by the Vim layer, so it can safely reach a screen-local binding.

## Desired behavior

- In browse mode, `q` closes the AXE lumberjack/chop editor and returns `None`, discarding unsaved edits just like
  closing the panel with `Esc`.
- In preview mode, `q` closes the editor directly; unlike `Esc`, it does not return to the property sheet.
- In a focused cell's INSERT mode, `q` remains literal input. After leaving INSERT for NORMAL mode, `q` closes the
  editor.
- While a plan/apply/reload transaction is busy, `q` is consumed by the modal but does not dismiss it, matching the
  existing busy guard on `action_back`.
- `Esc` keeps its existing staged navigation semantics.
- The local action must never invoke ACE's application quit lifecycle.

## Implementation

1. In `src/sase/ace/tui/modals/axe_entry_editor_modal.py`, add a non-priority, screen-local `q` binding with a distinct
   editor-quit action and a `Quit` description. Implement that action on `AxeEntryEditorModal` so it honors `_busy` and
   otherwise dismisses the modal with `None` directly, rather than delegating to `action_back` or the app-level
   `action_quit`.

   Keep the binding non-priority so a focused `VimTextArea` consumes printable `q` in INSERT mode. Once the editor is in
   NORMAL mode, the unhandled key should bubble to the modal and invoke the local action.

2. In `src/sase/ace/tui/modals/axe_entry_sheet.py`, make the new command discoverable in the browse and preview hint
   strings, including the narrow browse variant. Label `q` as quit and retain the existing `Esc` wording so the
   distinction between direct quit and staged back navigation is clear. Do not advertise `q` in cell-mode hints, where
   it is valid text in INSERT mode, or in the busy-only `Working…` hint.

3. Update `docs/ace.md` under “AXE Property Sheet” to document `q` separately for browse, cell NORMAL mode, and preview:
   direct panel close for `q`, existing staged back behavior for `Esc`, and literal typing in INSERT mode.

   This is a modal-scoped, fixed keymap, so do not add it to `ace.keymaps.app` in `src/sase/default_config.yml` or to
   the global ACE help/footer keymap. The property-sheet hints and its dedicated documentation are the relevant
   discoverability surfaces.

4. Extend `tests/ace/tui/test_axe_entry_editor_modal.py` with mounted interaction coverage that proves:
   - `q` dismisses the modal with `None` from browse mode without quitting the `AcePage` app;
   - `q` dismisses directly from preview instead of returning to edit;
   - a focused INSERT-mode cell accepts `q` as text and stays open, while `q` after switching that editor to NORMAL mode
     dismisses the modal;
   - the busy guard keeps the modal open; and
   - the existing multi-stage `Esc` test continues to pass unchanged.

   Add a focused assertion for the binding/action mapping if that makes failures easier to diagnose.

5. Extend `tests/ace/tui/test_axe_entry_sheet.py` to cover the exact mode-aware hint contract: browse (wide and narrow)
   and preview advertise `q` quit, cell and busy hints do not, and the existing `Esc` guidance remains present.

6. Regenerate only the intentional AXE editor PNG goldens affected by browse/preview hint text in
   `tests/ace/tui/visual/snapshots/png/`, using `tests/ace/tui/visual/test_ace_png_snapshots_axe_editor.py`. Review the
   changed images or generated diff artifacts to confirm that only the hint line changed; cell-mode and unrelated AXE
   snapshots should remain byte-identical.

## Validation

Run `just install` before the repository checks, then:

1. Run the focused behavioral tests:

   ```bash
   just test -- tests/ace/tui/test_axe_entry_editor_modal.py tests/ace/tui/test_axe_entry_sheet.py
   ```

2. Update the intentional editor snapshots, inspect the resulting image diffs, and immediately re-run the same visual
   file without update mode:

   ```bash
   just test-visual -- --sase-update-visual-snapshots tests/ace/tui/visual/test_ace_png_snapshots_axe_editor.py
   just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_axe_editor.py
   ```

3. Run the mandatory full repository gate:

   ```bash
   just check
   ```

## Acceptance criteria

- `q` closes only the AXE entry editor from every non-busy, non-INSERT editor state and returns `None`.
- Typing `q` into an INSERT-mode scalar or YAML cell remains possible.
- `Esc` retains its existing staged behavior, including preview-to-edit and cell-to-browse transitions.
- Browse and preview hints and the AXE property-sheet documentation accurately distinguish `q` quit from `Esc` back.
- Focused behavior tests, revalidated PNG snapshots, and `just check` all pass.
