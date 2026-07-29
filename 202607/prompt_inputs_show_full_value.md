---
tier: tale
title: Prompt Inputs panel shows each field's full value at all times
goal: 'Every field in the "Fill in this prompt" panel (raw placeholders and declared inputs) soft-wraps and grows so the
  entire value the user typed is visible at all times, instead of collapsing into a scrollbar that hides the value
  completely.

  '
create_time: 2026-07-29 16:58:32
status: done
---

- **PROMPT:** [202607/prompts/prompt_inputs_show_full_value.md](prompts/prompt_inputs_show_full_value.md)

# Plan: Prompt Inputs panel shows each field's full value at all times

## Problem

Submitting a prompt that contains raw `<placeholder>` tags from the prompt input widget opens `InputCollectionModal`
("Fill in this prompt", `src/sase/ace/tui/modals/input_collection_modal.py`). Typing a value longer than the field makes
the value **completely invisible** — see `.sase/home/tmp/screenshots/20260729_164011.png`, where
`Make PreviewPanelModal a real reader — M, highest daily value per line changed` renders as two solid colored bars and
zero glyphs.

The colored bars are not text. They are scrollbars:

- Every field is a `SingleLineVimTextArea`, globally styled `height: 3` with a border (`src/sase/ace/tui/styles.tcss`,
  the `SingleLineVimTextArea` rule) — exactly **one** content row.
- `SingleLineVimTextArea.__init__` hardcodes `soft_wrap=False` (`kwargs.setdefault("soft_wrap", False)`), and `TextArea`
  is a `ScrollView`, whose `DEFAULT_CSS` sets `overflow-x: auto; overflow-y: auto`.
- So a value wider than the field turns on the horizontal scrollbar, which consumes the widget's only content row. The
  vertical scrollbar then turns on too (nothing fits vertically any more) and eats columns.

Measured in a headless `AcePage` with the screenshot's prompt and value (modal container 64 wide → field 54 wide, value
79 cells):

```
size=Size(54, 1)  virtual_size=Size(79, 1)  scrollable_content_region=Size(52, 0)
show_horizontal_scrollbar=True   show_vertical_scrollbar=True   scroll_offset=Offset(27, 0)
```

`scrollable_content_region` height `0` is the bug: the value has nowhere to render. The wide orange run is the
horizontal scrollbar thumb, the maroon run its track, and the orange sliver on the right edge is the vertical scrollbar.

Even without that pathology a one-row, non-wrapping field could only ever show a ~52-character window of the value,
which fails the requirement that the _full_ value be visible at all times.

## Approach

Presentation-only Textual/CSS work in this repo — no Rust core boundary crossing. Every measurement below was taken
against the pinned `textual==8.0.1` with the real modal driven through `sase.ace.testing.AcePage`, so the follow-up
should reproduce these numbers as it goes.

### 1. Make the panel's fields soft-wrap and auto-grow

- `input_collection_modal.py`: give `_InputCollectionInput` an `__init__` that does
  `kwargs.setdefault("soft_wrap", True)` before `super().__init__(...)`. `SingleLineVimTextArea.__init__` uses
  `setdefault` for `soft_wrap`, so the subclass value wins. `_PathField` subclasses `_InputCollectionInput` and inherits
  this, so placeholders, required inputs, optional inputs, and `path` fields all behave the same.
- `styles.tcss`: add `height: auto;` to the existing `InputCollectionModal SingleLineVimTextArea` rule (it already sets
  `width: 100%; margin: 0`). That selector outranks the global `SingleLineVimTextArea { height: 3; }` rule, and
  `ScrollView.get_content_height()` returns `virtual_size.height`, which `TextArea._refresh_size()` sets to
  `wrapped_document.height` when soft-wrapping — so the box tracks the wrapped row count.

Verified behavior after this change:

| value                          | field height                                       | scrollbars |
| ------------------------------ | -------------------------------------------------- | ---------- |
| empty                          | 1 content row (3 with border — identical to today) | none       |
| `short`                        | 1 content row                                      | none       |
| the screenshot's 79-cell value | 2 content rows, both fully rendered                | none       |
| that value repeated 4×         | 16 content rows                                    | none       |

Because empty and short fields keep exactly today's 3-row box, the existing PNG goldens for the freshly-opened and
error-state modal should not move.

- The document stays a single logical line: newline flattening, `o`/`O`/`ctrl+j` suppression, and Enter-submits all live
  in `SingleLineVimTextArea` and are untouched. Update that class's module/class docstring, which currently asserts "A
  one-line box never soft-wraps": the _document_ is one line; the _display_ may wrap, and `soft_wrap` is a supported
  host-configurable knob.
- Two wrap side effects to keep and document in the docstring:
  - `ctrl+a` / `ctrl+e` and vim `0` / `$` stay logical-line based, because `VimTextArea.action_cursor_line_start` /
    `action_cursor_line_end` navigate via `self.document.get_line(row)` rather than Textual's wrap-aware navigator.
  - NORMAL-mode `j` / `k` now move between wrapped rows of the single logical line (Textual's navigator is wrap-aware).
    That is the right behavior in a wrapped box; it was previously a no-op.
  - Spot-checked with the value loaded: `escape` → NORMAL, `0`, `w`, `dw` all edit correctly, Enter still submits and
    advances, and the modal stays open.

### 2. Keep the growing field's cursor row visible

`TextArea.scroll_cursor_visible()` calls `scroll_to_region()` on the _text area itself_; Textual never propagates that
to ancestors. So as a field grows inside `VerticalScroll#input-fields` (`max-height: 24`), the newly added rows fall
below the fold. Measured with three placeholders and a 16-row value in the last one: `#input-fields` stayed at
`scroll_offset=(0, 0)` with `max_scroll_y=9`, and the cursor sat at screen `y=37` while the container window was
`y=7..30` — the user cannot see what they are typing.

Add a small modal helper that scrolls the fields container to the focused editor's cursor row, and call it from
`on_text_area_changed` (only when that editor has focus, so programmatic updates and unfocused validation do not move
the view) and, via `call_after_refresh`, at the end of `_focus_field`:

```python
fields = self.query_one("#input-fields", VerticalScroll)
window = fields.scrollable_content_region
target_y = editor.cursor_screen_offset.y - window.y + fields.scroll_offset.y
fields.scroll_to_region(Region(0, target_y, 1, 1), animate=False, immediate=True)
```

The screen→virtual conversion is required: `scroll_to_region` interprets its argument in the container's own scroll
space, and `editor.virtual_region` is relative to the editor's `#field-block-<idx>` parent, not to `#input-fields`.
Verified: the call above scrolls by exactly `(0, 7)` and lands the cursor at screen `y=30`, inside the window. Wrap the
lookups in the same defensive `try/except` style the rest of this modal uses so a missing node can never break typing.

Per the TUI performance rules this adds only synchronous O(1) geometry to a change handler — no I/O, no worker, no new
timer — and Textual only re-lays-out when the wrapped row count actually changes.

### 3. Safety net: a one-row editor must never surrender its content row

Add `scrollbar-size: 0 0;` to the global `SingleLineVimTextArea` rule in `styles.tcss`. Every host of this widget has a
1-row content area, so a scrollbar there always costs 100% of the visible text; a zero-size scrollbar costs nothing and
loses no information.

This deliberately keeps `overflow: auto` rather than switching to `hidden`, because `Widget.allow_horizontal_scroll`
requires `show_horizontal_scrollbar` — with `hidden` the cursor would stop scrolling into view. Verified on a
non-wrapping field with the same 79-cell value: the content row survives, the text renders, `show_horizontal_scrollbar`
stays `True`, and cursor-follow still works (`scroll_offset.x == 25` with the cursor at the end, `0` after `ctrl+a`).

This is the difference between "shows a window of the value" and "shows nothing" for the other single-line hosts (agent
name, tag, workspace, command-input modals, the AXE entry editor, and so on).

### Non-goals

- **Not** widening the 64-column `#input-collection-container`, and **not** touching the truncated
  `.placeholder-context` line visible in the screenshot (`…anges described in the "<section>" section of the artifac`).
  Both are separate asks. Note for later: wrapped row count scales inversely with panel width, so widening is the lever
  if the row count ever feels tall.
- **No** per-field height cap. The requirement is that the whole value shows at all times, so the field grows to fit and
  `#input-fields` (`max-height: 24`) plus the container's `max-height: 90%` absorb the overflow by scrolling.

## Files expected to change

- `src/sase/ace/tui/modals/input_collection_modal.py` — `_InputCollectionInput.__init__` soft-wrap default; cursor-row
  reveal helper wired into `on_text_area_changed` and `_focus_field`.
- `src/sase/ace/tui/styles.tcss` — `height: auto` on the `InputCollectionModal SingleLineVimTextArea` rule;
  `scrollbar-size: 0 0` on the global `SingleLineVimTextArea` rule.
- `src/sase/ace/tui/widgets/single_line_vim_text_area.py` — docstring/contract update only (single logical line vs.
  wrapped display, supported `soft_wrap` host config, which motions stay logical-line based).
- Tests listed below.

## Testing

- `tests/ace/tui/modals/test_input_collection_modal.py`:
  - A long placeholder value leaves `soft_wrap` `True`, both `show_horizontal_scrollbar` and `show_vertical_scrollbar`
    `False`, `size.height == wrapped_document.height > 1`, and
    `scrollable_content_region.height >= wrapped_document.height`.
  - The full value is actually rendered: joining `editor.render_line(y).text` for every row and normalizing whitespace
    contains the whole value (this is the assertion that fails on today's code, where the visible region is 0 rows
    tall).
  - Empty and short values still occupy exactly one content row — the regression guard for the unchanged PNG goldens.
  - Growth keeps the cursor visible: several placeholders, a very long value in the last one, then assert
    `#input-fields` scrolled and `editor.cursor_screen_offset.y` lies inside `#input-fields.content_region`.
  - Enter still advances to the next field and the last Enter confirms, with the dismissed `PromptInputValues` carrying
    the complete multi-row value (guards the single-logical-line invariant now that wrapping is on).
- `tests/ace/tui/widgets/test_single_line_vim_text_area.py`:
  - With `soft_wrap=True`, a linewise paste still flattens to one document line, Enter still posts `Submitted`, and `dw`
    / `0` / `$` / `ctrl+a` / `ctrl+e` still act on the logical line.
  - With the zero-size scrollbars and no wrapping, a value wider than the box still renders (content row preserved) and
    the cursor still scrolls horizontally into view.
- Visual snapshots (`just test-visual`):
  - Add a snapshot to `tests/ace/tui/visual/test_ace_png_snapshots_prompt_inputs.py` for a placeholder holding a
    multi-row value (new golden, e.g. `prompt_inputs_long_value_120x40.png`).
  - Re-run the existing goldens — `input_collection_modal_120x40.png`, `input_collection_modal_error_120x40.png`,
    `prompt_inputs_placeholders_only_120x40.png`, `prompt_inputs_mixed_literal_120x40.png` — which the measurements say
    should be unchanged (empty/short fields keep the same 3-row box). If any of them does shift, inspect the
    actual/expected/diff artifacts in `.pytest_cache/sase-visual/` before accepting anything with
    `--sase-update-visual-snapshots`.
- `just install` first (ephemeral workspace), then `just check`.

## Risks / constraints

- Wrapping is opt-in per host, so only this modal changes shape. The global `scrollbar-size: 0 0` does reach every
  single-line editor, but it only removes zero-information scrollbars and was verified not to break horizontal
  cursor-follow.
- `height: auto` on a `ScrollView` subclass relies on `ScrollView.get_content_height()` returning `virtual_size.height`
  and on `TextArea._refresh_size()` setting that from `wrapped_document.height` under soft wrap. Both verified against
  the pinned `textual==8.0.1`; a Textual bump must re-run the new tests.
- Layout feedback loop: when `#input-fields` gains its vertical scrollbar the field width drops (measured 54 → 52) and
  the value rewraps, which can add a row. It converges within one Textual layout pass, so tests must assert the settled
  state after `page.pause()` cycles rather than an intermediate frame.
- Keymaps, footer conditions, and the help popup are unaffected — no bindings are added or changed, so the ACE footer /
  help-modal conventions need no update.
