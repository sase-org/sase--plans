---
tier: tale
title: Keep Ctrl+J from clearing a lone empty prompt bullet
goal: In the ACE prompt input widget, INSERT-mode Ctrl+J on an empty `- ` marker only
  exits the list when the line above that marker already belongs to a hyphen bullet;
  on a lone empty marker it instead opens a second sibling `- ` line below, so the
  exit still happens on the next press.
create_time: 2026-07-25 14:20:51
status: wip
---

- **PROMPT:** [202607/prompts/lone_bullet_ctrl_j.md](prompts/lone_bullet_ctrl_j.md)

# Plan: Keep Ctrl+J from clearing a lone empty prompt bullet

## Problem

In the `sase ace` prompt input widget, INSERT-mode `Ctrl+J` currently treats **any** marker-only line (zero or more
leading spaces followed by `- `) as a request to leave the list: it wipes the marker and inserts a bare newline.

That is the right behavior when the user has been building a list and the empty marker is the trailing "next item" that
Ctrl+J itself created. It is wrong when the empty marker is the _first_ bullet the user just typed, because it destroys
the marker they deliberately wrote.

Reproduction (matches the reported screenshot, where the prompt held `#gh:sase`, a `%w:` line, a blank line, and a lone
`- ` on the last line):

```
#gh:sase
%w:sase-9m.land Can you help me ...

- ▌
```

Pressing `Ctrl+J` here clears the `- ` line and adds a newline. Expected: a second `- ` line is opened below.

## Desired behavior

With no selection and the cursor on a marker-only line:

| Situation                                                | Ctrl+J result                                                      |
| -------------------------------------------------------- | ------------------------------------------------------------------ |
| The line above the empty marker belongs to a `- ` bullet | Unchanged: replace the marker with a bare newline (leave the list) |
| Otherwise (no bullet above)                              | New: open a sibling marker on the next line                        |

So from the screenshot state the presses now read:

1. `Ctrl+J` → `- \n- ▌` (cursor after the new marker's content column, still INSERT)
2. `Ctrl+J` → `- \n\n▌` (the second marker is cleared and a newline added, cursor at column zero)

The first empty marker is intentionally left in place by step 2; that mirrors today's exit path, which only ever clears
the marker the cursor is on.

## Ownership rule for "a bullet above"

Reuse the widget's existing bullet-ownership model rather than inventing a second one: the line above counts as a bullet
when `prompt_bullet_sibling_prefix(lines, row - 1)` returns a prefix. That helper already resolves both direct `- `
markers and physical continuation lines owned by an earlier marker, and it already stops at blank lines, Markdown
fences, thematic breaks, unsupported markers (`*`, `+`, `1.`, `>`), tight dashes, and tab indentation.

Consequences worth stating explicitly, since they are the interesting edges:

- Row 0 has no line above, so a marker-only first line always opens a sibling.
- A blank line above (the reported case) means no bullet above → open a sibling.
- `- item` / `  wrapped` above → bullet above → exit, exactly as today.
- A prose line at column zero between an earlier bullet and the empty marker breaks ownership under the existing model,
  so that case opens a sibling. This is deliberate: it keeps one ownership definition in the widget.
- A nested empty marker directly under its parent (`- outer` then `  - ▌`) has a bullet above, so it exits. Keeping the
  rule at "any owning bullet above" also preserves the useful dedent-then-exit flow (`- a` / `  - b` / `- ▌`).

Do **not** change NORMAL-mode `o` / `O`, `Tab` / `Shift+Tab` bullet shifting, the selection path, or bullet
highlighting. Only the collapsed-selection marker-only branch of `action_insert_newline` moves.

## Scope notes

- This is prompt-widget editing behavior whose helpers already live in Python
  (`src/sase/ace/tui/widgets/_prompt_bullet_editing.py`); no `sase-core` Rust change is in scope.
- `PROMPT_INPUT_SECTION` in `src/sase/ace/tui/modals/help_modal/binding_common.py` has no `Ctrl+J` row today, so the
  help modal needs no edit. Verify this is still true before finishing; if a `Ctrl+J` row has appeared, update it and
  keep the description at 32 characters or fewer.

## Implementation

### 1. Add the pure predicate

`src/sase/ace/tui/widgets/_prompt_bullet_editing.py`

Add a helper next to `prompt_bullet_sibling_prefix` and list it in `__all__` (the list is alphabetized):

```python
def prompt_bullet_row_has_bullet_above(
    lines: Sequence[str],
    cursor_row: int,
) -> bool:
    """Return whether the line above *cursor_row* belongs to a hyphen bullet.

    An empty marker keeps its exit-the-list behavior only when an earlier
    bullet already owns the preceding line; a lone marker instead grows one
    sibling first.
    """
    if cursor_row <= 0:
        return False
    return prompt_bullet_sibling_prefix(lines, cursor_row - 1) is not None
```

### 2. Branch the marker-only path

`src/sase/ace/tui/widgets/_prompt_text_area_actions.py`, `action_insert_newline` (currently around line 322) — import
the new helper alongside the existing two and split the marker-only branch:

```python
    def action_insert_newline(self) -> None:
        """Insert a newline, continuing or exiting a prompt hyphen bullet."""
        row = self.cursor_location[0]
        start, end = self.selection
        line = self.document.get_line(row)
        if start == end and is_prompt_bullet_marker_only(line):
            if prompt_bullet_row_has_bullet_above(self.document.lines, row):
                self._replace_via_keyboard("\n", (row, 0), (row, len(line)))
            else:
                line_end = (row, len(line))
                self._replace_via_keyboard(f"\n{line}", line_end, line_end)
            return

        prefix = prompt_bullet_sibling_prefix(self.document.lines, row)
        insert = f"\n{prefix}" if prefix is not None else "\n"
        self._replace_via_keyboard(insert, start, end)
```

Notes on the new branch:

- `line` is exactly the sibling prefix here (the line is marker-only), so the inserted text is `"\n" + line`.
- The edit is anchored at the end of the marker line rather than at the cursor. The existing exit branch already ignores
  the cursor column inside a marker-only line, and anchoring keeps a cursor parked at column zero or one from producing
  `\n- - `. The resulting cursor lands after the new marker.
- Keep using `_replace_via_keyboard` so the insertion stays one undo checkpoint and dot-insert capture is unaffected.
- If the extra nesting trips a lint complexity or symvision rule, factor the branch into a small private method on the
  mixin rather than reaching for a pragma.

### 3. Refresh the widget docstring

`src/sase/ace/tui/widgets/prompt_text_area.py` — the class docstring currently says Ctrl+J "continues a containing
non-empty hyphen bullet and exits when invoked on an empty marker." Reword so the exit is conditioned on a bullet above,
e.g. "Ctrl+J continues a containing hyphen bullet, and exits from an empty marker only when the line above is already
part of the bullet."

## Tests

All in `tests/ace/tui/widgets/test_prompt_bullet_editing.py`, matching the file's existing parametrized style and its
`PromptPage` helper usage.

1. Pure-helper coverage for `prompt_bullet_row_has_bullet_above`, parametrized over lines/row → expected bool:
   - `["- "], 0` → `False` (no line above)
   - `["", "- "], 1` → `False` (blank line above)
   - `["#gh:sase", "%w:agent text", "", "- "], 3` → `False` (the reported screenshot shape)
   - `["- item", "- "], 1` → `True`
   - `["- outer", "  - nested", "  - "], 2` → `True`
   - `["- item", "  wrapped", "- "], 2` → `True` (continuation line above is owned)
   - `["- item", "", "- "], 2` → `False` (blank line breaks ownership)
   - `["```", "- "], 1` → `False` (fence above)
   - `["prose", "- "], 1` → `False`
2. Widget behavior: a lone `- ` at row 0 with the cursor at `(0, 2)` — one `Ctrl+J` yields `"- \n- "` with cursor
   `(1, 2)` and INSERT mode; a second `Ctrl+J` yields `"- \n\n"` with cursor `(2, 0)`.
3. Widget behavior for the reported document shape: text `"intro\n\n- "` with cursor `(2, 2)` → `Ctrl+J` gives
   `"intro\n\n- \n- "`, cursor `(3, 2)`; a second press gives `"intro\n\n- \n\n"`, cursor `(4, 0)`.
4. Nested lone marker: text `"  - "` with cursor `(0, 4)` → `Ctrl+J` gives `"  - \n  - "`, cursor `(1, 4)`.
5. Cursor inside the marker: text `"- "` with cursor `(0, 0)` → `Ctrl+J` gives `"- \n- "` (never `"\n- - "`), cursor
   `(1, 2)`.
6. Undo granularity for the new path: from a lone `- `, press `Ctrl+J`, `Ctrl+J`, then `escape` and `u` twice, showing
   the sibling insertion and the exit are separate checkpoints back to `"- \n- "` and then `"- "`.
7. Regression guards that must keep passing untouched:
   - `test_prompt_insert_ctrl_j_twice_exits_bullet_and_undoes_separately` (both the top-level and nested ids) — these
     still exit because a bullet sits above the created marker.
   - `test_prompt_insert_ctrl_j_marker_selection_uses_replacement_path` — a selection still takes the generic path.
   - `test_prompt_insert_ctrl_j_splits_with_structural_prefix` and the `o` / `O` tests.

## Documentation

`docs/ace.md`:

- INSERT-mode key table row for `Ctrl+J` (around line 2569): change "or leave the list from an exact empty marker" to
  make the exit conditional, e.g. "or leave the list from an empty marker that already follows a bullet".
- The prose paragraph that begins "INSERT-mode `Ctrl+J` and prompt NORMAL-mode `o` / `O` continue a containing
  space-indented `- ` bullet" (around lines 2601-2609): rewrite the exit sentence so it states the new condition, and
  add the lone-marker case — a marker-only line whose preceding line is not part of a bullet grows a sibling marker
  instead, so from a freshly typed `- ` the sequence is Ctrl+J to add the second marker and Ctrl+J again to exit. Keep
  the existing statements about separate undo checkpoints, the selection path, and the excluded marker shapes.
- Keep the file's Prettier-compatible wrapping; `just fmt` normalizes it.

`CHANGELOG.md`: add one bullet under `## Unreleased` → `### Bug Fixes`, following the existing `- **ace:** ...` style,
e.g. "**ace:** keep prompt `Ctrl+J` from clearing a lone empty `- ` marker; it now adds a sibling bullet and exits on
the next press."

## Verification

Run from the workspace checkout:

```bash
just install                                                   # ephemeral workspace may be stale
just test tests/ace/tui/widgets/test_prompt_bullet_editing.py  # fast focused loop while iterating
just check                                                     # required before finishing
```

Manual smoke check in `sase ace`: type `- ` into an empty prompt, press `Ctrl+J` (expect a second `- ` line), press
`Ctrl+J` again (expect the second marker cleared and the cursor on a fresh blank line).
