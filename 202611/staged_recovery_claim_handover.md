---
tier: tale
title: Finish staged-handoff recovery landing
goal:
  A crashed runner's recorded successor takes over the origin workspace claim and
  launches exactly once, holds found by recovery send one deduplicated readopt notice,
  and epic sase-1pj.6.1 and its parent phase sase-1pj.6 are closed.
size: medium
proposed_by: bbugyi200.athena.sase-1pj.6.1.land
proposed_with: claude/opus
bead: sase-1pj.6.1
create_time: 2026-11-01 13:37:28
status: wip
---

- **PARENT:**
  [202610/staged_handoff_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202610/staged_handoff_recovery.md)
- **BEAD:**
  [sase-1pj.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pj/sase-1pj.6.1.md)

# Finish staged-handoff recovery: successor claim handover and recovery hold notices

This tale finishes the landing of epic `sase-1pj.6.1` ("Recover staged handoffs without
repeating adoption", plan `plan:202610/staged_handoff_recovery.md`). All eight phases
(`sase-1pj.6.1.1` through `sase-1pj.6.1.8`) are closed and on master. The land audit
found two remaining gaps that this epic caused. Fix both, then run the epic's closeout
in the same turn (the final step below).

Before starting, run
`sase bead read sase-1pj.6.1 -r "Need epic notes and land triage before finishing the landing"`
and read its notes; the last note records the land agent's follow-up triage. Read the
`lint_and_test.md` memory note before verification. Do not create task beads for the
already-triaged follow-ups, and do not run `just check-full`.

## Gap 1: detached successor delivery cannot claim the dead origin's workspace

Kinds `pipe`, `plan`, and `questions` hand off to a recorded successor. When the
originating runner is dead, the reconciler chop (`staged_handoff_reconcile`) and
`sase agent readopt` deliver that successor through `deliver_staged_successor_detached`
in `src/sase/axe/run_agent_successor_delivery.py`. That function launches through
`spawn_agent_session_successor` or `spawn_detached_child`
(`src/sase/agent/detached_child.py`). Both calls pass `transfer_from_pid=None` and the
origin's own `workspace_num`; the staged identity builders copy it from
`ctx.workspace_num`.

With `transfer_from_pid=None`, `src/sase/agent/launch_spawn.py` claims the workspace
fresh with `claim_workspace`. In sase-core, `plan_claim_workspace_from_content`
(`crates/sase_core/src/agent_launch/workspace_claims.rs`) refuses any existing RUNNING
row with the same workspace number, even when that row's PID is dead. After a runner
crash, the dead origin's RUNNING row is still present. `_staged_origin_retains_claim` in
`src/sase/ace/scheduler/stale_running_cleanup.py` deliberately never releases a row
whose ace-run artifacts hold `.sase_handoff_staged.json`. The launch therefore fails
with "workspace #N is already claimed".

The failure is permanent. `delivery_state` was already set to `submitted` before the
launch, and every later attempt returns held with "successor launch was already
submitted but no live or terminal destination can be proven". That breaks the epic's
acceptance rows "Pipe/auto continuation prepared, runner dies" and "Inline success
receipt written, runner dies before successor starts". No test caught it because
`tests/test_staged_reconcile_delivery.py`, `tests/test_staged_recovery_service.py`, and
`tests/axe/test_run_agent_successor.py` stub the spawn and never touch a real RUNNING
field. Gate and monitor recovery are unaffected: `gate_turn/start_claim.py` and
`monitor/start_claim.py` transfer from the creator PID or release the claim.

Required behavior: under the adoption lock, decide claim ownership in
`deliver_staged_successor_detached` before it writes `delivery_state="submitted"`.

- Home mode, or a successor `workspace_num` of 0 or missing: unchanged.
- The workspace's RUNNING row is the origin's own claim and the origin runner is not
  live. "Origin's own" means the same workspace number, `pid` equal to the origin's
  `agent_meta.json` pid, and `artifacts_timestamp` equal to the origin artifacts
  directory name; also match `cl_name` when the row records one. Reuse the existing
  liveness and process-identity check further down that function. Launch with
  `transfer_from_pid=<origin pid>` so `transfer_workspace_claim` moves the row
  atomically to the successor. This is the same claim-continuity path that
  `monitor/followup` and `run_agent_retry_spawn` use. Apply it in both spawn branches.
- No RUNNING row for that workspace: keep the fresh claim (`transfer_from_pid=None`).
- Any other owner holds the workspace, such as a different PID or timestamp, a
  gate/monitor workflow row, or a reassigned workspace: return a `held`
  `SuccessorDeliveryResult` whose reason names the workspace number and the conflicting
  owner. Do not mark the record `submitted`, so recovery can retry after the conflict
  clears. Never release or overwrite another owner's claim.
- The transfer stays PID-matched and atomic, so a race between this decision and the
  launch fails safely. Keep the existing sticky `submitted` semantics for launches whose
  outcome is genuinely uncertain.
- Pinned origin rows: Rust `transfer_to` clones the row, so `pinned` carries over. A
  pinned origin row comes from sase-1pj.7.1.3 runner retention or its fail-closed pin.
  Find out whether `sase.running_field` already has a path that clears a pin on the
  transferred row, and use it so the live successor does not inherit a stale retention
  pin. If no such path exists, explain in a comment, and pin with a test, why a pinned
  successor row is safe. Add a sase-core API only if no existing path works. Such an API
  belongs in `agent_launch/workspace_claims.rs` and needs the normal binding and
  `sase-core-revision.txt` pin workflow.

Keep this policy next to the existing Python claim preflight precedent
(`monitor/start_claim.py`). Do not add another claim store or scheduler.

Regression tests use a real temporary project file and real RUNNING-field functions
(`claim_workspace`, `get_claimed_workspaces`). Fake only the OS process boundary, at the
lowest launch seam that still runs the real claim step (see how the existing
launch/spawn tests fake process creation). Cover:

1. Dead origin row, then reconcile with `reconcile_staged_handoffs`: the successor
   launches once, and the RUNNING row now belongs to the successor's PID and launch
   timestamp, with no origin row left. The receipt is adopted, the successor record
   reaches `submitted` with its `launch_pid`, and a second pass is a no-op.
2. The same scenario through explicit recovery
   (`recover_staged_handoff(..., mode="explicit")`, the `sase agent readopt` service).
3. Both spawn branches: plain detached child and agent-session attach plan.
4. The workspace is held by another live owner: held with the precise reason, no spawn,
   and `delivery_state` stays below `submitted`. After that owner releases, recovery
   delivers exactly once.
5. A pinned origin row, with whatever pin outcome you chose above.

## Gap 2: holds found by recovery send no readopt notification

The epic's readopt phase was meant to update the held-handoff notification "including
failures discovered by recovery" and to "deduplicate unchanged notifications". The
reconcile phase added "repeated failures must not flood notifications". Today only the
runner path (`_held_staged_handoff_note` in `src/sase/axe/run_agent_runner_finalize.py`,
used by `send_completion_notification`) tells the user about a held handoff. When the
runner is dead and the automatic sweep in `src/sase/axe/staged_handoff_reconcile.py`
writes a `handoff_not_adopted` receipt, the hold reaches only a chop log line. The user
is never told to run `sase agent readopt`.

Required behavior:

- After `recover_staged_handoff(origin, mode="automatic")` returns a non-adopted result,
  read the origin's adoption receipt. If it is a `handoff_not_adopted` receipt for the
  current `staged_id`, send one notification for that origin. It names the origin agent,
  the handoff kind and stage, the receipt reason, and the exact
  `sase agent readopt <origin-agent>` command.
- Reuse the runner's note builder for the text. Move it to a shared helper if needed,
  but leave the runner's existing notification text and tests unchanged. Do not write a
  second phrasing.
- Deduplicate on origin, `staged_id`, and receipt reason, across passes and across
  scheduler processes. Unchanged holds never notify twice; a changed reason notifies
  once more. Pick the smallest mechanism the notification store supports.
  `sase.notifications.senders.notify_workflow_complete` accepts a `notification_id`;
  check whether the store dedupes on it. Otherwise persist the last-notified key next to
  the reconcile checkpoint, which `atomic_write_json` already writes.
- No notification when automatic recovery declines without writing a receipt: a live or
  unknown runner, a busy lock, unfinished settlement, a kill or discard with no new
  receipt, or an already-adopted stage. Explicit `sase agent readopt` already prints its
  result and does not notify.
- A notification failure is logged as a pass error and never aborts the sweep.

Tests: two passes over one held origin notify once, and a changed reason notifies again.
Ineligible origins never notify. The text contains the exact readopt command.

Update the `staged_handoff_reconcile` job description in `src/sase/default_config.yml`
and its entry in `docs/reference/automation/scheduler-housekeeping.md` with one sentence
on the hold notification. Read `docs_writing.md` with your memory-read skill before
editing docs.

## Verification

Run the focused suites: the new regression tests,
`tests/test_staged_reconcile_delivery.py`, `tests/test_staged_handoff_reconcile.py`,
`tests/test_staged_reconcile_crashes.py`, `tests/test_staged_recovery_service.py`,
`tests/test_agent_readopt_cli.py`, `tests/test_run_agent_runner_notifications.py`,
`tests/axe/test_run_agent_successor.py`, `tests/test_stale_running_cleanup.py`, and
`tests/agent/test_detached_child.py`. Then run `just fix` and `sase tool run check`. Run
it through `/sase_monitor` if it may exceed your synchronous limit. If sase-core
changes, run `sase tool run check` there too and follow the core pin workflow. Do not
weaken assertions. If a failure reproduces identically on the clean base, check the
existing owners before treating it as yours, for example `sase-1ic` for the TUI import
budget.

## Final step: close out epic sase-1pj.6.1 and its parent phase

Do this in the same turn as the code. The closeout must not wait for this tale's own
commit, push, or CI. Never use `--force`.

1. Run `sase bead epic-symbols sase-1pj.6.1`. Resolve every listed `--epic-symbol`
   entry: wire it up, privatize it, add a non-test pragma, or delete it per the
   Symvision epic-whitelist policy. Re-key an entry only if a still-open later bead
   truly needs it. The audit found none, and new public symbols from this tale must be
   consumed, not whitelisted.
2. Close the epic: `sase bead close sase-1pj.6.1 --note "<verification>"`. The note
   states what was verified:
   - all eight phases and their commits (d3d26a5b55, b674079c2b, 9c17b8454b, d9129cfd4b,
     6a2c90f41b, 406c57f126, 288d2afeeb, a19243cdf6);
   - the sase-core recovery commits 73ec04a0 and 16a3ed0f, both contained in the
     `sase-core-revision.txt` pin;
   - integration with the concurrent retention work (sase-1pj.7.1.1-7.1.3) and lost-gate
     withdrawal (sase-1qk.4), plus the refactor splits;
   - the follow-up triage outcomes recorded in the epic notes;
   - this tale's two fixes and their tests, with the check ToolRun id.
3. Run `just symvision` and confirm it is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   printed by `sase bead read sase-1pj.6.1`, `202610/staged_handoff_recovery.md` in the
   plans sidecar. Open the sidecar with `/sase_repo` (`sase repo open plans`) and edit
   only the printed path.
5. The epic's `parent_bead` is phase `sase-1pj.6` (adoption-recovery). Its scope is
   idempotent adoption for every kind, reconciliation of staged handoffs whose runner
   died after finalization, and a re-adopt command for held handoffs. Confirm the closed
   child epic covers all three, and confirm `sase bead epic-symbols sase-1pj.6` is
   empty. Then close only that phase:
   `sase bead close sase-1pj.6 --note "<what you verified>"`. Do not close the
   containing epic `sase-1pj` or any other ancestor; its own land agent owns that.
