---
tier: tale
title: Finalize the first commit into an empty remote
goal:
  An agent's first commit into a brand-new, empty repository is committed and pushed by
  the host finalizer. A failure there reports its real cause, never "dirty work
  vanished". An empty, unborn nested clone never quarantines a workspace slot.
size: medium
proposed_by: bbugyi200.athena.1d2
proposed_with: claude/opus
create_time: 2026-11-16 15:41:02
status: wip
---

# Finalize the first commit into an empty remote, and stop the misleading failures around it

## Why

On 2026-11-16, epic phase `tailnet-x.1` on apollo bootstrapped a brand-new, still-empty
GitHub repo (`bbugyi200/nut`). The agent wrote a 17-file Rust skeleton into its
`sase repo open nut` checkout (`sase/repos/external/projects/nut`, an empty clone whose
HEAD was unborn) and declared a commit. The host commit finalizer then failed with:

> Commit finalizer failed: dirty work vanished without an attributable commit … nut …
> (HEAD did not advance) HEAD: 3db10f5 -> 3db10f5

That message was wrong: `3db10f5` was the finalizer's own new root commit. When the
agent was relaunched, the launch failed with
`WorkspacePreparationError … workspace #10 … is quarantined: nested preservation refused for …/sase/repos/external/gh/bbugyi200/nut … git rev-parse --verify HEAD failed`.
That path was a second, untouched empty clone the agent had opened with
`sase repo open gh:bbugyi200/nut`. The failed `tailnet-x.1` blocks the whole epic, and
the unpublished commit was lost when the workspace was removed by hand.

There are four independent defects. Each one alone turns "first commit to an empty repo"
into a failure or a misdiagnosis.

## Root causes

1. **Commit dispatch assumes the remote has a default branch.** `vcs_create_commit` in
   `src/sase/vcs_provider/plugins/_git_commit_dispatch.py` runs `git commit`, which
   succeeds and creates a root commit on an unborn branch. It then calls
   `_sync_with_origin_after_commit`, which runs `git fetch origin <default> --quiet`. On
   a remote with no branches, that exits 128 with `fatal: couldn't find remote ref main`
   (reproduced with a bare `git init --bare -b main` remote). The dispatch returns
   failure and leaves the commit local and unpushed.
   `_push_current_branch_with_rebase_retry` has the same fetch in its retry loop.
   `_get_default_branch` (`_git_core_ops.py`) falls back to `"main"` when `origin/HEAD`
   is missing.
2. **The unpushed-commit marker is gated on the error text.** On dispatch failure,
   `CommitWorkflow.run` (`src/sase/workflows/commit/workflow.py`) calls
   `_record_unpushed_commit_marker_if_present`
   (`src/sase/workflows/commit/_workflow_tracking.py`). That function records the
   `pushed=false` marker only when `classify_dispatch_failure(...) == "push_failed"`,
   which is a substring match on "push". The fetch error classified as `other`, so no
   marker was written, the checkpoint was deleted, and nothing tied `3db10f5` to the
   run. On the provider side, only the push-failure path goes through
   `_format_unpushed_commit_failure`/`_record_unpushed_commit_marker`.
3. **An unborn HEAD at declaration time is indistinguishable from "head not recorded".**
   `host_repository_records` (`src/sase/finalizers/declaration_store.py`) stores
   `head=None` when `git_head_commit_id` returns `UNKNOWN_HEAD_SENTINEL`, which includes
   an unborn HEAD. `already_clean_fingerprint_before`
   (`src/sase/finalizers/commit_transient.py`) treats `None` as a legacy record and
   substitutes the _current_ HEAD. On the finalizer's next pass the repo was already
   clean, so before == after. `discarded_dirty_work_evidence`
   (`src/sase/llm_provider/commit_finalizer_git_progress.py`) then reported
   `head_not_advanced`. That blamed the host's own commit and hid the real fetch error.
4. **Launch-time nested preservation cannot snapshot an unborn repo.**
   `read_source_state` in `src/sase/workspace_provider/_snapshot_capture_source.py`
   raises `PreservationError` when `git rev-parse --verify HEAD` fails. An empty clone
   of an empty remote has an unborn HEAD and no work at all, yet
   `preserve_nested_repositories` raises `NestedPreservationError`.
   `prepare_launch_workspace_repos` (`src/sase/axe/runner_workspace_protection.py`) then
   quarantines the workspace slot and fails the launch, with "Dirty paths: (none
   listed)".

## Changes

### 1. Publish the first commit into an empty remote

In `src/sase/vcs_provider/plugins/_git_commit_dispatch.py`:

- Add a helper such as `_remote_is_empty(cwd)`. It runs `git ls-remote --heads origin`
  with the same bounded timeout as `_remote_branch_exists` and returns `True` only when
  the command succeeds and lists no heads. A failed `ls-remote` returns `False`, so
  today's behavior stays.
- `_sync_with_origin_after_commit`: when the remote is empty, there is nothing to rebase
  onto. Return `(True, None)` before the fetch, the rebase, and the rebase-gate capture.
  Apply this only to a truly empty remote. A non-empty remote that lacks the detected
  default branch keeps today's failure.
- `_push_current_branch_with_rebase_retry`: when the remote is empty, push with
  `-u origin <branch>` so the new branch gets an upstream. Later `@{upstream}`-based
  checks, such as `git_upstream_ahead_count`, then see it.
- Audit the other paths a commit finalizer or `sase stitch create --resume` can reach
  for the same fetch-first assumption: `vcs_finalize_commit`,
  `vcs_create_pull_request`'s push, and the resume path. Apply the same empty-remote
  guard where they can hit an empty remote. Do not broaden to unrelated sync commands.
- Confirm the pre-dispatch steps tolerate an unborn local HEAD and degrade instead of
  failing. These are `capture_pre_commit_diff` (which may run `git diff HEAD`),
  `resolve_head_commit_sha`, `_changed_files_repo_wide`, and `_validate_staged`. The
  incident shows `git commit` itself works.

### 2. Never lose track of a local commit when dispatch fails

- In `_record_unpushed_commit_marker_if_present`, replace the
  `classify_dispatch_failure(...) != "push_failed"` gate with a check on state: HEAD
  resolves, and it differs from the pre-dispatch revision. Conflict and in-progress-sync
  states are already handled earlier in `run()` and return `CONFLICT`, so they never
  reach this path. Keep it that way.
- Unborn pre-dispatch HEAD: add a field such as `pre_dispatch_unborn: bool = False` to
  `CommitCheckpoint` and set it only when the repo is proven unborn before dispatch.
  Proven unborn means `git symbolic-ref -q HEAD` succeeds and
  `git rev-parse --verify -q HEAD` fails. Persisted checkpoints without the field must
  still load. Treat a resolvable post-dispatch HEAD as a new local commit when
  `pre_dispatch_unborn` is set. Never infer a new commit from a `None` revision that
  came from a resolution error.
- Provider side: in `vcs_create_commit`, a `_sync_with_origin_after_commit` failure that
  is not a rebase conflict must go through the marker-recording path. Return an accurate
  message such as `commit <sha> created locally; git fetch origin main failed: …`, not
  the bare fetch error. Leave the rebase-conflict and rebase-gate returns as they are.
- The workflow's existing "created local commit … but failed before publishing it"
  message and `--resume` guidance then also cover sync failures.

### 3. Record an unborn HEAD explicitly in declaration host records

- Add `head_unborn: bool = False` to `HostRepositoryRecord`. In
  `host_repository_records`, set it only when HEAD is proven unborn, using the same test
  as change 2. A HEAD that cannot be read for any other reason keeps
  `head=None, head_unborn=False`.
- Write it in `write_host_repository_file` and read it in `read_host_repository_file`.
  On read, treat it as optional: absent means `False`, and a non-boolean value is a
  malformed record. Keep `schema_version: 1`. The reader validates only known keys, so
  older readers are unaffected.
- In `already_clean_fingerprint_before`, use `UNKNOWN_HEAD_SENTINEL` as the before-HEAD
  when `head_unborn` is set. `discarded_dirty_work_evidence` then takes the attribution
  path: `_new_commit_records` already logs from `after_head` alone when the before-HEAD
  is the sentinel. Keep the legacy `None` → current-HEAD fallback only for records that
  lack the flag.
- Verify that the attribution path accepts the finalizer's own root commit, via the
  `SASE_AGENT`/tracked commit tags, the run ledger, or the `pushed=false` marker from
  change 2. A finalizer retry after a failed push must converge or report the real
  publication failure, never "dirty work vanished".
- Audit other consumers of a `None` head, for example
  `_failed_close_recovery_assess.py`. Change them only if they confuse "unborn" with
  "unknown" in the same way.

### 4. An empty unborn nested clone must not quarantine a workspace

In `src/sase/workspace_provider/_snapshot_capture_source.py`, detect an unborn HEAD
before raising in `read_source_state`, or in `_snapshot_git_repo` before it calls
`read_source_state`.

- **Proven no-work:** `snapshot_git_repo` returns `None` when all of these hold:
  - HEAD is unborn.
  - The repo has no refs at all (`git for-each-ref` is empty). This catches orphan
    checkouts whose other branches hold commits.
  - There is no stash.
  - The index is empty (`git ls-files --stage` lists nothing).
  - There are no untracked, non-ignored files.

  `preserve_nested_repositories` then skips the repo and launch eviction proceeds.

- **Any other unborn state** keeps failing closed, because it is real work that cannot
  yet be preserved. The error must name the unborn state and the nested path, not the
  raw `fatal: Needed a single revision`.
- Keep `snapshot_source_fingerprint` consistent with the no-work decision. It must not
  raise for a proven-empty unborn repo if any caller fingerprints one.

## Tests

- **Git dispatch** (extend the existing tests that cover `_git_commit_dispatch.py`). Use
  a real-git fixture: a bare empty remote (`git init --bare -b main`), a clone of it,
  and a new file. `vcs_create_commit` must succeed, the remote's `main` must equal the
  local HEAD, and the upstream must be set. Add a negative case: a non-empty remote that
  lacks the detected default branch keeps today's failure.
- **Commit workflow:**
  - A dispatch failure after a local commit with an `other`-class error writes the
    `pushed=false` marker and keeps the checkpoint.
  - A proven-unborn pre-dispatch HEAD plus a new commit writes the marker.
  - A `None` pre-dispatch revision without the unborn proof writes no marker.
- **Finalizer** (extend `tests/test_finalizers_discard_guard_before_head.py`):
  - A declaration snapshotted with an unborn HEAD, then a root commit by the host, then
    an already-clean pass, yields no `dirty_work_discarded`.
  - A legacy record without `head_unborn` keeps the old fallback.
  - The host repository file round-trips with and without the field.
- **Nested preservation** (alongside `tests/workspace_provider/test_rescue/`):
  - An empty unborn clone makes `snapshot_git_repo` return `None`, and
    `preserve_nested_repositories` succeeds.
  - An unborn clone with a staged or untracked file still raises, with the new message.
  - An unborn HEAD where another local branch holds commits still raises.
- If it is cheap, add one regression test closest to the incident: the finalizer commits
  a dirty empty-remote clone end to end and leaves no discarded-work diagnostic.

## Verification

Run `just install-venv` if the venv is stale, then `sase tool run check`. Do not run
`just check-full`.

## Out of scope

- The approved `sase project create` gate that failed with a `gate.answer` vs
  `project.create` operation-sidecar mismatch is already fixed on master by `a79f27e1dd`
  ("stop gate commands inheriting the gate.answer proc sidecar"). Do not touch gates.
- After launch eviction, a clean external clone that holds unpublished local commits on
  a branch with no remote counterpart may be removed without rescue. Check whether the
  existing rescue path in `src/sase/workspace_provider/_rescue_git.py` and the runner
  sidecar rescue covers that case. If it does not, file a task bead through
  `/sase_new_task` rather than widening this plan.
- No `sase-core` change is expected: every touched behavior lives in Python in this
  repo. If the implementation finds a Rust-owned piece, follow the core-boundary rule
  instead of duplicating logic.
