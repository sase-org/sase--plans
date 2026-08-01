---
tier: tale
title: Prompt NORMAL-mode J strips a pulled-up hyphen bullet marker
goal:
  Pressing J in a prompt pane joins the next line onto the current one with its leading `- ` bullet marker removed, so
  folding a bullet into the line above no longer leaves a stray dash mid-sentence.
create_time: 2026-07-29 08:15:27
status: done
---

- **PROMPT:** [prompts/202607/prompt_join_strips_bullet_marker.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/prompt_join_strips_bullet_marker.md)
- **AGENTS:**
  - [bbugyi200.athena.o0--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.o0.md#member-code)
  - [bbugyi200.athena.o0--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.o0.md#member-plan)
- **COMMITS:**
  - [a813226](https://github.com/sase-org/sase/commit/a8132265be0d7e27f93695c6ad3da8d3191ec217) — fix(ace): strip prompt bullet markers on join

# Plan: Prompt NORMAL-mode `J` strips a pulled-up hyphen bullet marker

## Motivation

In a prompt pane, `J` currently joins lines verbatim apart from whitespace collapsing:

```
- first bullet
- second bullet
```

`J` on the first row produces `- first bullet - second bullet`. The inner `- ` is noise: the user is folding two list
items into one sentence, so the pulled-up marker should disappear, yielding `- first bullet second bullet`.

This mirrors the prompt's existing bullet-aware editing (`Ctrl+J`, `o`, `O`, `Tab`/`Shift+Tab` all understand the
prompt's `- ` markers) and keeps the generic editor host behavior untouched.

## Current behavior

- `src/sase/ace/tui/widgets/_vim_normal_editing.py` dispatches `J` (in `_handle_normal_edit_key`) to `_join_lines`.
- `src/sase/ace/tui/widgets/_vim_normal_operator_exec.py` implements `_join_lines(count)`: per join iteration it
  `rstrip()`s the current line, `lstrip()`s the next line, joins them with a single space, deletes rows
  `(row, 0) .. (row + 1, len(next_line))`, reinserts the joined text, and parks the cursor at the join column.
- `_join_lines` lives on `VimNormalOperatorExecutionMixin`, which every `VimTextArea` host inherits — the config-edit
  modal textarea, frontmatter panel sub-editors, the AXE entry editor, and every `SingleLineVimTextArea` input, not just
  the prompt.
- `src/sase/ace/tui/widgets/_prompt_bullet_editing.py` owns the prompt's definition of a hyphen bullet marker
  (`_BULLET_MARKER_RE = ^( *)- `) plus the boundary regexes (`_THEMATIC_BREAK_RE`, `_UNSUPPORTED_MARKER_RE`,
  `_TIGHT_DASH_RE`, `_FENCE_RE`).
- `VimTextArea` already exposes a "host hooks" section (`src/sase/ace/tui/widgets/vim_text_area.py`) with safe defaults;
  `PromptTextAreaActionsMixin` (`src/sase/ace/tui/widgets/_prompt_text_area_actions.py`) overrides
  `_normal_open_below_insert_text` / `_normal_open_above_insert_text` there to make `o` / `O` bullet-aware. This change
  follows exactly that pattern.

`PromptTextArea` lists `PromptTextAreaActionsMixin` before `VimTextArea` in its bases, so a hook override in that mixin
wins the MRO.

## Target behavior

`J` (and `<count>J`, and `.` repeating either) in a **prompt** pane: when the next line's text is folded onto a current
line that has content, a leading prompt hyphen bullet marker on that next line is dropped along with the whitespace that
already gets collapsed today.

| Before (cursor on row 0)  | After `J`            | Note                                                    |
| ------------------------- | -------------------- | ------------------------------------------------------- |
| `- one` / `- two`         | `- one two`          | the primary case                                        |
| `hello` / `    - world`   | `hello world`        | indented marker; current line need not be a bullet      |
| `- one` / `  -   two`     | `- one two`          | extra spaces after the marker collapse as they do today |
| `- one` / `- `            | `- one`              | marker-only next line behaves like a blank next line    |
| `hello` / `-world`        | `hello -world`       | tight dash is not a marker                              |
| `hello` / `* world`       | `hello * world`      | only `- ` is in scope                                   |
| `hello` / `1. world`      | `hello 1. world`     | only `- ` is in scope                                   |
| `hello` / `- - -`         | `hello - - -`        | thematic break is not a marker                          |
| ``(blank) /`- two`        | `- two`              | nothing folds, so the marker survives                   |
| `- one` / `- two` / `- x` | `- one two x` (`3J`) | count strips each pulled-up marker                      |

Non-prompt `VimTextArea` hosts keep vanilla vim `J` (`- one` + `- two` → `- one - two`).

## Decisions

1. **Prompt-only, via a host hook.** Add a `VimTextArea` host hook rather than changing `_join_lines` unconditionally.
   The generic widget is documented as a reusable, host-agnostic vim layer, and `- ` bullets are a prompt-authoring
   convention; a config-file or AXE-script editor should not silently eat a dash. Precedent: the `o` / `O` open-line
   hooks.
2. **Marker definition is the repo's existing one** — `_BULLET_MARKER_RE` (`^( *)- `): space-only indentation, exactly
   one dash, exactly one following space required. Consequence: a **tab**-indented `- ` is _not_ stripped, matching the
   rest of the prompt bullet layer (`docs/ace.md`: "A tab-indented dash is not treated as a bullet marker"). This is the
   one place the behavior is narrower than a literal reading of "optional whitespace followed by `- `"; call it out at
   review time if tab-indented markers should also strip.
3. **Only strip when the current line contributes text.** If the current line is blank/whitespace-only, `J` merely pulls
   the next line up — no fold happens — so the marker is preserved and list structure survives.
4. **Thematic breaks are excluded.** `- - -` matches `^( *)- ` but is a horizontal rule; stripping it would leave `- -`.
   `---` never matched. Reuse `_THEMATIC_BREAK_RE`.
5. **No fenced-code awareness.** The prompt's bullet marker layer already treats a dash inside a fence as a marker (it
   even bolds it); a fence scan here would be new, inconsistent behavior. Out of scope.
6. **No opt-out key.** Vim's `gJ` is unavailable — `g`+`J` is already bound to "move pane down" in the prompt stack
   (`src/sase/ace/tui/widgets/_prompt_input_bar_g_prefix_actions.py`).
7. **Stays in Python.** Checked the Rust core boundary: `../sase-core` has no prompt/vim editing transforms (its
   `bullet` matches are plan-Markdown parsing in `crates/sase_core/src/plan/`). This is Textual-widget keybinding
   behavior and belongs beside the existing prompt bullet helpers.

## Implementation

### 1. `src/sase/ace/tui/widgets/_prompt_bullet_editing.py` — pure helper

Add a public helper next to the existing ones and register it in `__all__` (the list is sorted; it goes last):

```python
def strip_prompt_bullet_marker(line: str) -> str:
    """Return *line* without a leading prompt hyphen bullet marker.

    Lines that do not open with the prompt's supported space-indented ``- ``
    marker -- tight dashes, tab indentation, unsupported markers, thematic
    breaks -- are returned unchanged.
    """
    if _THEMATIC_BREAK_RE.match(line):
        return line
    marker = _BULLET_MARKER_RE.match(line)
    if marker is None:
        return line
    return line[marker.end() :]
```

The helper deliberately does not touch whitespace _after_ the marker; `_join_lines` already `lstrip()`s the result.

### 2. `src/sase/ace/tui/widgets/vim_text_area.py` — default host hook

In the `-- Host hooks (safe defaults; subclasses override to reach their host) --` section, beside
`_normal_open_above_insert_text`:

```python
def _normal_join_next_line_text(self, next_line: str) -> str:
    """Return the pulled-up line NORMAL-mode ``J`` folds in. Default: verbatim."""
    return next_line
```

### 3. `src/sase/ace/tui/widgets/_vim_normal_operator_exec.py` — call the hook

Declare the hook in the mixin's `if TYPE_CHECKING:` block (beside `_clear_prompt_search` / `_flash_yank`):

```python
def _normal_join_next_line_text(self, next_line: str) -> str: ...
```

In `_join_lines`, inside the join loop, feed the next line through the hook before the existing `lstrip()`, and only
when the current line has content (decision 3):

```python
cur_line = doc.get_line(row)
next_line = doc.get_line(row + 1)
# Strip trailing space on current, leading space on next.
stripped_cur = cur_line.rstrip()
# A fold into real content drops the pulled-up line's structural prefix;
# pulling a line onto a blank one folds nothing, so it stays verbatim.
folded_next = self._normal_join_next_line_text(next_line) if stripped_cur else next_line
stripped_next = folded_next.lstrip()
```

**Gotcha:** leave the delete range alone — `self.delete((row, 0), (row + 1, len(next_line)))` must keep measuring the
**original** `next_line`, not the hook's shortened result, or the trailing characters of the joined line are left
behind. Everything downstream (`joined`, `join_col`, cursor placement, `_record_mutation`) is unchanged.

### 4. `src/sase/ace/tui/widgets/_prompt_text_area_actions.py` — prompt override

Import `strip_prompt_bullet_marker` in the existing `_prompt_bullet_editing` import block (keep it sorted) and override
the hook next to the `o` / `O` overrides:

```python
def _normal_join_next_line_text(self, next_line: str) -> str:
    """Drop a pulled-up prompt hyphen bullet marker for ``J``."""
    return strip_prompt_bullet_marker(next_line)
```

Count and dot-repeat need no extra work: the hook runs per join iteration inside `_join_lines`, and `.` replays `J`
through the same path.

## Tests

All three files below already exist; extend them.

**`tests/ace/tui/widgets/test_prompt_bullet_editing.py`** — a parametrized unit test for `strip_prompt_bullet_marker`,
in the style of the existing `is_prompt_bullet_marker_only` block:

- stripped: `"- x"` → `"x"`, `"  - x"` → `"x"`, `"-   x"` → `"  x"` (helper leaves inner spacing; the caller lstrips),
  `"- "` → `""`, `"  - "` → `""`
- unchanged: `"-x"`, `"-"`, `"\t- x"`, `" \t- x"`, `"* x"`, `"+ x"`, `"1. x"`, `"> x"`, `"- - -"`, `"---"`, `""`,
  `"   "`, `"plain"`

**`tests/test_prompt_normal_mode_join.py`** — behavior through `PromptPage` (`PromptPage(text, cursor=(row, col))`), one
test per row of the target-behavior table above, plus:

- cursor lands at the join column: `PromptPage("- one\n- two")` → `J` → text `"- one two"`, cursor `(0, 5)`
- existing tests must still pass unmodified (plain joins, whitespace collapsing, last-line no-op, empty next line,
  count, count exceeding line count, dot-repeat, the no-background-formatter assertion)
- dot-repeat with markers: `"- a\n- b\n- c\n- d"` → `J` → `"- a b\n- c\n- d"`, then `.` → `"- a b c\n- d"`
- blank current line: `PromptPage("- one\n\n- two", cursor=(1, 0))` → `J` → `"- one\n- two"`

**`tests/ace/tui/widgets/test_vim_text_area.py`** — pin that the generic host is unaffected:
`VimEditorPage("- one\n- two", cursor=(0, 0))` → `J` → `"- one - two"`.

## Docs

`docs/ace.md`:

1. The prompt NORMAL-mode "Other Commands" table row (currently `| \`J\` | Join current line with next (supports count:
   \`5J\`) |`) — note that a pulled-up prompt hyphen bullet marker is removed.
2. The paragraph immediately after that table, which today explains `o` / `O` bullet continuation — add a sentence
   covering `J`, including that a blank current line keeps the marker and that non-prompt editors keep vanilla `J`.
3. The prompt hyphen-bullet section (the prose near the INSERT-mode `Ctrl+J` / `o` / `O` bullet explanation) — mention
   `J` as the inverse operation so the bullet-aware key set is listed in one place.

Table alignment is prettier-managed; `just fmt` reflows it. No help-modal change is needed: the `?` popup documents
ChangeSpec/agent/axe bindings, not the prompt vim layer, which lives only in `docs/ace.md`.

## Verification

```bash
just install
just fmt
just check
```

Targeted while iterating:

```bash
uv run pytest tests/test_prompt_normal_mode_join.py \
              tests/ace/tui/widgets/test_prompt_bullet_editing.py \
              tests/ace/tui/widgets/test_vim_text_area.py \
              tests/ace/tui/widgets/test_prompt_stack_keymaps_focus.py
```

(`test_prompt_stack_keymaps_focus.py` exercises `J` and `3J` in a prompt stack and must stay green.)

Manual smoke check in `sase ace`: type a two-item `- ` list in the prompt, press `Esc`, put the cursor on the first
item, press `J`, and confirm the result is a single bullet with no inner dash; press `u` and confirm one undo restores
both lines.

## Out of scope

- Other list markers (`*`, `+`, `1.`, `>`) and blockquote prefixes.
- Fenced-code awareness for `J`.
- A `gJ`-style raw-join escape hatch.
- Changing `J` for non-prompt `VimTextArea` hosts.
