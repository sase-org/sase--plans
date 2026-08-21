---
tier: tale
title: Fix resumed commit repository attribution
goal:
  Resumed commits are attributed to their checkpointed repository so the finalizer
  recognizes valid sidecar evidence without weakening fail-closed checks.
size: small
proposed_by: bbugyi200.athena.09o
create_time: 2026-08-21 11:48:47
status: wip
---

# Fix resumed commit repository attribution

## Problem

The `sase-ru.2` worker completed its implementation and the built-in commit finalizer
successfully committed both the primary repository and the `plans` sidecar. The sidecar
commit survived conflict resolution as `84aeb6a1`, but the agent was nevertheless marked
failed with:

> `sase stitch create completed for plans, but no commit_results.json entry was recorded`

The entry was actually present. Its message and SHA identified the `plans` commit, but
its `cwd` identified the primary workspace. During conflict repair, the agent continued
the rebase inside the sidecar and then invoked `sase stitch create --resume` from its
normal launch directory. Resume correctly loaded the checkpointed target repository, but
`write_result_marker()` ignored that checkpoint and derived `cwd` from ambient
`os.getcwd()`. The generic commit finalizer deliberately matches new ledger entries to
dirty repositories by exact normalized `cwd`, so it rejected the misattributed row and
failed closed.

Existing resume tests prove that the resumed SHA and tree are recorded, but they mock
marker writing and do not cover a resume command launched outside the checkpointed
repository. The focused baseline suites currently pass (45 tests), which confirms this
is a missing behavioral case rather than an already-detected regression.

## Implementation

1. Make commit-result repository identity explicit in
   `src/sase/workflows/commit/commit_tracking.py`.
   - Add an optional keyword-only commit-repository path to `write_result_marker()` and
     use it, when supplied, for the marker `cwd`, sidecar/external repository
     classification, commit-time lookup, and primary metadata ownership checks.
   - Preserve `os.getcwd()` as the compatibility default for direct callers that do not
     have a checkpointed repository.
   - Keep the existing ledger upsert and SHA/tree semantics unchanged.

2. Thread the checkpointed repository through `src/sase/workflows/commit/workflow.py`.
   - Pass `cp.cwd` to both the initial and final-with-entry-ID marker writes in
     `_run_tracking_steps()`.
   - This makes normal dispatch and resumed dispatch use the same authoritative
     repository identity regardless of the shell directory from which resume was
     invoked.
   - Do not relax `marker_matches_repo()` or the finalizer's missing-result and
     discarded-work guards; those checks should remain strict once producers emit
     correct evidence.

3. Add regression coverage around the production boundary.
   - Extend `tests/test_commit_result_marker.py` to prove an explicit repository path
     overrides a different ambient working directory in both `commit_result.json` and
     the accumulated `commit_results.json` ledger.
   - Extend `tests/test_commit_workflow_resume.py` with a checkpoint whose target
     repository differs from the process working directory, then prove both marker
     writes are attributed to the checkpointed repository and retain the resumed commit
     SHA/tree.
   - Update any existing marker-call assertions affected by the new keyword so normal
     dispatch behavior remains covered.
   - If needed for cross-layer confidence, add a focused assertion in
     `tests/test_finalizers_protocol_harness.py` that the correctly attributed resumed
     row is accepted for a sidecar while a row for another repository is still rejected.

## Verification

1. Run the focused regression suites: `tests/test_commit_result_marker.py`,
   `tests/test_commit_workflow_resume.py`, and
   `tests/test_finalizers_protocol_harness.py`.
2. Run `just check` as the required repository-wide lint and diff-scoped test gate. If
   it reports unusual test selection or escalates, inspect the selection explanation and
   use the documented full-check path only when warranted.
3. Confirm the working tree contains only the intended source and test changes, with no
   mutation of the historical `sase-ru.2` artifacts or weakening of finalizer
   fail-closed behavior.

## Acceptance criteria

- A conflict-resume launched outside the checkpointed repository records the
  checkpoint's repository as `cwd` in every commit-result marker.
- The built-in commit finalizer recognizes that new ledger row as evidence for the
  intended sidecar and does not raise `missing_commit_result` after a successful resume.
- Existing direct callers retain ambient-working-directory behavior when they do not
  supply an explicit repository path.
- Repository attribution remains exact, so an unrelated repository's marker cannot
  satisfy the finalizer obligation.
- Focused tests and `just check` pass.
