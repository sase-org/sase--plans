---
tier: tale
title: Make SDD/bead sidecar commits see staged-only changes
goal:
  Bead-sidecar commit paths commit staged-only changes (including staged deletions) so
  they agree with bead_state_is_clean and the commit finalizer never strands staged
  migration state.
size: medium
proposed_by: bbugyi200.athena.0y3
create_time: 2026-10-07 17:19:41
status: wip
---

# Plan: Make SDD/bead sidecar commits see staged-only changes

## Problem

Agent `bob-cli-5k.7.1.1` (project `gh_bobs-org__bob-cli`) did its work and its primary
stitch commit landed on `origin/master`. The `builtin@commit` finalizer then failed with
`dirty_after_commit_decisions`:

```
bead sidecar .../sase/repos/beads could not be reconciled
remaining files: .gitignore, issues.jsonl
reason (commit): bead auto-commit created no commit while 2 file(s) remain dirty
```

The sidecar was left with **staged-only** changes: `M  .gitignore` (the new
`issues.jsonl` ignore rule) and `D  issues.jsonl`. Nothing was left unstaged.

### Root cause (verified)

1. The installed sase checkout fast-forwarded mid-run to a revision that includes
   sase-1h8.11 (`feat(bead): take issues.jsonl off the per-mutation path`). Its
   idempotent `migrate_projection_off_track()`
   (`src/sase/bead/_projection_migration.py`) amends the store's `.gitignore` and
   deletes the tracked `issues.jsonl` worktree copy. Its docstring says it deliberately
   stages **nothing**, because the commit enumerator cannot see staged changes.
2. During the stitch, `handle_beads()` (`src/sase/workflows/commit/bead_hooks.py`) runs
   `sase bead sync`. That calls `BeadProject.sync()` and then `git_sync()`
   (`src/sase/bead/_sync_git.py`). `git_sync()` runs the migration and then
   `git add -- .gitignore issues.jsonl`. It is stage-only by contract and never commits,
   so the migration ended up staged but uncommitted.
3. Every later sidecar commit goes through `commit_sdd_files()`
   (`src/sase/sdd/_commit_store.py`). That function enumerates candidates with
   `changed_sdd_files()`, which runs
   `git ls-files --modified --others --deleted --exclude-standard`. That command
   compares the worktree with the index only, so staged-only changes are invisible to
   it. Two commits therefore missed the staged changes:
   - The bead-pages publication commit
     (`chore(beads): sync bead state and pages for bob-cli-5k`,
     `src/sase/bead_pages/publication.py`) committed only `pages/`.
   - The finalizer's bounded late pass
     (`src/sase/finalizers/reconciliation_bead_store.py::_auto_commit_bead_state`)
     created no commit.
4. `bead_state_is_clean()` uses `_list_bead_state_changes()`. That function **is**
   staged-aware (it also runs `git diff --cached`), so it reported the store dirty. The
   detector and the committer disagree, so `ensure_no_residual_dirt` failed the run.

A second latent defect affects the staged-aware path. `_list_bead_state_changes()`
returns staged entries, and its consumers then pass every entry to `git add -- <files>`.
If a staged deletion's path is absent from both the worktree and the index, `git add`
exits 128 with `fatal: pathspec 'issues.jsonl' did not match any files`. This was
verified in a scratch repo. The consequences:

- `_commit_bead_state()` (claim, claim release, epic checkpoint, task launch, rollback
  commits) raises `BeadWorkLaunchCommitError` on such a store.
- `git_sync()` runs its add with `check=False`, so it silently stages nothing, not even
  unrelated real changes.

`git commit -m … -- <paths>` (`--only` mode) handles staged deletions correctly. It
takes worktree content for listed paths, and a path missing from the worktree is
recorded as deleted. This was also verified.

The mid-run upgrade only made this deterministic. In any bead sidecar, whatever
`sase bead sync` stages becomes uncommittable through the SDD commit path. The migration
guarantees that `sase bead sync` has something to stage the first time new-code bead
state runs in a workspace clone whose `issues.jsonl` is still tracked.

## Goal

The SDD/bead commit paths treat "dirty" the same way `bead_state_is_clean()` does:
worktree changes **plus** staged changes, scoped by the same pathspecs. Staged-only
state (including staged deletions) is committed by the next bead commit instead of
stranding the finalizer. No commit path ever passes a path to `git add` that `git add`
cannot match.

## Changes

### 1. `src/sase/sdd/_commit_store.py`: staged-aware `commit_sdd_files`

- Add `staged_sdd_files(sdd_dir: Path, pathspecs: list[str]) -> list[str]`. It runs
  `git diff --cached --name-only -z -- *pathspecs` through `run_sdd_git` with
  `op="sdd.changed_files.staged"`.
  - Unborn `HEAD` (no commit yet): return `[]` instead of failing. Mirror the
    `_has_head_commit()` fallback in `src/sase/bead/_sync_git.py`; a small local helper
    is fine.
  - Any other non-zero exit raises `subprocess.CalledProcessError`, so the existing
    `except subprocess.CalledProcessError` blocks still convert it to
    `SddGitCommandError`.
  - It must use the **same** pathspecs as `changed_sdd_files()`. The `:(exclude)goals`
    pathspec then keeps staged goal-ledger changes out of bead commits, as the epic
    `sase-1bu` contract in `normalize_sdd_commit_pathspecs` requires.
- Leave `changed_sdd_files()` unchanged (worktree-only). Tests patch it by its dotted
  path (`tests/test_sdd_commit.py` around line 470), and `sase/sdd/files.py` re-exports
  it.
- In `commit_sdd_files()`, compute two lists:
  - `add_files = changed_sdd_files(...)`: the worktree changes, the only paths that may
    be passed to `git add`;
  - `commit_files`: the order-preserving union of `add_files` and
    `staged_sdd_files(...)`.

  Then:
  - Return `False` only when `commit_files` is empty.
  - Run `prepare_event_streams_for_commit(sdd_dir, commit_files)`. When it restores
    paths, recompute both lists, as the code does today for the single list.
  - Run `git add -- *add_files` only when `add_files` is non-empty.
  - Use `commit_files` for `git diff --cached --quiet`, `_capture_staged_sdd_diff`,
    `git commit -m … -- *commit_files`, and
    `_derive_artifact_links_for_commit(..., changed_files=commit_files)`.

- Update the docstring to say that staged-only changes under the target pathspecs are
  committed too.

### 2. `src/sase/bead/_sync_git.py`: split "add" from "commit" for bead commits

- Refactor `_list_bead_state_changes()` into a helper that returns the worktree list and
  the staged list separately, for example
  `_bead_state_change_sets(beads_dir, repo_root) -> tuple[list[str], list[str]]`.
  - Keep the existing `beads.db*` filtering and the unborn-`HEAD` fallback.
  - Keep `_list_bead_state_changes()` returning the de-duplicated union, so
    `bead_state_is_clean()` and its other callers are unchanged.
- In `_commit_bead_state()`:
  - Run `git add -- <worktree list>`, plus `.gitignore` when the migration reports
    `gitignore_updated` (that is a worktree modification, so adding it is safe).
  - Use the union, plus `.gitignore` in that case, for
    `prepare_event_streams_for_commit`, the `git diff --cached --quiet` probe, and
    `git commit -- …`.
  - Skip the `git add` call entirely when the worktree list is empty but staged entries
    exist.
- In `git_sync()`, stage only the worktree list. Staged entries are already staged, and
  including a staged deletion makes the whole `git add` fail silently.
- Update the docstrings to match.

### 3. Docstring touch-up in `src/sase/bead/_projection_migration.py`

The migration's behavior stays the same: it still stages nothing, which remains correct.
Its docstring paragraph says that a staged `.gitignore` or a `git rm --cached` deletion
is invisible to the commit enumeration, and that is no longer true after change 1.
Reword it so it no longer presents that as a hard constraint. For example: commit paths
now pick up staged entries too, but leaving the rule unstaged keeps the migration
side-effect-free for stage-only callers.

### Out of scope (intentionally)

- `sase bead sync` / `git_sync()` keep their stage-only contract. With change 1, the
  next sidecar commit in the same stitch commits the staged migration. That next commit
  is the bead-pages publication commit, which commits the whole beads root about 7s
  later, or else the finalizer's late bead pass. No CLI semantics change is needed.
- No `sase-core` change. The SDD commit enumeration lives only in Python
  (`sase.sdd._commit_store`, `sase.bead._sync_git`) and has no Rust counterpart, so the
  fix belongs here.
- Operational cleanup of the held bob-cli workspace and the parked sibling agents is a
  manual step for the user, not part of this change.

## Tests

Add regression tests next to the existing suites:

1. `tests/test_sdd_commit.py`
   - Staged-only commit: in a temp git repo, stage a modification to a tracked file and
     a deletion of a tracked file whose worktree copy is gone. Leave nothing unstaged.
     Then:
     - `commit_sdd_files(sdd_dir, "msg")` returns `True`;
     - the new commit contains both paths;
     - `git status --porcelain` is empty.
   - Mixed: one untracked new file plus one staged deletion. Both land in a single
     commit, and no `git add` failure occurs.
   - Targeted scope is preserved: with `paths=[...]` limited to one directory, a staged
     change outside it is **not** committed and stays staged.
   - Goals exclusion: with root pathspecs, a staged change under `goals/` is not
     committed.
2. `tests/test_bead/test_projection_migration.py`, or `tests/test_bead/test_sync.py` if
   fixtures fit better. This is the incident regression, for both the root (sidecar)
   layout and the nested `beads/` layout:
   - Build an event store (`events/` dir) with a tracked `issues.jsonl`.
   - Run `git_sync(beads_dir)`. Assert that `.gitignore` and the `issues.jsonl` deletion
     are staged.
   - Run `commit_sdd_files(repo_root, "msg", paths=[beads_dir])`, including the repo
     `.gitignore` for the nested layout. Assert that a commit was created,
     `issues.jsonl` is no longer tracked, the ignore rule is committed, and
     `bead_state_is_clean(beads_dir)` is `True`.
3. `_commit_bead_state` path, for example `commit_epic_graph_checkpoint`, on a store
   holding a staged `issues.jsonl` deletion plus a real event-stream change: it does not
   raise, and it commits both.
4. `git_sync()` with a pre-existing staged deletion plus a new worktree change: the
   worktree change ends up staged (today the whole add fails silently).
5. Finalizer late pass: in `tests/test_finalizers_commit_beads_residual.py` or
   `tests/llm_provider/test_commit_finalizer_auto_sdd_status.py`, whichever already
   builds a real sidecar fixture, add a case where the sidecar holds only staged
   changes. `auto_commit_separate_sdd_store_if_possible` must report `committed=True`
   with no `reconcile_error`.

## Verification

- Run `just check` (lint gates plus the diff-scoped test lane). Do not run
  `just check-full`.
