---
tier: tale
title: Exit prompt bullets from the marker content column
goal:
  Ctrl+J exits an existing prompt bullet list when the cursor is immediately after a supported hyphen marker, while
  preserving any text after the cursor and all existing continuation safeguards.
create_time: 2026-07-29 10:17:13
status: done
---

- **PROMPT:** [202607/prompts/ctrl_j_exit_populated_bullet.md](prompts/ctrl_j_exit_populated_bullet.md)

# Plan: Exit prompt bullets from the marker content column

## Context and behavior contract

`PromptTextAreaActionsMixin.action_insert_newline()` currently recognizes the exit-list gesture only when the active
line consists entirely of a supported space-indented `- ` marker. If the line is `- #plan` and the cursor is between the
marker and `#plan`, the generic continuation path instead leaves the old marker in place and inserts a second marker.

Generalize the existing exit behavior to a collapsed cursor positioned exactly at the content column of a supported
prompt hyphen bullet. For example, this input:

```text
- foo bar
- <cursor>#plan
```

becomes:

```text
- foo bar

<cursor>#plan
```

The exit remains conditional on the preceding physical line belonging to a hyphen bullet according to the existing
ownership rules. This preserves the protection for a lone marker that the user deliberately typed. The edit removes the
current line's indentation and `- ` marker, inserts one newline in their place, preserves the entire suffix after the
cursor verbatim, and leaves the cursor before that suffix. Indented markers therefore exit to an unindented line,
matching the current marker-only exit behavior.

Do not change the following behaviors:

- A marker-only line may still exit even if its cursor is inside the marker, and a lone marker still grows a sibling
  before a later press can exit.
- A populated lone bullet continues normally rather than losing its deliberately typed marker.
- A cursor before the content column, a cursor already inside bullet content, or an active selection uses the existing
  split/replacement path.
- Markdown thematic breaks, tight dashes, tab-indented markers, and unsupported Markdown marker forms do not become exit
  gestures.
- Normal-mode `o`/`O`/`J`, bullet indentation, dot replay, key bindings, and non-prompt text areas remain unchanged.

## Implementation

1. In `src/sase/ace/tui/widgets/_prompt_bullet_editing.py`, add a small pure helper that recognizes the end/content
   column of a supported leading hyphen marker on a possibly populated line. Reuse the module's existing marker and
   boundary definitions so the helper accepts space-indented `- ` bullets but rejects thematic breaks and all currently
   unsupported marker shapes. Keep `is_prompt_bullet_marker_only()` for the broader cursor-inside-empty-marker
   compatibility path, and export the new helper for the action and focused tests.

2. In `src/sase/ace/tui/widgets/_prompt_text_area_actions.py`, extend `action_insert_newline()` for collapsed
   selections. Preserve the existing marker-only branches first. For a populated supported bullet, enter the exit-list
   path only when the cursor column exactly equals the marker's content column and
   `prompt_bullet_row_has_bullet_above()` is true. Replace the range from column zero through the marker with `"\n"` via
   `_replace_via_keyboard()` so trailing text survives, the resulting cursor is at its start, and the whole structural
   edit remains one undo checkpoint. Otherwise fall through to the existing structural-prefix insertion.

3. Update the `PromptTextArea` class documentation in `src/sase/ace/tui/widgets/prompt_text_area.py` to describe exiting
   from the marker content column, without implying that only an empty marker can exit.

## Tests and verification

1. Extend `tests/ace/tui/widgets/test_prompt_bullet_editing_helpers.py` with parameterized coverage for the cursor-aware
   marker helper: top-level and indented populated bullets at their exact content columns, marker-only bullets, wrong
   cursor columns, thematic breaks, tight dashes, tabs, and unsupported Markdown markers.

2. Extend `tests/ace/tui/widgets/test_prompt_bullet_insert_editing.py` with focused `PromptPage` scenarios that:
   - reproduce the requested `- foo bar` / `- <cursor>#plan` transformation and assert the final cursor is immediately
     before `#plan`;
   - exercise an indented/nested marker and verify that its suffix is preserved while the marker indentation is removed;
   - undo the populated-line exit as one checkpoint;
   - prove a populated lone bullet retains the existing continuation behavior;
   - prove cursors before or after the exact content column and active selections keep the normal split/replacement
     behavior;
   - preserve the current marker-only first-press, second-press, cursor-inside, and separate-undo expectations.

3. Run `just install` before repository checks, execute the two focused prompt bullet test modules during iteration, and
   finish with the required `just check`.
