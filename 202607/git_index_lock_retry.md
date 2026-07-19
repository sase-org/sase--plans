---
tier: epic
title: Codebase-wide git index.lock retry and stale-lock recovery
goal: 'Every git invocation in sase survives transient .git/index.lock contention
  via bounded backoff retries and, when retries are exhausted, safe automatic removal
  of provably-stale lock files followed by one final retry — replacing today''s three
  inconsistent ad-hoc mechanisms with one shared policy.

  '
phases:
- id: core
  title: Shared git lock retry policy
  depends_on: []
  description: '''Shared git lock retry policy'' section: build the central classifier
    + backoff-retry + stale-lock-removal helper module with unit tests.'
- id: funnels
  title: Route high-traffic git runners
  depends_on:
  - core
  description: '''Route high-traffic git runners'' section: wire the vcs_provider
    CommandRunner, the SDD git write path, and the bare-git workspace-provider plugins
    through the shared policy.'
- id: longtail
  title: Migrate remaining git call sites
  depends_on:
  - core
  description: '''Migrate remaining git call sites'' section: convert the scattered
    per-module git runners and inline index.lock handling (ace TUI xprompt save, agent
    revert, dev_update, bead conflict resolver, commit finalizer, misc direct subprocess
    sites) to the shared policy and delete the duplicated logic.'
- id: verify
  title: Lock recovery exercises
  depends_on:
  - funnels
  - longtail
  description: '''Lock recovery exercises'' section: exercise stale-lock recovery
    end-to-end across representative flows with planted lock files and run the full
    regression sweep.'
  model: haiku
create_time: 2026-07-19 09:20:28
status: wip
---

# Plan: Codebase-wide git index.lock retry and stale-lock recovery

## Context

Git operations across sase intermittently fail with `fatal: Unable to create '<repo>/.git/index.lock': File exists.`
(exit code 128). The lock is almost always left behind by a crashed or SIGTERM-killed git process — common under sase's
agent lifecycle, where runners are killed mid-commit. Once present, the stale lock blocks _every_ subsequent git
operation in that clone. Operationally these locks have always been safe to delete.

Desired behavior everywhere git runs: on an index.lock failure, retry the command a few times with backoff (the
competing process usually finishes quickly); if all retries fail, delete the lock file (when it is provably stale) and
retry the operation one final time.

Today the codebase has **three inconsistent partial mechanisms** and a large unprotected majority:

1. `src/sase/sdd/_git_contention.py` — `run_sdd_git_write()` retries on lock errors with bounded backoff
   (`DEFAULT_GIT_LOCK_RETRY_DELAYS = (0.1, 0.2, 0.4, 0.8, 1.6, 3.2)`, overridable via `SASE_SDD_GIT_LOCK_RETRY_DELAYS`)
   and serializes store writes with an flock (`store_git_write_lock`). It never deletes a stale lock, so an abandoned
   lock still fails all SDD/bead writes.
2. `src/sase/axe/runner_workspace.py` — `clear_stale_git_index_lock()` deletes an index.lock older than 15s
   (`_STALE_GIT_INDEX_LOCK_MIN_AGE_SECONDS`), but only pre-emptively during `prepare_workspace()`.
   `git_index_lock_path()` here is the canonical lock-path resolver (handles worktree/submodule `.git` files via
   `git rev-parse --git-dir`).
3. `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt_git.py` — an inline `_run_git` closure that deletes
   the lock immediately (no backoff, no age check) and retries exactly once, with its own private `_is_index_lock_error`
   / `_remove_git_index_lock` copies.

Everything else — the vcs_provider plugin ops (commit/checkout/merge/rebase dispatch), the bare-git workspace-provider
plugins (clone/worktree/branch/merge/submit), agent revert, dev_update, the bead conflict resolver, the commit
finalizer, and a dozen misc `_run_git` helpers — has **no** protection at all.

There is no single funnel to patch: git execution is spread across per-domain runners
(`src/sase/vcs_provider/_command_runner.py` `CommandRunner._run`, `src/sase/sdd/_git.py` `run_sdd_git`,
`src/sase/version/_git.py` `run_git`, `src/sase/ace/revert_agent_git.py` `run_git`, `src/sase/core/shell.py` for generic
commands) plus direct `subprocess.run(["git", ...])` sites.

**Rust core boundary:** the Rust binding (`sase_core_rs`) only _parses_ git output (`src/sase/core/git_query_facade.py`,
`vcs_log_facade.py`); no git subprocess execution happens in Rust. Subprocess orchestration and lock-file recovery are
Python-side process glue, not shared domain behavior, so this effort deliberately makes no `sase-core` changes.

## Design overview

Introduce one shared policy module and route every runner through it. The policy, applied at each attempt boundary:

1. Run the git command. If it succeeds or fails for a non-lock reason, return as today.
2. If the failure is a **retryable lock error** (exit code 128 and stderr naming `index.lock`, or `unable to create` +
   `.lock` — the existing SDD classifier, promoted to shared code), record the lock file's identity (path, mtime,
   inode/size) on first failure, then sleep the next backoff delay and retry. Default schedule: the proven SDD delays
   `(0.1, 0.2, 0.4, 0.8, 1.6, 3.2)`.
3. When retries are exhausted, attempt **last-resort deletion** — but only when the lock is provably stale (see safety
   rules below) — then run the command one final time.
4. Surface the outcome (including a `lock_removed` flag) so callers can log/notify, and return the final result in the
   caller's native result type.

### Lock-deletion safety rules

Deleting a lock held by a _live_ git process can corrupt an in-flight index write, so deletion is guarded:

- Only files named `index.lock` inside the repository's resolved git dir are ever deleted — never ref locks,
  `config.lock`, `HEAD.lock`, etc. (those still get retry/backoff, just not deletion).
- Resolve the lock path preferentially by parsing the absolute path out of git's
  `Unable to create '<path>': File exists.` stderr line; fall back to the canonical `git_index_lock_path()` resolver.
- Delete only when the lock is provably stale: **either** the same lock file (unchanged mtime/inode) persisted across
  the entire backoff window (a healthy git index operation completes in well under the ~6.4s total backoff), **or** its
  age exceeds the existing 15s stale threshold. If the lock file churned during the window (live contention from other
  processes), fail with the original error rather than delete.
- Deletion failures (race with the owner's own unlink, permissions) degrade to returning the original error — never a
  new exception type from the recovery path.
- Callers with stronger knowledge (e.g. `prepare_workspace`, which runs in an exclusively-claimed workspace) keep their
  existing more-aggressive pre-emptive clearing; it is complementary.

### Observability

Every retry and every deletion logs at warning level with the repo path, attempt count, and lock age, consistent with
the existing structured git-operation logging used by `run_sdd_git` (`sase.logs`). The TUI xprompt-save path keeps its
user-facing toast (`git_index_lock_retry_message`) driven by the shared outcome flag instead of its private copy.

### TUI responsiveness constraint

Per the TUI perf rules, backoff sleeps must never run on the Textual event loop. All existing TUI git call sites already
run in tracked background tasks or worker threads, and keystroke paths are read-only — but the migration phases must
audit each converted call site and confirm no event-loop or keystroke path gains a sleep. The policy accepts a per-call
override (e.g. zero retries / delays) for any caller that must stay latency-bounded.

## Shared git lock retry policy

Phase goal: the reusable core, with no call-site changes yet.

- New module `src/sase/git_lock_retry.py` (flat top-level module, like `linked_repos.py`):
  - `is_retryable_git_lock_error(returncode, output) -> bool` — promoted from
    `sdd/_git_contention._is_retryable_git_lock_error`.
  - `git_lock_retry_delays()` — shared schedule with new env override `SASE_GIT_LOCK_RETRY_DELAYS`; same
    parsing/validation semantics as the SDD version.
  - `run_with_git_lock_retry(attempt, *, cwd, delays=None, allow_lock_removal=True, ...)` — the generic driver. It takes
    an _attempt callable_ plus a small adapter that extracts (returncode, stderr/stdout text) from the callable's
    result, so runners returning `subprocess.CompletedProcess`, `CommandOutput`, or `(success, error)` tuples can all
    adopt it without changing their public result types. Returns the final native result plus an outcome record
    (attempts made, `lock_removed`, lock path).
  - Stale-lock resolution and deletion implementing the safety rules above. Reuse `git_index_lock_path()` — either
    re-home it into this module with a back-compat re-export from `sase.axe.runner_workspace` (preferred; the TUI
    already imports it from there) or import it; the phase decides based on import-cycle constraints.
  - Backward compatibility: `SASE_SDD_GIT_LOCK_RETRY_DELAYS` remains honored on SDD paths (the SDD wrapper passes its
    resolved delays through), so existing configs and tests keep working.
- Unit tests (new `tests/test_git_lock_retry.py`): classifier truth table; delay parsing and env overrides;
  retry-until-success mid-schedule; deletion of a lock that persisted the whole window; no deletion of a young/churning
  lock; never deleting non-index.lock files; deletion-failure degrades to original error; outcome record contents. Tests
  use tiny delay schedules via parameters/env so the suite stays fast.

## Route high-traffic git runners

Phase goal: the funnels that cover most mutating git traffic adopt the shared policy.

- `src/sase/vcs_provider/_command_runner.py` — `CommandRunner._run()` already detects git commands
  (`_non_interactive_git_options`); wrap git invocations with `run_with_git_lock_retry`. This covers every vcs*provider
  plugin op (commit dispatch, core ops, query ops) in one place. Retry applies to any git command whose _error* is a
  lock error — no verb allowlist to maintain. `_run_streaming` (clone/long ops) may adopt classifier-only retry if
  practical; index.lock does not gate clones, so it is not required.
- `src/sase/sdd/_git_contention.py` — `run_sdd_git_write()` delegates its retry loop to the shared driver, gaining
  last-resort deletion (new behavior for SDD/bead writes). `store_git_write_lock` flock serialization stays as the first
  line of defense. Existing semantics preserved: `SddGitCommandError`, `check` handling, `always_log` on retries.
- `src/sase/workspace_provider/plugins/bare_git_init.py`, `bare_git_workspace.py`, `bare_git_submit.py` — route their
  direct `subprocess.run` git calls through the shared helper.
- `src/sase/axe/runner_workspace.py` — `prepare_workspace()` keeps its pre-emptive `clear_stale_git_index_lock()`; that
  function's deletion internals can now delegate to the shared module to avoid two unlink implementations.
- Update/extend the existing tests for these paths (`tests/test_sdd_git_contention.py`, `tests/test_sdd_commit.py`,
  `tests/test_axe_runner_utils.py`, `tests/test_vcs_provider_git_ops.py`, `tests/test_bare_git_workspace.py`), including
  new coverage that an abandoned lock is removed after retries exhaust on the vcs_provider and SDD write paths.

## Migrate remaining git call sites

Phase goal: the long tail adopts the shared policy and all duplicated lock code is deleted.

- `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt_git.py` — replace the inline `_run_git` closure's
  delete-immediately-retry-once logic with the shared driver (gaining backoff). Keep
  `GitCommitPushResult.index_lock_removed` and the toast in `models_panel_alias_edit.py` / prompt-bar mixin, now fed by
  the shared outcome. Delete the private `_is_index_lock_error` / `_remove_git_index_lock`.
- `src/sase/ace/revert_agent_git.py` `run_git` and `src/sase/ace/revert_agent_transaction.py` `run_git_for_revert`.
- Sweep the remaining runners and direct call sites, wrapping every runner that can execute a mutating git verb:
  `src/sase/dev_update/execute.py`, `src/sase/bead/conflict_resolver.py` (direct `git add`),
  `src/sase/llm_provider/commit_finalizer_git.py`, `src/sase/main/_init_chezmoi_deploy.py`,
  `src/sase/main/init_workspace_handler.py`, `src/sase/axe/image_attachments.py`, `src/sase/sdd/_store_adoption.py`,
  `src/sase/version/_git.py`, `src/sase/core/shell.py` workspace-command git users, and any other
  `subprocess.run(["git", ...])` / `_run_git` sites found by a final grep sweep.
- Read-only probes (doctor checks, rev-parse/status helpers) that cannot hit an index write lock may stay unwrapped;
  each deliberately-skipped runner gets a one-line code comment stating the constraint so the sweep is auditable.
- Audit per the TUI responsiveness constraint: no converted call site may introduce sleeps on the event loop or
  keystroke paths.
- Extend the affected tests (`tests/ace/tui/actions/test_prompt_save_xprompt.py`, `tests/test_bead/test_sync_remote.py`,
  revert/dev_update/finalizer suites) for the new backoff-then-delete behavior.

## Lock recovery exercises

Phase goal: prove the behavior end-to-end and guard against regressions.

- In throwaway git repos with planted `index.lock` files, drive representative flows and assert outcomes:
  - Transient contention (lock removed mid-backoff by a competing process): operation succeeds with retries and **no**
    deletion.
  - Abandoned lock (old mtime): retries exhaust, lock is deleted, final retry succeeds, warning logged; TUI xprompt-save
    path surfaces its toast flag.
  - Live churn (lock recreated fresh during the window): no deletion; original error surfaces.
  - Non-index `.lock` failure: retried but never deleted.
- Cover at least: a vcs_provider plugin commit op, an SDD store write, a bead sync write, the xprompt-save commit path,
  a bare-git workspace-provider op, and `prepare_workspace` self-heal.
- Run the full `just check` and the complete existing lock-related suites listed in the earlier sections; fix any
  regressions surfaced.

## Testing strategy

Each implementation phase lands its own unit/integration tests alongside the code (details in the phase sections above);
the final phase is a pure exercise-and-regression pass. All retry tests pin tiny delay schedules (parameter or env var)
so wall-clock stays negligible. No new CLI surface is added — configuration is env-var only — so no CLI docs/help
updates are required.

## Risks and constraints

- **Deleting a live lock** is the main hazard; mitigated by the identity/age staleness guard, the index.lock-only rule,
  and delete-only-after-retries ordering. The pre-existing 15s stale threshold and the ~6.4s persisted-through-backoff
  observation are both reused rather than invented.
- **Added latency on genuinely-held locks**: worst case ≈ sum of the backoff schedule (~6.4s) plus one final attempt,
  bounded and env-tunable; callers can opt down to zero retries.
- **TUI freezes**: sleeps must stay off the Textual event loop; existing call sites already run git in workers/tracked
  tasks, and the migration phases audit each conversion.
- **Behavioral compatibility**: `SASE_SDD_GIT_LOCK_RETRY_DELAYS` keeps working; `prepare_workspace` pre-clearing,
  `SddGitCommandError` semantics, and the xprompt-save user notification are all preserved. Existing lock tests continue
  to pass (extended, not replaced).
- **Concurrent sase processes sharing a repo** (SDD store, chezmoi) remain additionally serialized by the existing
  flock; the retry policy is the safety net beneath it, matching today's layering.
