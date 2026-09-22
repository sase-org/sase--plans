---
tier: epic
title: Self-healing agent workspace preparation
goal: 'A new SASE agent launch into a numbered ephemeral workspace never fails because
  of state an earlier run left behind (unpublishable sidecar commits, merge/rebase
  conflicts, in-progress git operations, dirty or diverged checkouts). Old state is
  rescued durably outside the workspace on a best-effort basis, the workspace is healed
  or re-created from the primary checkout, and the bead-merge bug that triggered the
  original failure is fixed at its root.

  '
phases:
- id: rescue-store
  title: Durable rescue store and non-refusing sidecar eviction
  depends_on: []
  size: medium
  description: 'rescue-store: add a best-effort rescue store outside the workspace
    (git bundles, worktree patches, manifests, retention, one notification per rescue)
    and change launch-time sidecar protection to publish once, rescue, and always
    proceed with eviction instead of raising _WorkspaceBeadEvictionRefused.'
- id: checkout-heal
  title: Self-healing checkout preparation for numbered workspaces
  depends_on:
  - rescue-store
  size: medium
  description: 'checkout-heal: give prepare_workspace an opt-in self-heal ladder for
    numbered workspaces that rescues state, aborts in-progress git operations, survives
    stash failures, replaces a conflicting sync rebase with rescue plus hard reset
    to the default branch, and verifies a clean postcondition.'
- id: reclone
  title: Last-resort workspace re-creation and non-holding setup failures
  depends_on:
  - checkout-heal
  size: medium
  description: 'reclone: when in-place healing still fails, rescue and move the numbered
    checkout aside, re-materialize it from the primary checkout, chdir into it, and
    prepare it once more (launch, retry, and linked-repo paths); release rather than
    hold workspaces whose setup ultimately failed.'
- id: core-merge
  title: Order-preserving bead event stream merge in sase-core
  depends_on: []
  size: medium
  description: 'core-merge: in the sase-core repo, stop the bead event-stream merge
    from re-sorting already-published upstream events by timestamp, and accept pure-reorder
    branches produced by the old merge so wedged clones can heal.'
- id: bead-sync
  title: Bead sync rollback and wedged-clone healing
  depends_on:
  - core-merge
  size: medium
  description: 'bead-sync: bump the sase-core pin, roll a clone back when the post-integration
    stream guard rejects an integration, abort cleanly on deadline timeouts, and mirror
    the pure-reorder tolerance in the Python stream-integrity guard.'
- id: visibility
  title: Truthful setup-failure reporting
  depends_on: []
  size: small
  description: 'visibility: make a recorded runner error beat the synthesized "Runner
    exited without recording an error" fallback in the TUI, and make runner stdout
    line-buffered so the output file interleaves stdout/stderr correctly.'
proposed_by: bbugyi200.athena.0pc
create_time: 2026-09-22 12:13:26
status: wip
bead_id: sase-16e
---

- **BEAD:** [sase-16e](https://github.com/sase-org/sase--beads/blob/main/pages/sase-16e/README.md)

# Plan: Self-healing agent workspace preparation

## Background: what failed

Two sibling research agents (`research.28.cld` and `research.28.mus`) failed during
workspace preparation, before the agent ever ran. Reconstructed from the run output
files, `~/.sase/notifications/notifications.jsonl`, and `~/.sase/bead_push_logs/`:

1. The numbered workspace's bead sidecar clone (`sase/repos/beads`) held 3–4 local bead
   commits (`chore(beads): note sase-165.x`) that had never been published.
2. `prepare_workspace` (`src/sase/axe/runner_workspace_prepare.py`) ran its tolerant
   sidecar pass: the bead-aware publication (`push_bead_work_launch`) failed, and a
   recovery ref was pinned _inside the clone_. The pass then returned "ok". Because
   `protect_workspace_bead_stores` adds a root to `unsafe_generic_roots` only when it
   returns False, the generic `protect_sidecar_repos` pass _also_ tried to publish the
   same bead clone with plain git. That produced a second recovery ref and the
   notification
   `sidecar integration failed … git rebase failed … semantic conflict resolution failed: validation: cannot merge non-append-only bead event stream sase-165: ours missing base event 39`.
3. The main checkout was cleaned and synced ("Workspace ready").
4. `prepare_launch_workspace_repos` then ran the strict pass
   (`refuse_on_unpublished=True`). It tried to publish a third time, failed, pinned a
   third in-clone ref, and raised `_WorkspaceBeadEvictionRefused`. The launch failed and
   the workspace was held.
5. The TUI showed "Runner exited without recording an error" and an output tail of
   stdout only. The real error was in `done.json` and in the notification.

**Root cause of the unpublishable commits.** The Rust bead event-stream merge
(`sase-core` `crates/sase_core/src/bead/events/merge.rs`) sorts the additions from
_both_ branches together by a timestamp-first key.

- Upstream streams are append-only but not timestamp-monotonic. Upstream appended event
  38 (`issue_closed`, 13:55) and then events 39–40 (`link_added`, 13:13 and 13:14).
- The merge therefore wrote `[..37, 39, 40, 38, note]`, which is not append-only
  relative to upstream.
- The Python post-integration guard rejected it (`HEAD missing ancestor events 39-39`)
  but left the rewritten HEAD in place (`src/sase/bead/sync_worker.py`).
- From then on, every rebase of that local commit fails with "ours missing base event
  39". The clone is permanently wedged, and every launch into that workspace is refused.

**Latent data-loss paths found along the way:**

- Recovery refs live only inside the sidecar clone, so any later deletion of the
  workspace or clone destroys them. The commits from this incident appear to be gone
  already, because the workspace directories were re-created later.
- `unpushed_bead_commit_count` fails open to 0 on git errors.
- `ensure_git_clone_at` `rmtree`s a checkout whose `git status` fails, and it does so
  without any sidecar protection.
- The main-checkout path (`run_sase_hg_clean` → `git stash push -u`, then
  `git checkout`, then `git fetch` + `git rebase origin/<default>`) has no handling for
  conflicts or in-progress operations. A failed sync rebase is never aborted. The next
  prep then fails at `git stash` (unmerged entries), so the workspace stays wedged until
  someone repairs it by hand.

## Policy (applies to every phase)

- **Launch first, rescue best-effort.** For a numbered ephemeral workspace
  (`workspace_num > 1`, the predicate `clear_workspace_repos` and `ensure_git_clone_at`
  already use), leftover state from an earlier run must never fail a new launch.
  - Try to preserve that state _outside_ the workspace, with bounded effort.
  - If preservation itself fails, warn loudly (stderr plus an inbox notification) and
    proceed anyway. Losing an old workspace's changes is preferred over failing a new
    launch. This deliberately reverses the strict refusal added in commit `d1b6f01a9`,
    and the docs must say so.
- **Ladder:** publish → rescue (bundle/patch outside the workspace) → heal in place →
  re-create the workspace from the primary checkout → only then fail.
- **Out of scope for healing:**
  - The primary checkout (`workspace_num <= 1`) and home mode keep failing closed. They
    may hold the user's own work.
  - `sase workspace open` keeps today's behavior. Self-heal is opt-in, and only the
    agent-launch and retry callers opt in.
- **Network failures are not healed.** A failing `git fetch` stays a hard failure with
  its own step name, and it never triggers re-creation.
- **One publication attempt, one rescue, one notification** per sidecar per launch.
- **No feature flag.** This is bug-fix hardening of existing launch behavior. The old
  refusal branch has no caller that needs it to stay reachable (see the flag policy in
  the `sase_flags` reference memory).
- **Rust boundary.** Workspace preparation, the git VCS mixins, and the new rescue store
  are runner-only host behavior that already lives in Python, and no other frontend has
  to match it, so they stay in this repo. The bead event merge is already core logic, so
  it is fixed in `sase-core` (core-merge), and sase moves its pin (bead-sync). If a
  later change adds a list/restore UI for rescues, that model belongs in `sase-core`.
- **Verification.** Each `sase` phase runs `just check` (not `check-full`) and follows
  the `lint_and_test` reference memory. The `sase-core` phase runs that repo's checks
  per its `AGENTS.md`. Tests must never write to the real `~/.sase`: redirect the rescue
  store and notifications to a tmp dir.
- Line numbers below are approximate (`~`). Locate code by symbol name.

## rescue-store: Durable rescue store and non-refusing sidecar eviction

### Rescue store module

Add a public module importable from both `sase.workspace_provider` and `sase.axe`
without import cycles. The suggested location is
`src/sase/workspace_provider/rescue.py`. Use lazy imports for notifications, as the
existing sidecar code does. Suggested surface (adjust names to fit the codebase):

- **`rescue_git_repo(repo_root, *, workspace_dir, workspace_num, label, reason, include_worktree) -> RescueRecord | None`**
  - Never raises. Returns `None` when there was nothing to rescue.
  - Every git call is non-interactive and time-bounded (about 60s each). Reuse
    `default_git_runner` / `non_interactive_git_env`.
  - **Local-only commits.** One `git bundle` of every local branch, every tag,
    `refs/sase/recovery/*`, a detached `HEAD`, and _every_ stash entry, not just
    `refs/stash`.
    - To capture stash entries, create temporary refs from `git stash list --format=%H`
      and delete them afterwards.
    - Exclude anything reachable from remote-tracking refs (`--not --remotes`). If there
      are no remote-tracking refs, bundle everything.
    - The error "Refusing to create empty bundle" means nothing needed rescue.
  - **Worktree** (when `include_worktree`). Write a binary patch of tracked, staged, and
    untracked-but-not-ignored changes relative to `HEAD`.
    - Build it with a temporary index (`GIT_INDEX_FILE=<tmp> git add -A`, then
      `git diff --cached --binary HEAD`), so the real index is untouched and unmerged
      entries don't break it.
    - Diff against the empty tree when `HEAD` is unborn.
    - Cap the size (for example 64 MiB). When the cap is exceeded, skip the patch and
      record a note.
  - **`manifest.json`.** Include:
    - schema version and timestamp
    - repo root, workspace dir and number, label, reason
    - branch, HEAD, upstream
    - operation markers present and `git status --porcelain` output
    - the files written
    - copy-pasteable restore commands, such as
      `git fetch <bundle> 'refs/*:refs/sase/rescued/<stamp>/*'` and
      `git apply --index <patch>`
- **`quarantine_directory(path, …) -> RescueRecord | None`**. Moves a whole directory
  into a rescue entry with `os.rename`. On `EXDEV` or any `OSError` it returns `None`.
  It is used for sidecar clones when bundling fails, and replaces the in-workspace
  `<ws>/.sase/sidecar-quarantine/` used by `_quarantine_damaged_sidecar_repo`.
- **Location.**
  `sase_projects_dir()/<project key>/rescue/<YYYYMM>/<UTC stamp>-ws<N>-<label>-<short hash>/`.
  - Derive the project key with the existing helpers that build other
    `~/.sase/projects/<key>/…` paths.
  - Fall back to a machine-level `~/.sase/rescue/` when no key resolves.
  - The location must never be inside the workspace directory.
- **Retention.** `reap_rescue_store()` deletes entries older than 30 days and
  quarantined directories older than 7 days. The disk is about 92% full, so directory
  quarantines must stay short-lived. Call it opportunistically after each rescue. It is
  best-effort and never raises.
- **Notification.** Send one inbox notification per rescue through
  `sase.notifications.notify_workflow_complete`, following the pattern in
  `_report_sidecar_eviction_failure`.
  - Sender: something like `workspace-rescue`.
  - Include what was rescued, why, the rescue dir in `extra_files`, and the restore
    hint.

### Sidecar launch policy

Files: `src/sase/axe/runner_workspace_prepare.py`,
`src/sase/axe/runner_workspace_beads.py`, `src/sase/axe/runner_workspace_sidecar.py`.

1. **`prepare_launch_workspace_repos` no longer raises for unpublished or unverifiable
   sidecar commits.** For each such sidecar:
   - Pin the in-clone recovery ref, best-effort.
   - `rescue_git_repo(..., include_worktree=True)`.
   - If the bundle fails, `quarantine_directory` the clone.
   - If that also fails, emit a loud warning and a notification saying the commits may
     be lost.
   - Proceed to `clear_workspace_repos` in every case.

   Also:
   - Delete `_WorkspaceBeadEvictionRefused` and its docstring references.
   - Replace `refuse_on_unpublished` with a mode that means "rescue, then allow
     eviction" (for example `evicting: bool`).
   - Route damaged-sidecar quarantine to the durable store.

2. **Publish once per launch.**
   - `protect_sidecar_repos` must skip _every_ root that `protect_workspace_bead_stores`
     handled, whatever the outcome. The plain-git generic path must never rebase a bead
     store.
   - The eviction pass must not re-publish a sidecar whose publication already failed at
     the same HEAD earlier in this launch. Memoize by `(repo_root, HEAD sha)`, or pass
     the tolerant pass's result through.
3. **Bounded lock wait.** `_protect_bead_store` calls `push_bead_work_launch` with the
   default `worker_lock_wait=0.0`, so a concurrently running bead sync counts as a
   publication failure. Pass a bounded wait instead (about 30s, as
   `MUTATION_PUBLICATION_WORKER_LOCK_WAIT_SECONDS` does), then fall back to rescue.
4. **Unknown is not zero.** `unpushed_bead_commit_count`
   (`src/sase/bead/_sync_diagnostics.py`) returns 0 on git errors. Give the launch path
   a variant that distinguishes "unknown" from 0, and treat unknown as
   rescue-before-evict. Keep the existing function's semantics for any other caller that
   relies on them.
5. **Docs.** Update `docs/workspace.md` (~599-607), `docs/beads.md` (~1045-1056), and
   `docs/sdd_storage.md` (~317-323): launches rescue and proceed, and here is how to
   restore from a rescue bundle.

### Tests

- **Real-git unit tests for the rescue module:**
  - Fetching the bundle into a fresh clone yields the unpublished SHAs.
  - All stash entries are included.
  - The patch captures staged, unstaged, untracked, and conflicted content.
  - Nothing to rescue → `None`.
  - No remote-tracking refs → full bundle.
  - Size cap is honored.
  - Reaper deletes only aged entries.
- **Rewrite the refusal tests in
  `tests/test_bead/test_workspace_sidecar_bead_eviction.py`** (plans commits ~:499, bead
  commits ~:551, quarantine failure ~:230). Each should now assert that eviction
  proceeds, a restorable bundle holds the commits, and exactly one notification is sent.
- **Incident regression.** Stub `push_bead_work_launch` to fail with the
  semantic-conflict error. Then `prepare_workspace` followed by
  `prepare_launch_workspace_repos` succeeds, exactly one publication attempt is made per
  store (count the calls), and exactly one rescue entry exists.
- **Generic pass skips bead roots** even when the bead pass returned True.
- **Unknown count** (a git error) leads to rescue, never a silent eviction.

## checkout-heal: Self-healing checkout preparation for numbered workspaces

Files: `src/sase/axe/runner_workspace_prepare.py` (`prepare_workspace`,
`_prepare_workspace_locked`), `src/sase/vcs_provider/plugins/_git_core_ops.py`,
`src/sase/vcs_provider/plugins/_git_sync_ops.py`. Both the bare-git provider and the
`sase-github` `GitHubPlugin` inherit these through `GitCommon` without overriding
`checkout`, `sync_workspace`, or `stash_and_clean`.

1. **Opt-in switch.** Add a keyword such as `self_heal: bool = False` to
   `prepare_workspace`. Pass `self_heal=workspace_num > 1` from:
   - `prepare_workspace_if_needed` and `prepare_linked_repo_workspaces_if_needed` in
     `src/sase/axe/run_agent_runner_setup.py` (for linked repos, use the linked repo's
     workspace number)
   - both `prepare_workspace` calls in `src/sase/axe/run_agent_exec_retry.py` (~401,
     ~450)

   Leave `src/sase/main/workspace_handler_list.py` at the default.

2. **Provider operations for the heal rungs.** Expose git-specific operations through
   the VCS provider, following the existing facade pattern: an unimplemented operation
   raises `NotImplementedError`, and prep then skips that rung, just as it does today
   for `sync_workspace`. Non-git providers keep today's behavior.
   - Share one public in-progress-operation helper instead of adding a third copy. The
     candidates are `_OPERATION_PATHS` in `src/sase/sdd/_repository_health.py` and
     `_abort_in_progress_operations` in `src/sase/workspace_provider/reset_replay.py`,
     which lacks revert, am, bisect, and sequencer handling.
3. **Self-heal ladder** (only when `self_heal`):
   1. Clear a stale `index.lock` (existing behavior).
   2. Inspect the checkout: operation markers (rebase-merge, rebase-apply/am,
      MERGE_HEAD, CHERRY_PICK_HEAD, REVERT_HEAD, bisect, sequencer), unmerged paths,
      detached HEAD, and dirty status.
   3. **Rescue first.** If the checkout is mid-operation, has unmerged paths, or has a
      detached HEAD holding commits no branch or remote contains: call
      `rescue_git_repo(..., include_worktree=True)` _before_ touching anything. That
      preserves in-progress conflict resolutions outside the workspace. A plain dirty
      worktree needs no rescue entry; the in-clone stash below stays its backup, which
      avoids notification spam.
   4. **Abort every in-progress operation**: `rebase --abort`, `am --abort`,
      `merge --abort`, `cherry-pick --abort`, `revert --abort`, `bisect reset`, and the
      sequencer `--quit`.
      - If an abort fails, use the `--quit` variant.
      - Then remove leftover _known_ marker paths from the git dir.
      - Then `git reset --hard HEAD`.
   5. **Clean.** Keep today's `git stash push --include-untracked -m <name>` backup. If
      the stash fails (for example, unmerged entries left by a stash-pop conflict with
      no marker):
      - Make sure a rescue patch exists.
      - Run `git reset --hard HEAD` and `git clean -fd`. Do not use `-x`: ignored files
        such as `.venv` must survive.
   6. **Checkout.** If `git checkout <branch>` fails, retry with `git checkout -f`. For
      the default parent, fall back to `git checkout -B <default> origin/<default>`.
   7. **Sync** (default parent only). Run `git fetch origin`. A fetch failure stays a
      hard failure with a distinct step (for example `fetch`) and is marked ineligible
      for re-creation. Then run `git rebase origin/<default>`:
      - A clean rebase keeps today's behavior: local commits are carried forward.
      - If the rebase fails: `git rebase --abort`, rescue the local-only commits (bundle
        plus recovery ref), `git reset --hard origin/<default>`, and print a warning
        naming the rescue dir.
   8. **Postcondition.** No operation markers, no unmerged paths, HEAD attached to the
      expected branch, and a clean `git status --porcelain` (ignored files allowed). A
      failure raises `WorkspacePreparationError(step="verify")`.
4. **Log every heal action** as one concise stdout line, for example
   `Aborted stale rebase in <dir>` or
   `Reset master to origin/master after rebase conflict; local commits rescued to <path>`,
   so the run log explains what happened.
5. **Classify failures for re-creation.** Extend `WorkspacePreparationError` (a field
   such as `reclone_eligible`, or a documented step taxonomy) so the reclone phase can
   tell local-state failures (eligible) from fetch, network, or primary-missing failures
   (not eligible).

### Tests

Use real git and reuse the fixtures in `tests/workspace_provider/test_reset_replay.py`
(`_init_origin`, `_clone`, `_advance_origin`, `_put_checkout_in_stale_rebase`).

- **Healed with `self_heal=True`:** each of the following ends at `origin/<default>`,
  clean, with a rescue entry where the ladder calls for one:
  - mid-rebase with conflicts
  - stash-pop conflict (unmerged paths, no marker)
  - stale `rebase-merge` directory with no conflicts
  - merge, cherry-pick, revert, and `am` in progress
  - local default branch diverged with a conflicting commit (the bundle contains it)
  - detached HEAD with an orphan commit
- **Unchanged behavior:** a non-conflicting divergence is still rebased, and a plain
  dirty worktree still produces only a stash entry.
- **`self_heal=False` keeps failing** for representative bad states.
- **Fetch failure** is a hard failure marked ineligible for re-creation.
- **Non-git provider:** the heal rungs are skipped.

## reclone: Last-resort workspace re-creation and non-holding setup failures

1. **Re-creation helper**, for example `recreate_managed_workspace(...)` near
   `ensure_workspace_checkout` in `src/sase/workspace_provider/_utils_checkout.py`, or
   in a new public module. For `workspace_num > 1` it does the following, and must
   refuse `workspace_num <= 1`:
   1. Rescue the main checkout: `rescue_git_repo(..., include_worktree=True)` for local
      branches, stashes, a detached HEAD, and the worktree.
   2. Rescue every `sase/repos/<role>` sidecar clone with the eviction-mode protection
      from rescue-store (publish or rescue, never refuse).
   3. Move the checkout aside by renaming it to a trash path next to it, in the same
      parent directory (same filesystem, O(1)). Delete it in the background the way
      `clear_workspace_repos` does (`_delete_paths_in_background` in
      `src/sase/_linked_repo_workspaces.py`).
   4. Re-materialize through `ensure_workspace_checkout(primary, N, project_file=...)`.
      That way the store path policy, `share_git_objects`, the origin rewrite, the
      registry record, the checkout marker, and the SDD clone all follow the normal
      path. Resolve the primary with `get_primary_workspace_dir`.
2. **Runner integration** in `prepare_workspace_if_needed`
   (`src/sase/axe/run_agent_runner_setup.py`). On a re-creation-eligible
   `WorkspacePreparationError` from `prepare_workspace(self_heal=True)`:
   1. Print
      `Workspace #N could not be repaired in place (<reason>); re-creating it from the primary checkout...`.
   2. Re-create the workspace.
   3. `os.chdir(workspace_dir)` again. `claim_deferred_workspace`
      (`src/sase/axe/run_agent_phases.py` ~129) already chdir'd into the old directory,
      which is now in trash. Re-apply any cwd-derived environment set there.
   4. Re-run `prepare_workspace` **once**. A second failure raises the normal
      `RuntimeError`.

   Keep the occupancy guard semantics intact: re-run `_guard_workspace_not_occupied` if
   needed.

3. **Share one helper instead of duplicating it:**
   - Apply the same fallback to both retry-prep calls in
     `src/sase/axe/run_agent_exec_retry.py`.
   - For retained linked-repo clones in `prepare_linked_repo_workspaces_if_needed`,
     rescue-then-move just that clone and re-run `materialize_linked_repo_workspace`.
4. **Corrupt-checkout path.** `ensure_git_clone_at` currently runs `shutil.rmtree` after
   `git status` fails (~306). Make it run the sidecar rescue over the doomed checkout's
   `sase/repos/*` clones first (best-effort), and use the rename-then-background-delete
   primitive. Leave the broken SASE object-borrower refusal
   (`_refuse_existing_borrower_after_status_failure`) as it is. That case is reached at
   claim time, where the claim already falls back to another workspace number.
5. **Do not hold these workspaces.** Tag final workspace-preparation failures (after
   self-heal and one re-creation) with a new exec outcome, for example
   `setup_workspace_failed`, in `launch_agent_run`
   (`src/sase/axe/run_agent_runner_launch.py` ~231). Add it to
   `_NON_HOLD_FAILURE_OUTCOMES` in `src/sase/axe/run_agent_runner_lifecycle.py`. The
   agent never ran and anything valuable was rescued, so the workspace is released
   instead of "held (visible failed run)".

### Tests

- **A heal that still fails** (force the postcondition or a rung to fail) leads to
  re-creation. Then prep succeeds, cwd is the new checkout, the rescue store holds the
  old branches and stashes, and an unpublished sidecar commit in the doomed workspace is
  bundled.
- **Fetch failure** triggers no re-creation.
- **A second failure** fails the launch. A lifecycle test shows the workspace is
  released for the new outcome.
- **Retry path and linked-repo path** use the same helper.
- **`ensure_git_clone_at` corrupt path** rescues sidecars before deleting.
- **End-to-end regression.** Build a numbered workspace that combines three problems:
  1. a bead or plans sidecar with an unpublishable local commit
  2. a main checkout mid-rebase with conflicts
  3. a diverged local default branch

  `prepare_workspace_if_needed` followed by `prepare_launch_workspace_repos` succeeds,
  one notification is sent per rescue, and every bundle restores.

## core-merge: Order-preserving bead event stream merge in sase-core

Work in the `sase-core` linked repo: open it with `sase repo open sase-core` and read
its `AGENTS.md`. The code is in `crates/sase_core/src/bead/events/merge.rs`.

- The merge (~90-160) collects both branches' additions into a `BTreeMap` and sorts them
  _together_ by `event_union_key` (timestamp first).
- `validate_append_only_branch` is at ~466-497.

1. **Order-preserving interleave.**
   - The merged output is the base events followed by an order-preserving interleave of
     the two branches' additions. Each side's additions keep their own relative order.
   - When picking between the heads of the two sides, use `event_union_key` to keep the
     result deterministic.
   - An event present on both sides appears once.
   - If both sides share additions in orders that cannot be reconciled, prefer the order
     of the side that represents published upstream. Confirm which argument that is from
     the Python caller: `src/sase/bead/conflict_resolver.py` passes
     `(base, local, upstream)`, and `conflict_resolver_git.py` maps rebase stages. Then
     document it on the function.
   - **Invariant:** the result always contains base and the upstream side as exact
     ordered subsequences, and also contains the local side whenever such an order
     exists.
2. **Pure-reorder tolerance.**
   - Today `validate_append_only_branch` rejects a branch that contains every base event
     exactly once but in a different order. Streams written by the old merge look
     exactly like this.
   - Accept that case and canonicalize it to base order plus that branch's genuine
     additions.
   - Keep rejecting base events that are really missing, rewritten, or duplicated.
   - Make sure relocation (`split_duplicate_creations`) still works.
3. **Tests.** Add Rust unit tests plus parity fixtures under
   `crates/sase_core/tests/fixtures/bead/` (`tests/bead_event_parity.rs`):
   - **Incident shape:** base `[..e37]`; upstream appends e38 (t=13:55), e39 (t=13:13),
     e40 (t=13:14); local appends a later note. The merge must yield
     `[..e37, e38, e39, e40, note]`, and it must pass
     `validate_append_only_branch(upstream, merged)`.
   - Pure-reorder acceptance.
   - Missing and rewritten events are still rejected.
   - Existing determinism properties are preserved.
4. **No wire or API change is expected** (behavior only). If one turns out to be needed,
   update the binding and its tests in the same repo.

## bead-sync: Bead sync rollback and wedged-clone healing

1. **Pin bump.** Move `sase-core-revision.txt` to the pushed sase-core commit from
   core-merge. Use `just ratchet-core-revision` when that commit is the remote HEAD;
   otherwise write the SHA. See "The CI source revision pin" in `docs/rust_backend.md`.
2. **Roll back when the post-guard rejects an integration.**
   - In `src/sase/bead/sync_worker.py`, the push loop runs `integrate_sdd_repository`
     and then the post-guard `refuse_unpublished_event_stream_shrink` (~184-192). When
     the guard fails, it returns `_failure` and leaves the rewritten HEAD in place,
     which wedges the clone.
   - Record the pre-integration HEAD. On a post-guard failure, restore it with
     `reset --hard` under the store write lock, mirroring `_abort_and_verify` in
     `src/sase/sdd/_repository_integration.py`.
   - Pin a recovery ref for the rejected integrated HEAD, and log both in the sync log.
3. **Deadline timeouts.** `_git_runner_for_deadline` (~334-377) lets
   `SddGitCommandTimeout` escape mid-rebase with no abort. Make the integration
   transaction abort and restore on timeout.
4. **Python pure-reorder tolerance.** Apply core-merge's rule in the stream-integrity
   guard: `src/sase/bead/_stream_integrity_analysis.py` (~147-190), used by
   `refuse_unpublished_event_stream_shrink` in `src/sase/bead/_stream_integrity.py`.
   - When every ancestor event is present and only the order differs, that is not a
     shrink.
   - Keep refusing genuinely missing or rewritten events.
   - This lets clones already wedged by the old merge integrate and publish their
     pending notes.
5. **Tests** (real git, `tests/test_bead/`):
   - Non-monotonic upstream timestamps plus a local note → sync publishes (needs the new
     core).
   - A crafted wedged clone (a local commit whose stream reorders ancestor events) heals
     and publishes.
   - A forced post-guard failure restores the starting HEAD exactly, with no rebase
     markers.
   - A deadline timeout mid-rebase leaves the clone restored.

## visibility: Truthful setup-failure reporting

Read the `tui` reference memory before changing TUI code.

1. **Recorded errors win in the TUI.**
   - `dedup_running_vs_workflow` (`src/sase/ace/tui/models/_dedup.py` ~414) copies the
     `done.json` row's error only when the WORKFLOW row has none. But the WORKFLOW row
     already carries the synthesized `_FAILURE_EXPLANATION`
     (`src/sase/ace/tui/models/_loaders/_workflow_failure_fallback.py`), so the real
     error (`_WorkspaceBeadEvictionRefused: …`) was dropped.
   - Make a recorded error beat the synthesized fallback. Mark the fallback as synthetic
     explicitly (a model flag, say) rather than string-matching.
   - Check the snapshot loader path (`_workflow_snapshot_loaders.py` ~117-163) too. If
     this merge is mirrored in a sase-core agent-scan path, fix it there as well.
   - Tests: extend `tests/test_agent_loader_dedup_merge.py` and
     `tests/test_workflow_failure_fallback.py`.
2. **Line-buffered runner stdout.**
   - The runner's stdout and stderr share one output file (Rust
     `spawn_prepared_detached_process`). stdout is block-buffered, so every stderr
     diagnostic lands before the stdout lines.
   - As a result, the TUI's "Last output" tail shows only stdout, and the late stdout
     flush overwrites the finalizer's appended `AGENT_RUN_COMPLETE` marker.
   - Make runner stdout line-buffered at entry (for example
     `sys.stdout.reconfigure(line_buffering=True)` early in
     `src/sase/axe/run_agent_runner.py::main`).
   - Flush stdout and stderr before the finalizer appends its marker
     (`src/sase/axe/run_agent_runner_lifecycle.py` ~340-346).
   - Add a subprocess test showing that interleaved stdout/stderr lines keep their order
     and the marker is last.

## Non-goals and follow-ups

- **No healing** for the primary checkout, home mode, or `sase workspace open`.
- **No re-creation on network/fetch failures.** The existing retry machinery owns
  transient failures.
- **No `sase` CLI to list or restore rescues.** The manifest carries restore commands.
  If the rescue-store worker thinks one is warranted, it should file a task bead via
  `/sase_new_task`.
- **No changes to the broken SASE object-borrower refusal at claim time.**
