---
tier: tale
title: Exit prompt bullet lists with a second Ctrl+J
goal: Let users end an auto-continued prompt bullet with Ctrl+J Ctrl+J while preserving
  existing continuation, selection, and undo behavior.
create_time: 2026-07-24 15:58:18
status: done
---

- **PROMPT:** [prompts/202607/ctrl_j_exit_bullet_list.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/ctrl_j_exit_bullet_list.md)
- **AGENTS:**
  - [bbugyi200.athena.jl](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.jl/README.md)
  - [bbugyi200.athena.jl--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.jl.md#member-code)
- **COMMITS:**
  - [4f783d4](https://github.com/sase-org/sase/commit/4f783d4b6efcae81eae4014d53154993b6083693) — feat(ace): exit prompt bullets with ctrl+j

# Exit Prompt Bullet Lists with a Second Ctrl+J

## Goal

Make consecutive `Ctrl+J` presses in the multiline ACE prompt editor provide the conventional way to leave a
hyphen-bullet list. The first press after a non-empty bullet should keep inserting a correctly indented sibling marker,
as it does today. If `Ctrl+J` is then pressed on that marker-only line—spaces for indentation followed by exactly `- `
and no content—the editor should remove the marker, leave that line blank, add one more newline, and place the cursor at
column zero two physical lines below the original bullet.

Keep the behavior local to `PromptTextArea`; do not change the `Ctrl+J` binding, the default keymap configuration,
normal-mode `o`/`O`, or the bare/single-line text-area widgets.

## Current Behavior and Design

`PromptTextArea.action_insert_newline()` in `src/sase/ace/tui/widgets/_prompt_text_area_actions.py` currently asks
`prompt_bullet_sibling_prefix()` for the current line's containing hyphen-bullet prefix and inserts `\n<prefix>` through
`_replace_via_keyboard()`. A line that contains only an auto-created marker still qualifies as a bullet, so the second
press creates another marker instead of ending the list.

Implement this without timing or mutable “last key” state. A marker-only line is sufficient evidence for the exit
operation and makes the interaction deterministic even when key events are delayed. Keep marker recognition with the
existing pure Markdown-list helpers in `src/sase/ace/tui/widgets/_prompt_bullet_editing.py`, using the same spaces-only
hyphen-marker grammar (`^( *)- $`) already used for sibling prefixes. In the newline action, apply the exit path only
when the selection is empty. Replace the entire marker line with `\n` via `_replace_via_keyboard()` so the line is
cleared, a new blank line is created, the cursor lands at `(row + 1, 0)`, and the operation remains a normal TextArea
undo checkpoint. For a non-empty selection or any other current line, retain the existing selection replacement and
bullet-continuation behavior unchanged.

This is a synchronous, in-memory edit on an existing keystroke path. It must not introduce filesystem access,
subprocesses, timers, deferred callbacks, or other event-loop work.

## Implementation

1. Extend `src/sase/ace/tui/widgets/_prompt_bullet_editing.py` with a small exported predicate for the exact marker-only
   hyphen-bullet form. Reuse `_BULLET_MARKER_RE.fullmatch()` so top-level (`- `) and space-indented (` -`) markers are
   accepted, while bullets with content, tight dashes, unsupported marker styles, whitespace-only lines, and tab
   indentation are rejected.
2. Update `PromptTextAreaActionsMixin.action_insert_newline()` in
   `src/sase/ace/tui/widgets/_prompt_text_area_actions.py` to inspect the current selection and line before calculating
   a sibling prefix:
   - When the selection is collapsed and the whole current line is a marker-only hyphen bullet, replace `(row, 0)`
     through the line end with a single newline and return.
   - Otherwise, preserve the current `\n<prefix>`/`\n` insertion at the selection exactly as it works today.
3. Update the `PromptTextArea` behavior documentation in `src/sase/ace/tui/widgets/prompt_text_area.py` to say that
   `Ctrl+J` continues a containing non-empty hyphen bullet and exits when invoked on an empty marker. Leave its binding
   declaration unchanged.
4. Expand `tests/ace/tui/widgets/test_prompt_bullet_editing.py`:
   - Add pure predicate cases for top-level and nested marker-only lines and rejection cases for content-bearing,
     malformed, unsupported, and tab-indented lines.
   - Add prompt-widget cases that press `Ctrl+J` twice at the end of top-level and nested bullets, asserting the final
     text has two newlines after the original bullet, the marker is gone, the cursor is at column zero two rows below
     the starting line, and insert mode remains active.
   - Assert that each press remains independently undoable: the first undo restores the marker-only sibling line and the
     second restores the original single bullet.
   - Add or retain regression coverage showing non-empty bullets still continue with the correct indentation and
     non-empty selections still use the existing replacement path rather than triggering whole-line removal.

## Validation

1. Refresh the editable development environment, as required for ephemeral workspaces:

   ```bash
   just install
   ```

2. Run the focused prompt bullet-editing tests:

   ```bash
   .venv/bin/pytest -q tests/ace/tui/widgets/test_prompt_bullet_editing.py
   ```

3. Run the required repository-wide checks:

   ```bash
   just check
   ```

## Acceptance Criteria

- From a top-level bullet ending in content, `Ctrl+J Ctrl+J` produces the original bullet followed by two newline
  characters and leaves the cursor at column zero two rows below the original line.
- The same interaction works for a bullet indented with spaces, without leaking its indentation or marker onto the final
  blank line.
- A single `Ctrl+J` on a non-empty bullet or its owned continuation still creates the correctly indented sibling bullet.
- Content-bearing and unsupported markers do not activate the exit path.
- A non-empty selection preserves the existing `Ctrl+J` replacement behavior.
- The exit edit and the preceding sibling-marker insertion are separate, reversible undo checkpoints.
- The focused tests and `just check` pass.
