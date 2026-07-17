---
tier: tale
title: Reliable update-confirmation commit scrolling
goal: 'Long incoming-commit previews stay fully contained and accessible in update
  confirmation modals, with Ctrl+D and Ctrl+U providing discoverable, regression-tested
  half-page scrolling.

  '
create_time: 2026-07-17 11:36:40
status: done
prompt: 202607/prompts/update_confirm_commit_scrolling.md
---

# Plan: Reliable update-confirmation commit scrolling

## Context

The screenshot shows `PluginActionConfirmModal`, the shared confirmation modal used by SASE dev, mixed/managed, and
plugin update flows. Its `#plugin-action-commits` `VerticalScroll` already declares modal-local `ctrl+d` / `ctrl+u`
bindings, half-page actions, and an overflow-only border hint. The existing interaction test calls those action methods
directly, however, so it does not prove that real key events reach the modal.

More importantly, the current layout combines an auto-height modal and commits pane with independent maximum heights. At
a terminal size approximating the supplied screenshot, a long preview can lay the commits pane out below the modal's
clipping boundary: the key actions change `scroll_y`, but the pane's bottom border, hint, and trailing rows may remain
outside the visible modal. The work should preserve the existing background incoming-commit loader and repair the
presentation/interaction boundary rather than duplicate the already-present bindings or introduce another data path.

## Implementation

1. Make the commits-bearing variant of `PluginActionConfirmModal` allocate a bounded vertical viewport in
   `src/sase/ace/tui/styles.tcss`. Keep the confirmation summary and controls visible, let the incoming-commits pane
   consume the remaining space up to its intended viewport cap, and ensure the pane's complete border remains inside
   `#plugin-action-container` at both compact and tall/wide terminal sizes. Overflow must be represented by the inner
   `VerticalScroll`, not by clipped children of the outer modal.

2. Retain the modal-scoped `Ctrl+D` / `Ctrl+U` behavior in `src/sase/ace/tui/modals/plugin_action_confirm_modal.py`,
   routing both keys to the incoming-commits viewport in deterministic half-page increments without animation. Keep
   absent, hidden, and non-overflowing commit panes as safe no-ops. Recompute the pane subtitle only after loaded/error
   content has been laid out so `ctrl+d/u scroll` appears exactly when the user can scroll and remains within the
   visible border. Keep refresh callbacks synchronous and thin; no new I/O or work may enter the keystroke or render
   path.

3. Strengthen `tests/ace/tui/test_plugins_browser_pane_install.py` around the shared modal. Drive the feature through
   `AcePage.press("ctrl+d")` and `AcePage.press("ctrl+u")`, assert half-page movement and clamping with a long grouped
   commit list, and assert the commits pane stays geometrically contained by the modal. Cover short/absent content so
   the hint stays hidden and both keys remain harmless. These tests should exercise the shared widget rather than
   duplicate cases for every update caller.

4. Add or extend a Config Center plugin-action PNG snapshot with a long update confirmation at a representative small
   viewport. Capture the overflow hint and contained panel (and, if useful for clarity, the state after one scroll
   keypress) so future CSS changes cannot silently reintroduce clipping. Accept only the intentional golden change.

The bindings are local modal controls, consistent with other confirmation/preview screens, so this change should not add
a configurable global keymap or alter `src/sase/default_config.yml`.

## Validation

- Run the focused plugin-action modal tests, including real `Ctrl+D` / `Ctrl+U` pilot input and geometry assertions.
- Run the targeted PNG visual snapshot test, inspect actual/expected/diff artifacts, and update its golden only after
  confirming the contained long-content layout and visible hint.
- Run `just test-visual` for the complete ACE visual suite, then run the required `just check` repository gate.

## Risks and safeguards

- Textual resolves `auto`, fractional, and maximum heights differently as parent space changes. Test at more than one
  viewport size and wait for incoming content plus layout settlement before asserting scroll extents.
- Do not make the entire modal scroll if the existing inner commits viewport can own overflow; keeping confirmation
  context and controls stationary makes the key target predictable.
- Preserve the off-thread loader and the current thin `call_after_refresh` hint synchronization in accordance with the
  TUI responsiveness rules.
