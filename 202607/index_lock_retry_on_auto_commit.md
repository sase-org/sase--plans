---
create_time: 2026-07-08 12:50:46
status: wip
prompt: 202607/prompts/index_lock_retry_on_auto_commit.md
tier: tale
---
# Plan: Recover from a locked git index during TUI auto-commits

## Problem

When the `sase ace` TUI auto-commits a change to a git repo (most notably the chezmoi source repo after saving an
xprompt via the `<ctrl+g>x` keymap from the prompt input bar), the commit fails hard if a stale `.git/index.lock` file
is present:

```
fatal: Unable to create '/home/.../.git/index.lock': File exists.

Another git process seems to be running in this repository ...
remove the file manually to continue.
```

Today this surfaces as an error toast ("Commit failed: ...") and the user has to manually delete the lock file and
re-trigger the commit. We want the TUI to detect this specific failure, remove the stale lock, retry the commit once,
and **tell the user via a TUI toast** that it had to do so.

## Goal / desired behavior

When an auto-commit git step fails specifically because `.git/index.lock` already exists:

1. Delete the offending `index.lock` file.
2. Retry that git step exactly once.
3. If the retry (and the rest of the commit) succeeds, the normal success toast still fires **plus** a distinct
   `warning`-severity toast informing the user a stale git index lock was removed and the commit was retried.
4. If a lock was removed but the operation still ultimately fails, the user is still told the lock was removed (in
   addition to the error toast), so the removal is never silent.

No new keymap, option, or user-facing setting is introduced — this is a resilience improvement to an existing flow, so
the `?` help popup and keybinding footer do **not** need updating.

## Where this lives (code map)

The auto-commit path is entirely Python + `subprocess` inside the TUI process (no Rust, no `sase commit`/vcs_provider
dispatch):

- `<ctrl+g>x` → `request_save_as_xprompt` → `SaveAsXpromptRequested` message → the app writes the xprompt file →
  `_offer_git_commit` shows a confirm modal → `_submit_xprompt_commit_task` enqueues a tracked Textual worker task whose
  body calls `run_git_commit_push_sync(...)`.
- The shared commit helper is `run_git_commit_push_sync` in
  `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt_git.py`. It runs, in order, `git add` → `git commit`
  → `git pull --rebase` → `git push`, then optionally `chezmoi apply --force` (when `get_use_chezmoi()` is true). Each
  step is a `subprocess.run([...], check=False)`; any non-zero return short-circuits to
  `(False, "<step> failed: <stderr>")`.
- The same helper is reused by the model-alias commit flow: `_submit_commit_task` in
  `src/sase/ace/tui/modals/models_panel.py`.

Because the commit runs in a Textual worker thread and completion is delivered to the **UI thread** via the tracked-task
`on_complete` callback, we can raise the new toast directly with `self.notify(..., severity="warning")` — there is no
need for the cross-process notification store used by out-of-process runners.

### Prior art to reuse

`src/sase/axe/runner_utils.py` already has robust index-lock plumbing:

- `_git_index_lock_path(workspace_dir)` resolves the correct `index.lock` path, handling worktrees/submodules (`.git`
  file → `git rev-parse --git-dir`).
- `_clear_stale_git_index_lock(...)` removes a lock, but it is **age-gated** (only removes locks ≥ 15s old) and does
  **not** retry, and it is only called during workspace prep — not on the commit path.

We will reuse the path-resolution logic but implement the new error-triggered/delete/retry behavior separately (the
existing helper's age gate and no-retry semantics are deliberately different and should stay as-is for the
workspace-prep use case).

## Rust core boundary

No changes to the sibling `sase-core` Rust repo. Git **process execution, filesystem lock handling, and mutation** are
explicitly Python-side per the `git_query_wire.py` contract ("Process execution, timeouts, ... and every mutating ...
operation stay in Python"). The `sase_core_rs` binding is used only for pure parsing of git output. Detecting a lock
error, deleting a lock file, and retrying a subprocess are exactly the responsibilities that stay in Python, so this
change is consistent with the boundary rule.

## Design

### 1. Small, shared lock helpers

Promote `_git_index_lock_path` → public `git_index_lock_path` in `src/sase/axe/runner_utils.py` (update its single
internal caller; there are no other references, so this is safe), so the commit path can reuse the robust path resolver
instead of duplicating it.

Add two focused helpers next to `run_git_commit_push_sync` in `_prompt_bar_save_xprompt_git.py`:

- `_is_index_lock_error(text: str) -> bool` — true when git failed because the index lock already exists. Detection keys
  on the locale-independent substring `"index.lock"` in the captured stderr/stdout (the git file path is not translated,
  unlike the surrounding "File exists" / "Another git process" prose). The _actual_ deletion is additionally gated on
  the lock file existing on disk, so a spurious substring match can never delete anything real.
- `_remove_git_index_lock(git_root: str) -> bool` — resolve the lock path via `git_index_lock_path(git_root)`; if the
  file exists, `unlink()` it and return `True`; missing file or `OSError` → `False` (logged, never raised).

### 2. Retry wrapper inside `run_git_commit_push_sync`

Refactor the four sequential `subprocess.run` calls to go through an inner `_run_git(args)` step runner that:

1. Runs `git -C <git_root> <args...>`.
2. On success, returns the result.
3. On failure, if `_is_index_lock_error(stderr)` **and** `_remove_git_index_lock(git_root)` succeeds, sets a captured
   `index_lock_removed = True` flag and retries the same command **once**.
4. Returns the (possibly retried) result.

Retry is strictly once-per-step (no re-deletion loop) to avoid thrashing. The wrapper is applied uniformly to every git
step; only the index-mutating steps (`add`, `commit`, and `pull --rebase`) can realistically produce a lock error, but
running `push` through the same wrapper is harmless and keeps the code uniform.

### 3. Carry the "lock removed" signal to the toast

Change `run_git_commit_push_sync`'s return type from `tuple[bool, str]` to a small frozen dataclass:

```python
@dataclass(frozen=True)
class GitCommitPushResult:
    success: bool
    message: str
    index_lock_removed: bool = False
```

The tracked-task infrastructure already provides a `payload` field on `TrackedTaskResult`/`TrackedTaskCompletion` for
exactly this kind of structured hand-off to the UI-thread completion callback. Both callers thread the flag through:

- In `_submit_xprompt_commit_task._task` and `models_panel._submit_commit_task._task`: build
  `TrackedTaskResult[bool](success=..., message=..., payload=result.index_lock_removed, error=...)`.
- In each `_on_complete`: when `completion.payload` is truthy, fire an extra `self.notify(..., severity="warning")`
  toast **in addition to** the existing success/error toast. Fire it regardless of final success so lock removal is
  never silent. The toast names the repo for context, e.g.
  `"Removed a stale git index.lock in <repo-basename> and retried the commit."` (`git_root` is in scope in both
  completion closures).

## Concrete changes

1. `src/sase/axe/runner_utils.py`
   - Rename `_git_index_lock_path` → `git_index_lock_path` (public); update the internal call in
     `_clear_stale_git_index_lock`.

2. `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt_git.py`
   - Add `GitCommitPushResult` dataclass; add `_is_index_lock_error` and `_remove_git_index_lock` (lazy import of
     `git_index_lock_path`).
   - Refactor `run_git_commit_push_sync` to use the `_run_git` retry wrapper and return `GitCommitPushResult`; update
     `__all__` and docstring.
   - Update `_submit_xprompt_commit_task` (`_task` payload + `_on_complete` warning toast).

3. `src/sase/ace/tui/modals/models_panel.py`
   - Update `_submit_commit_task._task` to the new return type and `_on_complete` to raise the warning toast on lock
     removal.

4. Tests (`tests/ace/tui/actions/test_prompt_save_xprompt.py`)
   - Update the two existing call sites that unpack `success, message = ...` to use the new `GitCommitPushResult`.
   - Add coverage:
     - lock error on `commit`, lock file present → deleted → retry succeeds → `success` and `index_lock_removed` both
       true; verify the failing command is retried exactly once.
     - lock error but retry still fails → `success` false, `index_lock_removed` true (removal still reported).
     - stderr mentions `index.lock` but no lock file on disk → no deletion, no retry (guards against false-positive
       deletion).
     - `_is_index_lock_error` unit cases (real lock stderr vs. unrelated errors).
   - Optionally a focused test asserting `_on_complete` emits the warning toast when `payload` is true (mirroring the
     existing `_SaveHarness` notification pattern).

5. `tests/test_axe_runner_utils.py` — no behavior change; only touch if the promoted name needs updating there (current
   tests reference `_clear_stale_git_index_lock`, not `_git_index_lock_path`).

## Safety tradeoff (decision point)

Per the request, when the lock file exists and the error matches, we delete it unconditionally and retry — we do **not**
apply the 15s age gate that `_clear_stale_git_index_lock` uses. The tradeoff: an `index.lock` can legitimately belong to
a _live_ concurrent git process (e.g. the user running git in a terminal, `chezmoi apply` spawning git, or another sase
workspace committing to the same chezmoi repo), and deleting it could corrupt that process's index write. The risk is
low here because these auto-commits are short-lived and per-repo deduplicated within the TUI, and the requested UX is
explicitly "delete + retry + toast." If we later decide the risk is unacceptable, the drop-in mitigation is to gate
deletion behind a short staleness threshold (reusing the existing age-gate logic). Recommendation: implement the
unconditional delete-on-lock behavior as requested; the warning toast keeps the action visible to the user.

## Out of scope

- The other independent commit subprocesses (`sase init` chezmoi deploy in `src/sase/main/_init_chezmoi_deploy.py`, the
  SDD commit finalizer, the vcs_provider `sase commit` dispatch, revert-transaction commits). They have their own
  runners and are not part of the `<ctrl+g>x`/model-alias TUI auto-commit flow this request targets. If desired later,
  the same `_is_index_lock_error` + `git_index_lock_path` helpers can be reused there.
- Changing the age-gated workspace-prep lock clearing.

## Validation

- `just install` then `just check` (lint + mypy + tests) after implementation, per repo policy for source changes.
- Manual smoke: create a `.git/index.lock` in a scratch repo, run `run_git_commit_push_sync` against it, and confirm the
  lock is removed, the commit retries, and the result reports `index_lock_removed=True`.
