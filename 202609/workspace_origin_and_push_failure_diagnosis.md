---
tier: tale
title:
  Fix stale workspace-clone origins and the false "dirty work vanished" finalizer
  diagnosis
goal:
  Workspace provisioning never hands out a clone with a stale origin, and a stitch whose
  push fails surfaces the push error with its local commit recorded instead of a false
  vanished-work failure.
size: medium
proposed_by: bbugyi200.athena.06t
create_time: 2026-09-09 19:52:56
status: wip
---

# Fix stale workspace-clone origins and the false "dirty work vanished" finalizer diagnosis

## Problem

Two agent runs on this project recently failed at the commit finalizer with:

```
Commit finalizer failed: dirty work vanished without an attributable commit.
... HEAD did not advance ...
... no commit exists anywhere in this repo's history for the changed files below ...
```

In both cases the diagnosis was **false**: the agent's work had been committed locally
by the stitch itself (the commit was sitting at HEAD with a correct `SASE_AGENT`
trailer, working tree clean). The real failure was a rejected `git push`. The operator
was told to "recover or redo the changes" for work that was never lost, and one run's
~1,000-line feature commit sat stranded in its workspace clone until recovered by hand.

## Root-cause chain (verified from run artifacts)

1. **Some numbered workspace clones have a stale `origin`.** When several workspaces are
   provisioned in the same batch (clones created within the same second),
   `ensure_git_clone_at()` in `src/sase/workspace_provider/utils.py` sometimes leaves
   the clone's `origin` pointing at the **primary checkout's filesystem path** instead
   of the primary's real origin URL (`git@github.com:sase-org/sase.git`):
   - It reads the primary's origin with a plain `git remote get-url origin` (no
     `run_with_git_lock_retry`); under batch concurrency this can fail (config lock
     contention), yielding `real_url = ""`.
   - When `real_url` is empty, the `set-url` rewrite is **silently skipped** and the
     freshly cloned workspace is returned anyway.
   - The reuse path (existing dir + healthy `git status`) returns immediately and
     **never revalidates origin**, so a workspace born broken stays broken forever. This
     reproduced in two separate batches (one last night, one this morning): in each
     batch the first clone got the correct origin and several later clones kept the
     primary path. Several such broken clones exist right now.

2. **Local-path origin flips the VCS provider from `github` to `bare_git`.**
   `_classify_git_repo()` in `src/sase/vcs_provider/_registry.py` asks plugins first;
   the sase-github plugin only claims repos whose origin URL parses as a GitHub remote.
   A local-path origin falls through to the `bare_git` heuristic. The run then executes
   with `vcs_provider: "Git (bare)"` on a GitHub project (both failed runs show exactly
   this in `done.json`). Side effect: the bare-git flow also committed an "Initialize
   SDD" commit into the main repo, which had to be reverted on master manually.

3. **The bare-git push is refused by git policy, and rebase retries cannot help.**
   `_push_current_branch_with_rebase_retry()` in
   `src/sase/vcs_provider/plugins/_git_commit_dispatch.py` pushes the current branch to
   `origin` — here the primary **non-bare** checkout with that same branch checked out.
   Git refuses with `receive.denyCurrentBranch` ("refusing to update checked out
   branch"). This is a policy refusal, not remote movement, so every rebase retry fails
   identically and the stitch fails **after already creating the local commit**.

4. **A failed stitch records no commit marker, so the finalizer misdiagnoses.** The
   stitch commit exists at HEAD, but because the stitch returned failure, no marker for
   it lands in the run's commit results. On the finalizer's retry cycle the repo is
   clean, and `reject_discarded_dirty_work()` in
   `src/sase/finalizers/commit_validation.py` treats "accepted obligation + clean repo
   - no recorded marker" as discarded work. It raises `dirty_work_discarded`, which
     **masks** the earlier `stitch_failed` diagnostic (the one carrying the true push
     error — that one survives only inside `finalizer_result.json`). The surfaced
     message also asserts "no commit exists anywhere in this repo's history for the
     changed files" without ever checking history — provably false in both failures.

## Changes

### 1. Harden and self-heal origin provisioning (`src/sase/workspace_provider/utils.py`)

In `ensure_git_clone_at()`:

- Read the primary's origin URL using `run_with_git_lock_retry` (same helper already
  used for `set-url` and `fetch`).
- If the primary's origin URL still cannot be read, or the `set-url` rewrite ultimately
  fails, **raise `RuntimeError`** instead of returning a clone whose origin points at
  the primary path. A loud provisioning failure is strictly better than a workspace that
  fails at push time an hour into a run.
- In the early-return reuse path: read the existing clone's origin; if it equals the
  primary checkout path (or differs from the primary's current origin URL), rewrite it
  to the primary's origin URL before returning (best-effort with lock retry; log a
  warning if the heal fails, and only hard-fail when the clone's origin points at the
  primary path and cannot be fixed). This automatically repairs the broken clones that
  exist today the next time each one is handed out.

Tests (`tests/workspace_provider/test_utils.py`):

- Fresh clone: primary origin unreadable → provisioning raises (no silent skip).
- Reuse: clone origin == primary path → healed to primary's origin URL.
- Reuse: clone origin already matches → untouched (no set-url call).

### 2. Record unpushed commits and report the true failure (`_git_commit_dispatch.py`, `src/sase/finalizers/`)

- In the git commit dispatch: when the commit was created but the subsequent push fails,
  persist a commit marker in the run's commit results **before** returning failure —
  same shape as a successful marker plus a flag such as `"pushed": false` (include
  `commit_sha`, `cwd`, message). The local commit is real, attributable work and must be
  observable by validation and by operators.
- `reject_discarded_dirty_work()` (`src/sase/finalizers/commit_validation.py`): treat a
  repo with an unpushed marker as **proven**, not discarded. A clean tree plus an
  unpushed marker must never produce `dirty_work_discarded`.
- Surfaced error: when a stitch fails after committing locally, the failure the agent
  run reports must be the stitch/push error (e.g. "commit <sha> created locally; git
  push failed: <reason>"), not a vanished-work claim. Today the later
  `dirty_work_discarded` diagnostic wins; the earlier `stitch_failed` one must win.
- Retry behavior in `execute_commit_finalizer()` / `_run_budgeted_commit()`
  (`src/sase/finalizers/commit.py`, `controller.py`): when a retry cycle encounters a
  clean repo whose decision is `commit` and an unpushed marker exists, attempt to
  complete the push (the stitch-resume machinery and the preserved commit-message file
  already exist for this) rather than re-stitching or failing with a misdiagnosis. If
  the push fails again, fail with the push error and the marker intact.
- Message honesty: only emit "no commit exists anywhere in this repo's history for the
  changed files" after actually searching history for those paths since the run
  baseline; otherwise say precisely what was checked (e.g. "no commit marker was
  recorded for this repository during this run").

Tests: extend the coverage around `tests/test_finalizers_execution_ledger.py` /
`tests/test_finalizers_live_e2e_cycles.py` with the commit-created-push-refused
scenario: assert the unpushed marker is written, the surfaced error is the push error,
and no `dirty_work_discarded` is raised; plus a resume-cycle test where the retry
completes the push.

### 3. Fail fast on provider misclassification in gh setup (sase-github linked repo)

In the gh workflow setup step (`src/sase_github/scripts/gh_setup.py` in the
`sase-github` linked repo — open it with the `/sase_repo` skill): after the workspace is
resolved, resolve the VCS provider for the workspace directory and **fail the run
immediately** if a `gh` workflow would execute with a non-github provider. The error
must name the workspace's actual origin URL and the expected GitHub remote so the fix is
obvious. This converts a wasted full agent run plus misleading finalizer failure into an
immediate, actionable setup error. Add a plugin-side test.

Note: with Change 1's self-heal in place this guard should never fire, but it protects
against every other way a workspace's remote can rot (manual edits, future provisioning
paths, partially migrated projects).

## Coordination and verification

- The finalizer modules touched by Change 2 were recently modified by the
  persist-observations/stale-writer-fencing epic, and related work may still be in
  flight. Rebase onto latest master before starting and reconcile carefully with any new
  fencing semantics in `src/sase/finalizers/`.
- Run `just install` first if the workspace venv is stale, then `just check` after the
  changes (inline, or via the `/sase_monitor` skill if it runs long). The landing gate
  runs `just check-full` per the two-speed verification rule.
- Do not attempt to repair existing broken workspace clones by hand in this tale; Change
  1's reuse-path heal covers them mechanically.
