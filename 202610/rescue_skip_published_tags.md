---
tier: tale
title: Stop rescuing already-published annotated tags from linked clones
goal:
  Launch-time rescue skips tags a remote already has, so clean linked sase-core clones
  stop sending false rescue notifications.
size: small
decisions:
  cleanup:
    ask:
      Should the implementer also delete the existing false-positive linked-sase-core
      rescue entries?
    choices:
      keep: Leave them alone; the rescue reaper deletes them within 30 days
      delete: Delete only entries proven to hold nothing but pushed tags; irreversible
    default: keep
    why:
      They hold nothing worth keeping and use about 1 MB; deleting from the rescue store
      gains little
    answer: keep
proposed_by: bbugyi200.athena.108
decided_by: auto
create_time: 2026-10-13 17:58:59
status: wip
---

# Plan: Stop rescuing already-published annotated tags from linked clones

## Problem

Since commit `1a638eac95` ("feat(workspace): rescue linked and external clones before
eviction"), every agent launch in a numbered workspace sends a `workspace-rescue` inbox
notification. The notification reads "Rescued leftover state from linked-sase-core
before workspace healing: local commits (local-commits.bundle)" with the reason
"launch-time eviction of linked or external repo clone". These are false alarms. The
research report `research:202610/linked_sase_core_rescue_notifications.md` looked at
every manifest. In each one the clone was clean (`head == upstream`, empty
`status_porcelain`, no stash entries). Each bundle held exactly sase-core's 313
annotated release tags (`refs/tags/*`) and no branch. Every one of those tags is already
on GitHub.

## Root cause

`_bundle_refs()` in `src/sase/workspace_provider/rescue.py` collects every ref under
`refs/heads`, `refs/tags`, and `refs/sase/recovery`. `_write_bundle()` then runs
`git bundle create <path> <refs...> --not --remotes`, expecting the exclusion to drop
anything a remote already has. That works for commits but not for tag objects. Git's
bundle writer skips a positive tip only when it is a commit that the rev walk excluded,
and it always writes non-commit tips. No `refs/remotes/*` ref points at an annotated tag
object, so every annotated tag becomes a bundle head. The bundle is then non-empty, so a
rescue entry and a notification are created on every launch.

A minimal reproduction (already confirmed in a scratch repo):

```text
git bundle create b refs/heads/main --not --remotes          -> Refusing to create empty bundle
git bundle create b refs/tags/v1 refs/tags/v2 --not --remotes -> bundle listing both tags
```

Lightweight tags are unaffected because they point straight at commits.

## Fix

Filter out tags that are already published before they reach `git bundle`. A tag counts
as published when its peeled commit is reachable from at least one remote-tracking ref.
One `git for-each-ref` call answers this for all tags at once:

```bash
git for-each-ref --format=%(refname) --merged=<remote ref 1> --merged=<remote ref 2> ... refs/tags
```

Facts confirmed before writing this plan, with git 2.47:

- When `--merged` is given more than once, the result is a **union**: a tag reachable
  from any one of the refs is listed.
- `--merged` peels annotated tags, including nested tag-of-tag, down to their commit.
- In the real sase-core linked clone (313 tags, 97 remote-tracking refs), all 313 tags
  are classified as published. With 400+ tags the call takes about 20 ms.
- An unknown or non-commit `--merged` argument makes the command exit non-zero
  (`fatal: malformed object name`).
- A tag whose target is not a commit (for example a tag on a blob) is never in the
  `--merged` output, so it stays in the bundle. That is the safe direction.

### Changes to `src/sase/workspace_provider/rescue.py`

1. Add a small helper that lists remote-tracking refs and returns an empty list when the
   command fails:

   ```python
   def _remote_tracking_refs(repo_root: Path) -> list[str]:
       result = _git(repo_root, "for-each-ref", "--format=%(refname)", "refs/remotes")
       if result.returncode != 0:
           return []
       return [line.strip() for line in result.stdout.splitlines() if line.strip()]
   ```

2. Add `_published_tags(repo_root, remote_refs) -> set[str]`. It returns an empty set
   when `remote_refs` is empty. Otherwise it runs the `for-each-ref --merged=...` call
   above (one `--merged=<ref>` argument per remote ref, then `refs/tags`) and returns
   the refnames. If the command fails (non-zero exit, or `_git`'s synthetic 124 on
   OSError/timeout), it returns an empty set. **Fail safe**: when unsure, keep every
   tag, which is today's behavior. Give it a short docstring that covers three points:
   - why it exists: `git bundle ... --not --remotes` always writes non-commit tips;
   - the accepted tradeoff: git keeps no list of which tags a remote has, and rescue
     must stay offline (no `git ls-remote` during launch), so an annotated tag that was
     never pushed but sits on a pushed commit is not rescued; only the tag name and
     message are lost, never a commit;
   - why tags are kept when there are no remote-tracking refs: it matches
     `_write_bundle()`, which then bundles everything.

3. In `_bundle_refs()`, after collecting the three namespaces, drop every ref that is in
   `_published_tags(repo_root, _remote_tracking_refs(repo_root))`. Leave the
   detached-HEAD and stash temporary-ref logic unchanged.

4. In `_write_bundle()`, replace the inline
   `_git(repo_root, "for-each-ref", "--format=%(refname)", "refs/remotes")` check with
   `if _remote_tracking_refs(repo_root): args.extend(["--not", "--remotes"])`. This
   keeps one definition of "has remote-tracking refs" for both the tag filter and the
   bundle exclusion. Do not refactor anything else in `_write_bundle()`.

Nothing else needs to change. After the filter, a clean clone whose only extra refs are
pushed tags produces `bundle_refs == ["refs/heads/<branch>"]`. `git bundle create` then
refuses to create an empty bundle, and the existing branch in `_rescue_git_repo()` (the
"Bundling found nothing" `shutil.rmtree` path) removes the entry directory and returns
`None`, so no manifest and no notification. When real local commits exist, the bundle
still gets written but no longer carries the 313 published tags.

The fix sits in the shared `_bundle_refs()`, so it also applies to sidecar rescues
(plans, beads, agents, …) and checkout healing. That is intended: a published tag is
never leftover state, whatever kind of clone it lives in.

`rescue.py` is currently 661 lines, and the repo's `toobig` gate warns at 700. Keep the
helpers compact, with brief docstrings and no new module, so the file stays under 700
lines.

## Tests

### `tests/workspace_provider/test_rescue.py` (real-git unit tests)

Use the existing helpers (`_seed_remote_clone`, `_commit_file`, `_redirect_state`,
`_bundle_lists_sha`, `_clone_and_fetch`, `_rescue_entries`). The session-wide test git
config (`tests/conftest.py`) already isolates global git config, so `git tag -a` needs
no signing overrides.

1. `test_rescue_ignores_annotated_tags_already_on_remote` (the regression from the
   report): in the clone, create two annotated tags on the pushed `main` commit plus one
   lightweight tag, and push the tags to origin with `git push origin --tags`. This
   mirrors release-plz tags fetched into a linked clone. Assert that
   `rescue_git_repo(..., include_worktree=True)` returns `None`,
   `_rescue_entries(home) == []`, and `notified == []`.
2. `test_rescue_bundles_local_tags_but_not_published_ones`: create a pushed annotated
   tag (as above), then make an unpushed commit on `main` and an annotated tag on it.
   Rescue must return a record with a bundle. Run `git bundle list-heads` on that bundle
   and assert it contains `refs/heads/main` and the local tag but **not** the pushed
   tag. Then use `_clone_and_fetch` and assert that
   `refs/sase/rescued/x/tags/<local tag>^{commit}` resolves to the unpushed commit's
   SHA.
3. Extend `test_rescue_bundles_everything_without_remote_tracking_refs` with an
   annotated tag on the offline commit, and assert the tag appears in `list-heads`. This
   pins the fail-safe behavior: with no remote-tracking refs, tags are still rescued.

### `tests/test_axe_runner_workspace_repo_rescue.py` (linked-clone path from the report)

4. Add `test_linked_clone_with_only_pushed_release_tags_needs_no_rescue`. Model it on
   `test_clean_fully_pushed_linked_clone_needs_no_rescue`, but before running
   `rescue_repo_clones_before_eviction(workspace, WORKSPACE_NUM)`, add annotated tags to
   the linked clone and push them to its origin. Assert there are no rescue entries and
   no notifications.

## Docs

In `docs/workspace.md`, the paragraph on launch-time eviction currently says "Linked and
external clones are rescued there too, with a bundle of local-only commits and a
worktree patch". Add one short sentence after it. The sentence should say that tags
whose commit is already reachable from a remote-tracking branch, such as fetched release
tags, count as published and are not rescued. Run `just fmt` (or `just fmt-md`) so the
Markdown wrapping stays canonical. `CHANGELOG.md` is generated by release-please, so do
not edit it.

## Cleanup of existing entries

> [!decision] cleanup = keep Do nothing outside the repo. About 16 entries named
> `<stamp>-ws<N>-linked-sase-core-<hash>` exist under `~/.sase/rescue/202610/`. The
> reaper deletes them after 30 days.

> [!decision] cleanup = delete After the code fix and tests pass, look at each
> `~/.sase/rescue/2026*/*-linked-sase-core-*` entry. Delete an entry only when all of
> these hold:
>
> - `manifest.json` has an empty `status_porcelain` and `stash_entries == 0`;
> - there is no `worktree.patch`;
> - `git bundle list-heads local-commits.bundle` lists only `refs/tags/*`;
> - every one of those tags also exists, with the same object id, in the current
>   sase-core linked clone. Open the clone with `sase repo open sase-core` first.
>
> Leave every other entry untouched, and report how many were deleted and how many kept.

## Out of scope

- Do not call `git ls-remote` or touch the network to tell apart local tags on pushed
  commits. Rescue runs on every launch with a 60 s per-call timeout and must stay
  offline.
- Do not move rescue logic into `sase-core`. This is a host-side bug fix in an existing
  Python module, not new shared backend behavior.
- While researching, it showed up that these rescues land in the machine-level fallback
  `~/.sase/rescue/` and not in `~/.sase/projects/<project>/rescue/`, because
  `scan_projects_for_cwd()` returns `None` for numbered workspace paths. Do not fix that
  here. If the implementer confirms it is unintended, file it through `/sase_new_task`.
- The notifications already in the inbox are left for the user to dismiss.

## Verification

- Run the targeted tests first:
  `pytest tests/workspace_provider/test_rescue.py tests/test_axe_runner_workspace_repo_rescue.py`.
- Run `sase tool run check`, the agent-default gate (lint plus diff-scoped tests). Do
  not run `check-full`.
- Optional manual sanity check: open the sase-core linked clone
  (`sase repo open sase-core`), call `rescue_git_repo()` on it against a temporary
  `SASE_HOME` with the notifier monkeypatched in a throwaway Python snippet, and confirm
  it returns `None`.
