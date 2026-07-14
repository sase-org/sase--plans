---
create_time: 2026-07-14 08:57:58
status: wip
prompt: 202607/prompts/yank_highlight.md
tier: tale
---
# Plan: Flash-highlight yanked text in the prompt input widget

## Goal & product context

When a user yanks text with the `y` operator in the `sase ace` prompt input (vim NORMAL/VISUAL modes), the copied region
should briefly flash with a distinct highlight and then fade back to normal — the same affordance the user already
relies on in Neovim, where `TextYankPost` fires `vim.highlight.on_yank({ higroup = "IncSearch", timeout = 200 })`.

This is a small, purely visual confirmation with an outsized UX payoff: it makes "yes, that got copied, and _this_ is
exactly what got copied" instantly legible, which is especially valuable in a TUI where there is otherwise no visible
cue that `y` did anything. The design goals, in priority order:

1. **Reliable** — never leaves a stuck highlight, never flashes stale/wrong regions, safe under rapid repeated yanks,
   cheap enough to not affect prompt responsiveness.
2. **Intuitive** — the flash covers _exactly_ the characters that were copied, for every yank flavor (charwise,
   linewise, visual, visual-line), and matches the user's existing Neovim muscle memory (~200 ms, then gone).
3. **Beautiful** — a crisp, theme-adaptive highlight that is visually distinct from the existing search/syntax overlays
   and reads as a positive "captured" confirmation.

### Scope / non-goals

- Applies to the **prompt input widget** (`PromptTextArea`) only. The single-line vim widget used in modals
  (`SingleLineVimTextArea`) intentionally does **not** flash (it gets an inert no-op hook). This matches the user's
  request and keeps the surface small; the hook is trivially reusable later if desired.
- Only the **`y` (yank)** operator flashes. `d`/`c` also copy to the clipboard, but the text is removed, so there is
  nothing to highlight — and the user asked specifically about yank.
- No new user-facing config knob. The duration is a single module constant (`0.2 s`, matching the Neovim reference). A
  config toggle is noted as a possible future extension but is out of scope here to keep the feature simple and
  predictable.
- **No fade animation.** A single crisp flash-then-clear (as Neovim does) is the reliable, beautiful choice; per-frame
  color interpolation is deliberately avoided to protect TUI responsiveness. Noted as a future option below.

### Architecture boundary note

This is entirely presentation: a transient Textual overlay span plus a timer, layered on the TUI's existing rendering.
It does **not** cross the Rust core backend boundary — no web/CLI/other frontend needs this behavior to match, so no
`sase-core` change is involved. All work lives in this repo's TUI layer.

## How the current code works (grounding)

- **The prompt widget** is `PromptTextArea` (`src/sase/ace/tui/widgets/prompt_text_area.py`), a Textual `TextArea`
  assembled from a mixin tower on top of `VimTextArea` (`src/sase/ace/tui/widgets/vim_text_area.py`).

- **Every yank funnels through two methods** in `src/sase/ace/tui/widgets/_vim_normal_operator_exec.py`:
  - `_execute_charwise_operator(start, end, op)` — the `op == "y"` branch (~lines 56–63) copies
    `_get_text_in_range(start, end)`. Handles `y{motion}`, `yiw`, `yf)`, and visual charwise `y`. `start`/`end` are
    normalized `(row, col)` locations.
  - `_execute_linewise_operator(first_row, last_row, op)` — the `op == "y"` branch (~lines 149–160) copies the joined
    rows. Handles `yy`, `Y`, counts, `yae`, and visual-line `y`. These two branches are the single choke point for all
    yank flavors, and the exact copied range is in scope at each.

- **Transient highlight overlays** are a well-established pattern. The widget layers Rich-styled spans on top of
  TextArea's syntax highlighting via `_append_highlight_span(start, end, style_name)` (absolute char offsets;
  `_jinja_highlight.py`), appended inside chained `_build_highlight_map()` overrides. Four uniform sibling mixins
  already do this: `JinjaHighlightMixin`, `SearchHighlightMixin`, `AltSyntaxHighlightMixin`,
  `XPromptSyntaxHighlightMixin`. Each registers its styles into a shared `TextAreaTheme` via a
  `_register_*_text_area_theme` method, re-registers on `_app_theme_changed`, and re-adds its styles when the base Jinja
  theme re-registers (`_register_jinja_text_area_theme` override). `SearchHighlightMixin` is the closest analog
  (transient, toggled spans).

- **`VimTextArea` already defines inert "host hooks"** with safe no-op defaults that `PromptTextArea` overrides (e.g.
  `_clear_search_highlights`, `_clear_prompt_search`, `_start_prompt_search`). This is the idiomatic seam for "base vim
  layer signals an event; the prompt layer decides how to present it."

- **Offset + timer utilities**: `_absolute_offset((row, col))` converts a location to an absolute char offset
  (`vim_text_area.py`). Transient timed state is done with `self.set_timer(seconds, cb)` guarded by a monotonically
  increasing generation counter (see `_prompt_soft_completion.py`), so a stale timer fire is ignored.

## Design

Two coordinated pieces: a **trigger** (fire an event from the yank choke point) and a **presenter** (a new overlay mixin
that flashes the region and clears it).

### 1. Trigger — a new `_flash_yank` host hook

Add an inert host hook on `VimTextArea`, alongside the existing no-op hooks:

```python
def _flash_yank(self, start: tuple[int, int], end: tuple[int, int]) -> None:
    """Briefly highlight a just-yanked region. Default: no-op."""
```

Call it from the two `op == "y"` branches in `_vim_normal_operator_exec.py`, passing the _exact_ copied range as
`(row, col)` locations:

- **Charwise** branch: after the copy, `self._flash_yank(start, end)` — guarded by the same non-empty condition the copy
  uses (`if text:`).
- **Linewise** branch: `self._flash_yank((first_row, 0), (last_row, len(last_line)))` so the flash spans the full yanked
  lines (matching Neovim's linewise highlight). Guard on the region being non-empty (skip a bare empty-line `yy`).

Passing raw locations keeps the operator-exec change minimal and pushes all presentation logic (offset conversion,
spans, timing, color) into the presenter mixin. `SingleLineVimTextArea` and a bare `VimTextArea` get the no-op and never
flash. (Add a `TYPE_CHECKING` stub for `_flash_yank` in `_vim_normal_operator_exec.py`, mirroring the existing
`_clear_prompt_search` stub, so mypy resolves the call.)

### 2. Presenter — a new `YankHighlightMixin`

New file `src/sase/ace/tui/widgets/_yank_highlight.py`, built as a faithful sibling of `SearchHighlightMixin`:

- **Transient state** (init): `_yank_flash_span: tuple[int, int] | None = None`, `_yank_flash_generation: int = 0`,
  `_yank_flash_timer: Timer | None = None`. Module constant `_YANK_FLASH_SECONDS = 0.2`.

- **`_flash_yank(start, end)`** (overrides the base hook): convert both locations to absolute offsets via
  `_absolute_offset`; if the region is empty, do nothing; otherwise store `_yank_flash_span`, bump the generation,
  `.stop()` any prior timer, repaint the overlay, and schedule
  `set_timer(_YANK_FLASH_SECONDS, lambda: self._clear_yank_flash(gen))`. Stopping the prior timer + the generation guard
  together make rapid repeated yanks safe (each new yank restarts the window cleanly; a stale fire is ignored).

- **`_clear_yank_flash(generation)`**: bail if `generation` is stale or the widget is unmounted; else clear the
  span/timer and repaint. This guarantees the highlight always tears itself down (no stuck highlight).

- **`_build_highlight_map()`** override: call `super()`, then, if a flash span is set (and within the same
  `_MAX_OVERLAY_BYTES` / `_MAX_OVERLAY_LINES` guards the other overlays use), append it via
  `_append_highlight_span(start, end, "yank.flash")`. `_append_highlight_span` already splits multi-line spans, so
  multi-line linewise yanks flash correctly.

- **Theme registration** (`_register_yank_text_area_theme`, `_resolve_yank_base_theme`, plus `on_mount` /
  `_app_theme_changed` / `_register_jinja_text_area_theme` overrides): register a single `"yank.flash"` style into the
  shared theme, exactly as the sibling overlays do, so it stays theme-adaptive and survives theme/Jinja re-registration.

- **`_refresh_yank_overlay()`**: `_build_highlight_map()` + `self.refresh()` (same as `_refresh_search_overlay`).

**Style / color choice.** `Style(color=app_theme.foreground, bgcolor=app_theme.success, bold=True)` — i.e. a "green
sibling" of `search.current` (which is `foreground` on `warning`). Rationale: green reads as a positive "captured ✓"
confirmation; it is clearly distinct from the search overlays (blue `accent` / amber `warning`) and the Jinja/xprompt
syntax colors, so a flash is never confused with a search hit; and it is derived from the active theme, so it adapts
across light/dark themes. `bold` adds punch without reflow (TextArea is a fixed monospace grid). This is a one-line
tunable if a different hue is preferred (e.g. amber to mirror Neovim's `IncSearch` more literally — noted, but rejected
here because amber already means "current search match").

### 3. Wire the mixin into `PromptTextArea`

Add `YankHighlightMixin` to `PromptTextArea`'s base list as the **first** overlay mixin (just before
`AltSyntaxHighlightMixin`). Because each overlay appends its spans _after_ calling `super()._build_highlight_map()`, the
earliest overlay in the MRO appends last and therefore paints on top — so the yank flash visually dominates
search/syntax overlays for its brief lifetime, which is what we want. Initialize its state in `PromptTextArea.__init__`
(or the mixin's own `__init__`, consistent with how `SearchHighlightMixin` seeds its state).

## Behavior matrix (what flashes)

| Input                               | Path                   | Flash region               |
| ----------------------------------- | ---------------------- | -------------------------- |
| `yw`, `yiw`, `yf)`, `y$`, …         | charwise exec          | exactly the yanked chars   |
| `viw` then `y`, visual charwise `y` | charwise exec          | the selected chars         |
| `yy`, `Y`, `3Y`, `yae`              | linewise exec          | the full yanked line(s)    |
| `V…` then `y` (visual-line)         | linewise exec          | the full selected line(s)  |
| `dd`, `cw`, paste, case ops         | not `op == "y"`        | no flash (by design)       |
| `yy` on an empty line               | linewise, empty region | no flash (nothing to show) |

Note: visual-mode yank collapses the native `self.selection` _before_ the exec methods run, so the flash is driven off
the explicit range passed to `_flash_yank`, never off `self.selection`.

## Testing

Behavioral tests (deterministic, no real-time waits) modeled on `tests/test_prompt_normal_mode_yank_paste.py` (via the
`PromptPage` harness) and `tests/ace/tui/widgets/test_prompt_search_highlight.py` (via `CompletionTestApp`

- the `_highlight_names` helper). New file, e.g. `tests/ace/tui/widgets/test_prompt_yank_highlight.py`:

* After `y w` / `y i w` / `y f )`: `"yank.flash"` appears in `ta._highlights` and `_yank_flash_span` covers exactly the
  yanked range; `page.text` unchanged.
* After `y y` / `2 Y` / visual-line `y`: the flash span covers the full line(s).
* After visual charwise `v … y`: the flash span covers the selection.
* **Expiry**: calling `_clear_yank_flash(current_generation)` removes the span and the `"yank.flash"` name; the
  buffer/register are untouched.
* **Stale-fire guard**: calling `_clear_yank_flash(old_generation)` after a second yank does _not_ clear the active
  flash.
* **Rapid re-yank**: a second yank replaces the span and restarts the timer (prior timer stopped); only the newest
  region is highlighted.
* **Coexistence**: with Jinja/search overlays active, `"yank.flash"` and the other overlay names all appear (theme +
  highlight map), confirming the new style registers without clobbering siblings.
* **Large-buffer guard**: no `"yank.flash"` span past the overlay size limits.
* **No flash on `dd`/`cw`** and on empty-line `yy`.
* **`SingleLineVimTextArea` no-op**: yanking there does not raise and produces no `"yank.flash"` span (exercises the
  inert base hook).

Optionally add a PNG visual snapshot (the repo has an ACE PNG snapshot suite) of a prompt with `_yank_flash_span` set,
to lock the appearance in — noting the snapshot must render while the flash is active (set the span directly rather than
racing the timer). Treat this as a nice-to-have on top of the behavioral tests.

## Files touched

- `src/sase/ace/tui/widgets/vim_text_area.py` — add inert `_flash_yank` host hook.
- `src/sase/ace/tui/widgets/_vim_normal_operator_exec.py` — call `_flash_yank` from the two `op == "y"` branches; add a
  `TYPE_CHECKING` stub for it.
- `src/sase/ace/tui/widgets/_yank_highlight.py` — **new** `YankHighlightMixin`.
- `src/sase/ace/tui/widgets/prompt_text_area.py` — add the mixin to the MRO (first overlay) and seed its state.
- `tests/ace/tui/widgets/test_prompt_yank_highlight.py` — **new** tests (+ optional visual snapshot).

## Risks & mitigations

- **Stuck highlight** → the clear callback always runs on the timer and always tears down; the generation guard +
  `.stop()` handle rapid re-yanks and unmount races. Covered by expiry/stale-fire tests.
- **Wrong region on visual yank** (selection already collapsed) → flash is driven by the explicit range, not
  `self.selection`. Covered by visual-mode tests.
- **Theme regression** (new style dropped when Jinja/theme re-registers) → mirror the exact
  `_register_jinja_text_area_theme` / `_app_theme_changed` chaining the sibling overlays use. Covered by the coexistence
  test.
- **Perf** → the flash reuses the existing overlay machinery, is gated by the same size limits, and paints twice per
  yank (on + off). No polling/animation.
- Run `just install` then `just check` (ruff + mypy) and `just test` / `just test-visual` before finishing.

## Future extensions (out of scope)

- Optional `default_config.yml` toggle / duration knob.
- Optional subtle fade-out (two-stage timer) if a softer feel is later desired.
- Reuse the `_flash_yank` hook to flash yanks in the modal single-line widget.
