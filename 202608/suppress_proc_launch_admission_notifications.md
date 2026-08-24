---
tier: tale
title: Suppress redundant standalone-proc launch-admission notifications
goal:
  Clean proc-only admissions launch without a generic completion notification while
  actionable and agent-bearing admissions remain observable.
size: small
proposed_by: bbugyi200.athena.0cp
---

# Plan: Suppress redundant standalone-proc launch-admission notifications

## Goal

Stop creating the generic `Launch admission finished` SASE notification when a typed
launch contains only standalone proc units and every unit is launched cleanly. Keep the
launch-admission receipt and journal unchanged, because they remain the durable audit
record, and keep the proc row as the user-facing lifecycle/status surface.

Preserve completion notifications when they remain actionable: agent-only launches,
mixed Agent/Proc launches, and proc-only admissions with a skipped unit, condition
error, launch error, cancellation, or other non-launch terminal outcome. Preserve the
existing rule that AXE chop typed requests never send this generic notification.

## Current behavior and evidence

- The reported notification came from direct ACE request
  `launch-d49e43dd-f1c9-4e4a-8f7c-ef41e8281eea`. Its persisted request is a
  `direct_typed_launch` containing one `ProcUnit` for
  `#gh:sase %w:0ck %proc("just install && just check")`; the unit waited, then the
  detached coordinator launched it successfully.
- `src/sase/agent/launch_admission.py` calls `_notify_admission_complete` from both the
  immediate dispatch path and `run_coordinator_in_bundle`. Its current eligibility
  predicate excludes AXE chop requests but does not inspect launch-unit kinds or the
  terminal summary, so a clean proc-only launch produces the same generic notification
  as a mixed or failed admission.
- A `launched` ProcUnit means the native proc supervisor accepted the proc; it does not
  mean the command finished. Direct launch handling already reports the accepted or
  immediate admission result, and the durable `proc-shell` row with origin
  `xprompt-proc` owns subsequent command status. The generic admission notification is
  therefore redundant for the clean proc-only case.

## Notification policy

| Completed typed admission                            | Generic completion notification       |
| ---------------------------------------------------- | ------------------------------------- |
| Non-empty proc-only plan; every unit launched        | Suppress                              |
| Proc-only plan with any skipped/error/cancelled unit | Keep                                  |
| Agent-only plan                                      | Keep                                  |
| Mixed Agent/Proc plan                                | Keep                                  |
| AXE chop typed request                               | Suppress, preserving current behavior |
| Empty or unclassifiable plan                         | Keep (fail open for observability)    |

## Implementation

1. Refine the centralized completion-notification eligibility helper in
   `src/sase/agent/launch_admission.py` so it receives the already-decoded
   `LaunchPlanWire` and completed `AdmissionProgress` in addition to the request
   metadata. Do not reparse the request or duplicate the typed launch grammar.
2. Retain the AXE chop exclusion first. For all other requests, identify a proc-only
   plan with the canonical `ProcUnitWire` payload type and require a non-empty plan plus
   an all-launched terminal summary before suppressing notification creation. Treat an
   empty plan, mixed payloads, skipped units, condition errors, launch errors, and any
   inconsistent summary as notification-worthy.
3. Apply the same helper at both completion sites: synchronous completion in
   `dispatch_typed_launch_request` and detached/reloaded completion in
   `run_coordinator_in_bundle`. Do not change coordinator detachment, dispatch,
   notification sender formatting, receipt/journal writes, proc settlement behavior, or
   direct `run.launch` operation results.
4. Keep the behavior unflagged: this is a ready, narrowly scoped correction to noisy
   output, not an incomplete beta branch or a permanent user-selectable mode.

## Tests

Add focused regression coverage around `tests/test_launch_admission.py` and, where the
durable direct bundle is the clearer fixture, `tests/test_direct_typed_launch.py`:

1. A clean immediate proc-only plan launches through an injected proc dispatcher,
   persists its normal receipt, and does not call `notify_workflow_complete`.
2. A clean proc-only plan completed through `run_coordinator_in_bundle` (using a durable
   direct bundle and injected/mocked proc dispatch) also does not notify. This covers
   the path exercised by the reported waited request rather than testing only the
   synchronous branch.
3. Proc-only plans with a launch error and with a skipped or condition-error outcome
   still notify, and their notification success/error state and receipt attachment stay
   consistent with the terminal summary.
4. An agent-only plan still notifies. Retain the existing mixed-matrix assertion that a
   mixed Agent/Proc plan notifies with matching counts and receipt attachment.
5. Retain the existing AXE chop assertions that the generic notifier is not called.

Prefer behavior-level call-site tests over direct assertions against a private helper,
so both notification branches and persisted outcomes remain covered.

## Verification

1. Run `just install` because this workspace may have stale editable dependencies.
2. Run the focused admission/direct-launch suites:
   `just test tests/test_launch_admission.py tests/test_launch_admission_mixed_matrix.py tests/test_direct_typed_launch.py tests/test_axe_chop_proposal_launch.py tests/test_axe_chop_proposal_launch_clan_dispatch.py`.
3. Run `just check` and repair any lint, type, or scoped-test failures until it passes.
   This notification-policy change does not touch the documented broadening set, so a
   full-suite monitor is not required unless `just check` escalates or reports unusual
   selection.

## Acceptance criteria

- Reproducing a clean proc-only direct launch, including one that waits and completes in
  the detached coordinator, creates no `launch.admission` notification.
- The proc is still recorded and supervised normally, and the admission journal,
  per-unit receipt, operation result, and Proc UI/status remain intact.
- Mixed and agent-bearing admissions still send their existing generic completion
  notification.
- Any non-clean proc-only admission remains observable through the generic completion
  notification with its existing summary and receipt attachment.
- Existing AXE chop notification suppression remains unchanged, and focused tests plus
  `just check` pass.
