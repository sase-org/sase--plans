---
tier: epic
title: Serialize SDD store git writers and retry index.lock contention
goal: 'Epic launches and bead-store commits no longer fail on transient git index.lock
  contention in shared SDD sidecar store repos: SASE-managed git writers are serialized
  against each other, transient lock collisions are retried instead of aborting the
  launch, and any residual git failure surfaces git''s stderr so it is diagnosable
  from the task output.

  '
phases:
- id: retry
  title: Index.lock-aware retry and stderr surfacing in store commit helpers
  depends_on: []
- id: serialize
  title: Inter-process write lock shared by sync workers and foreground committers
  depends_on:
  - retry
- id: sequence
  title: Consolidate epic-launch commits to one deferred store push
  depends_on: []
bead_id: sase-67
---

# Plan: Serialize SDD store git writers and retry index.lock contention

## Incident

An epic launch (`sase bead work <plan>.md --yes`, driven from the Admin Center Tasks tab) failed after validation,
bead-store creation, and plan archiving had all succeeded. The failing step was the bead-link commit:

```
Error: Command '['git', 'commit', '-m', 'Link approved epic plan to its bead:
runner_silent_failure_visibility\n\nSASE_TYPE=sdd', '--',
'202607/runner_silent_failure_visibility.md']' returned non-zero exit status 128.
ERROR: Epic launch failed with exit code 1
```

The task output showed only "exit status 128" with no reason. The actual git stderr was recorded in
`~/.sase/logs/tui_git_ops.jsonl` (event `sdd_git_operation`, op `sdd.commit`):

```
fatal: Unable to create '.../repos/plans/.git/index.lock': File exists.
Another git process seems to be running in this repository...
```

Timeline evidence from `tui_git_ops.jsonl` and `~/.sase/bead_push_logs/`:

- The launch's `git add` for the same pathspec succeeded ~5 ms before the failing `git commit`, so this was a live
  concurrent writer, not a stale lock file.
- Managed sync workers (fetch/rebase/push) ran in the same shared plans sidecar repo seconds before, and another agent
  process pushed to the same remote from its own clone in the same window.
- ~17 s after the failure, the rollback bead-store commit in the same repo succeeded — the contention had already
  cleared. A short retry would have saved the entire launch; instead the epic bead was created, rolled back, and the
  launch surfaced as failed.

## Root cause

Many SASE processes write concurrently to one shared SDD sidecar store repo (the plans repo also contains the bead store
under `beads/`), and none of the foreground git writers are serialized or retry on contention:

- `commit_sdd_files` (`src/sase/sdd/_commit_store.py`) runs `git add` + `git commit` with `check=True`; any 128 aborts
  the caller. It backs the plan archive commit, the bead-link commit, and `auto_commit_bead_store` (which fires after
  every `sase bead` create/update/close/remove in every agent process).
- `git_sync` and `commit_bead_work_launch` (`src/sase/bead/sync.py`) run raw `subprocess.run` git add/commit with no
  telemetry, no lock, no retry. `BeadProject.sync()` calls `git_sync` after every bead mutation.
- The managed sync worker (`src/sase/bead/sync_worker.py`) takes a `sase-bead-sync.lock` flock, but that only serializes
  workers against each other. Its `git rebase @{upstream}` holds `index.lock` and rewrites the worktree while foreground
  committers run unsynchronized in the same repo. Its clean-worktree check before rebasing is a TOCTOU guard only.

With dozens of live agents sharing one plans/beads repo, `index.lock` collisions are statistically routine. Most writers
are best-effort and swallow the failure silently; the epic-launch link commit is the one path that (correctly) treats
commit failure as fatal, so that is where the race became visible.

Two distinct defects fall out of this, plus one aggravator:

1. **No recovery from transient contention.** A single `index.lock` collision — inherently transient — aborts an epic
   launch and rolls back the created beads.
2. **No mutual exclusion between SASE's own writers.** Beyond the visible failure, a foreground `git commit` that lands
   mid-rebase (rebases run with a detached HEAD) can be silently orphaned when the rebase finishes — a quiet data-loss
   hazard.
3. **Undiagnosable errors.** `commit_sdd_files` raises bare `CalledProcessError`, whose string omits stderr, so the
   surfaced task error hides the "index.lock: File exists" reason entirely.

## Scope note: Python, not Rust core

Bead mutations go through the Rust core facades, but git commit/push/sync orchestration for store repos is an existing
Python layer in this repo (`sase/sdd/_commit_store.py`, `sase/bead/sync.py`, `sase/bead/sync_worker.py`). This plan
hardens that existing layer in place; it changes no wire contract or domain behavior, so nothing moves to `sase-core`.

## Design

### Phase `retry` — index.lock-aware retry and stderr surfacing

Add a small helper module under `src/sase/sdd/` (e.g. `_git_contention.py`) providing:

- A predicate for retryable git lock contention: exit code 128 and stderr matching git's lock-file diagnostics (at
  minimum `index.lock` / `Unable to create` + `.lock`; keep the match narrow enough to never retry identity, pathspec,
  or repository-state fatals).
- A bounded retry wrapper for git _write_ invocations (add/commit): retry with short exponential backoff (on the order
  of 100 ms doubling to a total of ~5–10 s), env-overridable so tests can shrink the schedule. Lock-failure attempts are
  safe to retry because git makes no changes when it cannot take the lock. Each retry should be visible in the existing
  `sdd_git_operation` telemetry.

Adopt it in every SASE-managed store writer:

- `commit_sdd_files` in `src/sase/sdd/_commit_store.py` (`sdd.add`, `sdd.commit` ops).
- `git_sync` and `commit_bead_work_launch` in `src/sase/bead/sync.py`. While here, route their raw `subprocess.run` git
  calls through `run_sdd_git` so they gain the same telemetry and timeout bounds as other SDD git ops.

Error surfacing: when retries are exhausted (or a non-retryable failure occurs), the exception raised out of
`commit_sdd_files` must carry git's stderr, so `PlanFileWorkError` / the Tasks-tab output shows "Unable to create ...
index.lock" instead of a bare exit status. Preserve the existing return-value contract (False for benign no-ops, True on
commit).

Testing (extend `tests/test_sdd_commit.py`, `tests/test_bead/test_sync_remote.py`):

- Pre-create `.git/index.lock` in a fixture repo, release it from a timer thread, and assert the commit succeeds after
  retrying.
- Hold the lock past the (test-shrunk) retry budget and assert the raised error message contains git's stderr.
- Assert a non-retryable 128 (e.g. bad pathspec) fails immediately without retries.

### Phase `serialize` — one write lock for all SASE store-repo writers

Extend the same helper module with an inter-process store write lock: an flock on `<git-dir>/sase-store-write.lock` with
bounded blocking acquisition.

- The managed sync worker (`src/sase/bead/sync_worker.py`) acquires it only around its local integration section (the
  status check, `rebase`, conflict-repair `rebase --continue` loop, and `rebase --abort`), not around `fetch`/`push`
  network calls, keeping hold times sub-second in practice. Lock ordering is fixed and documented: `sase-bead-sync.lock`
  (outer, worker-only) then `sase-store-write.lock` (inner); foreground writers only ever take the inner lock, so
  deadlock is impossible.
- Foreground writers (`commit_sdd_files`'s add/diff/commit sequence, `commit_bead_work_launch`, `git_sync`) acquire it
  with a bounded wait (~10 s) and proceed fail-open on timeout — flocks vanish with their holder process, so waiting is
  safe, and the phase-`retry` backoff still covers anything that slips through. This closes both the visible index.lock
  race and the mid-rebase orphaned-commit hazard between SASE processes.

Testing:

- Unit-test the lock helper's bounded wait and fail-open timeout.
- A two-writer test (thread or subprocess) asserting a foreground commit blocks until a held write lock is released
  rather than failing.
- A sync-worker test asserting the worker takes the write lock for the rebase section (e.g. via a probe that holds the
  lock and observes the worker wait) and that fetch/push run outside it.

### Phase `sequence` — one deferred push per epic launch

`work_from_plan_file` (`src/sase/bead/cli_work_from_plan.py`) currently lets each intermediate commit (archive,
bead-link, rollback) trigger its own fetch/rebase/push cycle via `push_sdd_store_after_commit`, multiplying the
contention window inside its own multi-commit sequence and adding remote round-trips mid-launch.

Change `_commit_plan_file` callers so intermediate commits pass `push_after_commit=False`, and trigger a single async
store push once the launch sequence completes (after `create_and_launch_epic_from_plan` returns, and likewise after the
resume path). Failure/rollback paths keep their existing best-effort push behavior so a failed launch still converges
with the remote. `--no-push` continues to suppress every push. Behavior contract: exactly one push trigger per
successful launch, none before the final commit of the sequence.

Testing (extend `tests/test_bead/test_cli_work_from_plan.py`):

- Record push invocations via monkeypatch and assert intermediate commits do not push, one async push fires at the end
  of a successful launch, and `--no-push` yields zero pushes.
- Assert the rollback path still pushes best-effort.

## Risks and mitigations

- **Retrying a genuinely wedged repo delays failure** by the retry budget (~10 s worst case). Acceptable for
  launch-critical commits; the budget is bounded and env-tunable.
- **Over-broad retry classification** could mask real fatals. Mitigated by a narrow stderr match and a test asserting
  non-lock 128s fail fast.
- **Lock scope bugs** (worker holding the write lock across network I/O) would stall foreground commits. Mitigated by
  scoping the lock to the rebase section only and by the foreground fail-open timeout.
- **Deferred push widens the unpushed window** during a launch by a few seconds. Bead-store convergence already
  tolerates this (TTL-gated background refresh; workers rebase on divergence).

## Out of scope

- Non-SASE git writers in store repos (a human running git manually): the retry in phase `retry` degrades gracefully
  there, which is sufficient.
- Moving git orchestration into `sase-core` (see scope note).
- The Admin Center Tasks tab UI itself; better error text falls out of the stderr surfacing in phase `retry`.
- The workers' broader fetch/rebase policy (`sdd.push_after_commit`, `bead_refresh` config semantics) stays as is.
