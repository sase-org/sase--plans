---
tier: tale
size: small
title: Make SASE monitors notification-neutral
goal:
  Starting, handing off to, settling, stopping, timing out, losing, or reconciling a
  SASE monitor adds no SASE notification rows, while monitor state and follow-up
  failures remain durable and inspectable and notifications independently emitted by the
  monitored command or a follow-up agent remain unchanged.
proposed_by: bbugyi200.athena.zz
create_time: 2026-08-13 16:25:28
status: wip
---

# Make SASE monitors notification-neutral

## Context and confirmed behavior

Epic `sase-kp` introduced monitors as command-running members of an agent family. Its
archived design explicitly added a monitor completion notification and a failure
notification for a dropped `--next` launch. That was the wrong product contract:
monitors are an execution and handoff mechanism, not a new notification-producing
workflow.

The current tree has two independent sources of monitor-added notifications:

1. `src/sase/monitor/supervise.py` and `src/sase/monitor/reconcile.py` both call
   `notify_monitor_complete()` after terminal artifacts are durable.
   `src/sase/monitor/settlement.py` turns that into a routine `sender="monitor"`
   notification for every terminal state and, when a follow-up cannot launch, calls
   `notify_monitor_followup_dropped()` for an additional alarm notification.
2. An in-agent monitor start kills the starter runner, whose handoff handler returns
   `outcome="monitored"` and resets the killed signal. The ordinary runner shutdown path
   therefore still invokes `send_completion_notification()` for the starter, producing a
   `sender="user-agent"` completion notification even though the agent merely handed
   control to a monitor.

This is observable at user scale: a read-only inspection of the 2026-08-13 inbox found
29 non-silent `sender="monitor"` rows, including routine command completions, epic
launch completions, and failed-follow-up reports.

Monitor failures do not need a notification row to remain diagnosable. Settlement
already persists command state, exit code, timeout details, output, follow-up outcome,
degraded reason, prompt path, and follow-up error into `agent_meta.json`, `done.json`,
the retained monitor log, and the agent artifact index. Those records power the Agents
UI and `sase monitor list|show`.

## Behavioral contract

The monitor mechanism itself must append zero SASE notifications:

- A starter agent that successfully hands off with `outcome="monitored"` does not send
  its ordinary agent-completion notification.
- Normal completion, command failure, timeout, explicit stop, dead-supervisor failure,
  pre-reboot loss, and reconciliation do not send a monitor completion notification.
- A failed or degraded `--next` launch is recorded on the monitor but does not send a
  routine notification or a special alarm notification.
- Do not replace these rows with `silent=True` notifications; no notification record
  should be created.
- A command run by a monitor may still emit notifications under that command's own
  workflow contract (for example, `sase bead work` completing an epic launch), and a
  successfully launched follow-up agent may still send its normal completion
  notification. Those are not monitor-added notifications and must remain unchanged.
- Do not rewrite, dismiss, or delete historical notification rows as part of this
  change.

## Implementation

1. Remove monitor settlement notification emission without disturbing settlement
   ordering or diagnostics.
   - Delete the `notify_monitor_complete()` calls and imports from both the live
     supervisor and dead-supervisor reconciliation paths.
   - Remove `notify_monitor_complete()` from `src/sase/monitor/settlement.py`, update
     that module's description/exports, and leave `settle_claim_and_followup()`, claim
     release, follow-up outcome/error persistence, terminal marker writes, index
     updates, and ACE refresh pulses intact.
   - Remove the now-unused `notify_monitor_followup_dropped()` sender and its imports. A
     dropped follow-up remains represented by `monitor_followup_outcome`,
     `monitor_followup_error`, and (when available) `monitor_followup_prompt_path`.

2. Suppress the starter runner's handoff completion at the generic completion boundary.
   - Extend the early outcome guard in
     `src/sase/axe/run_agent_runner_finalize.py::send_completion_notification()` so
     `outcome="monitored"`, like the already-suppressed `plan_rejected` outcome, returns
     before constructing or sending a completion payload.
   - Keep the starter's successful done marker, saved handoff chat, family promotion,
     workspace-claim transfer, metrics, and `outcome="monitored"` classification
     unchanged. Only its notification side effect is removed.

3. Replace notification-positive tests with regressions for notification neutrality.
   - Remove the monitor sender and `notify_monitor_complete()` tests from
     `tests/notification_store/test_senders.py`; retain unrelated notification sender
     coverage.
   - Add focused completion-runner coverage proving `outcome="monitored"` never calls
     the workflow notification sender while ordinary successful and failed agent
     outcomes still do.
   - Exercise both monitor settlement entry points with an isolated notification store:
     at minimum cover a clean supervisor completion, a failed `--next` launch whose
     error remains in terminal artifacts, and dead/pre-reboot supervisor reconciliation.
     Assert that no notification row is appended while the existing metadata,
     done-marker, claim-release, and follow-up assertions still pass.
   - Keep or strengthen tests showing that follow-up failures remain discoverable via
     the persisted monitor fields, so removing the alarm cannot accidentally erase the
     diagnostic.

## Verification

1. Run `just install` before project commands, as required for an ephemeral workspace.
2. Run the focused runner, monitor supervisor/store, and notification-sender tests
   touched above. Confirm clean completion and error/reconciliation cases leave the
   isolated notification store empty while their terminal artifacts remain complete.
3. Search the Python tree for `notify_monitor_complete`,
   `notify_monitor_followup_dropped`, and monitor-specific calls to
   `notify_workflow_complete`; only intentionally historical/non-code references may
   remain.
4. Run `just check` for the required whole-repo lint gates and diff-scoped tests. If it
   escalates or reports unusual selection, follow the repository guidance and run
   `just check-full` through `/sase_monitor` with a continuation action.

## Non-goals

- Changing monitor lifecycle, status, timeout, output capture, claim ownership, or
  follow-up launch behavior.
- Suppressing notifications that the monitored subprocess or a later follow-up agent
  independently owns.
- Cleaning up notification history already written to the user's inbox.
