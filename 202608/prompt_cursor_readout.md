---
tier: tale
title: Show a cursor line/column readout for every prompt input pane
goal:
  Every prompt input pane advertises its cursor position — the active pane live on the bar's bottom
  border, each parked pane on its own separator rule — so the column is always visible and an
  unfocused pane's cursor is no longer invisible.
proposed_by: bbugyi200.athena.up
create_time: 2026-08-07 09:51:42
status: done
---

- **PROMPT:**
  [prompts/202608/prompt_cursor_readout.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/prompt_cursor_readout.md)
- **AGENTS:**
  - [bbugyi200.athena.up](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.up.md)
- **COMMITS:**
  - [6399389](https://github.com/sase-org/sase/commit/6399389e5994a9fef98dceb34f0452b350f36c5c) —
    feat(ace): show a cursor line/column readout for every prompt input pane

# Plan: Show a cursor line/column readout for every prompt input pane

## Problem

The ACE prompt input bar (`PromptInputBar`, one or more `PromptTextArea` panes) never shows the
cursor's **column**. It shows the line only implicitly:

- The gutter appears only when a pane has more than one line
  (`show_line_numbers = document.line_count > 1`).
- In NORMAL mode the gutter is **relative**; only the cursor row carries an absolute number.
- `highlight_cursor_line=False`, and an unfocused pane draws no cursor cell at all.

So in a multi-pane prompt stack, a parked pane's cursor is completely invisible — yet pane
navigation (`gj`/`gk`, `focus_relative`) is deliberately "a pure focus change; no pane is rebuilt,
so each pane keeps its cursor and edit state". The user returns to a pane with no idea where its
cursor will land.

## Design: the Cursor Readout

One readout per pane, rendered in a single shared visual language.

### 1. Placement — one home per role

| Pane role                    | Where the readout lives                               | Why                                        |
| ---------------------------- | ----------------------------------------------------- | ------------------------------------------ |
| Active (stack-selected) pane | Flush right on the bar's **bottom border** (subtitle) | One fixed screen home; never moves         |
| Parked (unselected) panes    | Right end of that pane's **separator rule**           | Rides the rule that already names the pane |

Rationale — this is the core design decision:

- The **live** readout answers "where is my cursor right now?", asked constantly. It gets a single,
  permanently stable screen position: the bottom-right of the bar. It does not move when you switch
  panes, reorder panes, add a pane, or go solo. Your eye learns one spot.
- A **parked** readout answers "where did that pane leave off?", asked while scanning the stack. It
  belongs next to the pane it describes, on the `─── ▍ agent 2 ───` rule that already identifies
  that pane.
- Exactly one readout per pane, never two. The active pane's separator stays byte-for-byte
  unchanged; its bright `▍ agent N` marker already says "this one is live", and its position is on
  the status line.
- A solo pane is just the active pane with no separators, so it needs no special case: its readout
  is on the bar border like every other active pane.

**Costs zero rows.** Both surfaces already exist and are already painted.

### 2. Format — identical everywhere

```
Ln 3, Col 12
```

Spelled out, not `3:12` (reads as a clock) and not `3,12` (reads as a number pair). Same string in
both surfaces: same words, same order, same meaning. Only the _emphasis_ differs.

Both numbers are **1-based** (`cursor_location` is 0-based, so add 1 to each). Column is the
**document** column — the character index within the logical line, not the soft-wrapped screen
column and not a cell offset. That matches what "column" means when editing text, and it stays
stable when the terminal is resized. Double-width characters therefore count as one column; that is
an accepted simplification (vim's `col` vs `virtcol` distinction is out of scope).

Empty pane → `Ln 1, Col 1`.

### 3. Color — the numbers are painted the color of that pane's cursor

This is what ties the feature into the existing visual system instead of bolting a widget onto it.
`PromptTextArea` already colors the cursor cell and the gutter by vim mode (see `styles.tcss`
`.text-area--cursor` rules and `_line_rendering.py`):

| Pane's vim mode          | Color     |
| ------------------------ | --------- |
| `normal`                 | `#D0A215` |
| `insert`                 | `#3AA99F` |
| `visual` / `visual_line` | `#CE5D97` |

The readout's **digits** use that same palette, keyed off the _pane's own_ `_vim_mode`. The literal
words `Ln`, `Col` and the comma stay dim/muted. The rule the user learns in one glance: _the numbers
are the same color as the cursor they describe._ Parked readouts render on a separator that CSS
already dims (`.prompt-stack-separator.inactive { text-style: dim; }`), so they recede without any
extra styling logic.

### 4. Graceful degradation — explicit, opposite priorities

- **Subtitle:** the readout wins. If the mode hints plus the readout exceed the usable border-label
  width, truncate the _hints_ (with an ellipsis) and keep the readout flush right. Justification:
  every hint in the subtitle is also reachable from the `?` help modal and the `g`-prefix hint
  panel, so it is redundant; the readout has no other home.
- **Separator rule:** the `agent N` label wins. If the rule cannot fit the readout without eating
  into the centered label, **omit the readout entirely** — never abbreviate it to a second format.
  Justification: `agent N` identifies the pane and has no other home.

### 5. Non-goals

- No config toggle. The readout costs zero rows and no measurable work; a setting would be more
  surface area than the feature.
- No `virtcol` / screen-column reporting, no selection-length readout, no byte offsets.
- Scope is `PromptTextArea` panes inside `PromptInputBar`. Other `VimTextArea` surfaces
  (`SingleLineVimTextArea`, the frontmatter raw editor, config-tab editors) are **out of scope**: a
  single-line input has no meaningful line dimension, and the config editors are a different surface
  with different chrome. Worth revisiting separately if the user wants it.

## Implementation

### New module: `src/sase/ace/tui/widgets/_prompt_cursor_readout.py`

Keep the formatting **pure** so it is unit-testable without a running app. Roughly:

- `CURSOR_READOUT_MODE_COLORS: dict[str, str]` — the mode → hex table above, defaulting to the
  insert color for unknown modes (mirrors `LineRenderingMixin._VIM_MODE_CLASSES`'s default).
- `cursor_readout_position(text_area) -> tuple[int, int]` — read `text_area.cursor_location`, return
  1-based `(line, column)`. `cursor_location` is already the selection's cursor end, so VISUAL mode
  needs no special handling.
- `format_cursor_readout(line, column, *, vim_mode, dim=False) -> Text` — build a `rich.text.Text`
  with `Ln `/`, Col ` spans dim and the digit spans in the mode color. Returns `Text`, not a markup
  string (see the markup hazard below).
- `cursor_readout_cell_width(line, column) -> int` — `cell_len` of the rendered string, for the fit
  checks.

### Separator: `_PromptStackSeparator` in `_prompt_input_bar_stack_rendering.py`

- Add `position: tuple[int, int] | None = None` and `vim_mode: str = "insert"` state, plus
  `set_position(position, vim_mode)` that **no-ops when nothing changed** and otherwise calls
  `refresh()` (same shape as the existing `set_active`).
- In `render()`, leave the existing centered-label math **untouched** — `left_width` and
  `right_width` must keep coming from the full width, so the `agent N` label never shifts when a
  readout appears, disappears, or changes digit count. Then, when `position is not None`, spend part
  of the _right_ rule run on the readout:

  ```
  chip_cells = cursor_readout_cell_width(...) + 2   # one space each side
  if right_width >= chip_cells + 2:                 # keep >= 1 rule cell on each side
      right rule becomes: "─" * (right_width - chip_cells - 1) + " " + readout + " " + "─"
  else:
      unchanged (readout omitted)
  ```

  The active pane always passes `position=None`, so its rule renders exactly as it does today.

### Bar: subtitle composition

Only three places write the bar's own `border_subtitle`, and they must all route through one
composer:

- `_prompt_input_bar_completion_panel.py:573` `set_prompt_mode_subtitle`
- `_prompt_input_bar_completion_panel.py:579` `show_soft_completion`
- `_prompt_input_bar_completion_panel.py:589` `hide_soft_completion`

(`_prompt_text_area_bar.py:104` sets `bar.border_subtitle` only as a fallback when the setter is
missing; leave that path but let the composer own the normal path.)

Add a single `_render_subtitle(base: str) -> Text` on the bar that appends `  ·  ` plus the active
pane's readout, width-aware:

```
usable = max(0, self.size.width - 6)   # border corners + label padding; VERIFY empirically
```

`textual/_border.py::render_border_label` truncates the label **from the right with an ellipsis**
(`text_label.truncate(width - cells_reserved, ellipsis=True)`), which is precisely why a naively
appended readout would be the first thing lost on a narrow terminal. Do not trust the `- 6`
arithmetic — pin it with the narrow-width test listed below and adjust the constant to whatever the
renderer actually allows.

If `base + divider + readout` does not fit, truncate `base` (ellipsis) to
`usable - len(divider) - readout_width`; if even the readout alone does not fit, emit the truncated
readout and drop `base`.

**Markup hazard — build `Text`, not a markup string.** Existing subtitles are plain strings such as
`"[Enter] send  [Esc] normal  [^C] cancel"`, which survive today only because Textual leaves
unparseable tags literal, while `_refresh_title` has to escape `[` for the same reason. Assigning a
`rich.text.Text` to `border_subtitle` bypasses markup parsing entirely and lets the readout carry
real styles; the repo already does this (`modals/zoom_panel_events.py:42`). No current subtitle
string relies on markup, so this conversion is safe. Verify the `Text` renders correctly under
Textual 8.0.1 and fall back to `textual.content.Content` if it does not.

### Bar: refresh triggers

Add `refresh_cursor_readouts()` on the bar that (a) recomputes the active pane's readout into the
subtitle and (b) sets/clears each separator's `position`. Call it from:

| Site                                                                          | Why                                                                            |
| ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `on_text_area_selection_changed` (`_prompt_input_bar_stack_rendering.py:485`) | Cursor moved                                                                   |
| `on_text_area_changed` (same file, `:468`)                                    | Belt-and-braces for edits                                                      |
| `_apply_active_classes` (`:233`)                                              | Covers `focus_item`, `focus_relative`, `on_descendant_focus`, `_after_rebuild` |
| `PromptInputBar.on_mount`                                                     | Initial paint, after `_cursor_to_end`                                          |
| `on_resize` (`:622`)                                                          | Width changed → truncation budget changed                                      |

Vim mode changes need no new call site: `_update_vim_mode_display` (`_prompt_text_area_bar.py:69`)
already routes through `set_prompt_mode_subtitle`, so recoloring falls out of the composer.

Guard every pane lookup with the same `try/except` idiom the surrounding code uses — the bar is
routinely asked to refresh while panes are mid-mount or mid-detach.

### Performance

Read `sase/memory/tui_perf.md` before starting. This work stays inside the rules:

- All of it is O(width) string building on the UI thread — no I/O, no workers, no timers.
- A cursor readout is in the same class as a highlight move: it must paint immediately, never
  debounced or deferred behind the activity gate (rule 7 / rule 13).
- Parked separators only re-render when their value actually changes, via the `set_position` no-op
  guard — so typing in the active pane never repaints another pane's rule.
- Do not add a new refresh path; reuse `refresh()` on the separator and the existing subtitle
  assignment.

### Docs

Add a short **Cursor Readout** subsection to `docs/ace.md` under `## Prompt Input Widget` (line
~3435), before `### INSERT Mode (Default)`: what the readout shows, that it is 1-based and
document-column, where it lives for active vs parked panes, and that the digits are painted in the
pane's vim-mode cursor color. Add one sentence to `### Prompt Stacks` (line ~3601) next to the
existing "Inactive panes stay compact" text noting that each parked pane's rule carries its cursor
position.

No help-modal or `default_config.yml` change: this adds no keybinding and no option.

## Tests

### Pure unit — `tests/ace/tui/widgets/test_prompt_cursor_readout.py` (new)

- 1-based conversion: `(0, 0)` → `Ln 1, Col 1`; `(2, 11)` → `Ln 3, Col 12`.
- Mode → color mapping for `normal` / `insert` / `visual` / `visual_line` / unknown.
- `cursor_readout_cell_width` matches the rendered `Text`'s `cell_len` for 1-, 2-, and 3-digit
  values.

### Widget — `tests/ace/tui/widgets/test_prompt_input_bar_cursor_readout.py` (new)

Use the existing `_PromptBarApp` harness from
`tests/ace/tui/widgets/_prompt_input_bar_stack_helpers.py`.

- Solo pane: subtitle ends with `Ln 1, Col 1`; moving the cursor updates it; typing updates the
  column.
- Solo pane in `mode="feedback"`: readout present (proves it is not prompt-mode-only).
- Multi-pane: exactly `len(stack) - 1` separators carry a readout; the stack-selected pane's
  separator carries none.
- `focus_item` moves the readout: the previously active separator gains one, the newly active
  separator loses one, and the subtitle now reports the new pane's position.
- Parked position is truthful across a focus round-trip: move the cursor in pane 2, focus pane 1,
  assert pane 2's rule reports pane 2's actual `cursor_location`, not `Ln 1, Col 1`.
- Soft completion visible (`show_soft_completion`) → the readout is still present in the subtitle;
  `hide_soft_completion` restores hints plus readout.
- Vim mode switch recolors the digits (assert on the `Text` spans, not on a rendered screenshot).
- **Narrow terminal** (e.g. `size=(44, 24)`): the subtitle still ends with the full readout while
  the hints are truncated, and the separator drops the readout while keeping `agent N` intact. This
  test is what pins the `usable` width constant — if it fails, the constant is wrong, not the test.
- `set_position` with an unchanged value does not trigger a repaint (assert the no-op guard, e.g. by
  spying on `refresh`).

### Visual PNG snapshots — expect broad, intentional churn

**Every** golden that shows the prompt bar changes: the subtitle gains the readout, and stacked
goldens gain parked readouts on their rules. That is a large slice of
`tests/ace/tui/visual/snapshots/png/` — the whole `prompt_*` family plus frontmatter panel,
completion panels (artifact/model/placeholder/vcs/word/history), preview panel, jump action, word
lookup, finder, inputs, and xprompt-save goldens.

Procedure:

1. `just test-visual` and read the failure list.
2. Inspect `.pytest_cache/sase-visual/` actual/expected/diff artifacts and confirm **every** diff is
   only the readout. Any golden that changes in some other way is a real regression — most likely
   the separator label shifted, which means the centered-label math was not left alone.
3. Re-run with `--sase-update-visual-snapshots` to accept.
4. Add two new goldens with fixtures in `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py`:
   - `prompt_cursor_readout_solo_120x40` — solo pane, cursor parked mid-document (not at the end),
     NORMAL mode, so the gold digits and the relative gutter are both visible.
   - `prompt_cursor_readout_stack_120x40` — three panes, each parked at a different position, so the
     two parked rules show different values while the active readout sits on the bar border.

## Verification

```bash
just install        # ephemeral workspace — required before anything else
just check          # whole-repo lint gates + diff-scoped tests
just test-visual    # PNG snapshot suite (excluded from just check / just test)
just check-full     # before landing
```

Then run `sase ace`, open a prompt (`,` leader → prompt), and confirm by hand:

- Solo pane: readout bottom-right, tracks typing and `hjkl` motion, recolors gold/cyan/magenta as
  you switch NORMAL/INSERT/VISUAL.
- `g-` to add panes, move each pane's cursor somewhere distinct, then `gj`/`gk` between them: each
  parked rule keeps reporting that pane's real position, and the bar-border readout always reports
  the pane you are in.
- Shrink the terminal narrow enough to truncate: the bar-border readout survives; the separator
  readout disappears before the `agent N` label does.

## Done when

- Every mounted prompt pane advertises its cursor line and column exactly once — active on the bar
  border, parked on its own rule.
- Position values are 1-based document coordinates and stay truthful across pane focus changes,
  reorders, and rebuilds.
- Digits are painted in the owning pane's vim-mode cursor color.
- `agent N` labels do not shift when a readout appears or changes width.
- New unit, widget, and narrow-width tests pass; the visual suite is regenerated with every diff
  reviewed and attributable to the readout; `just check-full` is green.
