---
tier: tale
size: medium
title: Stop reporting SDD git timeouts as unsafe repositories
goal:
  A git timeout during SDD clone integration is retried quietly instead of being
  reported as repository damage.
decisions:
  escalate_repeated_timeouts:
    ask:
      Should a clone whose integration keeps timing out still reach the axe error digest
      eventually?
    choices:
      never:
        Timeouts only log; the digest stays quiet even if one clone times out every tick
      streak:
        Report once after 3 wall-clock timeouts in 30+ minutes, worded "timed out
        repeatedly"
    default: never
    why: git-op telemetry already logs every timeout; the digest should stay actionable
    answer: never
proposed_by: bbugyi200.athena.1dw
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 09:01:49
status: wip
---

# Stop reporting SDD git timeouts as unsafe repositories

## Problem

The `sdd-sidecar` / `workspace_sdd_clone_recovery` axe error keeps showing up in error
digests with messages like:

```
SDD repository ~/.sase/projects/gh_bbugyi200__body/repos/plans is not safe to write:
not a valid Git worktree; detached HEAD; could not list unmerged index entries:
deadline expired before git operation. Finish or abort the existing Git operation ...;
machine-managed recovery failed: the checkout is not a valid Git worktree
```

The repository is fine. When it was checked after the 2026-11-18 report, it was on
`main`, clean, and up to date with `origin/main`. Apollo's digests have about 20 of
these reports since 2026-10-05, and nearly all of them are timeouts mislabeled as
repository damage:

- **`deadline expired before git operation`** (most common). These hit the hidden
  machine clones under `~/.sase/projects/<key>/repos/{plans,research,beads}`.
- **`git operation 'sdd.health.conflicts' timed out after 30.0s`**. These hit the
  primary `repos/plans` checkout.
- **`could not inspect SDD repository ...: git operation 'sdd.health.git_dir' timed out after 0.0s`**.
- **Rollback failures after a deadline expiry**, such as
  `completed SDD integration does not contain upstream: deadline expired before git operation; rollback verification failed: not a valid Git worktree, detached HEAD, ...`.

Earlier epic `sase-9r` fixed the lock-contention version of this error (now
`LOCK_UNAVAILABLE`). The timeout version was never addressed.

### Mechanism

1. Machine artifact-link chops (`agent_publication`, `artifact_link_backfill`, and
   publication retry) give `ensure_sidecar_sdd_clone` → `pull_sdd_clone` a work-budget
   `deadline` (`src/sase/sdd/_artifact_link_machine_store.py`,
   `_ensure_hidden_document_root`).
2. `pull_sdd_clone` checks the deadline once on entry. It then spends time on the
   cooldown, unpushed-commit, and freshness probes before it integrates.
3. `_git_runner_for_deadline` in `src/sase/sdd/_store_integration.py` behaves in two
   ways when the budget runs out:
   - Once the budget is gone, it returns a synthetic
     `CompletedProcess(returncode=124, stderr="deadline expired before git operation")`.
   - When the budget runs out during a local git command, it lets `SddGitCommandTimeout`
     propagate.

   `default_git_runner` in `src/sase/sdd/_repository_health.py` turns a per-op
   wall-clock `SddGitCommandTimeout` into the same kind of plain 124 result.

4. `inspect_sdd_repository` reads those results as real observations:
   - a failed `rev-parse --is-inside-work-tree` becomes `valid_worktree=False`;
   - a failed `symbolic-ref` becomes `branch=None` ("detached HEAD");
   - a failed `git diff` becomes `unmerged_error`.

   `sdd_state_blockers` then reports blockers, and the integration preflight returns
   `UNRECOVERABLE` "not safe to write" (`src/sase/sdd/_repository_integration.py`).

5. `integrate_machine_managed_sdd_repository` escalates `UNRECOVERABLE` into
   `recover_machine_managed_sdd_repository`. Recovery inspects again with the same
   exhausted runner, `managed_recovery_branch` says "the checkout is not a valid Git
   worktree", and the result is `UNRECOVERABLE`.
6. `pull_sdd_clone` calls `_append_recovery_error` for `UNRECOVERABLE` /
   `RECOVERY_FAILED`, which puts the false report in the axe digest.

Related timeout misclassifications on the same path:

- A timed-out `rev-parse --verify <upstream>` currently returns **`SUCCESS`** (it is
  treated as "no upstream").
- A timed-out `tracked_changes_error` probe returns `LOCAL_CHANGES`.
- A timed-out pre-rebase `merge-base --is-ancestor` returns
  `ABORTED_UNSUPPORTED_CONFLICTS`, which also records a failed-integration cooldown.
- A timed-out status probe in `_machine_upstream_alignment_error` returns
  `LOCAL_CHANGES`, and that **starts destructive machine-managed recovery** (snapshot
  plus `reset --hard`) because of a timeout.
- If the deadline expires mid-rebase, `_abort_and_verify` runs `rebase --abort`,
  `reset`, and the verification inspections on the same exhausted runner. The rollback
  cannot run, so a real mid-rebase wedge can be left behind and is reported as
  `UNRECOVERABLE`.

## Goal

A git timeout, whether the job budget ran out or one command exceeded its wall-clock
timeout, is reported as **"timed out; checkout state unknown; nothing was changed"**. It
is never reported as repository damage. It never starts destructive recovery, never
records a failed-integration cooldown, and never reaches the axe error digest. Once SASE
has started changing a checkout (a rebase or a recovery reset), the rollback and its
verification still get a small, bounded grace budget. Real unsafe states (genuine
detached HEAD, in-progress merge or cherry-pick, unmerged entries, a stale rebase) keep
today's fail-closed behavior, recovery, and reporting.

Non-goals:

- Do not change the bead sync worker's private deadline runner
  (`src/sase/bead/_sync_worker_run.py::_git_runner_for_deadline`). It relies on
  `SddGitCommandTimeout` being raised so it can run its own
  `restore_pre_integration_head` rollback. The changes below must still handle its
  _raised_ timeouts correctly, but its runner stays as it is.
- Do not change chop work budgets or chop timeouts.
- Do not move SDD repository integration into `sase-core`. This is existing Python git
  plumbing, and the fix is a classification correction inside it.

## Design

### 1. A typed timeout result and one shared deadline runner

In `src/sase/sdd/_repository_types.py` add:

- `class SddGitTimeoutResult(subprocess.CompletedProcess[str])`
  - `returncode` stays `124`, so every existing `returncode != 0` branch still sees a
    failure.
  - It carries `timeout_reason: Literal["deadline", "wall_clock"]`.
- `def is_git_timeout(result) -> bool`, an `isinstance` check.

Add `SddRepositoryInspectionTimeout(SddGitCommandTimeout)` next to
`SddRepositoryHealthError` in `src/sase/sdd/_repository_health.py`.
`SddGitCommandTimeout` lives in `src/sase/sdd/_git_config.py` and is re-exported from
`sase.sdd._git`. Making the new exception a subclass means every existing
`except SddGitCommandTimeout` handler also catches it.

Create `src/sase/sdd/_deadline_git_runner.py` with
`class DeadlineGitRunner(deadline: float)`. It is a callable with the `GitRunner`
signature `(repo_root, args, *, op, network=False)` and replaces
`_store_integration._git_runner_for_deadline`, keeping the same timeout math:

- No budget left → return
  `SddGitTimeoutResult(reason="deadline", stderr="deadline expired before git operation")`.
- The local or network write raises `SddGitCommandTimeout` → return
  `SddGitTimeoutResult`, with `reason="deadline"` when the effective timeout was clipped
  by the remaining budget and `"wall_clock"` otherwise. Use the exception text as
  stderr.
- `with_grace(seconds: float) -> DeadlineGitRunner` returns a runner whose deadline is
  `max(self.deadline, time.monotonic() + seconds)`.
- Add a module constant `SDD_ROLLBACK_GRACE_SECONDS = 20.0`. One rollback is about 15
  local git calls. Each budgeted chop is SIGKILLed at 5m with at least 60s of headroom
  over its work budget, and `pull_sdd_clone` refuses new work once the deadline has
  passed, so there is at most one grace window per chop run.
- Add a helper `rollback_runner(runner: GitRunner) -> GitRunner`. It returns
  `runner.with_grace(SDD_ROLLBACK_GRACE_SECONDS)` for a `DeadlineGitRunner` and returns
  `runner` unchanged otherwise.

`_store_integration.py` uses `DeadlineGitRunner(deadline)` when a deadline is set.
`default_git_runner` returns `SddGitTimeoutResult(reason="wall_clock")` for
`SddGitCommandTimeout`. `OSError` keeps its current plain-124 result, because a missing
git binary is not a timeout.

### 2. Inspection fails as "unknown", never as "unsafe"

`inspect_sdd_repository` raises `SddRepositoryInspectionTimeout` (message names the op
and the stderr) as soon as any probe result `is_git_timeout`. That covers worktree,
git_dir, branch, head, conflicts, and status. `unmerged_paths` gets the same check.
`SddRepositoryState` and `blockers` do not change.

Raising is deliberate. Any call site this plan misses still falls into its existing
`except Exception` → fail-closed branch, so unknown state is never treated as clean.
`require_sdd_repository_health` lets the exception propagate.

### 3. New status `TIMED_OUT`

Add `SddIntegrationStatus.TIMED_OUT = "git_timed_out_state_unknown"` with a docstring
comment: "a git probe or operation timed out before SASE changed the checkout, or SASE's
rollback was verified; the checkout is presumed as-found and the attempt should simply
be retried later."

- It is not in `succeeded`.
- `structurally_healthy` stays `status is not UNRECOVERABLE`.
- Callers outside this module only branch on `succeeded` or name specific other
  statuses. That includes `bead/_sync_refresh.py`, `bead/_sync_worker_run.py`,
  `goals/*`, `_store_adoption.py`, `_store_materialization.py`, and
  `axe/runner_workspace_sidecar.py`, so they need no changes. Confirm this with a grep
  for `SddIntegrationStatus.` while implementing.

### 4. Integration transaction (`_repository_integration.py`)

Map timeouts **before any mutation** to
`SddIntegrationOutcome(TIMED_OUT, upstream_present=<known so far>, error="git timed out ...: <detail>")`.
These are:

- preflight inspection and locked starting inspection: catch `SddGitCommandTimeout`
  before the existing `except Exception`;
- `rev-parse --verify <upstream>` when the result `is_git_timeout`. This replaces the
  current false `SUCCESS`;
- `tracked_changes_error`: return a typed signal instead of folding a timeout into the
  `LOCAL_CHANGES` message. A small change is enough, such as returning a
  `(message, timed_out)` pair or raising `SddGitCommandTimeout` for a timeout result;
- pre-rebase `merge-base --is-ancestor`.

The fetch path keeps its current shape: a timed-out fetch already becomes
`REMOTE_UNAVAILABLE` after a healthy locked inspection.

**After a mutation** (the `rebase`, the continue loop, and `_successful_integration`):

- `_abort_and_verify` swaps in `rollback_runner(runner)` at its first line, so the
  abort, reset, and both inspections run on the grace budget.
- Add a `timed_out: bool = False` parameter. Set it at every call whose trigger was a
  timeout: the `except SddGitCommandTimeout` handler, an `is_git_timeout` result from
  `rebase`, `rebase --continue`, the conflicts probe, `verify_ancestor`, or an
  inspection timeout in `_successful_integration`.
- When the rollback verifies and `timed_out` is set, return `TIMED_OUT` with
  `restored=True` and `upstream_present=True` instead of
  `ABORTED_UNSUPPORTED_CONFLICTS`, so no failed-integration cooldown is recorded.
- When the rollback cannot be verified, keep `UNRECOVERABLE`. That includes an
  inspection that times out even under grace. It stays fail-closed because a real rebase
  may be in progress.
- The nested "rollback also timed out" handler stays as it is.

### 5. Machine-managed integration and recovery

In `_repository_transaction.py`:

- Add `TIMED_OUT` to the set of statuses that return without recovery, next to
  `LOCK_UNAVAILABLE`.
- In `_machine_upstream_alignment_error`, return `TIMED_OUT` for timed-out results from
  the upstream, status, and merge-base probes instead of `LOCAL_CHANGES`. That way a
  timeout can never start a snapshot plus `reset --hard`. The helper currently returns
  `(error, upstream_present)`. Extend it, or have it return an outcome, so the caller
  can tell a timeout from local changes.

In `recover_machine_managed_sdd_repository` (`_repository_recovery.py`):

- If the starting inspection raises `SddGitCommandTimeout`, return `TIMED_OUT` with
  error `_recovery_error(original, "recovery deferred: git timed out: ...")`. This is
  exactly the path in the 2026-11-18 report.
- Every **pre-mutation** step after that also returns `TIMED_OUT` instead of going
  through `_managed_recovery_failure`: `configured_upstream`, `branch_head`,
  `update_and_verify_ref` (the additive `refs/sase/recovery/*` ref), recovery `fetch`,
  and upstream SHA resolution. Leave the recovery-attempt marker in place; the existing
  cooldown handles the retry.
- From `rebase --abort` / `_snapshot_managed_changes` / `reset --hard` onward, swap to
  `rollback_runner(runner)` for the rest of the function, including the verify helpers
  and `_managed_recovery_failure`. Failures there keep today's
  `RECOVERY_FAILED`/`UNRECOVERABLE` semantics.
- Give `verify_rebase_cleared`, `verify_managed_reset`, and `managed_recovery_failure`
  in `_repository_recovery_git.py`, and the inspections in
  `_repository_recovery_snapshot.py`, no new timeout leniency. They run on the grace
  runner and stay fail-closed.

### 6. `pull_sdd_clone` (`_store_integration.py`)

- Check `_deadline_expired(deadline)` again immediately before
  `integrate_machine_managed_sdd_repository` and return `False` quietly. This closes the
  entry-check-to-preflight gap.
- For `TIMED_OUT`, return `False` without calling `record_failed_integration_marker`.
  `_records_failed_integration_cooldown` already excludes the status; keep it that way
  and test it. Do not call `_append_recovery_error`. Log at INFO through the existing
  `admit_recovery_notice` gate, saying "Deferred workspace SDD clone integration for %s:
  %s". With `strict=True`, keep raising `SddIntegrationError` through the existing
  `strict and status not in {REMOTE_UNAVAILABLE}` branch. The message now says "timed
  out" instead of "not safe to write".

> [!decision] escalate_repeated_timeouts = streak Also persist a per-clone
> wall-clock-timeout streak (count plus first timestamp) next to the existing
> failed-integration marker in `_repository_recovery_markers.py`. Increment it only for
> `TIMED_OUT` outcomes whose timeout reason is `wall_clock`. Budget exhaustion
> (`deadline`) neither increments nor resets it. Carry the reason on the outcome
> (`SddIntegrationOutcome.timeout_reason: str | None`). Any non-`TIMED_OUT` outcome
> clears the streak. When the streak reaches 3 attempts spanning at least 30 minutes,
> call `_append_recovery_error` once per streak with "SDD repository <path>: git timed
> out repeatedly (<n> attempts since <ts>): <last detail>". Under `never`, none of this
> is built.

## Tests

Add `tests/sdd_store/test_repository_timeouts.py`, reusing the git-repo fixtures in
`tests/sdd_store/_repository_transaction_helpers.py`, and extend
`tests/sdd_store/test_sidecar_clone_pull_refresh.py`. Cover:

1. A runner returning `SddGitTimeoutResult(reason="deadline")` for every op:
   `integrate_sdd_repository_transaction` returns `TIMED_OUT`, its error does not
   contain "not safe to write" / "not a valid Git worktree", and the repo HEAD, refs,
   and worktree are unchanged.
2. The same runner through `integrate_machine_managed_sdd_repository`: `TIMED_OUT`, no
   recovery-attempt marker under the git dir, no `refs/sase/recovery/*` ref.
3. `pull_sdd_clone` where the deadline expires after the entry check (monkeypatch the
   clock or `time.monotonic`): returns `False`, `sase.axe.state.append_error` is not
   called, and no failed-integration marker is written.
4. `default_git_runner` with `run_sdd_git_write` monkeypatched to raise
   `SddGitCommandTimeout` for `op` ending in `.health.conflicts`: `TIMED_OUT`, not
   `UNRECOVERABLE`.
5. Timed-out `rev-parse --verify <upstream>`: `TIMED_OUT`, not `SUCCESS`.
6. Timed-out status probe in machine upstream alignment
   (`require_upstream_alignment=True`): `TIMED_OUT`, recovery not invoked, worktree
   untouched.
7. Mid-rebase expiry on a real diverged repo. Use a `DeadlineGitRunner` subclass or
   wrapper whose budget is forced to expire right after the `.rebase` op starts a
   conflicting rebase. The rollback runs on the grace runner, the rebase is aborted, the
   outcome is `TIMED_OUT` with `restored=True`, and HEAD, branch, and markers match the
   starting state.
8. Recovery whose starting inspection times out returns `TIMED_OUT`. Recovery whose
   pre-mutation fetch times out returns `TIMED_OUT` and never runs `reset --hard`.
9. Regression: a genuinely detached HEAD and a genuine `MERGE_HEAD` still produce
   `UNRECOVERABLE` and still reach `_append_recovery_error`. Existing tests in
   `test_repository_transaction*.py`, `test_repository_recovery_*.py`,
   `tests/test_bead/test_sync_wedged_healing.py`, and
   `tests/test_bead/test_sync_conflict_claims.py` must pass unchanged, except where they
   assert the old false `UNRECOVERABLE`/`SUCCESS` for a timeout.
10. `DeadlineGitRunner` unit tests: expired budget gives `reason="deadline"`; a
    wall-clock timeout under a large budget gives `reason="wall_clock"`; `with_grace`
    extends an expired deadline.

Under `escalate_repeated_timeouts = streak`, also test that the streak reports once
after 3 wall-clock timeouts spanning 30+ minutes, that deadline timeouts don't count,
and that a success clears the streak.

## Verification

Follow the lint-and-test memory note (`just check` through `sase tool run`). After
landing, the next apollo digests should stop showing `workspace_sdd_clone_recovery`
entries whose error contains `deadline expired before git operation` or
`timed out after`. Genuine unsafe-state reports should still appear.
