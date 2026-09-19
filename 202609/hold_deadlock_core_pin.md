---
tier: epic
title: Ratchet the hold-deadlock source pin
goal: SASE's pinned Rust core contains the hold-deadlock binding used by production
  Python code.
parent_bead: sase-11l.11
phases:
- id: source-pin
  title: Ratchet and verify the hold-deadlock core revision
  description: 'source-pin: move SASE''s source pin to a current core revision containing
    the deadlock binding and verify the pinned build.'
  size: small
  depends_on: []
proposed_by: bbugyi200.athena.sase-11l.11.land
create_time: 2026-09-19 04:37:21
status: wip
bead_id: sase-11l.11.5
---

- **PROMPT:** [prompts/202609/hold_deadlock_core_pin.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/hold_deadlock_core_pin.md)
- **PARENT:** [202609/hold_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/hold_landing_repairs.md)
- **BEAD:** [sase-11l.11.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11l/sase-11l.11.5.md)

# Ratchet the hold-deadlock source pin

## Context

The landing audit for `sase-11l.11` found that phase `sase-11l.11.4` added a production
call to `sase_core_rs.agent_hold_deadlock_reaches` in SASE commit
`388d5160308367121467539eb3114bc342833ffc`, while that same commit pinned
`sase-core-revision.txt` to core commit `8261449c5f30870604d369635f6214125e8da0f1`
(`v0.34.61`). The binding itself landed one core commit later, in
`0a7301ca435d7ace7dfd732455a4997ad34b3624`. Core master has since released `v0.34.62` at
`093eb2dc296ebdd6568ebe809bd37a5d9e3d82a7`, which contains that binding.

This mismatch is caused by the epic and must be repaired before its landing can resume.
The separate published-package problem remains owned by ready task `sase-10d` and active
epic `sase-12y.4`: PyPI still exposes only incomplete `sase-core-rs 0.34.48`, so do not
change the dependency floor or `uv.lock` in this tale.

## Implementation

1. Recheck current SASE master and the current core remote head before editing, because
   `sase-12y.4` is concurrently working on the core floor. Open the linked core through
   `/sase_repo`; do not guess its path. Confirm that the target core revision is a
   descendant of `0a7301ca435d` and therefore exports the `agent_hold_deadlock_reaches`
   PyO3 binding.
2. If `sase-core-revision.txt` does not already name such a descendant, use the
   repository's supported `just ratchet-core-revision` workflow to move it to the
   current core remote HEAD. Do not hand-edit the pin and do not change `pyproject.toml`
   or `uv.lock`. If concurrent work already fixed the pin, retain that newer compatible
   revision and make no duplicate edit.
3. Install/build against the exact pinned core revision, then run the focused
   hold-deadlock regression suite and SASE `just check`. Verify
   `tools/ratchet_core_revision --check` reports no pending source-pin ratchet at the
   observed remote head, and verify the installed binding exposes
   `agent_hold_deadlock_reaches`.

## Acceptance

- `sase-core-revision.txt` resolves to a core commit containing `0a7301ca435d` and the
  `agent_hold_deadlock_reaches` binding.
- The focused tests in `tests/test_run_agent_wait_slot_hold_deadlock.py` pass against
  that pinned core build.
- `just check` passes.
- The published-package floor and lockfile remain untouched; their independent release
  work stays on `sase-10d` / `sase-12y.4`.
- The worker records the exact SASE pin and core commit verified so the `sase-11l.11`
  land agent can resume its close audit through the `parent_bead` link.
