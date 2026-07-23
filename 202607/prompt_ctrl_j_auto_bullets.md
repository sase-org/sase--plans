---
tier: tale
title: Auto-continue prompt bullets with insert-mode Ctrl+J
goal: Make prompt INSERT-mode Ctrl+J split a supported bullet into a correctly indented
  sibling while preserving existing non-bullet and editor behavior.
create_time: 2026-07-23 10:26:21
status: wip
---

- **PROMPT:** [202607/prompts/prompt_ctrl_j_auto_bullets.md](prompts/prompt_ctrl_j_auto_bullets.md)

# Auto-continue prompt bullets with insert-mode `Ctrl+J`

## Outcome

When `Ctrl+J` inserts a newline in a prompt input pane while the cursor's current physical line belongs to a supported
hyphen bullet, insert a sibling `- ` marker at the owning bullet's indentation and leave the cursor after that marker in
INSERT mode. This should reuse the recently added NORMAL-mode `o` ownership rules, including nested bullets and
Prettier-wrapped continuation lines, while preserving ordinary newline splitting everywhere outside a supported bullet.

## Current behavior and constraints

- `PromptTextArea.BINDINGS` maps `ctrl+j` directly to `action_insert_newline`; the action currently replaces the active
  selection with a bare `\n`. The binding is prompt-local, which is why it wins over the Agents-tab
  `next_agent_metadata_section` binding.
- `PromptTextAreaActionsMixin._normal_open_below_insert_text()` already calls
  `prompt_bullet_sibling_prefix(document.lines, row)` for prompt NORMAL-mode `o`. The pure helper recognizes only
  space-indented `- ` markers and resolves the nearest owning bullet across supported physical continuation lines. Its
  existing boundary contract rejects blank/dedented prose, fences, thematic breaks, tight dashes, unsupported list
  markers, and tab indentation.
- Unlike `o`, which always opens after the current physical line, `Ctrl+J` splits or replaces at the current selection.
  The new prefix must be computed from the cursor row before mutation and inserted in the same replacement as the
  newline so trailing text, selection replacement, cursor placement, and undo history retain TextArea semantics.
- This is prompt-only presentation/editing behavior. It belongs in the Python Textual widget layer, not the shared Rust
  core. It must remain an in-memory, side-effect-free keystroke path with no parsing subprocess, file I/O, asynchronous
  work, or new refresh path.
- `ctrl+j` is a class binding on `PromptTextArea`, not an `ace.keymaps` configuration value. The similarly named Agents
  metadata keymap in `src/sase/default_config.yml` is app-level and should remain unchanged.
- `SingleLineVimTextArea` explicitly swallows `ctrl+j`, and non-prompt `VimTextArea` hosts do not inherit the
  `PromptTextArea` binding or action. They must remain unchanged.
- NORMAL-mode `o` treats its automatically generated prefix as structural text and has special dot-repeat normalization.
  `Ctrl+J` is an insert-mode edit rather than a NORMAL-mode structural command; do not alter the `o` host hook, mutation
  capture, or dot-repeat machinery as part of this focused change.

## Behavioral contract

1. In INSERT mode, `Ctrl+J` on a line whose leading marker is zero or more ASCII spaces followed by exactly `- ` inserts
   `\n`, the same leading spaces, and `- ` at the selection replacement point.
2. On a physical continuation line, use the existing `prompt_bullet_sibling_prefix()` ownership algorithm so a top-level
   or nested continuation creates a sibling at its nearest owning bullet's marker indentation, including after Prettier
   wrapping and after a continuation has dedented back to an outer bullet.
3. Preserve split-at-cursor behavior: any text after a collapsed cursor moves after the new marker, and the cursor lands
   immediately after the inserted marker. For a nonempty selection, continue replacing the same selection range in one
   keyboard edit, deriving context from `selection.end` (the TextArea cursor) before the document changes.
4. When the current line has no supported bullet owner, insert the existing bare newline with unchanged cursor and
   selection semantics.
5. Keep the existing `Ctrl+J` binding and prompt-local priority. Do not affect Enter submission, app-level Agents
   metadata navigation, NORMAL/VISUAL modes, single-line editors, other Vim text areas, or the established `o` behavior.
6. Insert the newline and automatic prefix with one `_replace_via_keyboard()` call so TextArea records them as one
   atomic edit/checkpoint; subsequent user-typed text keeps its existing undo grouping.

## Implementation plan

1. Reuse the existing prompt-bullet ownership helper in the insert-newline action.
   - In `src/sase/ace/tui/widgets/_prompt_text_area_actions.py`, have `action_insert_newline()` capture
     `self.cursor_location[0]` before mutation and ask `prompt_bullet_sibling_prefix(self.document.lines, row)` for the
     sibling marker.
   - Build either `\n<prefix>` or the existing `\n`, then pass that complete payload and the unchanged selection
     endpoints to a single `_replace_via_keyboard()` call.
   - Keep detection independent of syntax highlighting and reuse the exact helper rather than introducing a second
     interpretation of bullet nesting or boundaries.
   - Broaden only stale comments/docstrings in `_prompt_bullet_editing.py` or the action mixin that incorrectly describe
     the shared ownership logic as exclusive to NORMAL-mode `o`; leave the `o`-specific replay normalizer accurately
     scoped.

2. Add focused prompt-widget regression coverage in `tests/ace/tui/widgets/test_prompt_bullet_editing.py`.
   - Parameterize `PromptPage` INSERT-mode `Ctrl+J` cases for a direct top-level bullet, a top-level wrapped
     continuation, a nested marker, a nested wrapped continuation, and ordinary prose.
   - Include split-in-the-middle assertions so trailing text moves to the new sibling after the marker and the cursor
     lands after the correct prefix, rather than testing only end-of-line insertion.
   - Add a selection-replacement case that locks in the existing replacement range while confirming ownership comes from
     the active cursor row.
   - Exercise `Ctrl+J`, typed text, `Escape`, and `u` to verify the automatic prefix is a single structural keyboard
     edit/checkpoint and does not disturb subsequent insert undo grouping.
   - Retain and run the existing helper boundary matrix, NORMAL-mode `o`/`O`, bare `VimTextArea`, and `o` dot-repeat
     tests to prove the shared ownership contract and old behavior remain intact.
   - Retain and run `test_agents_prompt_input_ctrl_j_keeps_local_newline_priority` plus the single-line `ctrl+j`
     regression to verify app-level binding precedence and newline suppression are unchanged.

3. Update the user-facing prompt documentation in `docs/ace.md`.
   - Change the INSERT-mode `Ctrl+J` entry from a generic newline description to note contextual hyphen-bullet
     continuation.
   - Explain that both INSERT-mode `Ctrl+J` and prompt NORMAL-mode lowercase `o` reuse the containing bullet's
     indentation across Prettier-wrapped continuation lines, while non-bullet lines keep ordinary newline/open-below
     behavior.
   - Leave `O`, Enter, the Agents metadata shortcuts, and default-keymap documentation unchanged.

## Validation

1. Run `just install` first, as required for an ephemeral SASE workspace.
2. Run `pytest -q tests/ace/tui/widgets/test_prompt_bullet_editing.py` for the shared ownership, INSERT-mode `Ctrl+J`,
   NORMAL-mode `o`, undo, and dot-repeat coverage.
3. Run the focused existing integration regressions for Agents-tab `Ctrl+J` prompt priority and `SingleLineVimTextArea`
   newline suppression.
4. Run repository-required `just check` for formatting, lint, type checking, the full test suite, and PNG snapshots.
5. If visual snapshots change unexpectedly, inspect the actual/expected/diff artifacts rather than accepting new
   goldens; this text-editing behavior should not require intentional snapshot updates.

## Acceptance criteria

- `Ctrl+J` on direct, wrapped, and nested supported prompt bullets inserts one correctly indented sibling `- ` and
  leaves the cursor after it in INSERT mode.
- Splitting mid-line preserves the trailing text after the automatic marker, and selection replacement still uses the
  active TextArea selection in one undoable keyboard edit.
- Bullet continuation does not cross the existing blank/dedent/fence/unsupported-marker/tab boundaries, and ordinary
  prompt lines still receive a bare newline.
- Prompt-local `Ctrl+J` continues to outrank the Agents metadata shortcut, while Enter submission and single-line editor
  suppression remain unchanged.
- NORMAL-mode `o`/`O`, non-prompt Vim text areas, `o` undo/dot-repeat behavior, and default keymap configuration are
  unchanged.
- Documentation matches the shipped behavior, focused regressions pass, and `just check` succeeds.
