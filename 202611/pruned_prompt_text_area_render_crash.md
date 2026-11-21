---
tier: tale
title: Stop the TUI crashing when it repaints an unmounted prompt text area
goal:
  The TUI no longer crashes when a full repaint reaches a PromptTextArea that Textual is
  still removing.
size: small
decisions:
  detach_layout:
    ask: Should prompt-bar detach also request an immediate layout refresh?
    choices:
      both:
        Guard plus refresh; the stale bar leaves the compositor map on the next paint
      guard_only:
        Guard only; the stale bar stays in the map until Textual's async prune ends
    default: both
    why: The refresh also protects the bar's other widgets, not just the text area.
    answer: both
proposed_by: bbugyi200.athena.1eu
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-21 10:11:08
status: wip
---

# Fix: TUI crash `KeyError: "No 'text-area--gutter' key in COMPONENT_CLASSES"`

## Problem

The TUI on apollo crashed at 2026-11-21 14:56:28 (`~/.sase/logs/tui.log` on apollo). The
full traceback logged there is:

```
textual/timer.py _tick
  -> textual/screen.py _on_timer_update            (line 1255: self._compositor_refresh())
  -> textual/screen.py _compositor_refresh
  -> textual/_compositor.py render_update -> render_full_update -> _render_chops -> _get_renders
  -> textual/widgets/_text_area.py render_lines    (theme.apply_css(self))
  -> textual/_text_area_theme.py apply_css         (get_style("text-area--gutter"))
  -> textual/widget.py get_component_rich_style
  -> textual/dom.py get_component_styles
KeyError: "No 'text-area--gutter' key in COMPONENT_CLASSES"
```

The widget being rendered was
`PromptTextArea(id='prompt-input-g0-p0', classes='-vim-insert solo prompt-input active')`
with its `sase-jinja-prompt` theme. The theme object already had resolved
`gutter_style`, `cursor_style`, and so on, so this widget had rendered successfully
before.

## Root cause

This is a lifecycle race between Textual widget removal and compositor repaint. It
reproduces on both the pinned dev Textual (8.0.1) and the Textual that `uv tool`
installs on apollo and athena (8.2.8):

1. `Widget._message_loop_exit` (Textual) runs when a widget is pruned. It calls
   `self._detach()`, which sets `_parent = None`, and then
   `self._component_styles.clear()`.
2. The screen's compositor map (`Compositor.visible_widgets`) is rebuilt only on a
   layout pass (`Screen._refresh_layout`). Textual requests that pass only after the
   whole removed subtree has finished pruning (`App._prune` → `post_mount` →
   `parent.refresh(layout=True)`). Until then the pruned widget stays in the map.
3. `TextArea.render_lines` calls `self._theme.apply_css(self)` on every render, and
   `apply_css` calls `get_component_rich_style("text-area--gutter")` with no default. On
   a pruned TextArea that raises `KeyError`.
4. In the crash, the screen's update timer ran a **full** repaint (`render_full_update`)
   with `_layout_required` still False. A pruned `PromptTextArea` inside a prompt bar
   was still in the compositor map, so `render_lines` ran on it and the KeyError escaped
   the timer and killed the app.

sase makes the window wider. `_detach_prompt_bar`
(`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_mount_lifecycle.py`) and
`_discard_prompt_bar_spare` (`.../_prompt_bar_mount_spare.py`) unlink the bar from
`parent._nodes` synchronously, to free its id right away, and then call `bar.remove()`.
Neither requests a layout. The screen keeps compositing the detached bar's stale
geometry until the async prune of the entire bar subtree finishes. Any full repaint
during that time renders the already-pruned `PromptTextArea`. Prune order is children
first, so the text area is cleared before the bar finishes. The window also includes the
bar's own unmount hooks. Event-loop stalls, which are common on apollo per
`~/.sase/logs/tui_stalls.jsonl`, widen it further.

Deterministic reproduction (plain Textual, no sase code needed):

```python
async with app.run_test(size=(60, 12)):
    ta = app.query_one("#ta", TextArea)
    await ta.remove()                       # prune finished; layout requested but not yet run
    assert ta in app.screen._compositor.visible_widgets   # still in the map
    app.screen._compositor.render_update(full=True)       # KeyError: 'text-area--gutter'
```

No upstream Textual report or fix was found.

## Changes

### 1. Make sase's TextAreas safe to render once Textual has detached them (the crash fix)

File: `src/sase/ace/tui/widgets/_line_rendering.py` (`LineRenderingMixin`). It sits in
the MRO of `VimTextArea` ahead of Textual's `TextArea`, so a fix there covers
`PromptTextArea`, `SingleLineVimTextArea`, `SecretVimTextArea`, and every other
VimTextArea host. No other sase mixin overrides `render_lines`.

- Add a `render_lines(self, crop: Region) -> list[Strip]` override. When the widget has
  been detached (`self.parent is None`), return `crop.height` blank strips of
  `crop.width` and do **not** call `super().render_lines` (that is where `apply_css`
  runs). Otherwise delegate to `super().render_lines(crop)` unchanged.
  - Use `self.parent is None` as the detached test. It is O(1), uses the public API, and
    flips at exactly the point Textual clears `_component_styles`: `_detach()` and the
    clear happen together in `_message_loop_exit` with no await between them. Do not use
    `is_attached`, which walks ancestors on every call.
  - Style the blank strips with the theme's base background when available
    (`getattr(getattr(self, "_theme", None), "base_style", None)`), so the one stale
    frame matches the editor background instead of the terminal default. Pass `None`
    otherwise. Use `Strip.blank(width, style)`.
- Add the same guard at the top of the existing `LineRenderingMixin.render_line`. The
  compositor's mouse path (`Compositor.get_widget_and_offset_at`) calls `render_line`
  directly, and `TextArea._render_line` reads the `text-area--suggestion` component
  style when a suggestion is set. Return a blank strip of `self.size.width` there. Keep
  the check O(1), because this runs per line on the typing path.
- Write a docstring comment that names the Textual race (pruned widget still in the
  compositor map, component styles cleared) so nobody later removes the guard as dead
  code.

### 2. Drop a synchronously detached prompt bar from the compositor on the next paint

> [!decision] detach_layout = both

This is the sase-owned part of the root cause. Apply it in both places where sase calls
`parent._nodes._remove(bar)`:

- `_detach_prompt_bar` in `_prompt_bar_mount_lifecycle.py`
- `_discard_prompt_bar_spare` in `_prompt_bar_mount_spare.py`

Immediately after the synchronous `parent._nodes._remove(...)` and before
`bar.remove()`, call `parent.refresh(layout=True)`. Textual's own `post_mount` makes the
same call after a prune; this makes it at unlink time instead.
`Widget.refresh(layout=True)` sets the screen's `_layout_required` synchronously, and
`Screen._on_timer_update` always runs `_refresh_layout` (reflow) before compositing. The
next timer-driven paint therefore rebuilds the map from `_nodes`, which no longer
contain the bar, before rendering anything. This protects every widget in the bar
subtree, not only the TextArea. For example, OptionList-based completion panels also
read component styles while rendering. It also removes the stale ghost-bar frame after
submit or cancel.

- Keep the spare path's existing best-effort `try/except` style. For the spare, which is
  `display=False` and never in the map, the refresh is harmless and coalesces with the
  fresh bar mount that follows in `_show_prompt_input_bar`.
- Consider a tiny shared helper, for example `unlink_prompt_bar_from_parent(bar)`, that
  does `_nodes._remove` plus the layout refresh, so the two call sites cannot drift.
  Optional; inline is fine if a helper would need a new module.
- Perf note: dismissal is not a per-keystroke path. At most one extra reflow can happen
  per dismissal, when the prune finishes after the first tick; otherwise Textual
  coalesces the two requests. The existing prompt perf smoke test must stay green.

If the reviewer picks `guard_only`, skip this section and its test (test 3 below).

## Tests

Add to `tests/ace/tui/widgets/test_prompt_input_bar_detach.py`, which already covers the
synchronous-detach race family. Keep each test deterministic: no sleeps, and drive the
compositor directly.

1. **Pruned `PromptTextArea` survives a full repaint.** Mount a `PromptInputBar` (reuse
   `PromptDetachTestApp`), grab its active `PromptTextArea`, mirror the detach
   (`parent._nodes._remove(bar)`, then `await bar.remove()`), and assert that the text
   area is still in `app.screen._compositor.visible_widgets`. This guards that the test
   exercises the race. Then assert that
   `app.screen._compositor.render_update(full=True)` does not raise. Without change 1
   this raises `KeyError: "No 'text-area--gutter' key in COMPONENT_CLASSES"`.
2. **Detached render contract.** For the same detached text area,
   `render_lines(Region(0, 0, w, h))` returns `h` strips each of cell length `w`, and
   `render_line(0)` does not raise. Also assert that a mounted, attached text area still
   renders its real text, so the guard does not affect live widgets.
3. **Synchronous detach requests layout (change 2).** Build a small
   `class _DetachApp(PromptBarMountMixin, App[None])` that composes a `PromptInputBar`
   with id `prompt-input-bar`, stubbing whatever `_detach_prompt_bar` reaches for, as
   `tests/ace/tui/test_prompt_bar_submit_no_cancel_save.py` does. Call
   `app._detach_prompt_bar(bar)` and then, with no awaits, run
   `app.screen._on_timer_update()`. Assert that neither the bar nor its text area is in
   `app.screen._compositor.visible_widgets`. Without change 2 the bar is still in the
   map at that point. If the harness cost gets out of hand, an acceptable fallback is to
   assert `app.screen._layout_required` right after the call.
4. Add a matching spare-path assertion if `test_prompt_bar_hot_spare.py` already
   provides a cheap harness for `_discard_prompt_bar_spare`. Otherwise one test in the
   detach file that calls `_discard_prompt_bar_spare` on the same `_DetachApp` with a
   bar stored in `_prompt_bar_spare` is enough.

Existing suites that must stay green: `tests/ace/tui/widgets/test_vim_text_area.py`,
`tests/ace/tui/widgets/test_single_line_vim_text_area.py`,
`tests/ace/tui/test_prompt_bar_hot_spare.py`,
`tests/ace/tui/test_prompt_bar_submit_no_cancel_save.py`,
`tests/ace/tui/test_agent_hijack_prompt_bar.py`, and
`tests/ace/tui/test_prompt_key_perf_smoke.py`.

## Verification

- `just fmt`, then `sase tool run check`. That runs the lint gates and diff-scoped
  tests; do not run `check-full`.
- Run the new tests directly and confirm tests 1–3 fail on the pre-change code: stash
  the source change or comment out the guard and the refresh. This proves they reproduce
  the crash and the stale-map window.

## Out of scope

- Plain Textual `TextArea` widgets composed inside modal screens (bead editor, memory
  and snippet add forms, and similar). A dismissed modal stops being the current screen
  before its children prune, so its compositor never repaints them. No change needed.
- Reporting the `TextArea.render_lines` / `apply_css` fragility upstream to Textual.
  That is optional follow-up; this plan does not depend on it.

## Rollout note

apollo runs an editable `uv tool` install from its own sase checkout, and a running TUI
keeps the code it imported at startup. After this lands, pull on apollo and restart the
TUI there. athena too, for its running TUI.
