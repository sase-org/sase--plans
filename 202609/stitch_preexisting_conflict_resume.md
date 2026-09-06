---
tier: tale
title: Stop misattributing pre-existing rebase state and support no-commit stitch resume
goal:
  "A leftover rebase/merge in a shared repo no longer fails an agent run after a
  successful conflict repair: sase stitch create refuses to blame pre-existing conflict
  state on itself, and --resume finishes cleanly when no commit exists to resume."
size: medium
proposed_by: bbugyi200.athena.0gr.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0gr.f0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0gr.f0.md)
  - [bbugyi200.athena.sase-v7](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-v7/README.md)
- **COMMITS:**
  - [b103017](https://github.com/sase-org/sase/commit/b1030177a4794d0bde0fb7ef32095080d22127e8)
    — docs(memory): record two-speed CI decision

# Stitch: Stop Misattributing Pre-Existing Rebase State And Support No-Commit Resume

## Problem

Agent run `0gr--code` (2026-09-06, run id `20260906142017`) failed with:

```
WorkflowExecutionError: Step 'main' failed: Error: sase stitch create --resume failed
for chezmoi: ❌ Could not find the expected commit at HEAD (subject mismatch). Re-run
sase stitch create from scratch.
```

The run's real work had already landed (its `sase-telegram` commit, research links, and
sdd commits all succeeded). The failure was pure commit-workflow bookkeeping, caused by
a cascade with two independent defects:

1. A **different project's** earlier run left `/home/bryan/.local/share/chezmoi` in a
   paused `git rebase` (its `chezmoi update -a --force` after-commit hook hit a rebase
   conflict replaying an unpushed local commit and was never repaired).
2. When `0gr--code`'s commit finalizer later ran `sase stitch create` in that repo,
   `git add -A` found nothing stageable, so dispatch failed with
   `"No staged changes to commit"` — **no commit was created**.
3. `CommitWorkflow.run()` (`src/sase/workflows/commit/workflow.py`, dispatch-failure
   branch) saw `is_conflict_state()` true — because of the _pre-existing_ leftover
   rebase — and misclassified the failure as a merge conflict caused by this stitch. It
   returned `RunResult.CONFLICT` and kept a checkpoint expecting the payload's commit
   subject at HEAD.
4. The finalizer's conflict-repair LLM turn correctly finished the foreign rebase (the
   replayed commit was dropped as already-upstream; repo ended clean, in sync,
   `just check` passing).
5. `sase stitch create --resume` (`src/sase/workflows/commit/workflow_resume.py`,
   subject-mismatch guard) then hard-failed: the expected commit does not exist at HEAD
   and never will, because step 2 never created one. The resume path has no way to
   represent "there is no commit to resume".
6. `resolve_commit_conflict` (`src/sase/finalizers/commit_repair.py`) treats any nonzero
   resume as `stale_conflict_checkpoint` and raises `BuiltinCommitFinalizerError`,
   failing the whole agent run.

Evidence (host artifact paths, for reviewer reference only):
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/06/20260906142017/finalizers/commit/`
(`attempt-1.chezmoi.stdout` shows the misclassified "No staged changes to commit"
conflict; `attempt-1.chezmoi.inputs.json` shows `dirty_fingerprints: []` and HEAD
already at `origin/master` before dispatch; `conflict_repair_response.md` documents the
repair turn's dead end), and
`~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202609/06/20260906140830/finalizers/commit/attempt-1.chezmoi.stderr`
(the hook failure that stranded the paused rebase).

## Goal

Make the commit workflow honest about pre-existing conflict state and make the resume
path able to finish cleanly when there is no commit to resume, so a leftover rebase in a
shared repo costs at most one repair turn instead of failing the run after the repair
succeeded.

All changes are host-side Python in this repo. No Rust-core (`sase-core`) changes: the
commit workflow, finalizer, and git VCS plugin live entirely in
`src/sase/workflows/commit/`, `src/sase/finalizers/`, and
`src/sase/vcs_provider/plugins/` with no `sase_core_rs` involvement.

## Changes

### 1. Pre-dispatch conflict guard in `CommitWorkflow.run()`

File: `src/sase/workflows/commit/workflow.py`

Shortly after `cwd = os.getcwd()` is resolved and the payload is validated — and
**before** bead handling (`handle_beads`), plan handling, and
`run_before_commit_hook(cwd)` mutate the worktree — resolve the VCS provider and check
`is_conflict_state(provider, cwd)`:

- If the repository is **already** in a conflicted/sync-in-progress state, do not run
  hooks and do not dispatch. Save a minimal `CommitCheckpoint` (method, payload, cwd,
  and the new `no_commit_dispatched=True` field from change 2) so `--resume` has
  something to load, print a warning that clearly attributes the state to a previous
  operation, e.g.:

  > `<method>` cannot start: the repository already has an in-progress rebase/merge left
  > by a previous operation. No commit was created. Resolve or abort the in-progress
  > operation, then run `sase stitch create --resume`.

  and return `RunResult.CONFLICT`. Increment `VCS_OPERATIONS` with
  `operation="commit_conflict_detected"` (matching the existing conflict branch) so
  telemetry still counts it; use a distinguishable `status` label value (e.g.
  `"pre_existing"`) instead of the existing `"ok"`.

- Returning CONFLICT (not FAILED) is deliberate: in the host finalizer this routes to
  the conflict-repair turn, which is exactly the right tool for cleaning up a stranded
  rebase (it did so correctly in the incident).

The existing dispatch-failure CONFLICT branch stays as-is; with the early guard, a
conflict state observed after a failed dispatch can only have been created by that
dispatch, so its current classification becomes correct.

Also set `no_commit_dispatched=True` on the checkpoint in the existing dispatch-failure
CONFLICT branch when the dispatch failure message is the no-staged-changes validation
failure (`"No staged changes to commit"`, see `_validate_staged` in
`src/sase/vcs_provider/plugins/_git_commit_dispatch.py`) — that failure is raised before
`git commit` runs, so no commit exists in that case either. Matching on the message
substring is acceptable; `classify_dispatch_failure` in
`src/sase/workflows/commit/workflow_support.py` already establishes that precedent.

### 2. `CommitCheckpoint.no_commit_dispatched`

File: `src/sase/workflows/commit/checkpoint.py`

Add `no_commit_dispatched: bool = False` to the dataclass. The default keeps old
serialized checkpoints loadable (missing key → default). Verify `checkpoint_load`
tolerates the new key appearing in files written by new code.

### 3. No-commit resume path in `resume_commit_workflow`

File: `src/sase/workflows/commit/workflow_resume.py`

Keep the existing "conflicts still in progress" early return. Then, before the
subject-mismatch guard, handle the no-commit case. Enter it when either:

- `cp.no_commit_dispatched` is true, or
- the subject-mismatch condition fires **and** the working tree has no stageable changes
  (empty `git status --porcelain` modulo SASE-reserved paths — reuse
  `dirty_path_fingerprints` from `src/sase/llm_provider/commit_finalizer_git_status.py`
  or an equivalent provider query). This legacy arm covers checkpoints written by old
  code and the sibling scenario where a genuine conflict resolution legitimately dropped
  the commit as already-upstream (`git rebase --skip` / empty continue).

In the no-commit path:

- If the working tree still has stageable changes (possible when a completed rebase
  reapplied an autostash): print an error explaining that the original stitch never
  created a commit and the changes must be re-committed with a fresh
  `sase stitch create`, delete the checkpoint (it can never resume), record
  `VCS_OPERATIONS` `commit_resume` with a distinct status (e.g.
  `"never_committed_dirty"`), and return `RunResult.FAILED`. This keeps today's outcome
  but with an accurate message and a non-poisonous checkpoint.
- Otherwise (repo clean, no conflict state): print a success-status message that the
  checkpointed stitch has nothing to finish (no commit was created, or the commit was
  dropped because its changes were already upstream), delete the checkpoint, record
  `VCS_OPERATIONS` `commit_resume` with status `"no_commit_needed"`, and return
  `RunResult.OK`. Skip `finalize_commit`, file hooks, after-hook, and tracking steps —
  there is no commit to track. Do not write a `commit_results.json` marker (change 4
  makes the finalizer accept this outcome explicitly instead of inventing a fake commit
  record).

The subject-mismatch guard's behavior for a genuinely divergent HEAD with dirty state (a
real stale checkpoint) is unchanged.

### 4. Finalizer accepts a clean no-commit repair outcome

Files: `src/sase/finalizers/commit_repair.py`, `src/sase/finalizers/commit_dispatch.py`

Today, after `resolve_commit_conflict` returns, `commit_dispatch` requires a new
`commit_results.json` marker for the repo and raises `missing_commit_result` otherwise.
With change 3, a successful resume may legitimately produce no marker.

- In `resolve_commit_conflict`: when the post-repair resume exits 0 but produced no new
  marker for the repo, verify the repository is actually settled — not in conflict state
  and no dirty paths (reuse the same cleanliness query as change 3). If settled, append
  `FinalizerOutcomeEvidenceWire(kind="conflict_repair", value="resolved_without_commit")`
  plus an evidence entry carrying the repo's current HEAD SHA, and signal the no-commit
  outcome to the caller (return-value flag, small result object, or an out-parameter —
  coder's choice, but explicit; do not infer it downstream by re-checking global state).
  If the repo is not settled, raise as today.
- In `commit_dispatch` (the `if not repo_markers:` block): when the no-commit outcome
  was signaled, skip the `missing_commit_result` raise and the
  `marker_evidence`/`reconcile_commit_file_hooks` calls for that repo, and continue with
  the existing `unexpected_path_resolver`/follow-up machinery (which already handles a
  repo left dirty after repair via `_attempt_post_repair_follow_up`).

Net effect on the incident scenario: the repair turn cleans the repo, resume exits 0
with `no_commit_needed`, the finalizer records `resolved_without_commit` evidence, and
the run succeeds.

### 5. Tests

- `tests/workflows/test_commit_workflow.py` (or a new sibling test module): a dispatch
  run against a repo already in conflict state returns `CONFLICT`, does not invoke the
  provider dispatch or the before-commit hook, and saves a checkpoint with
  `no_commit_dispatched=True`.
- `tests/test_commit_workflow_resume.py` /
  `tests/test_commit_workflow_resume_recovery.py` (existing helpers in
  `tests/_commit_workflow_resume_helpers.py`):
  - resume of a `no_commit_dispatched` checkpoint with a clean repo → `OK`, checkpoint
    deleted, no tracking steps run;
  - resume of a legacy checkpoint (flag absent) with subject mismatch and clean repo →
    `OK` via the legacy arm;
  - resume of a `no_commit_dispatched` checkpoint with stageable changes → `FAILED`,
    checkpoint deleted, message says to re-run from scratch;
  - genuine subject mismatch with a dirty repo and no flag → still `FAILED` (existing
    behavior preserved).
- `tests/test_commit_dispatch_conflict_repair_followup.py` (or sibling): after a repair
  turn, resume rc 0 with no new marker and a settled repo → finalizer succeeds with
  `resolved_without_commit` evidence; same but repo still has a conflict state or dirty
  paths → still fails.

### 6. Out of scope (deliberately)

- Re-dispatching the commit from within `--resume` when the repo is dirty after a
  no-commit repair (the finalizer's existing follow-up machinery and a fresh stitch
  cover this; the resume path only needs the accurate FAILED message).
- The `chezmoi update -a --force` after-hook design in other projects' configs (that
  hook stranding the shared repo mid-rebase is the environmental trigger, but it is
  project configuration, not this repo's code).
- The unrelated warnings visible in the same artifacts
  (`referenced-by write-back failed: sequence item 0: expected str instance, bytes found`;
  quarantined agent-hood publication requests).

## Verification

- `just install` first if the workspace venv is stale, then `just check` (whole-repo
  lint gates + diff-scoped tests) before finishing. Hand long runs to a monitor per the
  two-speed verification rule.
- Confirm no new Symvision violations (new helper symbols must be referenced by the code
  paths above and by tests).
