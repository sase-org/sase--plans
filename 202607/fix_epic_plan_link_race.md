---
tier: tale
title: Fix the approved-epic plan-link race and launch the epic
goal: Approved epic plan links are committed atomically, survive concurrent SDD recovery,
  and the beads-sidecar epic launches successfully.
create_time: 2026-07-27 15:26:52
status: wip
---

- **PROMPT:** [202607/prompts/fix_epic_plan_link_race.md](prompts/fix_epic_plan_link_race.md)

# Fix the approved-epic plan-link race and launch the epic

## Goal

Make an approved epic's `bead_id` plan update atomic with its targeted Git commit so a concurrent SDD integration or
machine-managed recovery cannot snapshot and reset the uncommitted link. Preserve the existing deterministic rollback
behavior, verify the fix under the observed race, install the fixed SASE build, and rerun:

```bash
sase bead work /home/bryan/projects/github/sase-org/sase/sase/repos/plans/202607/beads_sidecar_repo.md --yes
```

The retry must finish by launching the epic rather than creating and rolling back another duplicate graph.

## Diagnosis and invariant

The failed `sase-a7` launch wrote the plan link before creating ten phase beads and sixteen dependency edges. Its
canonical event stream timestamps that work from 19:15:39Z through 19:18:45Z. During that interval, at 15:17:51 local
time, machine-managed SDD recovery created recovery snapshot commit `c02bdd7c`, whose tree contains `bead_id: sase-a7`,
and then reset the plans clone to `origin/main`. Recovery held the cooperative store write lock, but the earlier plain
`Path.write_text()` did not. The later targeted plan commit therefore saw no changed file and returned false, producing
the reported error and the `sase-a7` rollback commit.

The required invariant is: every SASE-owned approved-plan worktree mutation that is intended to be committed must occur
inside the same `store_git_write_lock(..., mutates_worktree=True)` span as staging and commit. No recoverer, integrator,
or other cooperating writer may observe the intermediate plan contents.

## Implementation

1. Replace the current two-step plan-update contract in `src/sase/bead/epic_from_plan.py`, where the orchestration
   writes content and later invokes a commit callback, with a callback that receives the exact replacement content and
   owns both writing and committing it. Delay creation of the linked plan content until the complete epic DAG exists so
   the plan is not dirty during phase/dependency materialization. Use the same callback for rollback restoration,
   retaining the existing rules around when a false commit result is benign versus an incomplete rollback.

2. In `src/sase/bead/cli_work_from_plan.py` and `src/sase/bead/cli_work_from_plan_store.py`, implement the production
   callback as one transaction: acquire the plans repository's store write lock with `mutates_worktree=True`, write the
   supplied contents only after acquisition, and pass `already_locked=True` through the targeted SDD commit helper so it
   hands off rather than reacquires the same lock. Treat lock unavailability as an error without changing the plan. Keep
   in-tree and sidecar path routing, commit messages, commit-marker behavior, and push-after-launch sequencing
   unchanged.

3. Update unit tests and callback fakes for the new content-owning contract. Preserve assertions that the complete DAG
   exists before the linked plan is committed, creation failures restore the original bytes, linked graphs are retained
   after the runner spawn boundary, and a failed forward commit removes the just-created graph without launching.

4. Add a deterministic concurrency regression around the production plan update transaction. Pause after the transaction
   has acquired the SDD store lock and written the `bead_id`, start a competing recovery/integration-style writer using
   that same lock, and prove the competitor cannot enter until the plan commit completes. Then assert the archived plan
   retains the created epic ID and the launcher does not take its rollback path. Also cover lock acquisition failure and
   verify it leaves the pre-update plan bytes intact.

## Validation and launch

1. Run the focused approved-plan and store-transaction tests, including the new concurrency regression.
2. Run `just install`, then `just check` as required for SASE repository changes. Fix all failures and rerun until
   clean.
3. Reinspect the plans sidecar and active bead store before retrying: confirm the archived plan has no stale `bead_id`,
   neither rolled-back `sase-a6` nor `sase-a7` resolves as an active issue, and the repository is healthy.
4. From the installed workspace-0 SASE checkout, rerun the exact resume command shown above. Do not manually create or
   claim phase beads.
5. Verify the command succeeds, the archived plan carries the newly created epic ID, the epic has exactly ten phase
   children with the authored sixteen dependency edges, the graph is published, and phase plus land agents were
   launched. If the retry fails, preserve its recoverable state and diagnose the new terminal error rather than blindly
   running it again.
