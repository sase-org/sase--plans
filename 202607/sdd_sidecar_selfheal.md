---
tier: tale
title: Self-heal wedged SDD sidecar clones
goal: Machine-managed SDD sidecar clones recover from dirty-index and stale-rebase
  wedges without losing local state or flooding logs and axe error digests.
create_time: 2026-07-20 16:37:36
status: done
prompt: 202607/prompts/sdd_sidecar_selfheal.md
---

# Plan: Self-heal wedged SDD sidecar clones

## Context

Sidecar refresh currently delegates to the shared transactional SDD integration path. That path correctly refuses to
rebase a repository with tracked or staged changes and fails closed when it finds an in-progress Git operation. For
user-owned repositories and normal SDD write transactions, those safeguards must remain unchanged. For workspace-local
plans/beads sidecars, however, the checkout is machine-managed: a dirty index or rebase marker can wedge every periodic
refresh indefinitely, producing the same warning from multiple lumberjacks each tick.

This change will add an explicit recovery policy at the machine-managed sidecar boundary. It will preserve the
checkout's local state before any destructive Git command, restore the tracked branch to its configured upstream, retry
integration, and retain a discoverable recovery ref or stash for manual inspection. A durable per-clone cooldown will
coordinate independent processes and bound both repair attempts and identical reporting.

## Repository recovery contract

- Extend the SDD repository transaction layer with an opt-in machine-managed recovery operation or policy; do not make
  ordinary `integrate_sdd_repository` callers destructive. The sidecar clone path in `_store_link.py` will be the caller
  that opts in.
- Reuse the existing store write lock while inspecting and mutating recovery state. Resolve the attached branch and its
  configured upstream rather than assuming `main`, and return a typed outcome that distinguishes successful recovery,
  cooldown suppression, recoverable failure, and an unsafe repository.
- If a stale rebase is present, abort it and verify that `rebase-merge`/`rebase-apply` and unmerged entries are gone
  before continuing. Refuse automatic cleanup for unrelated or unprovable Git operations.
- When tracked, staged, or otherwise relevant dirty state remains, snapshot it to a timestamped `sase/recovery/...` ref
  or named stash, including untracked content when the chosen mechanism supports it. Verify that the snapshot exists and
  contains the local changes before resetting the managed branch. If snapshot creation or verification fails, leave the
  checkout untouched and surface the failure.
- After the snapshot is durable, hard-reset the managed branch to its fetched upstream, verify a clean attached worktree
  with no operation markers, and retry integration without discarding the recovery snapshot. A successful retry updates
  the normal integration freshness marker.

## Cooldown and reporting

- Persist recovery-attempt and failure-report timing under the clone's Git state directory so separate TUI and
  lumberjack processes share the same decision. Gate recovery before destructive work and record the attempt while
  holding the repository lock, allowing no more than one repair attempt per clone during the recovery window.
- Rate-limit identical `Failed to pull workspace SDD clone` warnings per canonical clone path. Successful recovery
  should emit one concise message naming the retained recovery ref; cooldown-suppressed ticks should stay quiet.
- On a terminal or repeated recovery failure, best-effort append one structured axe recent-error record for the clone
  and failure signature so the existing error-digest chop can notify the operator. Apply an independent durable
  reporting cooldown so repeated ticks neither append duplicate digest entries nor spam logs. Reporting failures must
  not hide the original repository outcome.
- Keep cooldown values internal and conservative unless an existing SDD refresh setting is a natural source of the
  interval; avoid adding public configuration solely for this repair.

## Tests and validation

- Add a fixture-repository regression with a diverged remote plus dirty tracked/staged content. Assert the first pull
  snapshots the local state, resets and integrates to the upstream head, leaves a clean healthy checkout, and keeps the
  dirty content reachable through the recovery ref/stash.
- Cover a clone left in a real rebase-in-progress state and assert recovery clears both rebase marker variants and
  unmerged entries before the successful retry.
- Cover safety failures: a failed snapshot or unverifiable recovery must not reset the branch, and the generic
  integration API must continue returning its non-destructive local-changes/unsafe outcome when recovery was not
  requested.
- Use a controlled clock or marker timestamps to call the failure path repeatedly and assert that recovery executes once
  per window, warning output is rate-limited, and only one axe recent-error record is produced. Advance the window and
  assert a later attempt/report is admitted.
- Run the focused SDD sidecar and repository transaction tests during development. Because source files will change, run
  `just install` and the repository-mandated `just check` before closing `sase-8g.5`; do not close its parent epic.

## Risks and boundaries

The destructive reset is safe only because this path owns machine-generated sidecar clones, so recovery must never
become the default for arbitrary SDD repositories. The snapshot must be verified before reset, the upstream must be
explicit and fetched, and every post-recovery state check must fail closed. Cross-process cooldown state belongs inside
Git metadata so it neither dirties the sidecar nor depends on one process remaining alive.
