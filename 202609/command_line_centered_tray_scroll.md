---
tier: tale
title:
  "Command Line panel: centered, completion tray below the input, ctrl+d/ctrl+u output
  scrolling"
goal:
  The `:` Command Line panel is a larger, vertically centered panel whose completion
  popup sits in a reserved tray beneath the input bar, so it never covers command
  output, and ctrl+d/ctrl+u scroll the transcript by half a page from INSERT or NORMAL
  mode, with follow-tail by default.
size: medium
proposed_by: bbugyi200.apollo.2f
create_time: 2026-09-27 14:50:43
status: wip
---

# Command Line panel: larger and centered, a completion tray below the input, and `ctrl+d`/`ctrl+u` output scrolling

## Goal

Make the `:` Command Line panel (`src/sase/ace/tui/command_line/`) easier to use for
reading command output:

1. **Larger and centered.** Today the frame is a bottom-anchored drawer that is 96% of
   the terminal wide (at most 160 columns) and 65% tall. Make it a vertically centered
   panel that is 96% wide (at most **200** columns) and **80%** tall. `ctrl+t` (full
   height) keeps working.
2. **Completion never covers the output.** Today the completion popup card and the
   doc-peek float on an overlay layer over the transcript, docked just above the input
   row, so they hide the most recent output (the user's screenshot shows the RECENT
   popup covering the output of the command that just ran). Move them into a dedicated,
   fixed-height **completion tray beneath the input bar** (below the input row and the
   one-line signature/hint row). Keep the Helix-style alignment of the card with the
   slot being completed.
3. **Scroll the output with `ctrl+d` / `ctrl+u`.** Half-page scroll of the transcript,
   working while typing (INSERT) and in NORMAL mode. The transcript should also follow
   new output by default so the newest output is visible. Today nothing in the panel
   scrolls the transcript at all, so a long transcript leaves the newest block below the
   fold.

## Design decisions (already made; do not relitigate)

- **Always-reserved tray.** The tray keeps a fixed height whenever the panel is open,
  even when the popup is hidden (for example, after a free-form argument with no
  candidates). The popup opens and closes on almost every keystroke, and its natural
  height changes with the candidate count. An in-flow card sized to its content would
  move the input row up and down while the user types. A reserved area avoids that. This
  is the same trade-off as prompt_toolkit's `reserve_space_for_menu`.
- **Order inside the frame, top to bottom:** transcript (`1fr`), input row (3 rows),
  signature/hint row (1 row), completion tray. The signature row stays directly under
  the input it describes. The completion card is the first thing in the tray, so it
  starts right below the signature row.
- **Tray height** comes from a pure helper. It is at most one full popup card:
  `POPUP_MAX_VISIBLE_ROWS` (8) + 2 section headings + 1 footer + 2 card border = **13**
  rows. It never takes more than half of the rows left after the input and hint rows,
  and it never drops below 3 rows (a border plus one row). Examples:

  | Terminal         | Frame | Transcript | Tray |
  | ---------------- | ----- | ---------- | ---- |
  | 120x40           | 32    | 13         | 13   |
  | ~262x79 (user's) | 63    | 44         | 13   |
  | 80x24            | 19    | 7          | 6    |

  The Transcript and Tray columns are rows.

- **`ctrl+d` / `ctrl+u` take over the input's readline keys in this panel.** Textual's
  `TextArea` binds `ctrl+u` to delete-to-line-start (and `ctrl+d` to delete-right). The
  vim NORMAL mode in `_vim_normal_motions.py` also handles both keys. The Command Line
  input must intercept them first, in both modes. The keys are configurable, and setting
  them to `unbound` restores the old editing behavior. `ctrl+w` and vim NORMAL
  `d0`/`S`/`cc` remain available for clearing text.
- **Follow-tail through Textual's anchor.** Use `Widget.anchor()`, which Textual 8
  provides on `CommandLineTranscript` (a `VerticalScroll`). It keeps the view at the
  bottom as content grows, `scroll_relative` releases it, and reaching the bottom again
  restores it.

## Changes

### 1. Frame geometry: `src/sase/ace/tui/command_line/screen.py` (`DEFAULT_CSS`)

- `CommandLineScreen { align: center middle; }` (was `center bottom`).
- `#command-line-frame`:
  - Set `width: 96%; max-width: 200; height: 80%;`.
  - Drop `height: auto` and `max-height: 65%`. The frame already always fills its max
    height because the transcript is `1fr`, so a fixed percentage keeps that behavior
    explicit.
  - Drop `layers: base overlay`.
- `#command-line-frame.full-height { height: 100%; }`.
- Remove the `layer: base` / `layer: overlay` lines, since nothing overlays the
  transcript any more.
- Replace the `#command-line-popup-float` overlay rule (`layer: overlay; dock: bottom`)
  with:
  - a `#command-line-completion-tray` rule: `width: 100%; height: 13;` (reassigned at
    layout, see §3).
  - a plain `#command-line-popup-float { width: auto; height: auto; }`.
- Update the module docstring and the `CommandLineScreen` class docstring. Both say
  "bottom-anchored drawer" and "popup floating over the transcript".

### 2. Compose: `CommandLineScreen.compose`

Wrap the existing float in the new tray, which is yielded after the hint row:

```python
with CommandLineFrame(id="command-line-frame"):
    yield CommandLineTranscript(id="command-line-transcript")
    with Horizontal(id="command-line-input-row"):
        ...  # unchanged prefix + CommandLineInput
    yield Static(COMMAND_LINE_IDLE_HINT, id="command-line-hint-row")
    with Vertical(id="command-line-completion-tray"):
        float_ = Horizontal(id="command-line-popup-float")
        float_.display = False
        with float_:
            ...  # unchanged card (popup + footer) and doc-peek
```

The tray itself is never hidden. Only the float inside it toggles `display`.

### 3. Tray geometry

**`src/sase/ace/tui/command_line/popup_layout.py`** (keep it pure and Textual-free):

- Delete `POPUP_BOTTOM_RESERVE` and drop it from `__all__`.
- Add these constants and export them:
  - `INPUT_BLOCK_ROWS = 4`: the input row (3) plus the signature/hint row (1).
  - `COMPLETION_TRAY_MAX_ROWS = 13`: the 8-row window, 2 section headings, the footer,
    and the card border. Its comment must name `popup.POPUP_MAX_VISIBLE_ROWS`. Do not
    import `popup.py`, which pulls in Textual.
  - `COMPLETION_TRAY_MIN_ROWS = 3`.
- Add and export `completion_tray_rows(content_height: int) -> int`, which returns
  `max(COMPLETION_TRAY_MIN_ROWS, min(COMPLETION_TRAY_MAX_ROWS, (content_height - INPUT_BLOCK_ROWS) // 2))`.
- Update the module docstring. The popup now sits in the tray beneath the input, and its
  left edge still follows the completion slot.

**`src/sase/ace/tui/command_line/screen_popup_layout.py` (`_layout_popup`):**

- Also query `#command-line-completion-tray`.
- First, compute `tray_rows = completion_tray_rows(frame.content_region.height)`. Assign
  `tray.styles.height` only when the value changed; cache the last value on the screen
  (initialize it in `CommandLineScreen.__init__`) to avoid needless layout passes.
- Do this before the early return that hides the float, so the tray is sized even while
  the popup is hidden.
- Replace
  `room = max(frame.content_region.height - POPUP_BOTTOM_RESERVE, _MIN_CARD_ROWS)` with
  `room = tray_rows`.
- Stop setting the float's bottom margin, but keep
  `float_.styles.offset = (geometry.x, 0)` for the Helix column alignment.
- Keep `card.styles.height = min(card_rows, room)` and
  `peek.styles.max_height = min(_PEEK_MAX_ROWS, room)`.
- Remove `_MIN_CARD_ROWS`, which `COMPLETION_TRAY_MIN_ROWS` replaces.
- Update the module docstring: the popup no longer lives on an overlay layer.
- `on_command_line_frame_resized` already calls `_layout_popup`, so resizes and the
  full-height toggle re-size the tray. Keep the layout synchronous and in memory
  (tui_perf rule 1).

**`src/sase/ace/tui/command_line/popup.py`:** fix the docstrings that say the popup
"floats over the transcript, anchored just above the input". The module docstring and
the `CommandLinePopup` class docstring both say this. Behavior is unchanged.

### 4. Scroll keys: keymap plumbing

Add two configurable `ace.keymaps.command_line` actions.

- `src/sase/ace/tui/keymaps/app_keymaps.py` (`CommandLineKeymaps`): add
  `scroll_transcript_down: str = "ctrl+d"` and `scroll_transcript_up: str = "ctrl+u"`
  right after `clear_transcript`. While there, update the class docstring ("drawer").
- `src/sase/default_config.yml` (`ace.keymaps.command_line`): add both keys with the
  same defaults. This file is the source of truth for bundled defaults, per the Default
  Keymap Config gotcha.
- `src/sase/config/sase.schema.json` (`command_line` properties): add both as
  `"type": "string"` with these descriptions:
  - "Scroll the Command Line output down half a page"
  - "Scroll the Command Line output up half a page"
- `src/sase/ace/tui/keymaps/metadata.py` (`_COMMAND_LINE_BINDING_META`): add
  `("scroll_transcript_down", "Scroll Output Down")` and
  `("scroll_transcript_up", "Scroll Output Up")`. These become screen bindings, so the
  keys also work when focus is not on the input, for example after clicking the
  transcript.

### 5. Scroll behavior

**Screen methods** (`src/sase/ace/tui/command_line/screen_navigation.py`, beside
`action_toggle_full_height`):

- `scroll_transcript(direction: int) -> None`: a half-page scroll with explicit anchor
  handling. Textual's own `_check_anchor` only re-anchors on an actual `scroll_y`
  change, so pressing `ctrl+d` at the bottom would otherwise silently stop following
  output. Steps:
  - `step = max(1, transcript.scrollable_content_region.height // 2)`.
  - If `transcript.max_scroll_y == 0`, return. There is nothing to scroll, and the
    method must not release the anchor.
  - If `direction > 0` and `transcript.scroll_y + step >= transcript.max_scroll_y`, call
    `transcript.anchor()` (scroll to the end and follow again) and return.
  - Otherwise call `transcript.scroll_relative(y=direction * step, animate=False)`,
    which releases the anchor on the way up.
- `action_scroll_transcript_down()` / `action_scroll_transcript_up()` call
  `scroll_transcript(1)` / `scroll_transcript(-1)`.
- `_after_selection`: after `_refresh_transcript()`, scroll the newly selected block
  into view. Add `CommandLineTranscript.scroll_block_into_view(block_id)` in
  `transcript.py`: it finds the `_CommandLineBlockWidget` and calls
  `self.scroll_to_widget(widget, animate=False)`. Schedule it with
  `self.call_after_refresh(...)` so that a just-repainted widget has its region. The
  callback is thin and synchronous (tui_perf rule 2). Textual's `scroll_to_region` does
  nothing when the block is already visible, so the anchor survives j/k within the
  viewport.

**Follow-tail:**

- In `CommandLineScreen.on_mount`, call `self.transcript.anchor()` right after the first
  `self._refresh_transcript()`. The panel then opens at the newest output, and restored
  blocks that the restore worker appends later keep it pinned to the bottom.
- In `screen_submission.py`, re-follow after appending a new block: call
  `self.transcript.anchor()` after `_refresh_transcript()` in `_submit_line` and in
  `_add_local_block`. The `_add_local_block` path covers denied, foreground and built-in
  blocks. A newly run command is then always visible, even if the user had scrolled up.
  Wrap the call best-effort like the neighbouring UI calls.

**Input routing** (`src/sase/ace/tui/command_line/input.py`,
`CommandLineInput._on_key`):

- Right after `keymaps = command_line_keymaps_for(self)`, and before the menu, history
  and block-nav routing, check `scroll_transcript_down` / `scroll_transcript_up` with
  `binding_matches_key`.
- On a match, in either vim mode:
  - call `self.screen.scroll_transcript(±1)` (through `getattr`, following the existing
    `handler` pattern);
  - then `event.stop()`, `event.prevent_default()` and `return`.
- This must run before `super()._on_key(event)`, so the `TextArea` bindings for
  `ctrl+u`/`ctrl+d` and the vim NORMAL half-page motion never see the key. Textual 8
  forwards a Key to the focused widget's `_on_key` before it checks non-priority
  bindings; the existing `ctrl+f` menu intercept relies on the same order.
- An `unbound` action never matches, so the key falls through to the old editing
  behavior.
- Update the module docstring to mention the two keys.

### 6. Key hints: `src/sase/ace/tui/command_line/screen_constants.py`

- Add a `^D/^U scroll` part, built with `_compact_key_display` from the live keymaps:
  - render `"{down}/{up} scroll"` when both keys are bound;
  - render only the bound key when just one is bound;
  - omit the part when both are unbound.
- Put it before `esc hide` in both `command_line_input_hints` (giving
  `⏎ run · ⇥ complete · ↑↓ history · ^R search · ^D/^U scroll · esc hide`) and
  `command_line_block_hints`.
- The bottom border already ellipsizes the hints before it drops the running count, so
  no chrome change is needed.

### 7. Docs: `docs/ace.md` → `## Command Line`

- Replace "a bottom-anchored drawer" with a centered panel.
- Update the frame-size sentence: 96% wide, at most 200 columns, 80% tall, centered;
  `Ctrl+T` toggles full height.
- Rewrite the "completion popup floats over the transcript…" bullet:
  - The popup (and, on terminals at least 140 columns wide, the doc peek) sits in a
    completion tray beneath the input and signature line.
  - It never covers command output.
  - The tray keeps a fixed height (up to 13 rows), so the input never moves as
    candidates change.
  - The candidate text still lines up under the slot being completed.
- Add a bullet about scrolling:
  - The transcript follows new output.
  - `Ctrl+D` / `Ctrl+U` scroll it by half a page from the input in INSERT or NORMAL
    mode, taking over the input's delete-char / delete-to-line-start keys in this panel.
  - Scrolling back to the bottom (or running a command) resumes following.
- Mention `scroll_transcript_down` / `scroll_transcript_up` in the "Panel keys are
  configurable under `ace.keymaps.command_line`" paragraph.
- Run `just fmt` afterwards; it also formats Markdown.

## Tests

**Update `tests/ace/tui/command_line/test_chrome_layout.py`:**

- Module docstring: the popup now sits in a tray beneath the input.
- Replace `test_frame_width_cap_and_full_height_toggle_keep_the_labels`:
  - At `(240, 40)` the frame is 200 wide and 32 tall.
  - It is vertically centered: `frame.region.y == 4`, and the gaps above and below the
    frame are equal.
  - At `(100, 40)` it is 96 wide.
  - `ctrl+t` gives height 40 and toggles back to 32.
  - The border labels keep `outer_size.width - 6` cells.
- Replace `test_popup_floats_over_the_transcript_without_reflowing_it` with a test that
  the tray sits below the input and never covers the transcript:
  - the card's `region.y` is at or below the hint row's `region.bottom`;
  - `not card.region.overlaps(transcript.region)`;
  - the card is inside `frame.content_region`;
  - the frame, transcript, input row and tray regions are identical with the popup open
    (`"bead "`) and hidden (`"zzzzzz"`). The tray is reserved, so nothing moves.
- Keep the column-tracking, narrow-clamp, reclamp-on-shrink and doc-peek tests. They
  assert x alignment and containment, which should still hold; fix them only if the
  geometry numbers legitimately moved.
- Add pure tests for `completion_tray_rows`:
  - `30 → 13`, `17 → 6`, a tiny height `→ 3`;
  - `COMPLETION_TRAY_MAX_ROWS == POPUP_MAX_VISIBLE_ROWS + 5`, which ties the literal to
    the popup window.
- Add a pilot test at `(120, 40)`: with the full 8-row window plus headings (the
  empty-state RECENT/FOR rows, or a `bead ` completion), the card height is at most the
  tray height and fits inside it.

**New scroll pilot tests** (a new `tests/ace/tui/command_line/test_transcript_scroll.py`
or additions to `test_transcript_blocks.py`, reusing the `_panel`-style harness and
seeding blocks through `command_line_session_for(app).add_block` plus `tail_text` /
`expanded=True` so the transcript overflows):

- The panel opens anchored at the bottom:
  `transcript.scroll_y == transcript.max_scroll_y`.
- INSERT mode with a non-empty line such as `bead list`:
  - `ctrl+u` scrolls up by half the viewport, and the input text is unchanged, so
    `ctrl+u` did not delete to line start;
  - `ctrl+d` scrolls back down.
- After a `ctrl+u`, appending a block or growing a block's `tail_text` and repainting
  leaves `scroll_y` where it was (not following). A `ctrl+d` back to the bottom
  re-follows: a further append keeps `scroll_y == max_scroll_y`.
- `ctrl+d` pressed while already at the bottom keeps following. This is the explicit
  `anchor()` branch.
- `ctrl+u` on a transcript that does not overflow is a no-op, and a later overflow still
  follows.
- NORMAL mode (after `escape` with text in the line) has the same scroll behavior.
- Submitting a line after scrolling up re-anchors to the bottom. Stub the submit worker
  the same way the existing submit tests in `test_panel_shell.py` do.
- With `scroll_transcript_up="unbound"` (override the registry scope as
  `test_keymap_config._open_panel` does), `ctrl+u` in INSERT mode falls through to
  `TextArea` delete-to-line-start.
- NORMAL-mode `k` from the input selects the last block, and `g` jumps to the first
  block and scrolls it into view (its region overlaps the transcript's
  `scrollable_content_region`).

**Update the keymap tests:**

- `tests/test_keymaps_defaults_panels.py::test_default_config_covers_all_command_line_keymaps`:
  add both defaults.
- `tests/ace/tui/command_line/test_keymap_config.py`:
  - `test_build_command_line_bindings_skips_unbound` stays green, since both actions are
    in the metadata;
  - the schema-sync test stays green once the schema is updated;
  - extend `test_hint_builders_render_live_names_and_omit_unbound` to cover
    `^D/^U scroll` in the input and block hints and its omission when both are unbound.

**Visual goldens.** Every `tests/ace/tui/visual/snapshots/png/command_line_*.png` golden
changes because the frame moves and grows. Regenerate only this module:

```bash
just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_command_line.py
```

Read its report and inspect the updated PNGs:

- `command_line_completion_popup_160x40.png` and `command_line_empty_state_120x40.png`
  must show the card below the input and signature row, with the transcript area above
  it uncovered.
- The frame must be centered.

Fix any assertion in that module that pinned the old geometry. If the tool reports
`partial`, say so rather than assuming the goldens are current.

## Verification

- Run `sase tool run check`, which is the agent default; see `lint_and_test` memory. Do
  not run `check-full`.
- Take a live capture to confirm the real layout at a wide size, for example
  `sase screenshot -o /tmp/cmdline.png -p colon -w "Command Line"`. With a few commands
  run, check that:
  - the panel is centered;
  - the tray sits under the input;
  - `ctrl+u` / `ctrl+d` scroll the output.

## Out of scope

- Making the frame size configurable.
- Changing the popup's 8-row window, the zsh menu-select keys, or the doc-peek content.
- Rust core / `sase_core_rs` changes. This is presentation-only Textual layout and
  keybinding work, so it stays in this repo per the Rust-core boundary.
