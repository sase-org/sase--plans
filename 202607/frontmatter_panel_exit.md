---
tier: tale
title: Make the xprompt property panel reliably exitable with q and add gj/gk pane
  jumps
goal: Pressing `q` (or `esc` / `g=`) in the ACE xprompt property panel always hands
  focus back to a visible prompt input pane from every panel mode, and `gj` / `gk`
  jump straight from the panel to the top / bottom prompt input pane.
create_time: 2026-07-27 06:56:39
status: done
---

- **PROMPT:** [202607/prompts/frontmatter_panel_exit.md](prompts/frontmatter_panel_exit.md)

# Plan: Make the xprompt property panel reliably exitable

## Problem

In the `sase ace` TUI the **Frontmatter Panel** (aka the "xprompt property panel",
`src/sase/ace/tui/widgets/frontmatter_panel.py`) renders directly above the prompt stack. The user reports that they
sometimes cannot quit it with `q`, and wants `gj` / `gk` to jump from the panel to the top / bottom prompt input pane.

There are four distinct defects behind this. All were confirmed by reading the code; the third one (`reserved_height`)
reproduces the reported screenshot exactly.

### RC1 — the panel under-reserves its own height, collapsing the prompt stack to zero rows

`FrontmatterPanel.reserved_height` (`src/sase/ace/tui/widgets/frontmatter_panel.py:176-187`) tells the host bar how many
rows the panel needs:

```python
base = self._content_lines + 2
if self._edit_mode in {"edit", "picker", "cell"}:
    return base + 3 + bottom_margin  # inline input plus its top margin
```

`self._content_lines` is only the `#frontmatter-rows` Static's line count
(`src/sase/ace/tui/widgets/_frontmatter_panel_rendering.py:72`). The formula ignores several things that really do
occupy rows:

- **The `#frontmatter-feedback` Static.** It is un-hidden whenever `_feedback` is non-empty
  (`_frontmatter_panel_rendering.py:73-92`, and in raw mode `_frontmatter_panel_raw.py:159`). It is set to `"Saved"`
  after a cell commit (`_frontmatter_panel_cell_editing.py:293`), `"Undid last frontmatter change"`
  (`_frontmatter_panel_editing.py:96`), `"No matching schema property"` (`:156`), etc. Its CSS is
  `height: auto; max-height: 4` (`src/sase/ace/tui/styles.tcss:3119-3124`). **It is never counted.**
- **`#frontmatter-inline`** is `height: 3; margin: 1 0 0 0` (`styles.tcss:3086-3091`) = **4** rows, but the `edit` /
  `picker` / `cell` branch adds only `3`.
- **`#frontmatter-rows` is clamped by CSS** to `max-height: 10` (`styles.tcss:3075-3080`) while `reserved_height` uses
  the unclamped `_content_lines`, so long frontmatter over-reserves.
- **Raw mode** hard-codes `15`, but `#frontmatter-raw` is `max-height: 12` (`styles.tcss:3097-3102`) plus 2 border rows
  plus a feedback block of up to 4 rows, and the whole panel is capped at `max-height: 18` (`styles.tcss:3024-3033`).

The host bar sizes itself from that number in `_update_height`
(`src/sase/ace/tui/widgets/_prompt_input_bar_stack_rendering.py:517-545`) as `visual_lines + 2 + panel_rows`, and
`#prompt-stack` is `height: 1fr` **with no floor** (`styles.tcss:3127-3132`). A single missing row is therefore enough
to hand the prompt stack **zero rows**.

That is precisely the reported screenshot: 2 content rows + a `"Saved"` feedback row → `reserved_height` returns
`2 + 2 + 1 = 5`, the panel actually occupies `2 (border) + 2 (rows) + 1 (feedback) + 1 (margin) = 6`, the bar is sized
to `1 + 2 + 5 = 8`, its 6 content rows are entirely consumed by the panel, and the prompt pane renders at height 0.

`q` _does_ fire, `Closed` _is_ posted, and focus _does_ move to the pane — but the pane is invisible, so nothing appears
to happen and the panel looks un-quittable.

### RC2 — `q` is only bound in rows mode

`FrontmatterPanel.on_key` (`frontmatter_panel.py:222-355`) early-returns for the `edit`, `picker`, `cell`, `content`,
and `raw` modes before reaching:

```python
elif key in ("escape", "q"):
    self._close()
```

In those modes `q` reaches the child `VimTextArea`, which swallows unhandled printable NORMAL-mode keys
(`src/sase/ace/tui/widgets/vim_text_area.py:119-134`) and never lets them bubble to the panel. `q` is also explicitly
excluded from picker accelerators (`_frontmatter_panel_editing.py:127` reserves `{"j", "k", "q"}`). So in every
sub-editor mode `q` is a genuine silent no-op — the user must discover `esc esc`.

The designated seam for fixing this already exists and already has a precedent:
`VimTextArea._host_claims_unhandled_vim_key` (`vim_text_area.py:244-251`), overridden by `_AxeValueVimHostMixin`
(`src/sase/ace/tui/modals/axe_entry_editor_rendering.py:119-124`) so that NORMAL-mode `q` reaches the AXE sheet's quit
binding.

### RC3 — the close handler can silently fail to hand focus back

`on_frontmatter_panel_closed` (`src/sase/ace/tui/widgets/_prompt_input_bar_frontmatter.py:480-495`) wraps the whole
handoff in a bare `except Exception: pass`:

```python
try:
    text_area = self.active_text_area()
    text_area.focus()
    text_area._enter_insert_mode()
except Exception:
    pass
```

If `active_text_area()` raises (stale pane id mid-rebuild, empty stack), focus stays on the panel with zero feedback —
the same "can't quit" symptom, from a different cause.

### RC4 — `gj` / `gk` do nothing useful in the panel

Rows mode owns a panel-local `g` prefix that recognizes **only** `=` (`frontmatter_panel.py:307-323`); any other
continuation falls through and is dispatched as a bare rows key. So `gj` currently just moves the row selection down and
`gk` moves it up. There is no path from the focused panel to a specific prompt pane.

For reference, in the prompt body `gj` / `gk` are `_g_focus_next_pane` / `_g_focus_prev_pane`
(`src/sase/ace/tui/widgets/_prompt_input_bar_g_prefix_actions.py:69-82`) delegating to `focus_relative(±1)`
(`src/sase/ace/tui/widgets/_prompt_input_bar_stack_navigation.py:31-55`), which wraps modulo via
`PromptStackState.move_focus` (`src/sase/ace/tui/widgets/prompt_stack.py:401-416`). Because the panel sits _above_ the
whole stack, the natural extension is `gj` → **top** pane (`items[0]`) and `gk` → **bottom** pane (`items[-1]`),
matching that wrapping intuition and the user's stated expectation.

### Bonus defect (in scope, one line)

`#frontmatter-feedback { color: $error; }` (`styles.tcss:3119-3124`) forces every feedback message red, overriding the
`"dim"` style the renderer picks for non-error messages (`_frontmatter_panel_rendering.py:83-88`). That is why a
successful `"Saved"` renders in alarming red in the reported screenshot.

## Non-goals

- **Do not** change when the panel is _hidden_. It stays visible while the frontmatter is non-empty and is only dropped
  when the model is empty (`_prompt_input_bar_frontmatter.py:484-487`). "Quit the panel" here means _hand focus back to
  a prompt pane_, not hide the panel.
- **Do not** add `gj` / `gk` to the panel's sub-editor modes (`edit`, `picker`, `cell`, `content`, `raw`). `g` is a live
  vim prefix inside those editors. Rows mode only.
- **Do not** touch the prompt body's own `gj` / `gk` semantics (relative, wrapping pane focus).
- **Do not** add a keymap-config entry. Every panel key and every prompt `g`-prefix key is hard-coded, not
  registry-driven; `src/sase/default_config.yml:228` `quit: "q"` is the unrelated app-level binding, which the panel
  already shadows via `event.stop()`.

## Before you start

- Run `just install` first (workspaces are ephemeral and dependencies drift).
- Use the `/sase_memory_read` skill for `sase/memory/tui_perf.md` — `_update_height` runs on the hot resize/keystroke
  path, so keep the new arithmetic allocation-free and avoid extra queries per call (cache the child widgets or reuse
  the existing `query_one` results).
- Use the `/sase_memory_read` skill for `sase/memory/symvision.md` if the new symbols trip lint.

## Implementation

### Step 1 — Correct `reserved_height` (fixes RC1, the actual "sometimes")

In `src/sase/ace/tui/widgets/frontmatter_panel.py`:

1. Introduce module-level constants mirroring the CSS so the two can never drift again, and add a short comment in
   `src/sase/ace/tui/styles.tcss` above each rule pointing at them:

   | Constant               | Value | CSS rule                                                                       |
   | ---------------------- | ----- | ------------------------------------------------------------------------------ |
   | `_PANEL_BORDER_ROWS`   | `2`   | `#frontmatter-panel { border: round … }` (`styles.tcss:3028`)                  |
   | `_PANEL_BOTTOM_MARGIN` | `1`   | `#frontmatter-panel { margin: 0 0 1 0 }` (`styles.tcss:3032`)                  |
   | `_PANEL_MAX_HEIGHT`    | `18`  | `#frontmatter-panel { max-height: 18 }` (`styles.tcss:3027`)                   |
   | `_ROWS_MAX_HEIGHT`     | `10`  | `#frontmatter-rows { max-height: 10 }` (`styles.tcss:3078`)                    |
   | `_INLINE_ROWS`         | `4`   | `#frontmatter-inline { height: 3; margin: 1 0 0 0 }` (`styles.tcss:3086-3091`) |
   | `_CONTENT_ROWS`        | `6`   | `#frontmatter-content { height: 6 }` (`styles.tcss:3110`)                      |
   | `_RAW_MAX_HEIGHT`      | `12`  | `#frontmatter-raw { max-height: 12 }` (`styles.tcss:3100`)                     |
   | `_FEEDBACK_MAX_HEIGHT` | `4`   | `#frontmatter-feedback { max-height: 4 }` (`styles.tcss:3122`)                 |

2. Track the feedback block's rendered height. `_refresh` (`_frontmatter_panel_rendering.py:63-93`) and
   `_refresh_raw_diagnostics` (`_frontmatter_panel_raw.py:150-161`) already decide whether the feedback Static is shown
   — have them also store `self._feedback_lines` (0 when hidden, otherwise
   `min(rendered_line_count, _FEEDBACK_MAX_HEIGHT)`; a single-line message is 1). Initialize `_feedback_lines = 0` in
   `__init__` and reset it in `set_frontmatter`.

3. Rewrite `reserved_height` to sum only the children that are actually visible in the current mode, then clamp to the
   panel's own `max-height` and add the bottom margin:

   | `_edit_mode`               | visible children                                                         | rows term                               |
   | -------------------------- | ------------------------------------------------------------------------ | --------------------------------------- |
   | `rows`                     | rows + feedback                                                          | `min(_content_lines, _ROWS_MAX_HEIGHT)` |
   | `edit` / `picker` / `cell` | rows + inline + feedback                                                 | same, plus `_INLINE_ROWS`               |
   | `content`                  | rows + content editor + feedback                                         | same, plus `_CONTENT_ROWS`              |
   | `raw`                      | raw editor + feedback (rows are hidden — `_frontmatter_panel_raw.py:51`) | `min(raw_line_count, _RAW_MAX_HEIGHT)`  |

   i.e. `min(_PANEL_BORDER_ROWS + children, _PANEL_MAX_HEIGHT) + _PANEL_BOTTOM_MARGIN`.

4. Every code path that changes the feedback text, the mode, or the rows content must already end in `_refresh()` /
   `_schedule_layout_update()`; verify that `_commit_cell_edit` (`_frontmatter_panel_cell_editing.py:289-293`, the
   `"Saved"` path) triggers a host height update, and call `self._schedule_layout_update()`
   (`frontmatter_panel.py:391-399`) if it does not.

### Step 2 — Guarantee at least one visible prompt row (defense in depth for RC1)

1. `src/sase/ace/tui/styles.tcss`: add `min-height: 1;` to `PromptInputBar #prompt-stack` (`:3127-3132`) so Textual can
   never allocate the stack zero rows even if the arithmetic drifts again. Clipping the panel is a strictly better
   failure mode than an invisible prompt.
2. `src/sase/ace/tui/widgets/_prompt_input_bar_stack_rendering.py:517-545` (`_update_height`): clamp the frontmatter
   reservation so the body always survives on a short terminal —

   ```python
   frontmatter_cap = max(0, max_height - 3 - completion_rows - g_prefix_rows - search_rows)
   frontmatter_rows = min(self._frontmatter_panel_reserved_rows(), frontmatter_cap)
   ```

   and push that cap onto the panel so it physically shrinks instead of being clipped: set
   `panel.styles.max_height = frontmatter_cap` (only when it actually changes, to avoid needless refreshes on the hot
   path). Leave the multi-pane branch `_apply_multi_pane_heights` (`:547-596`) using the same clamped `panel_rows` it
   already receives.

### Step 3 — Make `q` leave the panel from every mode (fixes RC2)

In `src/sase/ace/tui/widgets/frontmatter_panel.py`:

1. Rows mode: change `elif key in ("escape", "q"): self._close()` (`:350-351`) to route through `self.deactivate()` so
   there is exactly one exit path with one set of per-mode semantics. (In rows mode the two are equivalent today; this
   prevents them diverging.)
2. Sub-editor modes: add a `q` branch beside the existing `escape` branches in each of the `edit`/`picker`/`cell`
   (`:255-266`), `content` (`:279-288`), and `raw` (`:297-306`) blocks, using the **same guard already used for
   `escape`** — only act when the child editor is in NORMAL mode with no pending state
   (`editor._vim_mode == "normal" and not editor._has_pending_normal_state()`). On a match: `event.stop()`,
   `event.prevent_default()`, then `self.deactivate()`.

   `deactivate()` (`:196-220`) already encodes the right per-mode semantics: cancel the in-progress inline edit, or
   commit raw YAML and stay in raw mode if it does not parse.

3. For the panel's `on_key` to see `q` at all, the child editors must stop swallowing it. Add a small host mixin in the
   panel module (mirroring `_AxeValueVimHostMixin`, `src/sase/ace/tui/modals/axe_entry_editor_rendering.py:119-124`)
   that overrides `_host_claims_unhandled_vim_key` to return `True` for `event.character == "q"`, and use mixed-in
   subclasses for `#frontmatter-inline`, `#frontmatter-raw`, and `#frontmatter-content` in `compose`
   (`frontmatter_panel.py:118-136`).

   This only affects NORMAL/VISUAL mode (`vim_text_area.py:119-134` is reached only from those branches), so typing a
   literal `q` in INSERT mode is unaffected — verify this with a test.

4. Add `q done` to `_SUBTITLE_POPULATED` (`_frontmatter_panel_rendering.py:32`) — today the populated panel advertises
   no way out at all, while `_SUBTITLE_EMPTY` (`:33`) does say `esc done`. Keep the existing keys; the bar is full width
   so there is room.

### Step 4 — Make the focus handoff robust and targeted (fixes RC3 and RC4)

1. `FrontmatterPanel.Closed` (`frontmatter_panel.py:75-84`): add a `focus_target: str` field (`"active"` | `"top"` |
   `"bottom"`, default `"active"`). Thread an optional `focus_target` parameter through `deactivate()` (`:196-220`) and
   `_close()` (`_frontmatter_panel_editing.py:376-377`).
2. Rows-mode `g` prefix (`frontmatter_panel.py:312-323`): alongside the existing `=` continuation, handle `j` →
   `self.deactivate(focus_target="top")` and `k` → `self.deactivate(focus_target="bottom")`. Keep the existing
   fall-through for every other continuation so plain rows keys are still never swallowed.
3. Discoverability: add a `_SUBTITLE_PENDING_G` (e.g. `"g= done · gj top pane · gk bottom pane"`) to
   `_frontmatter_panel_rendering.py` and show it from `_refresh_chrome` (`:95-119`) while `_pending_g` is set, restoring
   the normal subtitle once the prefix resolves or is abandoned. `_pending_g` is cleared in `set_frontmatter`
   (`frontmatter_panel.py:154`) and `focus_panel` (`:170`), so make sure a chrome refresh follows those too.
4. Remember the pane the panel was entered from, so `"active"` genuinely means "the last prompt input widget". In
   `src/sase/ace/tui/widgets/_prompt_input_bar_frontmatter.py`, record
   `self._frontmatter_return_index = self._stack.selected_index` in `focus_frontmatter_panel` (`:219-232`) and in the
   focusing branch of `_show_frontmatter_panel` (`:202-217`).
5. Rewrite `on_frontmatter_panel_closed` (`:480-495`):
   - resolve the target index — `"top"` → `0`, `"bottom"` → `len(self._stack) - 1`, `"active"` → the recorded return
     index, falling back to `self._stack.selected_index`;
   - when it differs from the current selection, `self._stack.focus(index)`
     (`src/sase/ace/tui/widgets/prompt_stack.py:396-399`) and `self._apply_active_classes()`;
   - call `self._schedule_height_update()` **before** focusing so the target pane has been given real rows, then focus
     via `call_after_refresh` and `scroll_visible(animate=False)`;
   - drop the blanket `except Exception: pass`. If `active_text_area()` raises, fall back to the first mounted
     `PromptTextArea` in `#prompt-stack`; if there is genuinely no pane, blur the panel (`self.app.set_focus(None)`) so
     it visibly relinquishes focus rather than silently keeping it.
   - keep the existing empty-model behavior (clear `self._stack.frontmatter`, hide the panel) and the
     `_enter_insert_mode()` landing.

### Step 5 — Fix the misleading red feedback color

`src/sase/ace/tui/styles.tcss:3119-3124`: drop the blanket `color: $error` from `#frontmatter-feedback` so the `"dim"` /
`"red"` style chosen in `_frontmatter_panel_rendering.py:83-88` actually decides, or gate the red on an `.error` class
that the renderer toggles. Successful messages like `"Saved"` must not render as errors.

### Step 6 — Docs and help

Per `src/sase/ace/CLAUDE.md` ("Help Popup Maintenance"), keep every surface in sync:

- `docs/xprompt.md` §`Frontmatter Panel (ACE TUI)` (`:1854-1872`): document that `q` / `esc` / `g=` leave the panel and
  return focus to the pane you came from, from **every** panel mode, and that rows-mode `gj` / `gk` jump to the top /
  bottom pane.
- `docs/ace.md`: the prompt `g`-prefix table (`:2800-2803`) — note the in-panel meaning of `gj` / `gk`; and the `Ctrl+G`
  table entry for `Ctrl+G =` (`:2668`) if the panel row needs it.
- `src/sase/ace/tui/modals/help_modal/binding_common.py:46` and any frontmatter-panel section of the help modal: add the
  panel's `q` and `gj` / `gk` entries. Respect the 57-char box width and 32-char description limit documented in
  `src/sase/ace/CLAUDE.md`.

## Tests

Add to the existing suites (helpers already exist — `tests/ace/tui/widgets/_frontmatter_panel_helpers.py`,
`tests/ace/tui/widgets/_prompt_input_bar_stack_helpers.py`):

1. `tests/ace/tui/widgets/test_frontmatter_panel.py`
   - `q` in rows mode posts `Closed` and focus lands on the previously active pane.
   - `q` in `cell`, `edit`, `content`, and `raw` NORMAL mode leaves the panel (raw commits first; an unparseable raw
     buffer stays in raw mode with an error, matching `deactivate`).
   - `q` typed in a child editor's **INSERT** mode still inserts a literal `q`.
   - `gj` from rows mode focuses `items[0]`; `gk` focuses `items[-1]`; both update `PromptStackState.selected_index` and
     the active/inactive pane classes.
   - Every other `g<key>` continuation still falls through to its bare rows command (regression guard for
     `frontmatter_panel.py:319`).
2. `tests/ace/tui/widgets/test_frontmatter_panel_subeditors.py` (or a new `test_frontmatter_panel_height.py`):
   `reserved_height` for each mode, with and without a visible feedback block, with `_content_lines` above and below
   `_ROWS_MAX_HEIGHT`. Assert it never under-reports the panel's real outer height.
3. `tests/ace/tui/widgets/test_prompt_input_bar_stack.py` (or a focused new module): the reported regression — commit a
   cell edit so `_feedback == "Saved"`, then assert the active `PromptTextArea` still has a rendered height ≥ 1 and is
   the focused widget after `q`.
4. `tests/ace/tui/visual/test_ace_png_snapshots_frontmatter_panel.py`: a snapshot with the feedback line visible,
   proving the prompt pane is still rendered below the panel. New goldens go under
   `tests/ace/tui/visual/snapshots/png/`; generate them with `--sase-update-visual-snapshots` and eyeball the artifacts
   in `.pytest_cache/sase-visual/` before committing.

## Verification

```bash
just install
just check
just test-visual
```

Then confirm manually in `sase ace`:

1. Open a prompt with `xprompts:` frontmatter, `g=` into the panel, `e` into the `foo` cell, edit and commit so
   `"Saved"` appears — the prompt pane below must stay visible, and `q` must return focus to it.
2. With a multi-pane stack, `g=` into the panel and confirm `gj` lands on the top pane and `gk` on the bottom pane, each
   in NORMAL mode with the right pane marked active.
3. From `R` (raw) and from a `cell` edit, confirm NORMAL-mode `q` leaves the panel and that INSERT mode still accepts a
   literal `q`.
4. Shrink the terminal until the panel would overflow and confirm at least one prompt row survives.
