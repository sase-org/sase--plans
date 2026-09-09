---
tier: tale
title: Make Ctrl+] a reliable INSERT-to-NORMAL key in the vim text areas
goal:
  Pressing Ctrl+] in INSERT mode always transitions the prompt input widget (and every
  other VimTextArea editor) to NORMAL mode, and keys typed immediately after it execute
  as NORMAL-mode commands instead of being inserted as text.
size: small
proposed_by: bbugyi200.athena.0hi
create_time: 2026-09-09 13:41:13
status: wip
---

# Make `Ctrl+]` a Reliable INSERT → NORMAL Key in the Vim Text Areas

## Problem

Bryan presses `<ctrl+]>` to leave INSERT mode for NORMAL mode in the prompt input
widget. It does not reliably take effect on the first press, so the next few keys —
intended as NORMAL-mode commands — get typed into the prompt as literal text.

## Root Cause (diagnosed, with evidence)

Two coupled facts explain the symptom:

1. **The app has no INSERT-mode handler for `ctrl+]` at all.** Every terminal encodes
   `ctrl+]` as the single C0 byte `0x1d`, which Textual names
   `ctrl+right_square_bracket` (`textual/_ansi_sequences.py` maps `"\x1d"` →
   `Keys.ControlSquareClose`). The only INSERT → NORMAL triggers in the vim layer are
   `event.key == "escape"` checks:
   - `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py` (INSERT-mode `escape`
     branch near the end of `_on_key`, currently ~line 296), and
   - `src/sase/ace/tui/widgets/vim_text_area.py` (base `VimTextArea._on_key` INSERT
     branch, currently ~line 186).

   Verified empirically with the `PromptPage` harness: in INSERT mode, pressing
   `ctrl+right_square_bracket` leaves `_vim_mode == "insert"` and a following `k` is
   inserted as text. The only existing `ctrl+right_square_bracket` handler is the
   NORMAL-mode jump-to-definition in `src/sase/ace/tui/widgets/_vim_normal.py` (~line
   67).

2. **The key that does switch modes — `escape` (and its byte-identical alias `ctrl+[`) —
   is inherently racy in terminals.** A bare ESC byte is ambiguous: it may be the Escape
   key or the start of an escape sequence. Textual's `XTermParser` resolves this by
   waiting `ESCAPE_DELAY` (100ms, `ESCDELAY` env) for more bytes
   (`textual/_xterm_parser.py`, the `read1(constants.ESCAPE_DELAY)` loop). Any key typed
   within that window is merged into a pending sequence: the `escape` key event is never
   emitted, the first following key surfaces as a spurious `alt+<key>` chord (ignored by
   the widget), and the remaining keys land as plain text in the still-INSERT-mode
   widget. Verified: feeding the parser `"\x1b" + "k"` in one chunk yields no `escape`
   event, while `"\x1d" + "k"` yields `ctrl+right_square_bracket` then `k` immediately
   and unambiguously.

   Bryan's stack (kitty → ssh → tmux 3.5a with `extended-keys off`) always delivers ESC
   as a bare `0x1b` byte to the TUI, so fast typing after an ESC-based mode switch loses
   the switch. No layer of the stack (kitty conf, tmux conf, hammerspoon, nvim) remaps
   `ctrl+]`, so today the literal `ctrl+]` chord reaches the app and is silently ignored
   in INSERT mode.

The fix follows directly: handle `ctrl+right_square_bracket` in the vim layer as a
first-class INSERT → NORMAL chord. Because `0x1d` is a single unambiguous byte, it is
parsed instantly with **no** disambiguation window — it can never merge with subsequent
keystrokes. Key events are dispatched to the widget sequentially and
`_enter_normal_mode()` runs synchronously inside `_on_key`, so every key pressed after
`ctrl+]` is processed after the mode flip and acts as a NORMAL-mode command. **No
replay/queueing machinery is needed — do not build any.**

## Requirements

- In INSERT mode, `ctrl+right_square_bracket` must ALWAYS transition to NORMAL mode, in
  the prompt input widget and in every other `VimTextArea`-based editor (the base widget
  change covers the single-line and secret variants, filter bar, frontmatter cells, AXE
  entry editor, etc.).
- Keys pressed after `ctrl+]` must be preserved and act in NORMAL mode (this falls out
  of the synchronous mode flip; cover it with a test, e.g. `ctrl+]` then `x` deletes a
  character instead of inserting `x`).
- `ctrl+]` in INSERT mode must behave exactly like `escape` with respect to open
  completion UI: route through the same `_enter_normal_mode()` branch so completion
  menus, soft completion, and xprompt arg hints are dismissed identically.
- NORMAL-mode `ctrl+]` (jump-to-definition) must remain unchanged.

## Implementation

1. `src/sase/ace/tui/widgets/vim_text_area.py` — in the INSERT-mode branch of `_on_key`,
   extend the `escape` check to also match `ctrl+right_square_bracket` (e.g.
   `if event.key in ("escape", "ctrl+right_square_bracket")`). Update the method
   docstring, which currently documents Escape as the only INSERT → NORMAL key. Do NOT
   add it to the two-stage NORMAL-mode escape logic — in NORMAL mode `ctrl+]` stays
   jump-to-definition.

2. `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py` — in the INSERT-mode
   `escape` branch of `_on_key` (the one commented "INSERT mode: Escape dismisses any
   active completion UI and enters NORMAL mode"), match `ctrl+right_square_bracket` as
   well. Update the comment.

3. `src/sase/ace/tui/widgets/_prompt_text_area_key_g_prefix.py` — in
   `_handle_insert_g_prefix_key`, when the INSERT `^G` prefix is pending and the next
   key is `ctrl+right_square_bracket`, clear the prefix (like `escape`) **and** call
   `_enter_normal_mode()`, so the "always lands in NORMAL mode" guarantee holds even
   mid-`^G`-prefix. Leave `escape` behavior itself unchanged (it only cancels the
   prefix). Leave `_handle_normal_g_prefix_key` unchanged.

4. Keymap config: no change. The vim-layer mode keys are hardcoded in the widgets, not
   routed through the keymap registry, so `src/sase/default_config.yml` is intentionally
   untouched (checked against the keymap-config gotcha).

5. This is presentation-only Textual key handling — no Rust core (`sase-core`)
   involvement.

## Tests

Follow the existing style in `tests/ace/tui/widgets/`:

- `tests/ace/tui/widgets/test_vim_text_area.py` (base widget, mirrors
  `test_insert_escape_enters_normal_mode`):
  - INSERT + `ctrl+right_square_bracket` → mode is `normal`.
  - INSERT + `ctrl+right_square_bracket` followed immediately by NORMAL keys in the same
    `press(...)` call (e.g. `"x"` or `"d", "w"`) → the keys execute as NORMAL-mode
    commands, none are inserted as text.
- Prompt widget (place with the other `PromptPage` key tests):
  - INSERT + `ctrl+right_square_bracket` → NORMAL, following key acts in NORMAL mode.
  - INSERT with an open completion menu (mirror however the existing escape/
    completion-dismiss behavior is exercised) → menu dismissed and mode is `normal`.
  - INSERT + `ctrl+g` (pending `^G` prefix) + `ctrl+right_square_bracket` → prefix
    cleared, mode is `normal`.
  - NORMAL-mode regression: existing `test_prompt_normal_mode_jump.py` must still pass
    unmodified (jump-to-definition unchanged).
- `tests/ace/tui/widgets/test_single_line_vim_text_area.py`: INSERT +
  `ctrl+right_square_bracket` → NORMAL (inherited from the base widget).

## Documentation

- `docs/ace.md`:
  - The prompt vim-mode key table row for `Escape` ("Switch to vim NORMAL mode", ~line
    5272): add `Ctrl+]` (either in the same row or an adjacent row) with a note that it
    is the race-free alternative — unlike `Escape`/`Ctrl+[`, it cannot be swallowed by
    terminal escape-sequence disambiguation when keys follow quickly.
  - The "Press `Escape` in INSERT mode to enter vim-style NORMAL mode" prose (~line
    6307): mention `Ctrl+]` as an equivalent that always works under fast typing.
  - Keep the existing NORMAL-mode `Ctrl+]` jump-to-definition docs as-is.

## Verification

- Run the project's standard verification (`just check`) and the lint/test procedure
  required by the `sase/memory/lint_and_test.md` reference memory before finishing.
- Targeted:
  `pytest tests/ace/tui/widgets/test_vim_text_area.py tests/ace/tui/widgets/test_single_line_vim_text_area.py tests/ace/tui/widgets/test_prompt_normal_mode_jump.py`
  plus whichever file gains the prompt-widget tests.

## Non-Goals

- Do not try to "fix" the ESC/`ctrl+[` disambiguation race itself (e.g. by lowering
  `ESCDELAY`); that ambiguity is inherent to legacy terminal encoding and shrinking the
  window trades one failure mode for another. `Ctrl+]` is the reliable path.
- Do not change VISUAL-mode key handling. In vim, visual-mode `Ctrl+]` is a tag jump,
  not an exit; leaving it unbound there today matches the current NORMAL-mode jump
  semantics and stays out of this tale's scope.
- Do not touch the NORMAL-mode jump-to-definition binding or its guards.
