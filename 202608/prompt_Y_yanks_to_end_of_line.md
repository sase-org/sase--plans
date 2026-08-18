---
tier: tale
title: Make NORMAL-mode `Y` yank from the cursor to the end of the line
goal:
  Pressing `Y` in a vim NORMAL-mode text area (the ACE prompt input and its
  `VimTextArea` siblings) yanks from the cursor to the end of the line as a charwise
  register — matching Neovim's `Y` == `y$` — instead of yanking the whole line linewise.
size: small
proposed_by: bbugyi200.athena.05r
create_time: 2026-08-18 07:20:48
status: wip
---

# Plan: Make NORMAL-mode `Y` yank from the cursor to the end of the line

## Current behavior

`Y` is implemented as a plain `yy` synonym: it yanks whole lines, linewise, ignoring the
cursor column.

`src/sase/ace/tui/widgets/_vim_normal_editing.py:145`:

```python
if key == "Y":
    cur_row = self.cursor_location[0]
    last_row = min(cur_row + count - 1, self.document.line_count - 1)
    self._execute_linewise_operator(cur_row, last_row, "y")
    return True
```

So with `abc def` and the cursor on `d`, `Y` currently stores `"abc def"` with
`kind="linewise"`, copies `"abc def"` to the system clipboard as `"yanked lines"`, and
flashes the entire line. A following `p` then opens a **new line** below rather than
pasting inline, because the register is linewise.

This is the pre-0.6 Vim default. Neovim (and `vim-sensible` before it) define `Y` as
`y$`, which is what this plan implements. It also makes `Y` the yank sibling of the `C`
and `D` handlers that already live 4 lines below it in the same function and already
operate on `(row, col)` → `(row, len(line))`.

## Where the change lives (and what else it reaches)

`Y` is **not** configurable — it is hardcoded in the vim NORMAL-mode dispatch, not in
`src/sase/default_config.yml`. (The three `"Y"` entries in `default_config.yml` are
unrelated pane keymaps: `copy_source_path`, `files_copy_path`, `sync`.) So there is no
keymap-config change in this plan.

The handler lives in `VimNormalEditingMixin`, which reaches every vim text area, not
only the prompt input:

```
VimNormalEditingMixin  (_vim_normal_editing.py)
  └── VimNormalModeMixin  (_vim_normal.py)
        └── VimTextArea  (vim_text_area.py)
              ├── PromptTextArea            — the ACE prompt input (the reported case)
              ├── AxeValueTextArea          — axe entry editor
              └── SingleLineVimTextArea     — modal inputs (rename patch, model input,
                                              command input, xprompt name, group name,
                                              memory review, …)
```

**Change `Y` uniformly for all of them.** Two reasons: the whole point is muscle memory
consistent with the user's Neovim, and a per-widget divergence would mean `Y` means
different things in two panes of the same TUI. On a `SingleLineVimTextArea` the
practical effect is that `Y` yanks the tail of the single line instead of the whole line
— the same rule, just applied to a one-line buffer.

Visual mode is unaffected: `_vim_visual_keys.py` has no `Y` branch, so visual `Y` never
reached this code and stays as-is. `yy`, `V`+`y`, `yae`, `y{motion}` are all untouched.

## Decisions

1. **Charwise, not linewise.** `Y` routes through
   `_execute_charwise_operator(..., "y")`, so the register kind becomes `charwise`. This
   is the user-visible payoff beyond the shorter text: a following `p`/`P` pastes inline
   at the cursor instead of opening a line. Do not special-case the kind.

2. **Count follows Vim, not the local `C`/`D` precedent.** `{count}Y` == `{count}y$` ==
   charwise from the cursor through the end of the `(count-1)`-th line **below** the
   cursor, clamped to the last row. So on `aaa\nbbb\nccc\nddd` with the cursor at
   `(1, 0)`, `2Y` yanks `"bbb\nccc"` charwise.

   The sibling `C`/`D` handlers in this same function silently drop their count (they
   always use `(row, col)` → `(row, len(line))`). Diverging from them is deliberate:
   honoring the count costs two lines, `_get_text_in_range` and
   `_execute_charwise_operator` already handle multi-row charwise ranges correctly, and
   swallowing a typed count is a wart worth not copying. Bringing `C`/`D` in line is
   listed as a follow-up below, not done here.

3. **Empty range is a no-op.** With the cursor already at end-of-line (or on an empty
   line) and no count, `start == end`, and `_execute_charwise_operator`'s existing
   `start == end` guard returns before storing anything — the register keeps its prior
   contents, nothing is copied to the clipboard, and there is no flash. That is exactly
   how `D` behaves at end-of-line today; keep it rather than adding a special case to
   store an empty register.

4. **Cursor does not move.** The charwise yank path ends with
   `self.cursor_location = start`, and `start` is where the cursor already was, so `Y`
   leaves the cursor put — same as today.

5. **No feature flag.** Per `sase/memory/sase_flags.md`, a flag is for behavior reaching
   users _before it is ready_ (`beta`/`wip`) or for keeping a superseded branch
   reachable during migration (`sunset`). This ships ready and complete on the first
   commit, and there is no old `Y` branch anyone needs to keep reaching — `yy` already
   does exactly what old `Y` did, so the previous behavior remains one keystroke away
   with no code to preserve. Do not create a flag.

## Change 1 — the handler

File: `src/sase/ace/tui/widgets/_vim_normal_editing.py`, the `if key == "Y":` branch at
line 145 (leave it where it is, immediately above the `C`/`D` branch).

Replace the linewise call with a charwise one whose range runs from the cursor to the
end of the target line:

```python
if key == "Y":
    # Vim/Neovim ``Y`` is ``y$``, not ``yy``: yank the tail of the line
    # charwise so a following ``p`` pastes inline. ``yy`` remains the
    # linewise whole-line yank.
    row, col = self.cursor_location
    last_row = min(row + count - 1, doc.line_count - 1)
    self._execute_charwise_operator((row, col), (last_row, len(doc.get_line(last_row))), "y")
    return True
```

Notes for the implementer:

- `doc = self.document` is already bound at the top of `_handle_normal_edit_key`; use it
  (the `C`/`D` branch below already does) rather than re-reading `self.document`.
- Do **not** set `self._mutation_count`. `Y` is a yank, not a mutation; the existing
  code does not set it, and `_execute_charwise_operator`'s yank branch already ends with
  `self._mutation_key_buffer.clear()` so `Y` stays out of the dot-repeat register.
- No new symbols are introduced, so there is no Symvision surface here.
- Wrap the call to fit the line-length limit; `just fmt` will settle the exact layout.

Everything downstream is already correct for this range and needs no edit:
`_get_text_in_range` joins multi-row charwise ranges with `\n`; `_store_vim_register`
records `charwise`; `schedule_copy_delivery` copies with `copied_label="yanked text"` /
`task_name="sase-copy-vim-charwise-yank"`; `_flash_yank(start, end)` highlights exactly
the copied span.

`_execute_linewise_operator(..., "y")` is still reached from `yy`, `V`-mode yank, and
`yae`, so nothing becomes dead code.

## Change 2 — tests

### 2a. `tests/test_prompt_normal_mode_yank_paste.py`

Replace `test_Y_yanks_counted_lines` (lines 61–69) — its name, docstring ("Y is a yy
synonym"), and `kind == "linewise"` assertion all encode the old contract. Cover:

- **`test_Y_yanks_from_cursor_to_end_of_line`** —
  `PromptPage("aaa\n  bbb ccc\nddd", cursor=(1, 6))`, press `"Y"`. Expect `page.text`
  unchanged, register text `"ccc"`, `kind == "charwise"`, `page.cursor == (1, 6)`.
- **`test_Y_at_column_zero_yanks_whole_line_charwise`** — same buffer, `cursor=(1, 0)`,
  press `"Y"`. Register text is the full `"  bbb ccc"` but `kind == "charwise"`. This is
  the test that pins the difference from `yy`, whose sibling test
  (`test_yy_yanks_current_line_without_moving_cursor`, unchanged) asserts `"linewise"`
  for the same line.
- **`test_Y_with_count_spans_through_end_of_last_line`** —
  `PromptPage("aaa\nbbb\nccc\nddd", cursor=(1, 0))`, press `"2", "Y"`. Register text
  `"bbb\nccc"`, `kind == "charwise"`, cursor `(1, 0)`, buffer unchanged. (Same text as
  the old test asserted — only the kind changes — which is a useful signal that the
  count still means what it meant.)
- **`test_Y_at_end_of_line_leaves_register_untouched`** — the empty-range no-op. Prime
  the register first (e.g. `PromptPage("one two\n\nthree")`, press `"y", "w"` so the
  register holds `"one "`), then set `page.cursor = (1, 0)` (the empty line) and press
  `"Y"`. Assert the register still holds `"one "` — asserting `== ""` would be
  indistinguishable from `VimRegister`'s default (`text=""`, `kind="charwise"`) and
  would pass even if the guard broke.
- **`test_Y_register_pastes_inline`** — the behavioral payoff.
  `PromptPage("abc def\nxyz", cursor=(0, 4))`, press `"Y"` (register `"def"`, charwise),
  move to the second line, press `"p"`, and assert the paste landed **inside** that line
  rather than opening a new one. Contrast with the existing
  `test_p_pastes_linewise_below_cursor_on_first_nonblank`.

Follow the module's existing conventions: `async def` with no decorator
(`asyncio_mode = "auto"` in `pyproject.toml`), `async with PromptPage(...) as page`,
`await page.press(...)`, assertions on `page.text` / `page.ta._vim_register` /
`page.cursor`.

### 2b. `tests/ace/tui/widgets/test_prompt_yank_highlight.py`

The `("aaa\nbbb\nccc\nddd", (1, 0), ("2", "Y"), (4, 11))` row at line 77 sits in
`test_linewise_yanks_flash_full_lines`. Move it to the
`test_charwise_yanks_flash_exact_copied_range` parametrize list above it, which takes an
extra `expected_register` column:
`("aaa\nbbb\nccc\nddd", (1, 0), ("2", "Y"), (4, 11), "bbb\nccc")`.

The expected span is unchanged at `(4, 11)` — old linewise `Y` flashed `(1, 0)`→`(2, 3)`
and new charwise `Y` flashes the same two points — so this move is a pure
reclassification, and the fact that the number does not move is the check that the
flashed span still equals the copied span. Leave the `("y", "y")` and `("V", "j", "y")`
rows in the linewise test.

### 2c. Sweep

`Y`-in-a-vim-text-area is covered only by those two files; verified by grepping `"Y"`
across `tests/`, `smoke/`, `demos/`, and `tools/`. The other `"Y"` hits are unrelated
pane/modal bindings (notification modal copy-path, plan-approval copy, artifact-files
modal copy, glossary/preview modals, agent metadata search, keymap defaults) and must
not be touched.

## Change 3 — docs

`docs/ace.md:5183`, in the NORMAL-mode **Operators** table:

```
| `Y`   | Yank entire line                                                     |
```

becomes a description of the new behavior — e.g.
`Yank from the cursor to end of line (charwise, like ``y$``)` — so it reads as the yank
sibling of the `D` ("Delete to end of line") and `C` ("Change to end of line") rows two
lines above it. Leave the `yy` row at line 5186 ("Yank entire line") exactly as-is; it
is now the only whole-line yank in the table, which is the point.

`docs/ace.md` is hand-maintained (not generated), and `CHANGELOG.md` is release-please
managed from conventional commits, so neither needs anything else. No other document,
help-modal section, or in-TUI hint describes this key.

## Verification

- `just install` first — these workspaces go stale — then `just check`.
- The docs table is prettier-formatted; run `just fmt` after editing `docs/ace.md` so
  the column padding stays aligned, and re-run `just check`.
- Targeted while iterating:
  `just test tests/test_prompt_normal_mode_yank_paste.py tests/ace/tui/widgets/test_prompt_yank_highlight.py`
  (or the equivalent `pytest` invocation for this repo).
- `just test-visual` is not expected to be affected — no renderer or styling change.
- Manual smoke in `sase ace`: in the prompt input, `<esc>` to NORMAL, put the cursor
  mid-line, press `Y`, confirm the flash covers only the tail of the line and the toast
  reads "yanked text"; then `p` on another line and confirm it pastes **inline** rather
  than opening a new line. Repeat once in a single-line modal input (e.g. the rename
  patch modal) to confirm the shared-mixin behavior is sane there too. Confirm `yy` is
  unchanged.

## Notes / proposed follow-ups (not in scope here)

- **`C`/`D` swallow their count.** `3D` and `3C` in this implementation only ever affect
  the current line, where Vim extends through 2 more lines. Decision 2 above
  deliberately does not copy that limitation into `Y`, which leaves the three handlers
  inconsistent with each other until `C`/`D` are fixed. Worth its own task bead.
- **Empty charwise yank does not clear the register.** Vim's `y$` on an empty line sets
  the unnamed register to empty; here the `start == end` guard leaves the previous
  contents intact. This plan keeps the existing convention (`D` behaves the same way)
  rather than changing shared operator semantics for one key. Separate bead if it ever
  matters.
- **Visual-mode `Y`.** Vim's visual `Y` yanks the selected lines linewise; this codebase
  has no visual `Y` branch at all, so the key does nothing there. Unchanged by this
  plan, but a reasonable gap to fill later.
