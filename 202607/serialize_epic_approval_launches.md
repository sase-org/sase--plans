---
tier: tale
title: Serialize approved epic launches per bead store
goal: Ensure simultaneous epic approvals against one project cannot allocate conflicting
  bead IDs by completing each approved epic's archive, bead creation, commit, and
  push before the next launch mutates that store.
create_time: 2026-07-20 11:03:31
status: done
prompt: 202607/prompts/serialize_epic_approval_launches.md
---

# Plan: Serialize approved epic launches per bead store

## Context

All epic-approval surfaces eventually invoke `sase bead work <plan> --yes` in the canonical primary workspace. The TUI
and detached host launcher currently start one worker per approved plan, and their plan-specific deduplication keys
intentionally differ. Two workers for the same project can therefore pass store preflight together and enter plan
archival and bead creation concurrently. Existing SDD Git locking protects individual add/commit operations, but it does
not cover bead-ID allocation or the complete multi-commit launch transaction; the final sidecar push is also detached.
As a result, both workers can allocate from the same bead state and produce the same epic ID before either launch has
fully committed and synchronized its result.

The correctness boundary should be the shared plan-file launch workflow, not one UI queue. That covers TUI approvals,
headless/detached approvals, CLI approvals, and direct plan-file retries, including concurrent processes that do not
share an in-memory task queue.

## Store-scoped launch transaction

- Add a dedicated blocking, cross-process lock for epic plan launches, keyed by the canonical bead/plan store repository
  root and stored outside tracked content (prefer the repository Git metadata, with a safe non-Git fallback if the
  supported store model requires one). Keep it separate from the existing short-lived store Git write lock so nested
  archive and bead commits cannot deadlock.
- Acquire the launch lock after resolving the effective store but before any non-dry-run archive, linked-plan resume,
  bead allocation, plan-link mutation, agent-launch state mutation, or commit. Hold it until the complete launch or
  rollback path has finished its final store synchronization. Waiting workers must block rather than time out and
  proceed unsafely: serialization is the invariant, not a best-effort contention optimization.
- Scope serialization to one canonical store. Approvals for different projects/stores should remain independent, while
  all entry points targeting the same sidecar or in-tree bead store must converge on the same lock identity.
- Keep dry-run validation and preview pure and outside the lock. Preserve idempotent linked-plan resume behavior,
  `--no-push`, rollback/resume commands, and existing user-visible task ownership; queued TUI tasks may remain visible
  as separate tasks while their subprocesses serialize at the shared mutation boundary.

## Commit and push completion boundary

- Retain the existing strategy of suppressing intermediate pushes while the plan archive, `bead_id` link,
  phase/dependency creation, and launch-state commits are assembled.
- For a successful approved-plan launch, replace the final detached sidecar-store push with a blocking managed
  synchronization attempt while the launch lock is still held. The next queued launch must not begin bead creation while
  the prior launch's commit/push operation is still running. Apply the same terminal boundary to linked-plan resume when
  it performs launch-state work.
- Preserve the established best-effort remote-failure contract unless a stronger existing API contract requires
  otherwise: a failed remote push is surfaced clearly and the local commit remains available for recovery, but the lock
  is released only after the blocking attempt returns. `--no-push` continues to skip network synchronization while still
  serializing local mutation and commit work.
- Do not change the global SDD or bead push configuration for unrelated commands. This synchronous completion
  requirement is specific to the approved epic plan launch transaction.

## Regression coverage

- Add focused lock tests showing that two independent callers for the same canonical store cannot overlap, the second
  begins only after the first releases, and distinct stores are not globally serialized. Cover exception-safe release so
  a failed/rolled-back launch cannot strand later approvals.
- Add a concurrent plan-file launch regression using two distinct valid epic plans against one store. Coordinate the
  workers so they attempt to launch together, then assert that both complete with different epic IDs, both complete
  phase graphs and plan links survive, and the second bead-creation section starts only after the first launch has
  committed and completed its blocking push attempt.
- Update the existing plan-file push-order tests to require one synchronous terminal push rather than an asynchronous
  handoff, while retaining coverage for `--no-push`, resume, rollback, and push-failure behavior. Keep the lower-level
  configurable async-push tests for ordinary bead/SDD commands unchanged.
- Exercise the common command boundary rather than only the TUI helper, demonstrating that foreground, tracked, and
  detached approval paths inherit the fix without separate front-end queues.

## Validation

- Run the focused epic plan-file, epic-launch, sync, and SDD contention test modules while iterating.
- Run `just install` before repository checks as required for an ephemeral workspace, then run `just check` and resolve
  all failures.

## Risks and safeguards

- A lock that reuses the existing store-write lock would self-deadlock when the launch performs nested commits; use a
  distinct launch lock and document lock ordering.
- A bounded or fail-open lock would recreate the race under a slow launch; use blocking acquisition and ensure every
  exit path releases through a context manager.
- A lock keyed by plan path or TUI task identity would not serialize different plans for one store; derive identity from
  the resolved canonical store instead.
- Holding the lock across a network push increases wait time for later approvals, but that wait is intentional and
  should remain visible through the existing task/subprocess reporting. Existing network timeouts bound a failed
  synchronization attempt.
