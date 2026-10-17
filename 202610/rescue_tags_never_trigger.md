---
tier: tale
title: Stop tags from triggering workspace rescues
goal:
  Clean linked and external clones whose only extra refs are upstream tags no longer
  produce rescue entries or "Rescued leftover state" toasts, while real local work is
  still rescued.
size: medium
decisions:
  detached_tag_checkout:
    ask:
      Should a detached HEAD resting on a tag (git checkout <tag>) stop counting as
      local work?
    default: true
    why:
      Agents check out old tags to research versions; same false-positive class as tag
      bundles
    answer: true
proposed_by: bbugyi200.athena.115
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-17 15:24:50
status: wip
---

# Stop Tags From Triggering Workspace Rescues

## Problem

Users keep getting toasts like:

> Rescued leftover state from external-gh-gastownhall-gascity before workspace healing:
> local commits (local-commits.bundle).

They are false positives. Launch-time eviction (`rescue_repo_clones_before_eviction` in
`src/sase/axe/runner_workspace_protection.py`) calls `rescue_git_repo(...)`
(`src/sase/workspace_provider/rescue.py`) for every linked and external clone under
`sase/repos/` in a numbered workspace before each new launch. `_bundle_refs()` collects
every `refs/heads/*`, `refs/tags/*`, and `refs/sase/recovery/*` ref. `_published_tags()`
drops tags whose commit is reachable from a remote-tracking ref (the fix from 8581d2313f
/ sase-1km). After that, any remaining ref makes `git bundle create ... --not --remotes`
produce a non-empty bundle, which writes a rescue entry and sends a notification.

Third-party repos commonly carry upstream tags whose commit is on no remote branch, such
as release tags cut on since-deleted release branches or on version-bump commits made
off the default branch. `git clone` fetches those tags, but their commits are not
reachable from `refs/remotes/*`, so they pass `_published_tags()` and get bundled. Each
launch in a workspace that has such a clone repeats this.

Evidence from 2026-10-17:

- athena: all 8 `~/.sase/rescue/202610/*-external-gh-*` bundles list only `refs/tags/*`.
  Examples are tailscale (`v1.88.0`, ...), networkupstools/nut (6 tags, 3.7 MB),
  rsms/inter (3.0 MB), and astral-sh/uv. Every manifest shows `head == upstream`, an
  empty `status_porcelain`, and 0 stashes.
- apollo: all 25 recent `external-gh-*` bundles (mostly gastownhall/gascity and gastown,
  several per day) contain zero non-tag refs.
- A scratch repro behaves the same way. Clone a remote whose only extra tag `v1.0`
  points at a commit on no branch, stay on a clean `main`, then call `rescue_git_repo`.
  It returns a record, notifies, and the bundle lists only `refs/tags/v1.0`. A detached
  `git checkout v1.0` triggers the same false rescue through the detached-HEAD temp ref.

Real rescues still work and must keep working. For example, the linked bob-mac-capture
and bob-plugins entries carried genuine worktree patches.

Git keeps no offline record of which tags a remote has, and rescue must stay offline
(see the `_published_tags` docstring), so an upstream off-branch tag cannot be told
apart from a locally created tag. SASE itself never creates tags in workspace clones,
and agents never commit (completion is host-owned), so in practice a tag that is the
only anchor of a commit is upstream.

## Design

**Tags never trigger a rescue on their own.** They only ride along as labels for commits
that are already being rescued for another reason.

1. **Work anchors** are the refs that can hold this clone's own work: `refs/heads/*`,
   `refs/sase/recovery/*`, the detached-HEAD temp ref, and the stash temp refs.
   `_bundle_refs()` returns anchors and tags separately. A small dataclass or tuple is
   fine; keep it private.
2. **Rescue trigger.** A bundle is needed only when the anchors carry a commit that is
   not reachable from remote-tracking refs: a non-empty
   `git rev-list -1 <anchors> --not --remotes`. With no remote-tracking refs, any anchor
   counts, as today. When that check is empty, write no bundle at all, even if
   unpublished tags remain.
3. **Tags in a bundle.** When a bundle is written, include only tags that label rescued
   work: tags merged into at least one kept anchor
   (`git for-each-ref --merged=<anchor> ... refs/tags`, union semantics like
   `_published_tags`), minus `_published_tags()`. With no remote-tracking refs, keep
   today's behavior: bundle every tag. Upstream off-branch tags therefore no longer
   bloat bundles that are written for a real unpushed branch.
4. **Detached HEAD on a tag** (decision `detached_tag_checkout`). Before treating the
   detached HEAD as an anchor, drop it when
   `git rev-list -1 HEAD --not --remotes --tags` is empty, because HEAD is resting on,
   or behind, a tag checkout.

> [!decision] detached_tag_checkout = no Skip step 4: keep the detached-HEAD temp ref as
> an unconditional anchor and omit the detached-tag-checkout test below.

5. **Worktree-only rescues** (dirty tree or stashes with all anchors published) still
   write the patch and stash bundle as today. They no longer add a tag-only
   `local-commits.bundle`, so the toast no longer claims "local commits" when only
   worktree changes were saved.
6. Keep every existing guarantee: never raise, temp refs under `refs/sase/rescue-tmp/`
   are always deleted (including on early return), the manifest and restore commands are
   unchanged, and nothing is written to the store when there is nothing to rescue. The
   early `return None` must happen before `_new_rescue_dir()` so no empty entry
   directory is created. The current post-bundle `rmtree` fallback can stay as a safety
   net.

**Accepted trade-off.** Document it in the `_published_tags` / `_bundle_refs` docstrings
and the docs. A tag created locally on a commit that nothing else references (no branch,
stash, recovery ref, or detached HEAD ahead of every tag) is not rescued. This extends
the trade-off the module already accepts for never-pushed tags on pushed commits. In the
SASE workflow, commits are made on branches by host finalizers, so this state does not
arise in practice.

No Rust-core change is needed. Workspace rescue is Python launch-time glue under
`src/sase/workspace_provider/`, and no other frontend shares this behavior.

## Changes

### `src/sase/workspace_provider/rescue.py`

- Split `_bundle_refs()` into anchor and tag collection, and add the trigger check
  (step 2) and the tag filter (step 3). Reuse `_remote_tracking_refs()` and
  `_published_tags()`. Avoid redundant `for-each-ref` calls where it is easy, but
  correctness matters more than shaving git calls.
- Add the detached-HEAD tag check (step 4) unless `detached_tag_checkout = no`.
- In `_rescue_git_repo()`, decide `needs_bundle` from the anchor check. Return `None`
  early when `not needs_bundle and not has_worktree_changes`, after cleaning up any temp
  refs that were created. The detached and stash temp refs are currently created inside
  `_bundle_refs` and deleted in `_write_bundle`'s `finally`, so an early return must not
  leak them. Either create temp refs only after deciding to bundle, or delete them on
  every path.
- Update the module and helper docstrings to state the rule "tags never trigger a rescue
  on their own" and the accepted trade-off.

### Tests: `tests/workspace_provider/test_rescue.py`

Reuse the existing helpers (`_seed_remote_clone`, `_redirect_state`, `_rescue_entries`,
`_clone_and_fetch`, ...). Add:

- `test_rescue_ignores_upstream_tags_off_every_remote_branch`: the remote has a tag on a
  commit that is on no remote branch (create it on a temp branch, push only the tag,
  delete the branch, then make a fresh clone of the remote). On a clean `main`,
  `rescue_git_repo(...) is None`, there are no store entries, and nothing is notified.
- `test_rescue_bundle_omits_upstream_off_branch_tags_when_rescuing_branch_work`: the
  same off-branch upstream tag plus an unpushed commit on `main` with a local tag on it.
  The bundle lists `refs/heads/main` and the local tag, but not the off-branch upstream
  tag.
- `test_worktree_only_rescue_writes_no_tag_bundle`: an off-branch upstream tag plus an
  untracked file. The record has a `patch_path`, `bundle_path is None`, and the
  notification names only worktree changes.
- (Unless `detached_tag_checkout = no`)
  `test_rescue_ignores_detached_checkout_of_upstream_tag`:
  `git checkout <off-branch tag>` in a clean clone gives `None`. Also add a
  counter-test: a new commit made on top of that detached HEAD is still rescued and
  restorable through `_clone_and_fetch`.
- Assert that no `refs/sase/rescue-tmp/*` refs remain after an early-return rescue.

The existing tests must keep passing unchanged:
`test_rescue_bundles_local_tags_but_not_published_ones`,
`test_rescue_ignores_annotated_tags_already_on_remote`,
`test_rescue_bundles_everything_without_remote_tracking_refs`, and the stash, detached,
and patch tests.

### Tests: `tests/test_axe_runner_workspace_repo_rescue.py`

- Add an external-clone eviction test next to
  `test_linked_clone_with_only_pushed_release_tags_needs_no_rescue`. Use a nested
  external clone (as in
  `test_rescues_linked_worktree_changes_and_nested_external_clone`) whose only extra
  state is an upstream off-branch tag, and assert that
  `rescue_repo_clones_before_eviction` creates no entries and sends no notifications.

### Docs: `docs/reference/agents/workspaces.md`

Update the rescue paragraph (around the "Tags whose commit is already reachable from a
remote-tracking branch" sentence). State that tags never trigger a rescue on their own
and are bundled only when they label local-only commits that a branch, stash, recovery
ref, or detached HEAD is being rescued for, so upstream release tags in third-party
clones produce no rescue entries. Also state the accepted trade-off in one sentence.
Read `docs_writing.md` with `/sase_memory_read` before editing docs. Check
`docs/workspace.md` and `docs/reference/work-units/bead-storage.md` (Publication
Verification) for any statement the new rule makes false, and fix it.

## Bead bookkeeping

- When the change lands, close task bead `sase-1km` (the original "rescue bundles tags,
  false notification every launch" report, still `ready`) with a note that names this
  fix and the new tests: `sase bead close sase-1km --note "..."`.
- Task `sase-1md` (rescue store falls back to `~/.sase/rescue` because
  `_project_key_for_workspace` ignores the checkout marker) is a separate, related bug
  in the same file. Do **not** fold it in. Avoid restructuring
  `_project_key_for_workspace` / `_rescue_root_for_workspace` so that fix stays a clean,
  separate change.

## Verification

- Read `lint_and_test.md` with `/sase_memory_read` and run the verification it
  prescribes (`just check` through `sase tool run`; do not run `just check-full`).
- Targeted:
  `pytest tests/workspace_provider/test_rescue.py tests/test_axe_runner_workspace_repo_rescue.py`.
- Manual sanity check (optional): on a scratch clone of a repo with off-branch release
  tags, `rescue_git_repo` returns `None`, and an unpushed commit on `main` still yields
  a restorable bundle.

Out of scope: deleting the existing false-positive entries under `~/.sase/rescue/` (the
reaper removes them after 30 days, and the user can delete them by hand), and changing
how external clones are cloned (for example `--no-tags`), since agents researching
third-party repos need the tags.
