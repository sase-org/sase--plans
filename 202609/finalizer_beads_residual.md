---
tier: tale
title: Reconcile late beads-sidecar changes during commit finalization
goal:
  A landed agent commit completes when late bead-state changes can be safely committed
  and published, while unresolved changes receive a precise failure.
size: medium
proposed_by: bbugyi200.athena.0t0
create_time: 2026-09-26 18:01:00
status: wip
---

# Reconcile late beads-sidecar changes during commit finalization

## Problem and evidence

The `sase-1ah.8.2` run completed its adapter work, closed its assigned phase, and
recorded primary commit `f7886b1a64dab09bb7bc85846f6b838dd6b955bc`. The built-in commit
finalizer then committed bead state and pages, including sidecar commit
`4d6949ad94568ed8190f986ca080fb788afeadfe`, which was integrated and pushed. Its final
dirty-state check nevertheless reported `dirty_after_commit_decisions`: four bead event
streams (`sase-19x.11.5`, `sase-19x.11`, `sase-1ah.8`, `sase-1aq.10`) and `issues.jsonl`
had become dirty in the run's beads checkout. The run was marked failed even though its
primary commit landed. The 2026-09-26 run ID is `20260926133913`; its `error_report.md`,
`commit_results.json`, and bead-push log for 17:43:38 EDT contain the evidence.

`prepare_commit_dirty_state` auto-commits machine-owned bead state, while
`execute_commit_finalizer` treats all residual repositories in the returned snapshot as
a failure. That leaves a race window around reconciliation, publication, and other
sidecar writers. The exact writer of the late files is not proven by the surviving
artifacts, so the repair must use current Git and run-owned evidence instead of assuming
every residual path is foreign.

## Implementation

1. Build a deterministic real-Git regression in the commit-finalizer tests: land a
   declared primary commit, create and publish a beads-sidecar commit, then introduce
   append-only event and projection changes before the final residual check. Cover both
   a late machine-owned change and an unrelated writer's change. Assert the primary
   stitch runs once and its commit marker survives.
2. Add one bounded reconciliation pass for a newly dirty SDD/beads sidecar after commit
   decisions. Reuse the existing store lock, event-stream integrity guard, commit
   marker, and publication verification. Refresh the dirty snapshot after that pass;
   declare success only when every required checkout is clean and the newly created bead
   commit is published. Do not retry the primary stitch or erase sidecar files.
3. Stop swallowing the reason when `_auto_commit_bead_state` cannot reconcile a dirty
   sidecar. Return a typed diagnostic with the sidecar path, remaining files, and
   underlying commit, lock, integrity, or publication failure. Keep genuine uncommitted
   or unpublishable agent-owned work as a finalizer failure with an actionable recovery
   path.
4. Test the bounds and refusal cases: changes arriving during the extra pass, stream
   rewrite/integrity rejection, lock or push failure, and actual agent-owned residual
   edits. Check that no duplicate primary commit occurs, no event is discarded, and the
   final outcome distinguishes repaired dirt from unresolved dirt.

## Verification

Run the focused finalizer and bead-publication tests, then `just fix` and `just check`
in the sase checkout. Inspect commit markers and sidecar remote state in the regression
to verify that success means both the code and bead state are durable.
