---
tier: tale
title: Recover release-plz merge concurrency
goal:
  New release-plz merge jobs evict stale group owners and settle successfully without
  weakening release safeguards.
size: small
proposed_by: bbugyi200.athena.0lv
create_time: 2026-09-16 09:05:14
status: wip
---

# Recover release-plz merge concurrency

## Diagnosis

`actstat --repo sase-org/sase-core -n 5 --format json -vv` shows that the Rust CI jobs
are passing, while recent commits are classified as failures because their `Release-plz`
workflows end in cancellation. The affected workflows all stall before the
`Merge release PR` job executes any steps.

GitHub's concurrency-group API identifies the persistent blocker precisely:
`release-plz-merge-refs/heads/master` was last acquired on 2026-09-15 by run
`34984385698`, job `104432908118`. GitHub reports that member as `in_progress` even
though the job view reports it as runner-queued with no steps. The current run
`35098010136`, job `104800371260`, is therefore `pending` in the same group. The
workflow's `cancel-in-progress: false` preserves the wedged group owner; GitHub's
single-pending default then cancels each older pending run whenever a new one arrives.
Job `timeout-minutes` does not recover a job that never starts executing on a runner.

The release PR itself is healthy: PR #275 is mergeable and all CI and PR-title checks
pass. This is a concurrency recovery defect, not a Rust test failure or a
release-content failure.

## Implementation

1. In the linked `sase-core` repository, update only the `release-plz-merge` job's
   concurrency policy in `.github/workflows/release-plz.yml` so a newer merge reconciler
   cancels an in-progress predecessor in the same branch-scoped group. Keep the stable
   group key so merge jobs remain serialized, and leave the separate `release-plz-pr`
   serialization policy unchanged.
2. Add or update the nearby workflow comment to document why preemption is safe and
   necessary: every merge job looks up the current guarded release-plz PR rather than
   acting on run-specific output, so only the newest reconciler is useful; preemption
   also lets a fresh run evict a hosted-runner queue entry that otherwise owns the group
   indefinitely.

## Validation

1. Run `actionlint .github/workflows/release-plz.yml` for focused workflow syntax and
   expression validation.
2. Run the repository-mandated `just check` from the `sase-core` root and fix any
   failures caused by the change.
3. After the change reaches `master`, inspect the branch's new `Release-plz` run and the
   `release-plz-merge-refs/heads/master` concurrency group. Confirm that the stale group
   owner is cancelled, the newest `Merge release PR` job leaves `pending`, and the
   guarded release PR is merged only after its checks pass (or the job exits
   successfully if no release PR remains).
4. Re-run `actstat --repo sase-org/sase-core -n 2` after the replacement run settles.
   Confirm the latest commit is no longer reported as failed because of a cancelled
   `Merge release PR` job; if it is still red, inspect that run's job details, correct
   the remaining cause, and repeat the focused, full-repository, and live checks until
   green.

## Non-goals and safety constraints

- Do not weaken the guarded release-PR lookup, check waiting, or squash-merge
  conditions.
- Do not cancel or parallelize wheel builds, PyPI publication, or the `release-plz-pr`
  writer job.
- Do not manually edit release versions or dependency version pins.
