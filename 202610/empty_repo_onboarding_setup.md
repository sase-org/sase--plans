---
tier: tale
title: Onboarding setup works on empty remote repositories
goal:
  Adding or creating a project whose remote has no commits marks it SASE-managed instead
  of failing, and the New Project wizard never hides a failed setup behind the init
  hand-off.
size: medium
decisions:
  first_branch:
    ask:
      Which branch should setup create when it pushes the first commit to an empty
      GitHub repository?
    choices:
      local:
        The clone's own branch (your git default, master here); GitHub adopts it;
        sase-only
      provider:
        GitHub's configured default (main here) via a new sase-github REST probe
        fallback
    default: local
    why:
      Matches the clone and your git default, works for bare git, needs no extra API
      call
    answer: local
proposed_by: bbugyi200.apollo.8m
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-17 11:04:07
status: wip
---

# Fix project onboarding setup for empty remote repositories

## Problem and root cause

The user added `bbugyi200/tailnet` from the SASE Admin Center Projects tab (New Project
wizard, existing-repo "add" mode). The repository had been created on GitHub one minute
earlier with **no commits**. Evidence:

- Proc `c93sjfmjmm5x` ran
  `sase project add bbugyi200/tailnet --make-current --name tailnet --yes --json`. Its
  log shows `checkout` done, then `manage` failed with
  `the provider did not report a usable default branch`. The result status was
  `partial`, with `next_argv = sase init -p gh_bbugyi200__tailnet`.
- The onboarding journal recorded `status: failed` and `stopped_step: manage`, and every
  `management` field was empty.
- `gh repo view` on an empty repository returns `defaultBranchRef: {"name": ""}` and
  `isEmpty: true`. The sase-github probe passes the empty name through as
  `default_branch`.
- 13 seconds later the TUI showed the toast "tailnet is initialized · Config, Machine,
  Memory, Repos, Service, and Skills are current".
  `sase init -c -p gh_bbugyi200__tailnet` shows why: the repo initializer is _skipped_
  ("repository is not SASE-managed") but counts as `ok`. So the project was registered
  but never became SASE-managed, and the wizard ended on Ready instead of Failed.

There are two defects.

1. **Primary: `manage_repository` cannot handle an empty origin**
   (`src/sase/project_onboarding/manage.py`). It assumes origin already has a default
   branch with a tip commit:
   - it needs a non-empty provider `default_branch` (fails first on GitHub);
   - it runs `git rev-parse HEAD` (fails on the unborn branch of an empty clone);
   - it runs `_remote_oid` (`ls-remote` finds no heads);
   - it has `head == remote_oid` and parent-commit bookkeeping.

   A sandbox repro against an empty bare origin hit both failure modes:
   - provider branch `""` raises `the provider did not report a usable default branch`
     (GitHub);
   - provider branch `main` raises `failed to read checkout HEAD ... unknown revision`.
     This is the bare-git case: `ws_create_repo` for `git:NAME` runs `git init --bare`,
     so **`sase project create git:NAME` with setup always fails the same way**. Every
     bare-git CLI test uses `--no-setup`. A GitHub `create` with the `add_readme` option
     turned off would also hit this.

   Latent bug in the same file: `_valid_expected_commit` lists the commit's paths with
   `git diff-tree --no-commit-id --name-only -r`. That prints nothing for a root commit
   unless `--root` is passed, so a root marker commit can never validate.

2. **Secondary: the wizard treats `partial` like success**
   (`src/sase/ace/tui/modals/_new_project_apply_flow.py`, `_on_progress_result`). For
   `status in {"ok", "partial"}` it starts the `sase init` hand-off and finishes on the
   Ready stage, which hides the setup failure and drops the Failed stage's Retry. The
   CLI (`src/sase/main/project_handler_add.py`) offers init only on `ok`, and the docs
   say "Successful setup may hand off to `sase init`."

No sase-core change is needed. The Rust planner already receives `remote.is_empty`, and
the manage effect is Python-owned today.

## Changes

### 1. Empty-origin path in `src/sase/project_onboarding/manage.py`

Keep the existing non-empty path byte-for-byte equivalent in behavior. Add a root-commit
path:

- **Routing.** After the journal and `can_push` checks, call `_verify_origin` and decide
  whether origin is unborn. Take the new path when either condition holds:
  - the journal `management` state records `root_commit: true` and origin's
    `refs/heads/<commit_branch>` equals the recorded `commit_id` (re-entry after a
    completed push);
  - origin has **no heads** (`git ls-remote --heads origin` returns no rows; add a small
    `_remote_heads` helper). Only check this when the provider reported
    `is_empty is True` or no usable default branch, so non-empty repos pay nothing
    extra.

  If the default branch is still unusable and origin _does_ have heads, keep raising the
  existing `the provider did not report a usable default branch` error.

- **Branch choice.** Use the recorded `commit_branch` if present. Otherwise use the
  provider-reported default branch if it is usable (bare git reports its unborn `HEAD`
  symref). Otherwise use the checkout's unborn branch
  (`git symbolic-ref --quiet --short HEAD`, validated with `_normalize_branch`). If none
  is usable, raise a `ManagementError`. If the chosen branch differs from the unborn
  local branch, run `git symbolic-ref HEAD refs/heads/<branch>`. This is safe only while
  HEAD is unborn, so assert that first.

  > [!decision] first_branch = provider Also change `probe_gh_repo` in the linked
  > `sase-github` repo (`src/sase_github/workspace/onboarding.py`, open it with
  > `sase repo open sase-github`). When `isEmpty` is true and `defaultBranchRef.name` is
  > empty, fill `default_branch` from
  > `gh api repos/{owner}/{name} --jq .default_branch`. Treat a failure there as
  > best-effort and leave the field empty. Add a sase-github test for both outcomes and
  > run `sase tool run check` in that checkout too. The routing rule above then pushes
  > to that name.

- **Local HEAD guard.** If `git rev-parse --verify -q HEAD` succeeds, HEAD must be a
  valid root marker commit:
  `_valid_expected_commit(checkout, head, "", expected_hash, _MANAGEMENT_COMMIT_MESSAGE)`.
  Otherwise raise a `ManagementError` ("the checkout already has local commits but
  origin is empty; push or remove them, then retry setup"). Never push anything except
  the single marker commit.
- **Marker write.** Reuse the existing marker preparation and write logic. That is the
  block from `_read_config` through the `marker_written` evidence write, including the
  hash and journal recovery checks. Factor it into one helper shared by both paths
  instead of duplicating it. A missing `sase/sase.yml` is the expected baseline (empty
  bytes).
- **Evidence before effects.** Before committing, write `root_commit=True`,
  `commit_branch=<branch>`, `commit_parent=""`, `commit_content_hash`, and
  `commit_message`. `update_management` accepts arbitrary keys, so no schema change is
  needed. Then reuse the existing `commit_started` / `committed` / `push_started` /
  `push_verified` stages.
- **Commit.** Run `_stage_config`, then `git commit --only -m <msg> -- sase/sase.yml`.
  This works on an unborn branch and leaves other untracked files, such as a
  user-created README, uncommitted. After a failed or lost commit, recover with
  `_find_expected_commit(checkout, "", ...)`, since a root commit's `%P` is empty.
- **Push.** Immediately before pushing, re-read origin's heads.
  - If `refs/heads/<branch>` already equals the commit, record `push_verified` and
    return success.
  - If origin has any other heads, fail closed with a `ManagementError`. The message
    should say origin received commits while setup was creating the first commit, the
    local marker commit is preserved, and nothing was pushed.
  - Otherwise push `<commit>:refs/heads/<branch>`. Verify with `_remote_oid`, reuse the
    existing protected-branch failure hint, record `push_verified`, and return
    `_success(commit_id, changed=True)`.
- **Upstream.** After a verified push, set `branch.<branch>.remote=origin` and
  `branch.<branch>.merge=refs/heads/<branch>` only if they are missing. The clone sets
  them for its original unborn branch name, but not for a renamed one.
- **Root-commit validation fix.** Add `--root` to the `diff-tree` call in
  `_valid_expected_commit`. This does not change results for non-root commits.

`manage.py` is 661 lines. If the change would push it past the 850-line `toobig` warning
tier, move the git plumbing into a new sibling module under **public** names
(`_run_git`, `_git_output`, `_failure_message`, `_read_config`, `_sha256`, and
`_remote_oid`/`_remote_heads`), e.g. `src/sase/project_onboarding/manage_git.py`.
Symvision rejects cross-file imports of `_private` names.

### 2. Wizard: a `partial` result lands on Failed, not init

In `_new_project_apply_flow.py` `_on_progress_result`:

- Only `status == "ok"` records preferences and starts the init hand-off, or finishes
  Ready.
- `partial` (and `failed`/`blocked`) calls `_finish_failed()`.

In `_finish_failed` / `NewProjectFailed` (`new_project_failed.py`), show a lead line
"Project registered; setup incomplete" for a `partial` result. Use the same style as the
existing "Repository created; setup incomplete" create notice, which keeps priority for
create. Show it above the result message and the failed step's hint. `r` (Retry) already
re-runs Review and apply, which resumes the same journal. Do not change the CLI.

### 3. Docs

In `docs/guides/add-existing-repository.md` ("Register the project"), add one sentence.
It should say that for an empty repository, setup creates the first commit, containing
only `sase/sase.yml`, on the clone's branch and pushes it (which makes it the default
branch). Under `first_branch = provider`, say it uses the provider's configured default
branch instead. Read the `docs_writing` reference memory first and run the docs
formatters it names.

## Tests

`tests/project_onboarding/test_manage.py`: add an empty-origin fixture. Use an empty
bare repo plus a `git clone` of it, with user name and email configured and the same
`parse_repo_ref` monkeypatch and `record_intent` as `onboarding_repo`. Cover:

- remote `{"default_branch": "", "is_empty": True}` succeeds. Origin's branch then has
  exactly one root commit, whose only path is `sase/sase.yml` and which is SASE-managed.
  The journal shows `push_verified`, and a pre-existing untracked `README.md` stays
  uncommitted;
- a provider-reported branch that differs from the clone's unborn branch pushes to the
  provider branch and leaves the local HEAD on it with upstream config set;
- recovery when the push fails once (monkeypatch the push to fail): the first call
  raises, the retry pushes the same commit, and no second commit is created;
- recovery when the push succeeded but the evidence write was lost: the retry returns
  success with the recorded commit;
- origin gains a commit (pushed from a second clone) after the marker commit: the call
  fails closed, origin is unchanged, and the local commit is kept;
- local unrelated commits with an empty origin: `ManagementError`, nothing pushed;
- regression: a non-empty origin with `default_branch: ""` still raises the existing
  default-branch error.

`tests/project_onboarding/test_cli_bare_git.py`: add an end-to-end
`project create git:NAME` **with** setup, following the existing resume test's harness
but without `--no-setup`. Assert the result is `ok` and the bare repo's HEAD branch
holds the marker commit.

`tests/ace/tui/modals/test_new_project_apply.py`: a `partial` result with an init
`next_argv` lands on stage `failed`. Assert that `run_project_init` is never called
(monkeypatched recorder) and that the Failed body contains "setup incomplete". Update
any existing test that assumed partial goes to Ready.

Before editing TUI code, read the `tui` reference memory. The Failed stage already
exists, so no new visual goldens are expected.

## Verification

- Run `sase tool run check` in this repo (and in sase-github if
  `first_branch = provider`). Read the `lint_and_test` reference memory first.
- Optional manual smoke test in a temp dir only: an empty bare origin plus a clone,
  running `manage_repository` as the new tests do.

## Out of scope

- Do not touch the user's existing tailnet checkout or GitHub repository. The user has
  since pushed a README commit to both `main` and `master` by hand, and will recover it
  themselves.
- `sase init` reporting a skipped, unmanaged repo initializer as `current` is init's
  contract and stays unchanged. After change 2 the wizard no longer reaches it after a
  failed setup.
