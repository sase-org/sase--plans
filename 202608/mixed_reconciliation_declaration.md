---
tier: tale
title: Preserve finalizer declarations across mixed machine reconciliation
goal:
  Valid commit declarations survive attributable machine-owned partial commits without
  weakening stale-state rejection.
size: medium
proposed_by: bbugyi200.athena.0b3
create_time: 2026-08-22 16:25:43
status: wip
---

# Preserve finalizer declarations across mixed machine reconciliation

## Goal

Fix `builtin@commit` so a valid declaration is not made stale when finalizer preparation
auto-commits machine-owned files from a repository that also contains declared
user-authored work. Preserve the protocol's fail-closed behavior for real post-submit
edits, unexpected repositories, unproved cleanups, and unpublished machine-owned
commits.

## Root cause and current state

The `research.0w.cld` run completed its research response and submitted a commit
decision for the dirty `sdd:research` repository. At execution time,
`prepare_commit_dirty_state()` auto-committed the pending artifact-link index as
`b35b392` while leaving `202608/stand_alone_proc_shells.md` dirty. The executor then
compared the accepted whole-repository digest with the post-reconciliation dirty
snapshot. Removing the link index from that snapshot necessarily changed the digest, so
`_reject_stale_repository_obligation()` reported
`repository research changed after submit`. The bounded declaration-recovery turn was
forbidden to mutate repositories and refused the remaining note, which converted a
successful research turn into a failed agent.

The recent `cf72b00d1` fix is already on the fetched default branch and predates the
failed run. It snapshots `commit_results.json` before reconciliation and proves the case
where an auto-commit makes an accepted repository completely clean. Its tests do not
cover the mixed case where reconciliation commits only machine-owned paths and the same
repository remains dirty for the declared stitch. No later default-branch commit fixes
that ordering. The research output has since been recovered and published in the
research sidecar, so this work should fix the finalizer protocol rather than manipulate
the held failed workspace or recreate the report.

## Implementation

1. In `src/sase/finalizers/commit.py` and, if useful for keeping the snapshots atomic,
   `src/sase/finalizers/reconciliation.py`, distinguish the accepted pre-reconciliation
   repository state from the post-reconciliation state used to execute decisions.
   Validate each accepted obligation digest and the set of repository obligation IDs
   against a snapshot taken before any bead, plan-status, prompt-Q&A, or artifact-link
   auto-commit. Continue to order and run declared stitches against the refreshed dirty
   state after those machine-owned commits.

2. Make reconciliation transitions fail closed rather than merely bypassing the stale
   check. For a repository that remains dirty, allow preparation to remove only paths
   accounted for by a new run-owned reconciliation commit while requiring all remaining
   declared paths to retain their pre-reconciliation fingerprints; reject newly dirty
   paths, changed residual fingerprints, missing or wrong-checkout ledger markers, and
   repositories absent from the accepted context. Reuse the pre/post commit-ledger
   snapshots and repository normalization helpers introduced by `cf72b00d1` instead of
   weakening `_reject_stale_repository_obligation()` globally. Keep the existing
   already-clean/discarded-work and publication guards intact.

3. Add focused regression coverage in `tests/test_finalizers_commit_reconciliation.py`
   for a single accepted sidecar that starts with both a user document and a
   machine-owned link index. Simulate preparation committing the link index with a new
   matching ledger marker while the document remains dirty, and assert that the declared
   stitch runs once and the aggregate finalizer result succeeds. Add negative cases
   showing that the same flow still rejects an edited residual document, an unexpected
   residual path, and a reconciliation transition with no new marker or a marker for
   another checkout.

4. Add or extend the real-git coverage in
   `tests/llm_provider/test_commit_finalizer_auto_artifact_links.py`: create a sidecar
   with an ordinary dirty report plus a pending canonical artifact-link index, publish
   and submit the declaration, then run the controller. Assert that preparation records
   the scoped artifact-link auto-commit, the declared stitch commits the report, the
   repository ends clean, and no declaration-recovery turn is opened. Retain the
   existing publication-failure expectations.

5. Update the built-in finalizer protocol section of `docs/commit_workflows.md` to
   explain that declaration staleness is checked against pre-reconciliation state, while
   the post-reconciliation transition must be attributable before remaining declared
   work is stitched. Document both the fully-clean case covered by `cf72b00d1` and the
   mixed-path case from `research.0w.cld`.

## Verification

- Run the focused commit reconciliation, protocol harness, live finalizer, and
  artifact-link auto-commit test modules, including the existing real post-submit edit
  recovery test.
- Run `just install` before repository checks, then run `just check` as required for
  changes in this repository.
- If scoped selection escalates or reports unusual coverage, use `/sase_monitor` for
  `just check-full` with the required `TESTING`/`TESTED` statuses.

## Acceptance criteria

- A mixed sidecar containing a declared report and pending machine-owned artifact-link
  indexes completes without a false stale-declaration recovery turn.
- Both reconciliation and the later declared stitch have new, checkout-matching commit
  evidence, and the repository is clean and published at completion.
- A real edit after submission, an unexpected dirty path or repository, an unproved
  cleanup, or unpublished machine-owned state still fails or enters the bounded recovery
  path exactly as before.
- The focused tests and `just check` pass.
