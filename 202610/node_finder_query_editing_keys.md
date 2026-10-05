---
tier: tale
title: Fix dead editing keys in the Node Finder query input
goal:
  Backspace, ctrl+u, and every other Input editing or motion key work in the Node Finder
  SEARCH-mode query box, and keys still never leak to the host app.
size: small
proposed_by: bbugyi200.athena.0wm
create_time: 2026-10-05 07:29:37
status: wip
---

# Plan: Fix dead editing keys in the Node Finder query input

## Problem

In the Agents-tab Node Finder modal (`NodeFinderModal`), pressing `/` or `Tab` switches
to SEARCH mode and focuses the `#node-finder-query` `FilterInput`. In that input, typing
printable characters works. Every editing or motion key does nothing: `backspace`,
`ctrl+u`, `ctrl+w`, `ctrl+k`, `delete`, `left`/`right`, `home`/`end`/`ctrl+a`/`ctrl+e`,
and `FilterInput`'s own `ctrl+f`/`ctrl+b`.

## Root cause (confirmed)

The search-mode branch of `NodeFinderModal.on_key` in
`src/sase/ace/tui/modals/node_finder_modal.py` ends with an unconditional
`event.stop()`. Its purpose is to keep keys from reaching the host app ("invalid-key
flash with no app leak", from the modal's original commit).

Textual 8's `Input` handles keys in two ways:

- Printable characters are inserted in `Input._on_key`, which then stops the event. The
  event never reaches the screen, so typing works.
- Editing and motion keys are ordinary non-priority `BINDINGS` on `Input`, such as
  `backspace -> delete_left` and `ctrl+u -> delete_left_all`. Textual resolves these in
  only one place: `App._on_key` -> `App._check_bindings(key)`. That walks
  `screen._modal_binding_chain` (focused Input -> containers -> modal screen). It only
  runs after the `Key` event has bubbled from the Input up to the App.

The screen's `event.stop()` ends that bubble at the screen. `App._on_key` never runs, so
no Input binding ever fires. The screen's priority `BINDINGS` (tab, shift+tab, ctrl+n/p,
up/down, pgup/pgdown) still work because Textual checks priority bindings before it
forwards the key to the focused widget.

How this was confirmed:

- A pilot repro against the real modal: `/`, type `abc`, then press the key. With
  `backspace`, `ctrl+u`, and `ctrl+w` the value stayed `"abc"`. With `left` the cursor
  stayed at 3. The test host's `on_key` recorded no leaked keys, which shows the event
  died at the screen.
- The tests missed it because every existing search-mode test sets `query.value`
  directly (or posts `Input.Changed`). None of them presses an editing key.

This is not caused by the hints-mode `backspace`/`ctrl+u` branches. That code is
unreachable while `_search_mode` is true. It is not caused by keymap config or by
`FilterInput` either.

## Fix

Keep the modal's "no app leak" isolation. Before swallowing the key, run the focused
widget's own non-priority binding with public Textual APIs. This repeats what
`App._check_bindings(key, priority=False)` would have done for the modal chain.

In `src/sase/ace/tui/modals/node_finder_modal.py`:

1. Make `on_key` `async def on_key(self, event: Key) -> None`. Textual awaits coroutine
   handlers. No caller invokes `NodeFinderModal.on_key` synchronously (checked with grep
   across `src/` and `tests/`). Leave the hints-mode branch unchanged.
2. In the search-mode branch, keep `escape` (`_enter_hints`) and `enter`
   (`_jump_highlighted`) exactly as they are. For every other key, call `event.stop()`
   first and then `await self._run_focused_binding(event.key)`. Add a short comment
   explaining why: Textual resolves the Input's non-priority bindings only in
   `App._on_key`, after bubbling, and this screen deliberately stops that bubble.
3. Add the helper:

   ```python
   async def _run_focused_binding(self, key: str) -> None:
       active = self.active_bindings.get(key)
       if active is None or not active.enabled or active.binding.priority:
           return
       await self.app.run_action(active.binding.action, active.node)
   ```

   - `Screen.active_bindings` is public. It walks the modal binding chain from the
     focused widget up and splits multi-key bindings, so `home,ctrl+a` yields `ctrl+a`.
   - Skip priority bindings. Textual already dispatched them before forwarding the key,
     so running them again would fire them twice.
   - Unbound keys (for example `f5`) are still swallowed without a flash. That keeps
     today's search-mode behavior.

`Input.Changed` is a separate bubbling message, not a `Key`, so it is unaffected.
Backspace and `ctrl+u` therefore still go through
`NodeFinderFilteringMixin.on_input_changed` and trigger the latest-wins refilter. The
app's `on_input_changed` still records input activity.

Out of scope:

- The hints-mode key state machine.
- Rendering (legend, pill, layout). No PNG goldens change.
- Other modals. Only the Node Finder has this search-mode swallow pattern.

The fix was prototyped out of tree with a `NodeFinderModal` subclass that overrides
`on_key` this way. Every key in the test matrix below passed with `leaked == []`.

## Tests

Add these to `tests/ace/tui/modals/test_node_finder_modal.py`, reusing `_ModalHost`,
`_modal`, `_node`, and `wait_for`. If the file would trip the `toobig` line gate, put
them in a new sibling `tests/ace/tui/modals/test_node_finder_modal_search_keys.py` that
imports those helpers.

1. A parametrized `test_search_mode_editing_keys_edit_query`. Open the modal, press `/`,
   check that the query has focus, and press `a`, `b`, `c` through the pilot (do not set
   `.value`). Then press the key and assert `(query.value, query.cursor_position)`:
   - `backspace` -> `("ab", 2)`
   - `ctrl+u` -> `("", 0)`
   - `ctrl+w` -> `("", 0)`
   - `left` -> `("abc", 2)`
   - `ctrl+b` -> `("abc", 2)`
   - `ctrl+a` -> `("abc", 0)`

   Also assert `host.leaked == []` and `modal._search_mode` is still true.

2. `test_search_mode_backspace_refilters`. Use rows such as `alpha`, `alpine`, `beta`.
   Type `alp`, wait for `modal._view.query == "alp"`, press `backspace`, then `wait_for`
   `modal._view.query == "al"`. This proves the edit reaches the refilter path.
3. `test_search_mode_unbound_keys_do_not_leak`. In search mode, press an unbound
   non-printable key such as `f5`. Assert the query value is unchanged,
   `host.leaked == []`, and the modal was not dismissed. Also confirm that `escape`
   still returns to HINTS with the query value kept. The existing round-trip test covers
   that, so do not duplicate it if it already applies.

Existing tests must keep passing unchanged, especially:

- `test_invalid_keys_flash_without_dismiss_or_app_leak`
- `test_backspace_and_escape_cancel_pending_prefix` (hints-mode backspace)
- `test_tab_and_slash_round_trip_search_mode`
- `test_enter_jumps_in_search_mode`
- `test_cursor_wraps_and_skips_context_rows` (priority ctrl+n while searching)

## Verification

- `sase tool run check`. It must pass, and it covers ruff, mypy, symvision, and the
  scoped tests.
- No visual snapshot update is expected because the rendered output does not change.
- Do not run `just check-full`.
