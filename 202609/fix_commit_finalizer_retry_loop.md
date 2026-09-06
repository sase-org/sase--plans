---
tier: tale
title: Fix commit-finalizer infinite retry loop after a partially-failed stitch
goal: The commit finalizer's retry loop always terminates within its attempt budget,
  and a retry cycle that finds an accepted repo already committed by a prior attempt
  of the same run completes successfully instead of hanging the agent runner.
size: medium
proposed_by: bbugyi200.athena.0gt
status: done
---

# Fix Commit-Finalizer Infinite Retry Loop That Hangs Agents After A Partially-Failed Stitch

## Problem

A live incident on 2026-09-06 (agent `bob-cli-1h.f0`, project `gh_bobs-org__bob-cli`,
run artifacts at
`~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202609/06/20260906140830`) left
the agent runner spinning at ~100% CPU for 20+ minutes after the provider turn had
already completed successfully. The runner was stuck inside the commit finalizer's retry
loop, re-running the expensive full dirty-state scan (repo inventory across all
workspaces plus many git subprocesses) on every cycle, writing no artifacts, and never
terminating. `sase agent list` showed the agent as RUNNING the whole time.

Confirmed causal chain (via py-spy stack samples, `py-spy dump --locals`, and the run's
artifacts):

1. The agent's final commit declaration was accepted. Attempt 1 of the `commit`
   finalizer instance ran `sase stitch create` for the linked `chezmoi` repo. The commit
   itself landed and was pushed (its marker is in the run's `commit_results.json`, and
   `commit_state.json` records `completed_steps: ["dispatch", "file_hooks"]`), but the
   stitch exited non-zero because the after-commit hook (`chezmoi update -a --force`)
   hit a rebase conflict. That recorded a retryable `stitch_failed` on attempt 1 and
   consumed 1 of `max_attempts: 2` (see `finalizers:` in `src/sase/default_config.yml`).
2. The workspace clone later converged clean (HEAD equals the pushed commit, no rebase
   in progress), so on every retry cycle `execute_commit_finalizer`
   (`src/sase/finalizers/commit.py`) takes the `already_clean` path and raises
   `dirty_work_discarded` — a result that carries NO new attempt wire.
3. In `_run_budgeted_commit` (`src/sase/finalizers/controller.py`, the `while True:`
   loop around line 423), that exception's result is merged into the ledger and
   `is_retryable_result(merged)` is consulted. Because the fresh failure added no
   attempt, `merged.attempts[-1]` is still attempt 1 with `stitch_failed` (a retryable
   code), so the merged view is misclassified as retryable. No budget was consumed,
   `ledger.remaining() > 0` stays true forever, and the loop `continue`s indefinitely.
   `py-spy dump --locals` showed `consumed_before: 1` frozen across all samples.

Three distinct defects combine here:

- **Defect A — missing no-progress guard.** `run_budgeted_attempts`
  (`src/sase/finalizers/ledger.py`, around lines 151-166) bails out when a cycle
  consumed no budget (`if ledger.consumed == consumed_before: return ...`). The
  hand-rolled loop in `_run_budgeted_commit` captures `consumed_before` but never checks
  it in the exception path, so zero-progress cycles retry forever.
- **Defect B — retryability judged on stale merged state.** `_run_budgeted_commit`
  evaluates `is_retryable_result` on the merged ledger result. When the fresh failure
  has no attempt wire, the "latest attempt" is a prior cycle's attempt, so a
  non-retryable failure (`dirty_work_discarded`, `bead_state_unpublished`, ...) is
  treated as retryable.
- **Defect C — a commit landed by a prior attempt of the same run can never prove the
  repo clean.** In `execute_commit_finalizer` the `already_clean` guard (around
  `src/sase/finalizers/commit.py:212-236`) calls `reject_discarded_dirty_work`
  (`src/sase/finalizers/commit_validation.py`, around line 247) with
  `ledger_before=ledger_before_reconciliation`, which is snapshotted at the start of the
  SAME cycle (commit.py, around line 104). `proven` therefore only counts commit markers
  created within the current cycle, so attempt 1's landed commit is invisible on every
  retry cycle. Additionally, that call site passes no `fingerprint_before` to
  `discarded_dirty_work_evidence`
  (`src/sase/llm_provider/commit_finalizer_git_progress.py`, around line 72), so
  `progress_fingerprint(before)` computes the "before" head from the CURRENT worktree,
  guaranteeing `before_head == after_head` and thus a `head_not_advanced` discarded
  verdict for the accepted repo. The correct outcome for this converged state is
  success, not `dirty_work_discarded`.

## Changes

### 1. Add the no-progress guard to `_run_budgeted_commit` (Defect A)

In `src/sase/finalizers/controller.py`, in the `except BuiltinCommitFinalizerError`
branch of `_run_budgeted_commit`: only retry when this cycle actually consumed budget.
Mirror `run_budgeted_attempts` semantics — after `merged = ledger.record(exc.result)`,
require `ledger.consumed > consumed_before` (in addition to the existing retryable +
remaining-budget conditions) before `continue`. When no budget was consumed, re-raise
the terminal `BuiltinCommitFinalizerError` with the merged result exactly as the
existing non-retryable path does. This guard alone ends the infinite-loop class even if
other defects regress.

### 2. Scope retryability to the fresh cycle result (Defect B)

Base the retry decision on the failure recorded THIS cycle rather than the merged ledger
view: evaluate `is_retryable_result` against `exc.result` (whose latest attempt and
diagnostics belong to the current cycle) instead of `merged`. Keep merging into the
ledger for reporting/aggregation. `run_budgeted_attempts` in
`src/sase/finalizers/ledger.py` has the same theoretical misclassification but is
protected by its no-progress guard; align it the same way only if doing so keeps all
existing ledger tests passing — otherwise leave it and note why in the code.

### 3. Let run-owned commit markers prove an already-clean accepted repo (Defect C)

Goal: a retry cycle that finds an accepted repo clean, where a prior attempt of this
same run landed a commit for that repo (marker present in the run's
`commit_results.json`), must pass the `already_clean` guard and reach the success return
(the `attempt_id is None` path at the end of `execute_commit_finalizer`).

Preferred shape (coder may adjust if a cleaner seam exists, but keep the fail-closed
property):

- In `execute_commit_finalizer`, snapshot the commit ledger once at declaration-load /
  run-entry time and pass THAT as `ledger_before` for the `already_clean`
  `reject_discarded_dirty_work` call, so markers from earlier attempts of this run count
  as proof; OR equivalently, in `reject_discarded_dirty_work`, union `proven` with all
  markers in the run-owned `commit_results.json` (it is scoped to this run's artifacts
  dir) whose `cwd` matches the repo path.
- Do NOT weaken the protection for repos with no matching run-owned marker: dirty work
  vanishing without any attributable commit from this run must still fail closed with
  `dirty_work_discarded`.

With this fix, the incident scenario converges: cycle 2 proves the chezmoi repo via
attempt 1's marker, nothing is dirty, no publication errors remain, and the finalizer
returns success.

### 4. Follow-up for the unresumed after-commit hook

Attempt 1 left `commit_state.json` with the after-commit hook step pending
(`sase stitch create --resume` is the manual remedy). Automatically resuming pending
stitch steps on finalizer retry is out of scope here; capture it as a follow-up task
bead via the `/sase_new_task` skill (root cause: plain `stitch_failed` hook failures
never invoke `run_stitch_resume`, unlike the `EXIT_CODE_CONFLICT` repair path in
`src/sase/finalizers/commit_dispatch.py`).

## Tests

Add regression tests (see `tests/test_finalizers_execution_ledger.py` for the existing
budget-boundary patterns and `tests/finalizers_commit_reconciliation_test_helpers.py`
for reconciliation fixtures):

1. **Loop termination (A+B):** drive `_run_budgeted_commit` with a fake commit executor
   whose first call consumes one attempt and raises a retryable `stitch_failed`, and
   whose subsequent calls raise `BuiltinCommitFinalizerError` with a
   `dirty_work_discarded` result carrying no attempt wire and consuming no budget.
   Assert the loop terminates with a failure after a bounded number of executor calls
   (no infinite loop) and that the terminal result is not misclassified as retryable.
2. **Retry-within-budget preserved:** a cycle that consumes budget and fails with a
   retryable code still retries while budget remains (existing behavior must not
   regress).
3. **Converged success (C):** accepted repo with a commit marker recorded by attempt 1
   in the run-owned `commit_results.json`, worktree clean on the retry cycle → the
   finalizer returns success, not `dirty_work_discarded`.
4. **Fail-closed preserved (C):** accepted repo clean with NO matching run-owned marker
   → still fails with `dirty_work_discarded`.

## Verification

- All code involved is Python in this repo; no Rust core (`sase-core`) changes needed.
- Run `just install` first (ephemeral workspaces may have stale virtualenvs), then
  `just check` before finishing (hand it to the `/sase_monitor` skill if it runs long).
  This change touches retry/ledger semantics used by every agent run; if the scoped test
  selection escalates or looks unusual, run `just check-full` through `/sase_monitor`
  instead.
