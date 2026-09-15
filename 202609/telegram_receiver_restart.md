---
tier: tale
title: Prevent the Telegram receiver from delaying TUI update restarts
goal:
  Restart ACE promptly after updates when only the persistent Telegram receiver and
  monitor shells remain active, while preserving waits for ordinary work.
size: small
proposed_by: bbugyi200.athena.0lf
create_time: 2026-09-15 13:42:06
status: wip
---

# Prevent the Telegram receiver from delaying TUI update restarts

## Diagnosis

The user's suspicion is confirmed. This is an unnecessary wait capped at 60 seconds, not
an indefinite wait in the restart helper.

The investigation read the primary SASE checkout and opened `sase-telegram` through
`sase repo open sase-telegram`. No implementation files were changed.

- `src/sase/default_config.yml` maps the comma leader's `E` to `update_everything`.
  `actions/base.py::action_update_everything_shortcut()` submits the comprehensive
  update. `actions/update_run.py::_on_scoped_update_complete()` calls the shared restart
  helper when code changed. These action paths are under `src/sase/ace/tui/`.
- In the Telegram plugin, `src/sase_telegram/receiver.py::ensure_receiver_running()`
  submits a durable proc with `origin="telegram-receiver"`, no session attribution, and
  no command or idle timeout. Its request fingerprint and concurrency key ensure one
  receiver for the configured bot.
- `src/sase_telegram/scripts/sase_tg_inbound.py::_run_receiver()` loops indefinitely,
  rechecking whether Telegram is enabled and credentials remain available. The chop
  ticks only ensure that this independently supervised receiver exists.
- `src/sase/ace/tui/_proc_observer_store.py::store_proc_row()` preserves the origin.
  `ProcProjection.active_rows()` includes active unattributed rows in every session.
- `src/sase/ace/tui/update_restart.py::running_background_procs()` excludes only
  monitor-origin rows. It therefore treats the persistent receiver as work that must
  finish before restarting. `restart_after_update_when_ready()` polls every second and
  forces the restart at its 60-second deadline.
- The controlled exit path flushes TUI state and stops observation; it does not add
  another wait for this receiver. The proc supervisor launches independently of ACE.

### Read-only reproduction

The live proc snapshot contained exactly one active proc: `mnqrkm88fdgq`, labeled
`Telegram inbound long-poll receiver`, with origin `telegram-receiver`, kind `command`,
no session ID, and no timeouts. It had been running since `2026-09-14T20:30:30.456885Z`;
its supervisor liveness check passed.

Using the workspace virtualenv's Python, the investigation adapted that actual row with
`store_proc_row()`, placed it in a `ProcProjection`, and invoked the real restart helper
with fake timer, notification, and restart callbacks. The helper returned the receiver
as a blocker and emitted `restart queued until 1 proc finishes`. Advancing a mock
monotonic clock showed no restart at 59 seconds and a timeout warning followed by the
mocked ACE+AXE restart at 60 seconds. No real update, restart, proc mutation, or
Telegram request was performed.

## Scope and design

Use one small tale: the cause and fix are localized, and one coding agent can implement
and verify them directly.

Extend the existing ACE restart exemption to the exact durable origin
`telegram-receiver`. Keep the policy in `src/sase/ace/tui/update_restart.py`, next to
the existing monitor exemption, and explain why the receiver cannot be drained before a
restart. The existing origin is sufficient for already-running receiver records; the fix
must not require relaunching the receiver or changing its stored metadata.

This is an ACE presentation/lifecycle scheduling decision over existing read models. It
adds no shared proc lifecycle or storage behavior, so it needs no Rust core wire change
and no plugin implementation change. Do not import the optional Telegram package into
SASE. A new generic durable service-policy schema would add cross-repo work beyond the
demonstrated defect.

Preserve these boundaries:

- Keep ordinary durable procs and session-local workers as restart blockers, including
  unattributed work. Missing timeouts, `kind="command"`, and `shell_kind="proc"` also
  describe ordinary jobs and must not be used as daemon detection.
- Match the exact receiver origin, not labels, command substrings, or all Telegram
  origins. Finite Telegram gate-action procs still follow existing behavior.
- Preserve monitor exemptions, active-status/session filtering, the effective session
  overlay, the 60-second deadline, and `restart_axe=True`.
- Keep the receiver visible and active in the proc projection and Procs pane. This fix
  only changes restart blockers; proc counts, quit dialogs, receiver polling,
  deduplication, termination, and update deployment behavior retain their contracts.
- Keep restart timer callbacks in-memory; add no I/O, subprocess calls, or slow awaits.

## Implementation

1. Add a focused regression module, preferably `tests/ace/tui/test_update_restart.py`,
   using the actual `ObservedProc`/ `ProcProjection` path and fake clock/timer/restart
   callbacks. Establish the failure before changing the helper. Include a receiver
   fixture passed through `store_proc_row()` from a durable `Proc` to cover the real
   origin and session mapping.
2. Update `running_background_procs()` and its docstring to exclude the exact receiver
   origin alongside monitor shells. Keep the change local and reuse the current
   projection accessor and monitor predicate.
3. Cover the following behavior without real timers, Telegram credentials, live-state
   writes, or an actual TUI/AXE restart:
   - A receiver alone, and a receiver with a monitor, restart immediately exactly once,
     without queue/timeout notices or a scheduled restart poll. Parameterize the
     receiver across active statuses (`pending`, `running`, `settling`).
   - A receiver plus ordinary work queues for the ordinary work only. When that work
     finishes, the captured callback restarts even though the receiver is still active.
   - Ordinary work still reaches the existing deadline. Its timeout summary names the
     actual blocker and omits the receiver.
   - An ordinary proc with the receiver's display label but another origin still blocks;
     an exact receiver origin with another display label remains exempt.
   - Filtering leaves the original projection rows and activity counts intact.
4. Extend `tests/ace/tui/test_update_run_actions.py` with a code-changed completion that
   reaches the real `UpdateRunActionsMixin._restart_after_update()` and shared helper
   while a receiver is active. The existing `_Harness` overrides that method, so use a
   small derived harness that restores the production method and captures
   `_restart_tui()`/`set_timer()`. Continue mocking receipt persistence. Assert an
   immediate ACE+AXE restart and preserved receipt handling.

## Verification and acceptance

Read `lint_and_test.md` through `/sase_memory_read` before finishing implementation. Use
the workspace `.venv/bin/python`/pytest: the bare system Python is too old for this
checkout's syntax and does not provide the application environment.

Run the new regression tests plus the existing relevant suites:

```bash
.venv/bin/python -m pytest -q tests/ace/tui/test_update_restart.py tests/ace/tui/test_update_run_actions.py tests/ace/tui/test_proc_actions_session_workers.py tests/ace/tui/test_plugins_browser_pane_sase_update.py tests/ace/tui/test_feature_flags_pane.py
just check
```

Existing suites protect monitor/session behavior, plugin-pane update waits, and the
feature-flag restart caller. Run `just install` if required by dependency validation;
open any additionally needed repo with `/sase_repo` before accessing it. If checks
require a long handoff, use `/sase_monitor` as instructed by project memory.

Acceptance: all listed checks pass; the live-record-shaped receiver no longer causes a
restart wait; ordinary work retains its wait and timeout behavior; the app-level update
completion reaches an immediate ACE+AXE restart with receipt handling intact; and no
production proc is stopped or modified to make the tests pass. Summarize the confirmed
60-second delay, the exact-origin correction, and verification in the implementation
handoff. Submit completion through `/sase_final`; host finalizers own commits.
