---
tier: tale
title: Thin verified snapshots so dirty large checkouts can be preserved
goal:
  A verified checkout snapshot stores only the bytes that are not already in published
  remote history. A small dirty diff in a large repository (for example a nested sase
  clone) is then preserved in seconds, and launch eviction, self-heal, reclone, reset
  replay, and failed-close evidence stop refusing it. Unpublished work over the cap
  still fails closed.
size: medium
decisions:
  decision_record:
    ask:
      Add a decisions-web record saying verified snapshots treat published remote
      history as durable base?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.159
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-28 12:46:39
status: wip
---

# Thin verified snapshots so dirty large checkouts can be preserved

## Why: the bob-cli-do.4 failure and its siblings

`bob-cli-do.4` (project `gh_bobs-org__bob-cli`, 2026-10-28) waited 53 minutes in the
queue. It then failed 5 seconds into workspace preparation:

```
WorkspacePreparationError: ... during sidecar-protection: verified nested preservation
failed before launch eviction: could not preserve nested repository
<workspace>/sase/repos/external/projects/sase: snapshot payload
src/sase/ace/tui/actions/agents/_notification_hitl_modal.py is 9815 bytes; total
67116886 exceeds the 67108864-byte cap
```

The nested external `sase` clone had only a 13-line uncommitted diff in three files. The
diff was left by `bob-cli-df.land--code`. The same error failed four more bob-cli
launches that day: `bob-cli-dp.1` in the same workspace, plus three runs at 07:13,
09:15, and 09:18 in another workspace. The cascade, where a refusing workspace stays
assignable, is filed separately as `sase-1p9` and is out of scope here.

Root cause: `snapshot_git_repo` (added by the bob-cli-df.4.1 epic, commits `42512dc116`,
`55ebb614f2`, `c175d66667`, split in `bb9115b4e0`) builds a fully self-contained pack.
Its cost and its byte cap scale with repository size, not with dirty work. In
`src/sase/workspace_provider/_snapshot_capture_payload.py`:

- `_worktree_entries` reads and `git hash-object`s every tracked file plus every
  untracked file, one subprocess per file. It also charges every file's bytes to
  `SNAPSHOT_MAX_BYTES` (64 MiB, shared with `RESCUE_PATCH_MAX_BYTES`).
- `_copy_index_blobs` does a `cat-file` plus a `hash-object` for every index entry. It
  charges those bytes again, then discards the returned total.
- `_object_shas` packs `rev-list --objects --no-walk HEAD`, which is HEAD's entire tree.
  `_write_pack` then caps the pack itself at 64 MiB.

Your own sase checkout shows the scale. It has about 15k tracked files, about 316 MB of
tracked bytes, and a full-HEAD pack of about 211 MB. A dirty sase checkout therefore can
never be snapshotted. Because preservation correctly fails closed, every guarded
destructive path refuses permanently:

- launch eviction of nested clones (`prepare_launch_workspace_repos`)
- preparation clean and self-heal (`runner_workspace_heal.py`,
  `workflows/commit_utils/workspace.py`)
- reclone (`_utils_checkout.py`)
- reset replay (`reset_replay.py`)
- failed-close evidence (`finalizers/_failed_close_recovery_snapshots.py`)

The older best-effort rescue already uses the right boundary. `rescue_git_repo` bundles
only `--not --remotes` commits and patches the worktree against HEAD. Verified snapshots
should use the same durability boundary: objects reachable from a remote-tracking ref
are durable base, and the snapshot stores, caps, and verifies only what lies beyond it.

## Design

The snapshot pack holds only objects that are not recoverable from published history.
The manifest records that base so restore can fetch it.

**Base boundary.** Use every remote-tracking ref (`refs/remotes/*`), matching
`_rescue_git.py`'s `--not --remotes`. Define C as follows:

- When at least one remote-tracking ref exists, C is
  `git rev-list --objects HEAD --not --remotes`. C is empty when HEAD is published, and
  holds only the local-only commits and their new trees and blobs when HEAD is ahead.
  Boundary commits come from `git rev-list --boundary HEAD --not --remotes` (`-` lines),
  or are HEAD itself when C is empty.
- When there are no remote-tracking refs, keep today's self-contained behavior: C is
  `rev-list --objects --no-walk HEAD`, and the manifest records no external base.

**Tree thinning must be explicit.** Do not rely on
`git rev-list --objects <tree> --not --remotes`. On git 2.47.3, with no interesting
commit, it still lists every object of the tree. Compute H as the object ids of HEAD's
tree (`git ls-tree -r -t -z --full-tree HEAD` plus `HEAD^{tree}`). Compute T as the
objects of the worktree tree and the index tree (the trees themselves, all subtrees, and
blobs), plus every unmerged stage blob. Then the pack set is:

```
P = C ∪ (T − H) ∪ {worktree_tree, index_tree}
```

Every object in H is either in C or in a boundary commit's tree. Exclude gitlink
(`160000`) SHAs from every set. They are another repository's commits, and today they
would make `pack-objects` fail.

**Hash only dirty paths.** Build the worktree tree from the real index's stage-0
entries. Override only the candidate paths:

- every path in `status_paths(state.status_z)` (minus `is_managed_sidecar_rel`)
- every path in an unmerged index entry
- every index entry flagged assume-unchanged or skip-worktree (from `git ls-files -v`),
  because status cannot vouch for those

For each candidate, read and hash the live bytes into the snapshot object directory, or
drop the entry when the file is absent. Every other path keeps its index mode and SHA.
Keep the existing nested-repository refusal (`_refuse_nested_repositories`) and the
`sase/repos` exclusion. Capture should spawn a number of git processes proportional to
the dirty paths plus a constant, not to the tracked files.

**Integrity and the cap.** Replace `_copy_index_blobs` with one
`git cat-file --batch-check` over every non-gitlink index SHA. A missing object still
raises `PreservationError`, because corrupt sources still fail closed. Charge
`SNAPSHOT_MAX_BYTES` only for objects in P: one `--batch-check` of P with the snapshot
alternates env, summing object sizes, with each object and the total under the cap. Keep
the existing pack-size cap. Keep the `_account_bytes` message shape: it names the label
and sizes, and tests match on "cap". The 64 MiB value does not change. Unpublished work
over it must still refuse, and no cap may turn into permission to reset.

**Verification.** `_verify_pack` keeps unpacking into a throwaway repository. It then
checks that every id in P is present, not a fixed list that includes HEAD and the base
tree. When the base is published, capture records the evidence it decided on: the
boundary commits and the remote-tracking refs and SHAs used. The existing before/after
fingerprint race check stays as is.

**Manifest.** Add a `base` object that records:

- the mode (`published` or `self-contained`)
- the boundary commit SHAs
- each remote's name and fetch URL, with local-path URLs made absolute against the repo
- the remote-tracking refs and SHAs used

Update the `restore` instructions to start with "fetch the base commits from the
recorded remote". A manifest without `base` is a legacy self-contained snapshot and must
keep restoring exactly as today. Do not change `RESCUE_SCHEMA_VERSION` unless the rescue
store's readers require it.

**Restore.** `restore_git_snapshot(manifest, dest)` keeps its signature. For a
`published` base it works in this order:

1. Init `dest` if needed.
2. Check each boundary commit (`git cat-file -e <sha>^{commit}`). For any that are
   missing, fetch them from each recorded remote: first by SHA
   (`git fetch --no-tags <url> <sha>...`), then fall back to all branches into a private
   `refs/sase/snapshot-base/<snapshot_id>/*` namespace.
3. If any boundary commit is still missing, raise `PreservationError` naming the missing
   commits and the URLs tried, before clearing or writing `dest`'s worktree.
4. Unpack the pack and continue as today.

Both fetch forms are confirmed to work against a local bare remote. Update the module
docstrings in `snapshot.py` and `_snapshot_restore.py`. Today they say restore "uses
only objects recorded in the snapshot pack". The new text should say a published-base
snapshot depends on the recorded remote retaining its base commits. Like the legacy
rescue bundle, a later remote force-push or branch deletion can strand the base, and
restore then fails closed instead of guessing.

## Tests

Add these beside `tests/workspace_provider/test_rescue/test_snapshot.py` and
`test_nested_preservation.py`, using the existing `seed_remote_clone`, `redirect_state`,
and probe helpers.

1. Large published repo, small diff (the direct regression). Commit and push a tracked
   file larger than a monkeypatched `SNAPSHOT_MAX_BYTES`, for example a 256 KiB file
   under a 64 KiB cap. Modify a different small tracked file, stage a third, and add an
   untracked file. The snapshot must succeed with `base.mode == "published"`, its pack
   must be far under the cap, and the source must be unchanged. Restore into an empty
   directory (which fetches from the bare remote) and into a fresh clone of the remote.
   Compare exact worktree bytes and `ls-files --stage` with the source.
2. Local-only commits plus dirty edits. The pack carries the unpushed commits, the
   boundary equals the pushed tip, and restore into an empty directory reproduces HEAD,
   the worktree, and the index.
3. No remote at all. The manifest is self-contained and restore needs no fetch. Also
   strip `base` from such a manifest and prove the legacy path still restores.
4. Base unobtainable. Move the bare remote away before restoring a published-base
   snapshot. Expect a `PreservationError` naming the missing commit and URL, with no
   worktree written.
5. Process-count guard. In a repo with a few hundred clean tracked files and one dirty
   file, wrap `run_git_bytes` and assert the number of `hash-object` invocations is a
   small constant.
6. Nested launch eviction. A parent workspace has an external clone under
   `sase/repos/external/...` that holds a pushed file over the monkeypatched cap and a
   small diff. `preserve_nested_repositories` must succeed and return a locator that
   restores the diff. Exercise `prepare_launch_workspace_repos` too if the existing
   suites already drive it.
7. Keep the fail-closed guarantees. `test_snapshot_size_cap_fails_closed` (200 dirty
   bytes against a 16-byte cap) and the corrupt or missing-object, source-change,
   conflict-stage, binary, deletion, tracked-ignored, and manifest-failure tests must
   pass unchanged.

Every existing restore-based test in these suites must keep passing:
`tests/workspace_provider/test_rescue/`,
`tests/workspace_provider/test_reset_replay.py`,
`tests/test_axe_runner_workspace_self_heal.py`,
`tests/test_axe_runner_workspace_reclone.py`,
`tests/test_finalizers_failed_close_recovery*.py`, and
`tests/test_plan_archive_approval_recovery.py`. Most of them push to a local bare
remote, so their snapshots become published-base and their restores fetch. If one fails
only because its fixture removed the remote before restoring, fix the fixture. Never
weaken a restored-bytes assertion.

## Docs

Read `docs_writing.md` with `/sase_memory_read` first. Then correct
`docs/reference/agents/workspaces.md`, which predates strict nested preservation. Both
the summary near the top ("a failed rescue does not stop eviction") and the
workspace-maintenance paragraph ("Leftover state from an earlier run never fails a new
launch", "Rescue is best-effort", "Eviction always proceeds") are now false for dirty
content. State the current contract briefly:

- Before eviction or re-creation, every nested `sase/repos` clone and the main checkout
  get a verified snapshot.
- The snapshot stores only work not already on a remote.
- Unpreservable state fails preparation (`sidecar-protection`) and leaves the clone
  intact. That covers unpublished bytes over 64 MiB, corrupt objects, and dirty
  submodules.
- The legacy bundle-and-patch rescue still runs afterward.
- A restore fetches the recorded base first.

Stay within the page's budget.

> [!decision] decision_record Add one decisions-web strand through `/sase_memory_write`,
> for example `decisions/snapshots-anchor-at-published-history`. Its claim: a verified
> snapshot stores and caps only objects not reachable from remote-tracking refs.
> Alternatives rejected: a self-contained full-tree pack, which is about 211 MB per sase
> snapshot and un-snapshottable under the cap, and raising the cap. Cost: restore needs
> the recorded remote to keep its base commits. Reopen it if base commits are found
> stranded in practice. Then run `sase memory init`.

## Constraints and non-goals

- This is Python Git and filesystem orchestration in `workspace_provider`. No
  `sase-core` change is expected.
- Never mutate the source index, worktree, HEAD, refs, or operation state during
  capture. Keep writing new objects only into the snapshot's own object directory via
  `GIT_OBJECT_DIRECTORY` plus alternates.
- Do not change the callers' fail-closed or `reclone_eligible=False` semantics. Leave
  workspace-avoidance after a refusal to `sase-1p9`.
- `sase-1p5` is a separate, already-filed red node,
  `test_artifact_directory_operation_sites_are_reviewed`. It lists five unreviewed
  filesystem-operation sites in these `_snapshot_*` modules. Do not fix it here. Add no
  new artifact-directory operation sites, and if you rename or move one of the five,
  note the new name on `sase-1p5` with `sase bead note`.
- Restore materialization (one `cat-file` per blob) is left as is.

## Verification

Use the repo's isolated `.venv` (`just install-venv` if it is missing). Run the focused
suites above with `.venv/bin/python -m pytest`, then `sase tool run check` as described
in `lint_and_test.md`. Do not run `check-full`. As a manual probe, take a disposable
`git clone --no-local` of a sase checkout and leave a three-file diff in it. One
snapshot must finish in a few seconds with a pack of a few KB, and restoring it into an
empty directory must reproduce the diff.
