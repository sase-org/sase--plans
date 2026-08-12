---
tier: epic
title: Finish and land task bead gate convergence
goal: 'Restore the two promised convergence regressions missing from epic sase-k0,
  verify the combined implementation against intervening master changes, close the
  epic with a complete audit record, clean post-close Symvision findings, and mark
  its durable plan done.

  '
phases:
- id: convergence_regressions
  title: Complete the promised convergence regression coverage
  depends_on: []
  size: small
  description: 'convergence_regressions: add the removed-then-re-enabled fresh-g1
    and cross-project-key exact-convergence regressions promised by the original plan,
    fixing implementation defects if either regression exposes one, and rerun the
    focused task-gate suites.

    '
- id: land_task_gate_convergence
  title: Verify and close epic sase-k0
  depends_on:
  - convergence_regressions
  size: medium
  description: 'land_task_gate_convergence: verify the combined tree and intervening
    commits, close sase-k0 without force with the complete audit and follow-up outcomes,
    run post-close Symvision cleanup, finish any task made obsolete by that cleanup,
    run landing checks, and set status done in the original durable plan.

    '
proposed_by: bbugyi200.athena.sase-k0.land
parent_bead: sase-k0
create_time: 2026-08-12 12:46:14
status: wip
bead_id: sase-k0.4
---

- **PROMPT:** [prompts/202608/finish_task_gate_convergence.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_task_gate_convergence.md)
- **PARENT:** [202608/task_gate_convergence.md](https://github.com/sase-org/sase--plans/blob/main/202608/task_gate_convergence.md)
- **BEAD:** [sase-k0.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-k0/sase-k0.4.md)

# Plan: Finish and land task bead gate convergence

## Audit already completed

Epic `sase-k0` has three closed phases and no epic-level notes. The original durable
plan is `plans:202608/task_gate_convergence.md`. The landing audit read the plan, every
child bead and note, current source, tests, and all three epic commits:

- `07f050d3a28091a0b7ef28a4e7ca1502e7ec3398` (`sase-k0.1`) adds the shared pending gate
  resolver and routes TaskTriage lookup through it.
- `875f67b74da1e3829b9b2ec72be40df8e9be6726` (`sase-k0.3`) settles TaskTriage and
  BeadSnooze gates after a successful task-bead close, with a single scan and a zero-I/O
  non-task/no-op path.
- `95a9b457502c898d74c448219eec417e6800cd11` (`sase-k0.2`) sweeps inactive-project state
  and producer-owned gates forgotten by lane state while preserving unreadable projects
  and failing closed on an unavailable inventory.

The existing focused suites passed 54 tests after `just install`:

```text
tests/test_bead/test_gate_lookup.py
tests/test_bead/test_task_triage_lookup.py
tests/test_bead/test_cli_close_gate_settle.py
tests/test_axe_chop_bead_task_triage.py
tests/test_axe_chop_bead_task_triage_snooze.py
tests/test_axe_chop_bead_task_triage_presentation.py
```

The audit found one epic-owned gap: the original `chop_sweep` phase explicitly required
coverage for a swept project re-enabled with a fresh `g1` request and a regression in
which the same live bead exists under a different project key, but neither test is in
the combined tree.

The six non-epic commits after the first epic commit were reviewed:

- `ad11756e6ed919d83f30c69fdb82d3c37c59b955` removes the retired ACE Chats pane.
- `51996c57e0ea657de3202489af924f2e19d7055d` adds bead-list summaries.
- `d4c4efda57da358787c94801d3d8cdea038c05af` renders AXE overrun indicators.
- `d6f435c7a2a50678560e377d2604a08194408d7e` lazily materializes prompt sidecars.
- `fb33e3c1f9ba8122392eeec67aee1b05874c0e88` moves external mirrors to their own
  lumberjack and is the only intervening change overlapping `default_config.yml`.
- `e4391c373df946f87fe6f48b37338a0d3f7f25c7` fixes lazy imports in ACE/logging.

The final config retains both the external-mirror lane and the task-gate convergence
contract, and no other intervening change duplicates or conflicts with the epic.

Two `PROPOSED FOLLOW-UP:` entries were triaged through `/sase_new_task`:

- `sase-k0.1` proposed raising the stale `sase-core-rs` floor. The current floor probe
  independently reports `stale_actionable` for `0.24.0`, including
  `bead_needs_external_ref_migration`. This exactly duplicates ready task `sase-jj`,
  which received a +1, and was also recorded on active epic `sase-jx.5` because phase
  `sase-jx.5.3` causally owns the dependency-window ratchet. No new task was created.
- `sase-k0.2` proposed removing stale `sase-js` Symvision epic-symbol entries. This
  exactly duplicates ready task `sase-kc`, which received a +1. No new task was created.

`sase-k0.3` proposed no follow-up.

## Phase `convergence_regressions`: Complete promised coverage

Extend `tests/test_axe_chop_bead_task_triage.py` with two behavior-level regressions
using the existing real state helpers and narrow monkeypatch seams:

1. Start with lane state for a project and a pending gate at a generation greater than
   one. Run a healthy inventory pass where the project is absent, assert the pending
   gate is canceled and the entire project entry (including generations) is removed,
   then reintroduce that project with the live task and assert the newly created request
   ID ends in `-g1` and does not reuse the canceled request ID.
2. Seed stale state and a pending produced gate for a removed project key while the same
   bead ID is live under a different enabled project key. Run one reconciliation pass
   and assert the removed-project gate is canceled, the live project gets exactly one
   expected gate, the stale project disappears from state, and no second produced gate
   survives outside the state expectation set.

Do not weaken fail-closed behavior or producer ownership to make the tests pass. If a
test exposes an implementation defect, fix the smallest reconciler seam and add any
necessary focused assertion.

Run the six focused suites listed in the audit section. Record the exact result on phase
bead `sase-k0.2` as a note because these tests complete that phase's promised coverage,
while leaving its already-closed resolution intact.

## Phase `land_task_gate_convergence`: Close and clean up

Re-read `sase-k0`, all three children, and their notes after the regression phase. Check
the Git range from the first epic commit through current `master` again in case more
commits landed during execution, excluding commits whose `SASE_BEAD` belongs to
`sase-k0`; integrate any new caller, duplicate, or conflict before closing.

Run the focused suites and the repository's available pre-close checks. Then close the
epic without `--force`:

```bash
sase bead close sase-k0 --note "<complete verification, integration, and follow-up record>"
```

The close note must include the three epic commits and verified behaviors, exact test
results, the intervening-commit integration conclusion, the restored regression
coverage, and both proposed-follow-up outcomes with their proposing child IDs
(`sase-k0.1` -> duplicate `sase-jj` plus active epic `sase-jx.5`; `sase-k0.2` ->
duplicate `sase-kc`; `sase-k0.3` -> none). If close is rejected, finish or reopen the
named descendants; never force a successful resolution.

After the close, run `just symvision` as required. Remove every stale epic-symbol entry
and any unused code it reports, following the Symvision memory hierarchy. The audit
already established that the five closed-`sase-js` entries in `Justfile` duplicate task
`sase-kc`; if removing them makes `just symvision` pass and no underlying code cleanup
is required, note the verification on `sase-kc` and close that task as done so the
duplicate does not remain actionable. Do not alter `sase-jj`; active phase `sase-jx.5.3`
owns that dependency ratchet.

Run `just symvision`, `just check`, and `just check-full` on the cleaned combined tree,
using the repository policy for any unrelated failure. Finally, open the plans sidecar
through `/sase_repo`, set `status: done` in the frontmatter of
`202608/task_gate_convergence.md`, and verify the resulting plan link and repository
status. Preserve all unrelated worktree changes.
