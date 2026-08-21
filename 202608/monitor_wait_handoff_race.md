---
tier: tale
title: Prevent monitor settlement races from releasing waiters early
goal:
  Wait dependencies remain blocked until a monitor handoff successor actually reaches a
  successful terminal state.
size: medium
proposed_by: bbugyi200.athena.0a8
create_time: 2026-08-21 20:32:18
status: wip
---

# Prevent monitor settlement races from releasing waiters early

## Problem

`sase-rr.land.w1` declared its `%w:sase-rr.land` dependency satisfied while the
predecessor family was handing off from `sase-rr.land--mon-0` to `sase-rr.land--2`. The
waiter did not observe `ready.json`; its in-runner fallback scan released it at 20:19:04
UTC, while the successor did not start until 20:19:14 UTC.

The fallback scan builds a dependency index in multiple passes. It can read a monitor
artifact before `done.json` is written, retain that missing outcome, and then read the
same artifact's workflow state after monitor settlement has changed it to `completed`.
The generic plan-chain fallback treats a completed workflow without a terminal outcome
as resolved. For monitors, however, that workflow state is a post-settlement display
projection; the monitor's authoritative terminal signal is `done.json` (or its archived
equivalent). Combined with an artifact snapshot taken before the successor is visible,
the family can therefore appear resolved during the handoff gap.

## Implementation

1. Update `src/sase/core/wait_dependency_resolution/_artifact_state.py` so a monitor
   member with no resolved terminal outcome cannot use the generic completed-workflow
   handoff fallback. Identify monitor members through the shared
   `sase.monitor_state.is_monitor_member_role` helper so both explicit family roles and
   legacy monitor suffixes remain supported.
2. Preserve the existing semantics for non-monitor plan-chain handoffs, which
   legitimately may resolve from completed workflow state without `done.json`. Preserve
   monitor resolution from a valid `done.json` outcome and from the archived-completion
   lookup already performed by the index builder.
3. Add a resolver regression test in `tests/test_monitor_wait_dependency.py` that
   constructs the torn state from the incident: an otherwise completed family, a monitor
   whose workflow is `completed` but whose `done.json` is absent, and no visible running
   successor. Assert that the monitor candidate and family remain unresolved. Cover the
   legacy suffix form as appropriate to the helper's compatibility contract.
4. Add or extend coverage in `tests/test_run_agent_wait_deps.py` for the actual
   in-runner fallback entry point, asserting that this torn monitor state does not
   satisfy the dependency. Keep the existing positive cases for ordinary no-`done.json`
   plan handoffs and successfully settled monitor families intact.

## Validation

1. Run `just install` before repository checks so the workspace uses the current
   editable package and native extension.
2. Run focused tests for the shared resolver and fallback integration, including
   `tests/test_monitor_wait_dependency.py` and `tests/test_run_agent_wait_deps.py`.
3. Run `just check`, the required whole-repository lint and diff-scoped test gate. If it
   escalates or reports unusual selection, run `just check-full` through `/sase_monitor`
   as required by the repository instructions.

## Acceptance criteria

- A monitor artifact cannot be considered resolved solely because its workflow state is
  `completed` when no terminal or archived outcome was observed.
- `%wait` and the runner fallback continue blocking throughout a monitor-to- successor
  handoff, including the settlement race reproduced above.
- Ordinary plan-chain handoffs without `done.json` retain their current successful
  resolution behavior.
- Monitors with valid successful terminal or archived outcomes still resolve, and failed
  monitor handoffs continue following their successor until that successor reaches a
  successful terminal state.

## Non-goals

- Changing monitor settlement ordering or the workflow-state projection used by status
  displays.
- Redesigning dependency-index snapshotting beyond removing this monitor-specific misuse
  of the generic plan-chain fallback.
