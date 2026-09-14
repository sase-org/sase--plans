---
tier: tale
title: Restore epic-launch zero-weight supervision
goal: Epic approval supervision starts immediately without consuming weighted runner
  capacity, while ordinary monitors keep their existing inherited claims.
size: medium
proposed_by: bbugyi200.athena.sase-10h.land
bead: sase-10h
status: done
---

- **PARENT:**
  [202609/gate_admission_never_blocks_approval.md](https://github.com/sase-org/sase--plans/blob/main/202609/gate_admission_never_blocks_approval.md)
- **BEAD:**
  [sase-10h](https://github.com/sase-org/sase--beads/blob/main/pages/sase-10h/README.md)

# Restore The Missing Epic-Launch Zero-Weight Monitor Work

## Context

Epic `sase-10h` requires approved-epic supervision to start independently of the
weighted runner budget. Its Rust prerequisite and non-blocking gate-shell admission
landed in `sase-core` commit `e3e926f` and SASE commit `78570c0611`, respectively. Phase
`sase-10h.3` was then closed with a note claiming that the epic-launch monitor carried
explicit weight `0`, but no commit mentioning that phase exists and the current tree
does not contain the implementation: `StartMonitorRequest` has no weight override,
`start_epic_launch_monitor()` does not supply one, and monitor-member creation only
inherits the planner's weight.

The implementation must be integrated with drift since the epic began. In particular,
monitor start was split across `src/sase/monitor/request.py`, `start.py`, and supporting
modules; gate-capacity tests were split into focused fakey files; and capacity-record,
fleet, and TUI projection paths now live in smaller modules. The ordinary monitor path
must retain its current claim-holding and inherited-weight behavior.

## Implementation

1. Add an internal optional queue-weight override to `StartMonitorRequest`, include it
   in the monitor request fingerprint, pass it through `start_monitor()` to
   `create_monitor_member()`, and layer an explicitly authored override onto the member
   metadata after normal parent metadata inheritance. Preserve the current default:
   general monitors with no override inherit both `queue_weight` and
   `runner_claim_owner_key` exactly as they do today.
2. Have `start_epic_launch_monitor()` request weight `0` for its monitor-backed path. Do
   not add queue metadata to `_submit_epic_launch_task()` or otherwise change the
   fallback proc path, because ordinary procs are outside the agent capacity budget.
3. Carry explicit zero-weight records losslessly through the current scan, capacity,
   local TUI-enrichment, and remote fleet-projection adapters. Accept zero only when
   `queue_weight_explicit` is true; implicit zero, negative, non-finite, boolean, and
   malformed values must remain invalid/fail closed. Keep directive and waiter weight
   validation strictly positive, so user-authored `%q`/`%queue` semantics do not widen.
4. Ensure zero-weight monitor presentation stays quiet: it must neither increase
   `occupied_capacity` nor render a `w0` badge or misleading capacity usage. Keep the
   existing displays for positive non-default weights unchanged.

## Tests And Acceptance

- Add focused request/member tests proving the override affects request identity,
  explicit `0` replaces an inherited positive weight, and a general monitor still
  inherits its parent's positive weight and live claim owner.
- Extend epic-launch tests to assert that only the monitor request carries weight `0`
  and that the proc fallback carries no queue-weight metadata.
- Add projection tests for an explicit-zero filesystem/wire record and remote fleet row,
  plus invalid implicit-zero/negative/non-finite cases and suppression of a `w0` badge.
- Add or restore fakey acceptance around the production monitor-start/admission paths:
  at a saturated weighted limit, the epic-launch-shaped monitor starts with effective
  weight `0`; workers launched by its `sase bead work` command acquire or park using
  their own positive weights; and total occupied weight never exceeds the limit. Retain
  an explicit regression proving a general monitor still holds/transfers its inherited
  positive claim.
- Run the focused suites, `just install`, and the repository-required `just check`. Do
  not modify the Rust queue-directive policy; its existing explicit-zero record and
  non-reusable-lineage tests are already the landed prerequisite.
