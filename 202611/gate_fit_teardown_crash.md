---
tier: tale
title: Stop gate modal fit callbacks from crashing the TUI during teardown
goal:
  Dismissing a GateInputPanel or CustomGateModal while a queued body-fit callback is
  pending can no longer crash the ACE TUI with NoMatches; the fit callbacks return
  quietly on a partially torn-down modal, and regression tests pin that behavior.
size: small
proposed_by: bbugyi200.athena.1dm
proposed_with: claude/opus
create_time: 2026-11-17 15:43:45
status: wip
---

# Fix the gate modal fit-callback teardown crash

## Problem

The ACE TUI on apollo crashed (the whole app exited) on 2026-11-17 with this traceback,
logged to `~/.sase/logs/tui.log`:

```
File ".../textual/screen.py", line 1272, in _invoke_and_clear_callbacks
  await invoke(callback)
...
File "src/sase/ace/tui/modals/gate_input_panel.py", line 167, in _fit_body_scroll
  footer = self.query_one("#gate-input-footer", Static)
textual.css.query.NoMatches: No nodes match '#gate-input-footer' on GateInputPanel()
```

The machine was running current `master`, so the bug is still live.

## Root cause

`GateInputPanel._fit_body_scroll` (`src/sase/ace/tui/modals/gate_input_panel.py`) is a
layout callback scheduled with `call_after_refresh` from `on_mount`, from `on_resize`,
and from itself (it requeues itself whenever the computed max height changes). Its only
DOM guard checks for one node, then calls `query_one` on three:

```python
if not self.is_mounted:
    return
if not self.query("#gate-input-container"):
    return
shell = self.query_one("#gate-input-container", Container)
body = self.query_one("#gate-input-body", VerticalScroll)
footer = self.query_one("#gate-input-footer", Static)   # <- raised
```

That guard cannot catch a partially torn-down screen:

- Textual's `MessagePump._on_invoke_later` forwards the callback to whichever screen is
  active (`self.app.screen._invoke_later(...)`). The callback is not tied to the panel's
  lifetime, so it can still run after the panel is dismissed.
- `is_mounted` is never reset to `False` while a screen is removed.
- `App._prune` closes every descendant node's message pump concurrently. Each child
  detaches from its parent when its own task finishes, so for a short window
  `#gate-input-container` (and maybe `#gate-input-body`) is still attached while
  `#gate-input-footer` is gone.

During mount, all of the container's children are appended together. A state where the
container exists but the footer doesn't can only come from teardown. When
`_fit_body_scroll` runs inside that window, `query_one` raises `NoMatches`.
`SaseApp._handle_exception` (`src/sase/ace/tui/app.py`) treats any unhandled callback
exception as fatal, so the whole TUI exits over a purely cosmetic layout pass.

`CustomGateModal._fit_compact_review_scroll`
(`src/sase/ace/tui/modals/custom_gate_modal.py`) has the same shape and the same latent
crash. It is scheduled the same three ways, guards only `#custom-gate-review-scroll`,
then calls `query_one` on `#custom-gate-container`, `#custom-gate-review-scroll`, and
`#custom-gate-footer`. Fix both.

## Changes

### 1. `src/sase/ace/tui/modals/gate_input_panel.py`

In `_fit_body_scroll`, drop the single-node `if not self.query("#gate-input-container")`
guard. Do all three lookups inside one `try`, and return quietly on `NoMatches` (import
it from `textual.css.query`):

```python
try:
    shell = self.query_one("#gate-input-container", Container)
    body = self.query_one("#gate-input-body", VerticalScroll)
    footer = self.query_one("#gate-input-footer", Static)
except NoMatches:
    # The panel is mid-teardown: a queued fit callback outlived dismissal and
    # Textual has already detached some of the shell's children.
    return
```

Keep the existing `is_mounted` early return and everything after the lookups (region
math, `_body_max_height` memo, requeue) unchanged. Catch `NoMatches` specifically, not
bare `Exception`. Any other error in this method is a real bug and should still surface.
Do not add a broader app-level exception filter.

Update the method docstring to say it tolerates being run on a partially torn-down
panel, because callbacks scheduled with `call_after_refresh` can outlive dismissal.

### 2. `src/sase/ace/tui/modals/custom_gate_modal.py`

Apply the same change to `_fit_compact_review_scroll`. Keep the
`self._data.preview_text is not None or not self.is_mounted` early return. Replace the
`if not self.query("#custom-gate-review-scroll")` guard plus the three `query_one` calls
with one `try` / `except NoMatches: return` around the three lookups. Leave the rest of
the method unchanged.

Keep this a local fix in both files. Do not extract a shared fit helper in this change:
the two methods use different node ids, minimum row constants, and memo attributes, and
a crash fix should stay minimal.

### 3. Regression tests

The real race depends on Textual's prune scheduling, so these tests reproduce the exact
DOM state that crashed instead: a mounted panel with one of the queried children
detached.

**`tests/ace/tui/test_gate_input_panel_layout.py`**: add a parametrized async test over
`"gate-input-body"` and `"gate-input-footer"`:

1. Push a `GateInputPanel(_tall_request())` on a `StyledGateTestApp` and wait with the
   existing `_wait_for_panel` helper.
2. `await panel.query_one(f"#{node_id}").remove()`, then `await pilot.pause()`.
3. Assert `panel.is_mounted` is still `True` and `panel.query("#gate-input-container")`
   is still non-empty. This proves the old guard would have passed.
4. Call `panel._fit_body_scroll()` directly and assert it returns without raising.

Before changing the source, confirm the footer case fails on the old code with
`NoMatches`, so the test really covers the crash.

**`tests/ace/tui/test_custom_gate_modal_sizing.py`**: add the equivalent test for
`"custom-gate-footer"`. Use the existing `_modal(notes=_LONG_NOTES)` and
`_wait_for_mounted_modal` helpers. Remove the footer, pause, assert
`#custom-gate-review-scroll` is still present, then call
`modal._fit_compact_review_scroll()` and assert it does not raise. Before changing the
source, confirm this test fails on the old code.

A container case is unnecessary here. `#custom-gate-review-scroll` is nested inside
`#custom-gate-container`, so removing the container also removes the scroll, and the old
guard already returned early for that state.

No PNG goldens change: the layout math for a fully mounted panel is unchanged.

## Verification

- `uv run pytest tests/ace/tui/test_gate_input_panel_layout.py tests/ace/tui/test_custom_gate_modal_sizing.py tests/ace/tui/test_gate_input_panel.py -q`.
  This covers the new regression tests plus the existing fit, resize, and
  footer-inside-shell tests, which must still pass.
- `just check` (run through `sase tool run` as the repo's guarded-recipe rules require).

## Out of scope

- Other `call_after_refresh` callbacks across the TUI are not audited here. Only these
  two fit callbacks are known to requery a multi-node shell after an is-mounted guard.
- No change to `SaseApp._handle_exception`'s fatal-on-unhandled-exception policy.
- No memory or docs changes are needed.
