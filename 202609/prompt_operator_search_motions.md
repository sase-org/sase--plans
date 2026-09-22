---
tier: tale
title: Vim operator + search motions (d/ c/ y/ d? dn) in the prompt input
goal:
  In prompt NORMAL mode, any operator followed by `/{pattern}<Enter>` or
  `?{pattern}<Enter>` (and `n` / `N`) acts on exactly the text between the cursor and
  the chosen match. A live, operator-colored preview shows the region before Enter
  commits it, and the edit is undoable and dot-repeatable.
size: medium
proposed_by: bbugyi200.athena.0pb
create_time: 2026-09-22 10:16:57
status: wip
---

# Plan: Operator + search motions for the prompt input widget

## Summary

Today `d/` silently cancels the pending `d`. After this change, `d/foo<Enter>` deletes
from the cursor up to, but not including, the next `foo`. It works like Vim's exclusive
`/` motion, and every operator in the prompt's vim tower accepts it (`d`, `c`, `y`,
`gu`, `gU`, `g~`, `>`, `<`, `ys`). Reverse (`d?foo`), counts (`2d/foo`, `d3/foo`),
repeat-search operators (`dn`, `dN`), undo, and dot-repeat also work.

The feature follows three design principles:

1. **What you see is what gets operated on.** While you type the pattern, the exact
   region the operator will touch is tinted in the operator's color: red with
   strikethrough for `d`/`c`, green for `y`, and the theme's secondary color for the
   transform operators. The search panel says in words what Enter will do (for example
   `delete 42 chars · 3 lines`). This summary also covers huge prompts where the
   highlight overlay is skipped.
2. **Never surprising.** An operator search never wraps around the buffer. `d/` only
   reaches forward and `d?` only reaches backward. It stays inside the active prompt
   pane. If there is no valid target, Enter changes nothing and says why. Esc/Ctrl+C
   restores everything. The range is re-resolved against the live text at the moment of
   Enter, so a stale preview never drives an edit.
3. **Vim-faithful.** Matching is the same smartcase literal matching `/` already uses,
   and a match exactly at the cursor is skipped. The motion is exclusive, with Vim's
   `:help exclusive` / `exclusive-linewise` adjustment for matches at column 0. A
   successful operator search also records the query in the shared search register, so
   `n` / `N` continue from it.

This is editor-level vim behavior inside the Textual prompt widget. The whole vim tower
and the prompt search engine already live in Python in this repo, so this work does
**not** cross the Rust core boundary. No `sase-core` changes are needed.

## Look and feel (target UX)

While typing `d/final` with the cursor on `alpha` in line 1 of a three-line prompt:

```
╭─ delete to match ───────────────────────────────────────────────────────────╮
│  d /final█                          delete 94 chars · 3 lines   1/1          │
╰──────────────────────────────────────────── [enter] delete  [esc/^c] cancel ─╯
```

- **Panel border**: `$error` for `d`/`c`, `$success` for `y`, and `$secondary` for the
  transform family. Plain `/` keeps its existing `$accent` border.
- **Operator chip**: `d` (or `2d` / `gU`) is drawn before the `/` / `?` sigil in the
  operator color, with ink picked for contrast.
- **Panel title**: `{verb} to match` for `/`, `{verb} back to match` for `?`.
- **Panel subtitle**: `[enter] {verb}  [esc/^c] cancel`.
- **Right side of the panel, when a target exists**: the effect summary in the operator
  color, then the existing gold count segment. The count is **pane-local** here (for
  example `2/3`), because the operator cannot reach other panes. Plain `/` keeps its
  stack-global count.
- **Right side of the panel, when there is no target**: a dim message instead. The
  wording is exact:
  - `pattern not found` when the pane has zero matches.
  - `no match after cursor · 2 before` for `/` when matches exist only on the other
    side. For `?` it reads `no match before cursor · 2 after`. Drop the `· N …` suffix
    when the other side has none either.
  - `only 1 match after cursor` (or `before`) when a count asks for more matches than
    exist.
  - On a narrow panel, degrade first to count-only and then to nothing, so the text
    never wraps or clips mid-word.
- **In the text**:
  - All matches get the existing `search.match` highlight.
  - The landing match gets `search.current` (gold), and the preview cursor sits on it,
    the same as Vim incsearch.
  - The affected region is painted _under_ the match highlights. As a result, a backward
    `d?foo` shows the gold match struck through (it will be deleted), while a forward
    `d/foo` shows the gold match intact (the edit stops right before it).
- **During the operator search**: the committed-search pill on the bottom border is not
  shown. It keeps its existing meaning, "a committed plain search".

Operator families, verbs, and region styles:

| Operator | Family      | Verb          | Region style                                        |
| -------- | ----------- | ------------- | --------------------------------------------------- |
| `d`      | destructive | `delete`      | error-tinted background + strikethrough             |
| `c`      | destructive | `change`      | error-tinted background + strikethrough             |
| `y`      | yank        | `yank`        | success-tinted background (pairs with yank flash)   |
| `gu`     | transform   | `lowercase`   | secondary-tinted background                         |
| `gU`     | transform   | `uppercase`   | secondary-tinted background                         |
| `g~`     | transform   | `toggle case` | secondary-tinted background                         |
| `>`      | transform   | `indent`      | secondary-tinted background, full affected rows     |
| `<`      | transform   | `dedent`      | secondary-tinted background, full affected rows     |
| `ys`     | transform   | `surround`    | secondary-tinted background (delimiter key follows) |

"Tinted" means `surface.blend(role, 0.30)` made terminal-safe. The foreground is left
unset so syntax colors stay readable underneath.

## Semantics (exact)

Let `origin` be the absolute offset of the cursor when `/` or `?` was pressed, and let
`N` be the total count: operator count × motion count, default 1.

1. **Matching.** Use `find_search_matches(text, query)` on the active pane's text only.
   This is the same smartcase literal matching `/` uses. `dn` / `dN` pass the register's
   `whole_word` / `smartcase`.
2. **Candidates.**
   - Forward: matches with `start > origin`, nearest first.
   - Reverse: matches with `start < origin`, nearest first.
   - A match starting exactly at `origin` counts toward neither side.
   - There is no wrap.
3. **Target.** The N-th candidate. If there are fewer than N candidates, there is no
   target.
4. **Raw exclusive range.** Forward: `[origin, match.start)`. Reverse:
   `[match.start, origin)`. Both are non-empty by construction.
5. **Exclusive adjustment (Vim `:help exclusive`).** Let `(sr, sc)` be the row and
   column of the range start and `(er, ec)` the row and column of its end. Apply this
   only when `ec == 0 and er > sr`:
   - **Linewise case.** If `sc` is at or before the first non-blank column of row `sr`,
     the range becomes linewise over rows `sr .. er-1`. A blank or whitespace-only row
     counts every column as "at or before".
   - **Charwise case.** Otherwise the range stays charwise, and its end moves to the end
     of row `er-1` (before that row's newline). If this would make the range empty, keep
     the raw range instead.
6. **Affected rows.** `first_row = row(start)`. `last_row` is `row(end - 1)` for
   charwise ranges, and the last row for linewise ranges.
7. **Execution.**
   - Linewise ranges and the indent operators (`>`, `<`) run through
     `_execute_linewise_operator(first_row, last_row, op)`. It already routes indent and
     case operators to the linewise transform, and `ys` to a linewise surround.
   - Every other case runs through
     `_execute_charwise_operator(loc(start), loc(end), op)`.
   - The cursor lands where those helpers already put it. For example, `y?foo` leaves
     the cursor on the match, as in Vim.
   - Registers, the clipboard copy, the yank flash, and INSERT entry for `c` all come
     from the existing helpers unchanged.
8. **Preview region.** The painted span is `[start, end)` for charwise ranges. For
   linewise ranges and indent operators, paint the full rows `first_row..last_row`.
9. **Search register.** A successful operator search records
   `(query, direction, whole_word=False, smartcase=True)` through the existing
   `record_prompt_search` path, so `n` / `N` continue from it.
10. **Dot-repeat.**
    - `.` re-runs the same operator, query, direction, and count from the current
      cursor. A count on `.` replaces the recorded count.
    - For `c`, the captured insert text is replayed, mirroring `_replay_visual_dot`.
    - If the replay finds no target, it shows feedback and makes no edit.
    - `y` is not a mutation, so it is not recorded. `ys` through a search motion works,
      but its dot-repeat is out of scope. It must never replay a broken partial
      sequence: at worst `.` is a no-op.
11. **`dn` / `dN`.** The same resolver runs with the shared search register's query and
    flags. `N` inverts the recorded direction. With no register, show
    `no previous search`. With no target, show the same feedback strings as Enter. These
    go through the normal key-buffer `_record_mutation` path, so `.` naturally replays
    `d`, `n`.
12. **Undo.** One `u` restores the whole operator edit, because the existing helpers
    make a single edit.

Non-goals, to keep this bounded and predictable:

- Regex patterns and `/e`-style offsets. The query is literal, and `/` inside it is a
  literal slash.
- Empty `d/<Enter>` reusing the last pattern. Here it just cancels.
- Counted plain `3/`.
- VISUAL-mode `/` extending the selection.
- `d*` / `d#` / `gn`.
- Cross-pane operators.
- Wrapscan for operators.

## Implementation

### 1. Pure resolver: new `src/sase/ace/tui/widgets/_vim_search_motion.py`

This module has no Textual or widget imports. It holds the whole semantics above so they
can be unit tested exhaustively. Reuse `find_search_matches` and `SearchDirection` from
`_vim_search.py`, and `line_start_offsets` / `offset_to_row_col` from
`vim_search_controller.py`. The module contains:

- `SearchOperatorRequest(operator: str, count: int)`, a frozen dataclass.
- `SearchMotionRange(linewise: bool, start: int, end: int, first_row: int, last_row: int)`,
  a frozen dataclass. Offsets are absolute and `end` is exclusive.
- `SearchMotionResolution(spans, target_index: int | None, reachable: int, opposite: int, motion_range: SearchMotionRange | None)`,
  a frozen dataclass.
- `resolve_search_motion(text, origin, query, direction, *, count=1, whole_word=False, smartcase=True) -> SearchMotionResolution`,
  implementing semantics steps 1–6.
- `exclusive_motion_range(text, start, end) -> SearchMotionRange`, implementing step 5.
  Keep it separate so it can be tested directly.
- `SEARCH_MOTION_MUTATING_OPERATORS = frozenset({"d", "c", "gu", "gU", "g~", ">", "<"})`.
- `search_operator_family(op) -> Literal["destructive", "yank", "transform"]` and
  `search_operator_verb(op) -> str`, following the table above.
- `describe_search_motion_effect(op, motion_range) -> str`:
  - For linewise ranges and `>` / `<`, use `"{verb} {n} line(s)"`.
  - For single-row charwise ranges, use `"{verb} {n:,} char(s)"`.
  - For multi-row charwise ranges, use `"{verb} {n:,} chars · {rows} lines"`.
  - Pluralize correctly.
- `search_motion_miss_message(resolution, direction, count, query) -> str`. This is the
  one source of the no-target wording used by both the panel status and the Enter/`dn`
  toast. The toast form of the zero-match case is `pattern not found: {query}`.
- `SearchOperatorPreview` is a frozen dataclass the text area hands to the bar. It
  holds: `operator`, `count`, `direction`, `family`, `verb`, `effect: str | None`,
  `miss: str | None`, `ordinal: int | None`, and `total: int`.

### 2. Vim tower hooks (shared `VimTextArea` layer)

- `src/sase/ace/tui/widgets/_vim_normal_state.py`:
  - Add a `SearchMotionMutation(NamedTuple)` next to `VisualMutation`, with fields
    `operator`, `count`, `query`, and `direction`.
  - Declare `_last_search_motion_mutation` in the TYPE_CHECKING block, along with a stub
    for the host hook `_replay_search_motion_mutation(mutation, count, has_count)`.
  - In `_record_mutation`, reset `_last_search_motion_mutation = None` inside the
    existing `not self._replaying_dot and self._mutation_key_buffer` block.
  - In `_replay_dot`, dispatch to the hook first when `_last_search_motion_mutation` is
    set.
- `src/sase/ace/tui/widgets/_vim_visual_ops.py` `_record_visual_mutation`: also reset
  `_last_search_motion_mutation = None`.
- `src/sase/ace/tui/widgets/vim_text_area.py`:
  - Initialize `self._last_search_motion_mutation = None` next to
    `_last_visual_mutation`.
  - Add inert default host hooks next to `_start_prompt_search`:
    - `_start_prompt_search_operator(direction, operator, count) -> bool` returns
      `False`.
    - `_operate_to_search_register(operator, count, *, reverse) -> bool` returns
      `False`.
    - `_replay_search_motion_mutation(...) -> None` is a no-op.
  - With these defaults, single-line and secret text areas keep today's behavior (the
    operator is cancelled).
- `src/sase/ace/tui/widgets/_vim_normal.py` `_handle_normal_mode_key`:
  - Add a branch immediately after the existing `/` / `?` branch, so it runs before the
    mutation-buffer append.
  - The branch fires when the key is `/` or `?` (character or `slash` /
    `question_mark`), `self._pending_operator` is set, and `not self._pending_keys`.
  - It calls a new `_start_operator_search_from_normal_key(direction)`. That method
    computes the total count as `_pending_operator_count × int(_count_prefix or 1)` and
    calls `_start_prompt_search_operator`. When the host returns `False`, it returns
    `False` so the old cancel path runs.
  - Update the `_can_start_prompt_search_from_normal_key` docstring to mention the new
    operator route.
  - `dt/`, `df/`, and `ds/` must still treat `/` as a literal target, because
    `_pending_keys` is set for them.
- `src/sase/ace/tui/widgets/_vim_normal_motions.py`, `n` / `N` branch (around line 283):
  - With an operator pending, call `op_info = self._consume_pending_operator(count)` and
    then `_operate_to_search_register(op, eff, reverse=key == "N")`.
  - If the hook returns `False`, clear `_mutation_key_buffer`.
  - Return `True` without doing today's plain repeat.
  - Without a pending operator, keep the current `_repeat_prompt_search` behavior.

### 3. Prompt-side operator search: new `src/sase/ace/tui/widgets/_prompt_search_operator.py`

Add `PromptSearchOperatorMixin` to `PromptTextArea`'s bases directly before
`PromptSearchMixin` in `src/sase/ace/tui/widgets/prompt_text_area.py`. Follow the
TYPE_CHECKING-stub style of the sibling mixins. It implements:

- `_start_prompt_search_operator(direction, operator, count)`: open the search with
  `SearchOperatorRequest(operator, max(1, count))` through the shared opener (see 4),
  then return `True`.
- `_update_prompt_search_operator_preview()`:
  - Resolve against `self.text` from `_search_origin_offset`.
  - Call `_set_search_highlights(spans, current_index=target_index, readout=None)`.
  - Set or clear the operator region (see 5).
  - Move the preview cursor to the target match start, or back to the origin when there
    is no target.
  - Build a `SearchOperatorPreview`, cache it on `self._search_operator_preview`, and
    re-render the command line.
- `_confirm_prompt_search_operator()`:
  1. Re-resolve against the live text.
  2. Tear the search UI down _before_ editing: set the state inactive, hide the panel,
     clear highlights and the region, restore the cursor to the origin cursor, and reset
     `_search_operator` and `_search_operator_preview`. The edit's `delete()` /
     `_replace_via_keyboard` overrides call `_clear_prompt_search`, which must find
     nothing active.
  3. With no target: toast `search_motion_miss_message(...)`, clear
     `_mutation_key_buffer`, call `_update_count_display()`, and stop.
  4. Otherwise: record the register, then run the operator through
     `_run_search_motion_operator`.
  5. If `op in SEARCH_MOTION_MUTATING_OPERATORS`, set
     `self._last_search_motion_mutation = SearchMotionMutation(...)` _after_ the
     helper's own `_record_mutation` has run.
  6. Call `_update_count_display()`.
- `_run_search_motion_operator(op, motion_range)`: the linewise/charwise dispatch from
  semantics step 7. Convert offsets with `_location_from_absolute`.
- `_operate_to_search_register(op, count, *, reverse)`: the `dn` / `dN` path from
  semantics step 11. Read the register through
  `self._find_prompt_bar().prompt_search_register()`, tolerating a missing bar. Clear
  transient search highlights before editing.
- `_replay_search_motion_mutation(mutation, count, has_count)`: the dot path from
  semantics step 10.
  - Set `_replaying_dot = True` inside `try` / `finally`.
  - For `c`, replay `_last_mutation_insert` through
    `_insert_replayed_text_and_return_normal` when the mode is INSERT.
  - Re-record the register.

### 4. Existing prompt search: `src/sase/ace/tui/widgets/_prompt_search.py`

- Add `_search_operator: SearchOperatorRequest | None = None` and
  `_search_operator_preview: SearchOperatorPreview | None = None` to `__init__`.
- Extract the body of `_start_prompt_search` into
  `_open_prompt_search(direction, operator)`. `_start_prompt_search(direction)` becomes
  `_open_prompt_search(direction, None)`. The opener sets `_search_operator` and still
  clears pending/count state, completions, and highlights, and calls
  `prepare_search_command_line`.
- `_update_prompt_search_preview`: when `_search_operator` is set, delegate to
  `_update_prompt_search_operator_preview()` and return. This skips the stack-global
  readout work entirely.
- `_confirm_prompt_search`: when `_search_operator` is set, delegate to
  `_confirm_prompt_search_operator()`.
- `_cancel_prompt_search` and `_clear_prompt_search`: also reset `_search_operator` and
  `_search_operator_preview`. `_cancel_prompt_search` additionally clears
  `_mutation_key_buffer` when an operator was pending. The region is cleared by
  `_clear_search_highlights` (see 5).
- `_render_prompt_search_command_line`: pass `operator=self._search_operator_preview` to
  `bar.show_search_command_line`.

### 5. Region overlay: `src/sase/ace/tui/widgets/_search_highlight.py`

- Add `_search_operator_region: tuple[int, int, str] | None` (start, end, style name).
- Add a setter, `_set_search_operator_region(start, end, family, *, refresh=True)`.
  `_clear_search_highlights` also clears the region.
- In `_build_highlight_map`, append the region span **before** the match spans, so the
  match styles layer on top and keep the strikethrough underneath. Restructure the early
  `if not self._search_match_spans: return` so the region still paints. Honor the same
  `_MAX_OVERLAY_BYTES` / `_MAX_OVERLAY_LINES` guard. When the guard skips painting, the
  panel's effect summary is the safety net.
- In `_register_search_text_area_theme`, register `search.operator.destructive`,
  `search.operator.yank`, and `search.operator.transform`. Build them from
  `search_operator_palette(family, self.app.theme_variables).region`. Theme changes
  already re-run this registration.

### 6. Palette: `src/sase/ace/tui/widgets/_prompt_search_readout.py`

- Add `SearchOperatorPalette(chip: Style, effect: Style, region: Style)` and a cached
  `search_operator_palette(family, variables)`.
- Build it with the module's existing private helpers (`_theme_var`, `_terminal_safe`,
  `_ink_for`, `_ensure_contrast`), so no private names cross modules.
- Role colors: `error` for destructive, `success` for yank, `secondary` for transform.
  Add these to `_FALLBACK_COLORS` and `_THEME_VAR_NAMES`, and keep the existing readout
  cache keys working.
- `chip`: bold ink on the terminal-safe role background.
- `effect`: bold role color, contrast-adjusted against `surface` (minimum 3.0).
- `region`: background is `surface.blend(role, 0.30)` made terminal-safe, with
  `strike=True` only for the destructive family.

### 7. Command line and panel: `search_command_line.py`, `_prompt_input_bar_search.py`, `styles.tcss`

- `render_search_command_line(...)` gets an optional `prefix: Text | None = None`,
  appended before the sigil. Existing callers are unchanged.
- `show_search_command_line(*, direction, query, readout, operator=None)` in
  `src/sase/ace/tui/widgets/_prompt_input_bar_search.py`:
  - `operator=None` keeps today's output exactly. The existing snapshots must not
    change.
  - With an operator:
    - Set the title and subtitle as specified above.
    - Swap the CSS class to `operator-destructive`, `operator-yank`, or
      `operator-transform`.
    - Render the chip prefix: `{count}{op}` with the count omitted when it is 1.
    - Compose the status. For a target, use the effect text in the `effect` style plus
      `format_search_count_segment(ordinal, total, …)`. For no target, use the dim miss
      text.
    - Degrade on narrow widths as described.
  - `hide_search_command_line` removes the operator classes.
- `src/sase/ace/tui/styles.tcss`, next to `PromptInputBar #prompt-search-command`:
  - Add `.operator-destructive { border: round $error; }`.
  - Add `.operator-yank { border: round $success; }`.
  - Add `.operator-transform { border: round $secondary; }`.

### 8. Help and docs

- `src/sase/ace/tui/modals/help_modal/binding_common.py`, `PROMPT_INPUT_SECTION`, after
  the `g* / g#` row:
  - Add `("d/ / d? + Enter", "Operate up to a search match")`.
  - Add `("dn / dN", "Operate to next/prev match")`.
  - Descriptions must stay at 32 characters or fewer.
- `docs/ace.md`:
  - Add a new `#### Operator + Search` subsection under `### NORMAL Mode`, after
    `#### Text Objects`. Cover:
    - the key table: `{op}/{pat}<Enter>`, `{op}?{pat}<Enter>`, `{op}n` / `{op}N`, and
      counts;
    - WYSIWYG preview colors per family, and the struck-through match on `?`;
    - the effect summary;
    - no-wrap and pane-local behavior, with the exact miss messages;
    - the exclusive/linewise rule, with one short example;
    - dot-repeat and undo;
    - the non-goals.
  - Extend the `/` / `?` row in `#### Other Commands` with "after an operator, acts up
    to the match".
  - Add one sentence to `### Prompt Search` pointing at the new subsection.
- Do not edit `CHANGELOG.md` (it is generated). No `default_config.yml` change is
  needed, because prompt vim keys are not configurable keymaps. No feature flag is
  needed: this is additive, complete, and replaces a silent no-op.

## Tests

- **Pure resolver.** New file `tests/ace/tui/widgets/test_vim_search_motion.py`, table
  driven. Cover:
  - forward and reverse basics;
  - a match exactly at the cursor being skipped;
  - overlapping matches;
  - smartcase in both directions (lowercase query ignores case, uppercase query is
    case-sensitive);
  - `count` selecting the N-th match, and `count` exceeding the available matches;
  - no match in the requested direction, with the correct `opposite` count;
  - zero matches;
  - the charwise column-0 adjustment (start after the first non-blank);
  - the linewise adjustment (start at or before the first non-blank, and on a
    whitespace-only row);
  - the empty-adjustment guard (cursor at end of line with the match at the next row's
    column 0);
  - reverse with the origin at column 0 of a later row;
  - `first_row` / `last_row`;
  - the effect and miss strings, including pluralization.
- **Editing semantics.** New file `tests/test_prompt_normal_mode_search_operator.py`,
  using `sase.ace.testing.PromptPage` like
  `tests/test_prompt_normal_mode_char_search.py`. Cover:
  - `d/foo` + enter;
  - `d?foo`;
  - `c/foo` then typing and Esc;
  - `y/foo`: text unchanged, register kind charwise, cursor placement;
  - `gU/foo`;
  - `>/foo` on multi-row ranges;
  - `2d/foo` and `d2/foo`;
  - the linewise case, with a linewise register;
  - Esc and Ctrl+C cancel: text, cursor, and pending operator all restored and cleared;
  - Enter with no target and with a too-large count: no edit;
  - no wrap;
  - `u` restores the text in one step;
  - `.` repeat for `d/`, `c/` (with inserted text), and a counted `3.`;
  - `.` when no target remains: no edit;
  - `dn` / `dN` after `/foo` and after `*` (whole-word flags respected);
  - `dn` with no register;
  - regressions: `dt/` and `df/` still delete up to a literal slash, and plain `/` and
    `n` are unchanged.
- **Bar and panel.** New file `tests/ace/tui/widgets/test_prompt_search_operator.py`,
  modeled on the `_PromptSearchApp` harness in
  `tests/ace/tui/widgets/test_prompt_search_interactive.py`. Cover:
  - The panel shows the chip, `d/`, the query, the `delete to match` title, the
    `[enter] delete` subtitle, and the `operator-destructive` class.
  - For `y`, the panel has the `operator-yank` class.
  - The effect summary and the pane-local count are shown.
  - The miss messages are shown.
  - The region span and style name are right (`_search_operator_region`).
  - Other matches are highlighted and the preview cursor sits on the target.
  - Keys do not bubble to the app `slash` / `question_mark` bindings.
  - Enter does not submit the prompt.
  - The committed-search pill is absent during and after an operator search.
  - The register is recorded, so a following `n` works.
  - Operator classes are removed on hide.
  - A multi-pane stack where the pattern exists only in another pane reports
    `pattern not found` and edits nothing.
  - `render_search_command_line` with a `prefix` (unit-level), including narrow-width
    degradation.
- **PNG goldens.** New file
  `tests/ace/tui/visual/test_ace_png_snapshots_prompt_search_operator.py`, following the
  existing search snapshot at
  `tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py` (the
  `prompt_search_highlight_120x40` test) with `mount_prompt_bar` and `SEARCH_PROMPT`.
  Add two snapshots:
  - `prompt_search_operator_delete_preview_120x40` (textual-dark): a multi-row `d/final`
    preview from `alpha` on line 1.
  - `prompt_search_operator_yank_reverse_light_120x40` (textual-light): a `y?alpha`
    preview showing the green region and the gold, unstruck match.

## Verification

Follow the lint/test memory (`sase memory read lint_and_test.md`). Run `just install` if
the workspace venv is stale, then `just fix` (or `just fmt`), then
`sase tool run check`, which falls back to `just check`.

Generate and inspect the new goldens with a targeted
`just fix-tui-screenshots -- <new visual test file>`, through `/sase_monitor` if it runs
long. Also run the help-modal visual selectors, because the new help rows may shift
those goldens. Inspect every created or updated PNG in the retained report before
accepting it.

Optionally, confirm the real UX with `sase screenshot` (read `tui_screenshot.md` first):
press `Escape`, `g g`, `w`, `d`, `/`, `f i n a l`. Check that the red, struck-through
region stops just before `final`, that the panel border is red, and that the effect
summary is accurate.
