---
tier: tale
title: Repair condition-lease cleanup when marker persistence fails
goal:
  Condition admission releases its temporary workspace claim and fails visibly when
  recovery-marker persistence fails.
size: small
proposed_by: bbugyi200.athena.sase-tk.land
bead: sase-tk
create_time: 2026-08-25 13:05:04
status: wip
---

- **PARENT:** [202608/claimed_workspace_if.md](claimed_workspace_if.md)
- **BEAD:**
  [sase-tk](https://github.com/sase-org/sase--beads/blob/main/pages/sase-tk/README.md)

# Repair condition-lease cleanup when marker persistence fails

## Context

Epic `sase-tk` added a temporary operational workspace lease around every project-scoped
`%if` evaluation. The lease is acquired before `condition_workspace_lease.json` is
persisted so recovery can later settle it. The land audit reproduced an uncovered
failure window in `src/sase/agent/launch_condition_workspace.py`: when
`write_json_marker_atomic()` raises after `acquire_operational_lease()` returns, the
helper propagates the raw exception and never calls `release_operational_lease()`. The
admission engine only classifies `ConditionWorkspaceUnavailable` and
`ConditionWorkspaceError`, so this path can both crash admission without a durable
condition-error result and leak the `lease(launch-if:...)` RUNNING claim.

This plan covers only that remaining epic-caused repair. The broader claimed- workspace
behavior, documentation, external bugyi-chops contract, and ordinary lease/recovery
paths are already implemented and verified.

## Implementation

1. In `src/sase/agent/launch_condition_workspace.py`, make persistence of the unsettled
   condition-workspace marker part of the acquisition transaction. If marker persistence
   fails after the operational lease was acquired, release that concrete lease
   immediately and raise `ConditionWorkspaceError` with an operation-specific message.
   Do not return a lease, evaluate the predicate, fall back to `source_cwd`, or leave
   cleanup dependent on a marker that was never durably written.
2. Preserve the existing retry classification: only genuine operational-pool contention
   becomes `ConditionWorkspaceUnavailable`; a marker-write failure is terminal for that
   condition attempt and journals as `condition_error`. Keep successful acquisition and
   the existing idempotent settlement/recovery marker behavior unchanged.
3. Add focused regression coverage in `tests/test_launch_condition_workspace.py` that
   injects marker persistence failure after a fake operational lease is acquired and
   proves:
   - the acquired lease is released exactly once;
   - the helper exposes `ConditionWorkspaceError`, not raw `OSError`;
   - admission records a terminal condition error and never invokes the predicate or
     agent dispatcher; and
   - no unsettled condition-workspace claim/marker is treated as successfully acquired.

## Verification

- Run the focused condition-workspace and adjacent admission/runtime tests.
- Run `just check` as required for SASE source changes.
- Report the exact cleanup/error assertions and command results on the child plan bead
  so the `sase-tk` land agent can resume its interrupted audit and closure.

## Acceptance criteria

- Every failure after operational lease acquisition but before durable marker
  persistence releases the claim exactly once.
- Marker persistence failure is visible as the existing fail-closed `condition_error`
  outcome and cannot execute `%if` in any cwd.
- Existing contention retry, prepared-checkout evaluation, normal settlement,
  cancellation, and recovery regressions remain green.
