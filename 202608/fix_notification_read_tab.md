---
tier: tale
title: Fix notification-tab read dispatch
goal:
  Notification tab-wide read and related durable state actions execute through the
  registered CLI contract without losing tab scope.
size: small
proposed_by: bbugyi200.athena.0ab
create_time: 2026-08-22 10:37:53
status: wip
---

# Fix notification-tab read dispatch

## Problem and root cause

Pressing `R` in ACE's notification panel correctly reaches `action_read_tab`, asks for
confirmation, captures the active tab key and visible notification IDs, and submits a
durable notification-state proc. The key binding and the core `mark_tab_read` operation
are not the defect. The proc fails before it can mutate the store because the durable
operation migration left three mismatches in the Python integration:

1. `notification_modal_action_support._notification_state_argv()` appends `--json` to
   both `notify apply-state` and `notify apply-state-many`, but neither operation parser
   accepts that flag. Argparse therefore exits with status 2, matching the reported
   toast.
2. Once the invalid flag is removed, `main.notify_handler.handle_notify_command()`
   forwards only `apply-state` to the durable operation handler. The registered
   `apply-state-many` subcommand falls through to stale usage text and exits with
   status 1.
3. A tab read with exactly one currently loaded row selects `apply-state`, whose read
   implementation marks only that ID and ignores the request's `tab_key`. This violates
   the confirmation promise that the operation covers every unread row in the tab,
   including rows not currently loaded. Tab reads must always use the tab-aware bulk
   operation, even when the captured visible ID set has length one.

The bulk parser and durable `_run_apply_state_many()` implementation already exist, and
the notification store/core already implements tab-scoped reads. This is therefore a
Python CLI/TUI wiring repair; no Rust-core, keymap, default-config, or feature-flag
change is needed.

## Implementation

1. In `src/sase/ace/tui/modals/notification_modal_action_support.py`, make durable
   notification argv match the registered operation CLI:
   - remove the unsupported `--json` argument from single- and multi-row state commands;
   - include the optional `tab_key` when selecting the command shape, so any tab read
     uses `notify apply-state-many`, while ordinary one-row mute/snooze operations
     continue to use `notify apply-state` and ordinary multi-row operations continue to
     use `notify apply-state-many`;
   - retain the existing private request/result sidecar flow, concurrency key,
     fingerprint, captured IDs, and completion behavior.
2. In `src/sase/main/notify_handler.py`, route both `apply-state` and `apply-state-many`
   through `handle_notify_operation()`. Update the fallback usage text so it accurately
   and alphabetically lists the registered notification subcommands.
3. Update notification-modal tests to assert executable argv rather than the currently
   invalid argv:
   - tab reads with one or multiple captured rows always submit `apply-state-many`
     without `--json` and preserve the captured `tab_key` and IDs;
   - one-row mute/snooze commands still use `apply-state`, bulk mute/unmute/snooze
     commands use `apply-state-many`, and neither form includes `--json`.
4. Add a CLI regression test that crosses the parser and top-level notify dispatcher
   with a private durable request sidecar, proves `apply-state-many read` reaches the
   operation handler, invokes the tab-scoped store mutation, writes a successful typed
   result, and exits zero. Expand the notify help expectation to include
   `apply-state-many`; this closes the gap that allowed a registered-but-undispatchable
   command to pass existing unit tests.

## Validation

1. Run the focused notification modal and CLI tests covering read-tab dispatch,
   mute/snooze command selection, notify parsing/dispatch, and durable operation
   results.
2. Run `just install` before repository verification, as required for an ephemeral SASE
   workspace.
3. Run `just check` and resolve every lint, type-check, and diff-scoped test failure. If
   scoped selection escalates or reports an unusual selection, run `just check-full`
   through `/sase_monitor` as required by repository instructions.

## Acceptance criteria

- Pressing `R`, confirming the prompt, and completing the durable proc marks every
  unread notification in the active tab read without an exit-code toast.
- The tab-wide behavior remains correct when ACE currently has only one row from that
  tab loaded but the backing store contains additional matching rows.
- Single-row and marked-row mute/snooze mutations no longer fail on an unsupported
  output flag.
- `sase notify apply-state-many` is both present in help and reachable through the
  normal top-level CLI dispatcher.
- Focused regressions and `just check` pass.
