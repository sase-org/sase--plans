---
tier: tale
title: Restore uppercase G scrolling in the notifications panel
goal: Uppercase G consistently scrolls notification details to the bottom while preserving
  lowercase g and jump-mode behavior.
size: small
proposed_by: bbugyi200.athena.0gv
status: done
---

# Restore uppercase G scrolling in the notifications panel

## Outcome and scope

Make uppercase `G` consistently scroll the selected notification's detail pane to the
bottom. Preserve lowercase `g` scrolling to the top, notification selection, and the
existing precedence of apostrophe jump mode. This is a focused TUI event dispatch fix
that one coding agent can implement directly, so use a `tale` with implementation size
`small`. Shared backend behavior is unaffected; the change belongs in the Python
presentation layer and requires no Rust core changes.

## Diagnosis and evidence

- `src/sase/ace/tui/modals/notification_modal.py` already binds `g` to `scroll_file_top`
  and both `G` and `shift+g` to `scroll_file_bottom`.
- `src/sase/ace/tui/modals/notification_modal_attachments.py` implements those actions
  using `#notification-file-scroll` and `scroll_home(animate=False)` /
  `scroll_end(animate=False)`. The scroll action and production layout work when the
  expected binding is dispatched.
- `NotificationOptionMixin.on_key` in
  `src/sase/ace/tui/modals/notification_modal_options.py` immediately returns outside
  jump mode. It therefore ignores `event.character` during normal navigation. An
  uppercase event represented as `Key("g", "G")` reaches the lowercase `g` binding and
  scrolls to the top. When the pane is already at the top, the key appears to do
  nothing.
- This event representation is already documented by `normalize_jump_key` in
  `src/sase/ace/tui/actions/navigation/jump_hints.py`. Comparable handling exists in
  `config_center_modal.py` and `logs_pane_source_list.py`: the uppercase character is
  checked before the lowercase key name.
- A read-only diagnostic used a real `NotificationModal`, a long text attachment, the
  production `styles.tcss`, Textual 8.0.1, and a 120-by-40 headless viewport. Starting
  at scroll offset 100 with maximum offset 295, `Key("G", "G")` and
  `Key("shift+g", "G")` reached 295 and invoked the bottom action once. `Key("g", "G")`
  instead reached zero, invoked the top action once, and never invoked the bottom
  action. Lowercase `Key("g", "g")` correctly reached zero. Events were posted through
  the app's input path, not directly to the focused widget. The user's exact terminal
  event has not been captured, but this is a reproduced defect that explains the
  reported symptom.
- An in-memory prototype, without editing source files, handled uppercase events in
  `on_key` after the jump-mode branch and stopped their propagation. It passed all three
  bottom representations (`G`, `shift+g` without a character, and `g` with character
  `G`) with either the list or detail pane focused. Each invoked the bottom action
  exactly once and preserved selection. Lowercase `g` still reached zero. With 43
  selectable notifications, jump hints `g` and `G` selected their respective rows;
  `Key("g", "G")` selected the uppercase hint without invoking either scroll action or
  closing the modal.
- Existing tests in `tests/test_notification_modal_action_bindings.py` only inspect
  binding tuples and mock the scroll widget. They cannot detect this event-dispatch
  failure. Existing jump tests do not cover the mismatched lowercase key / uppercase
  character outside jump mode.

## Implementation

1. Add a focused regression in `tests/test_notification_modal_scroll.py` for the real
   input path. Reuse the notification test helpers and mount the actual modal with the
   production notification styles and a synthetic long attachment under `tmp_path`. Post
   `textual.events.Key("g", "G")` through `pilot.app.post_message` after layout has
   settled. Start between the top and bottom, assert `max_scroll_y > 0`, and assert the
   final offset equals the maximum while selection stays unchanged and the modal remains
   open. Confirm this case fails on the original implementation by going to zero.
2. Update `NotificationOptionMixin.on_key` locally. Keep the existing normalized
   jump-key handling first, with an explicit return after handling an active jump mode
   so a jump that exits the mode cannot fall through into scrolling. Outside jump mode,
   recognize `event.key` equal to `G` or `shift+g`, or `event.character` equal to `G`.
   Consume the handled event with `prevent_default()` and `stop()`, then invoke the
   existing `action_scroll_file_bottom()` exactly once. Leave lowercase `g` and
   unrelated keys on their existing binding path. Preserve the current binding tuples;
   making them priority bindings would bypass the required jump interception.
3. Expand regression coverage around the same behavior:
   - Parameterize uppercase key representations, including a missing character on
     `shift+g`, and test both list focus and detail-scroll focus.
   - Exercise lowercase `g` from a nonzero offset and assert it reaches zero. Ensure
     handling an uppercase event cannot subsequently invoke the top binding or invoke
     the bottom action twice.
   - Verify the non-scrollable/empty-detail case remains harmless and leaves the modal
     open; avoid assertions that pass merely because there was no overflow.
   - Extend `tests/test_notification_modal_jump.py` to cover `Key("g", "G")` while jump
     mode is active. With enough rows for both hints, assert uppercase `G` and lowercase
     `g` choose their distinct targets and neither detail scroll action is called.
     Preserve cancellation and unrelated-key handling. Selection changes may
     legitimately reset the preview, so do not assert an unchanged preview offset after
     a row jump.
4. Review `docs/notifications.md` and the modal's default/question/gate hint text. Add a
   brief clarification beside the existing keybinding table that `g`/`G` scroll the
   detail pane with either pane focused and that jump mode consumes hint keys first.
   Keep the advertised `g/G: top/bot` behavior consistent. Review
   `src/sase/default_config.yml`; the notification bindings are currently modal-local
   and the configured app `scroll_to_bottom: "G"` is already correct, so this fix needs
   no configuration-schema or default-key change.

## Verification and acceptance

- Read `lint_and_test.md` and `tui_perf.md` through `/sase_memory_read` in the
  implementation turn. If the checkout's virtualenv is stale, run `just install` before
  verification. Keep this keystroke handler synchronous and limited to in-memory UI
  dispatch; introduce no I/O, workers, or timers.
- Run the focused notification scroll regression before and after the fix. Then run the
  focused set with the checkout interpreter, for example:
  `.venv/bin/python -m pytest -q tests/test_notification_modal_scroll.py tests/test_notification_modal_action_bindings.py tests/test_notification_modal_jump.py`.
  Use normal project test isolation and synthetic notification data; do not mutate the
  user's notification store.
- Run `just check` after the implementation and documentation changes. If the scoped
  lane escalates or reports unusual selection, follow the required `just check-full`
  procedure through `/sase_monitor`; use a monitor as well if `just check` becomes
  long-running. No layout change or snapshot refresh is expected; inspect any unexpected
  visual drift instead of accepting it blindly.
- Acceptance: all supported uppercase event forms reach the actual detail-pane bottom
  from a nonzero starting offset, lowercase `g` reaches the top, focus and selection
  remain stable during detail scrolling, jump hints retain precedence, the modal remains
  open, and required checks pass.

## Limits and follow-through

The defect is in character-sensitive event routing, not notification ordering or
attachment rendering. Do not broaden this into an application-wide keymap rewrite. If
the reported issue persists after these regressions pass, capture the actual incoming
key/character pair and focus in the affected ACE session before choosing another change;
do not claim the user's terminal was directly reproduced.
