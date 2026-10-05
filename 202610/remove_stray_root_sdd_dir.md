---
tier: tale
title: Remove the stray root sdd/ directory and stop tests from re-creating it
goal:
  The sase repo no longer tracks a root `sdd/` directory, and neither the test suite nor
  SDD init can write or commit generated SDD scaffolding into a GitHub-backed checkout
  again.
size: medium
proposed_by: bbugyi200.athena.0x3
create_time: 2026-10-05 18:08:35
status: wip
---

# Plan: Remove the stray root `sdd/` directory and stop tests from re-creating it

## Problem

The sase repo root on GitHub tracks an `sdd/` directory. This project keeps its SDD
content in sidecar repos (`sase--plans`, `sase--research`, `sase--beads`, ...). The
in-tree `sdd/` was removed on purpose on 2026-07-10
(`db116de892 chore: Remove local sdd/ dir`).

These four files came back:

- `sdd/README.md`
- `sdd/plans/README.md`
- `sdd/research/README.md`
- `sdd/assets/sdd-directory-map.png`

They are the generic in-tree SDD templates from `src/sase/sdd/templates/`.

## Root cause (verified)

### A pytest process wrote and committed the files

- **The PNG proves it.** `sdd/assets/sdd-directory-map.png` is 26 bytes:
  `directory-map-placeholder\n`. That is byte-for-byte
  `tests/fixtures/directory-map-placeholder.bin`. Only the autouse fixture
  `_use_placeholder_directory_map_assets` in `tests/_conftest_environment.py` supplies
  it, through `SASE_DIRECTORY_MAP_ASSET_OVERRIDE`.
- **Only one function makes these commits.** Every one is authored
  `sase <sase@localhost>` with the message `Initialize SDD\n\nSASE_TYPE=init`. Only
  `commit_bare_git_sdd_init_paths()` in `src/sase/sdd/_commit_bare_git.py` produces
  that, reached through `ensure_bare_git_sdd_initialized()`.
- **They were made during `just check`.** Workspace reflogs show 5 such commits on the
  local `master` of agent workspaces between 2026-09-06 and 2026-09-08. Each was created
  while that agent's `just check` was running. One example: commit at 19:35:35 EDT,
  inside a `just check` that ran 19:21–19:36 EDT.
- **The finalizer published them.** The host finalizer's `pull --rebase` and push then
  carried each stray commit to `origin/master` together with the agent's real commit.

### History of the directory

1. `9f827b5c4b` created the directory.
2. `147da5943a` reflowed the files at 88 columns via `just fmt`.
3. `9f875ee114`, `fe7f84f038` and `42f6bb3d51` were further leaked "Initialize SDD"
   commits that restored the template wrapping.
4. The reverts `b2573ba139` and `1f0f8ff0a9` only undid those re-wraps. The directory
   itself was never removed.
5. `e962a3ea44` added `sdd/README.md` and `sdd/plans/` to `.prettierignore` to silence
   `fmt-md-check`. That entrenched the files.

### Why it stopped, and why deleting the directory re-arms it

SDD init only acts on drift. Once the tracked files matched the templates byte-for-byte,
and prettier stopped reflowing them, re-running the leaked init became a no-op.

**Deleting `sdd/` brings the drift back.** So the prevention below must land in the same
change as the deletion.

### The leak path is still live today

An instrumented full-suite run on current `master` traced every SDD-init call. Four
tests call `ensure_bare_git_sdd_initialized()` on the checkout that is running the
tests. Their bead-store resolution is not isolated, so under the sandboxed HOME it falls
back to the cwd checkout. The call chain is `get_read_view()` → `get_project()` →
`init_beads()` in `src/sase/bead/cli_common.py`.

The three `test_cli_doctor.py` tests get there via `cli_admin.handle_bead_doctor` →
`inspect_attachment_health` → `get_read_view`:

- `tests/test_bead/test_cli_doctor.py::test_plain_doctor_forwards_roots_without_planning_or_writing`
- `tests/test_bead/test_cli_doctor.py::test_fix_preview_cancellation_never_opens_mutation`
- `tests/test_bead/test_cli_doctor.py::test_stale_preview_performs_no_updates_or_commit`

The fourth gets there via `send_completion_notification` →
`format_agent_bead_display_for_name` → `lookup_bead_issue` → `get_read_view`. Sibling
tests in that file already stub `sase.agent.bead_display.lookup_bead_issue`; this one
does not, and its agent name `sase-x.3` looks like a bead id:

- `tests/test_run_agent_runner_notifications.py::test_completion_notification_does_not_truncate_dotted_standalone_name`

### What currently stops a commit

Only the remote-URL heuristic in `is_local_bare_git_workspace()`. It returns true when
both of these hold:

- `detect_vcs()` reports `bare_git`. That is the fallback whenever the `sase_github`
  plugin is not loaded.
- The origin URL does not start with `http://`, `https://`, `git@` or `ssh://`.

That lets through origins that are not SASE bare-git remotes:

- a local path to a **non-bare** checkout. Workspace clones are made with
  `git clone <primary checkout>` before their origin is rewritten.
- an scp-style SSH host alias such as `gh:org/repo.git`.

The exact condition that let the heuristic pass on 2026-09-06 through 09-08 could not be
reproduced. Neither current code nor leak-era code reproduced it under a real-like
environment. So the fix is defense in depth.

### Deleting the directory is safe

- `sase validate` (`init repo --check`) does not inspect the root `sdd/` for this
  sidecar-configured project. It passes today even though the tracked placeholder PNG
  differs from the packaged asset.
- Nothing else in the repo reads these tracked files.

## Changes

### 1. Delete the tracked root `sdd/` directory

- `git rm -r sdd`. This removes the 4 tracked files above.
- Do not touch gitignored local leftovers, such as the perf-check JSON reports under
  `sdd/plans/<YYYYMM>/perf_artifacts/`, or their `.gitignore` entries. They are out of
  scope; see Follow-ups.

### 2. Drop the workaround `.prettierignore` entries

- Remove the `sdd/README.md` and `sdd/plans/` lines. `e962a3ea44` added them only to
  hide the leaked files.
- Leave the older legacy `sdd/...` entries alone.

### 3. Isolate the four leaky tests

- **`test_cli_doctor.py`:** stub the bead-store read that these tests do not exercise.
  Check how `src/sase/bead/cli_admin.py` imports `inspect_attachment_health` and patch
  that name; other tests in the file may already show the idiom. These tests must not
  resolve a real bead store.
- **The notification test:** stub `sase.agent.bead_display.lookup_bead_issue` exactly
  like its siblings in that file.
- Do not add a blanket `chdir` to these tests or to `conftest`.

### 4. Harden `is_local_bare_git_workspace()`

In `src/sase/sdd/_commit_bare_git.py`, require positive evidence: the origin must
resolve to an existing local **bare** repository.

- Keep the fast `False` for network-looking URLs. Then also return `False` for:
  - scp-like `host:path` URLs;
  - schemes other than `file://`.
- Resolve `file://` URLs and plain paths:
  - expand `~`;
  - resolve relative paths against the workspace.
  - `src/sase/workspace_provider/plugins/bare_git_init.py` already resolves origin paths
    near `_assess_clone_dir`. Reuse or mirror that logic instead of inventing a variant.
- Confirm bareness with
  `run_sdd_git(["--git-dir", str(path), "rev-parse", "--is-bare-repository"], ...)` and
  require the output `true`. Any error or `SddGitCommandTimeout` means `False`. Pass a
  distinct `op=` label.
- Update the docstring to state the positive-evidence contract.
- Make sure every legitimate flow still qualifies:
  - `init_bare_git_project()` primary clones, whose origin is `BARE_REPO_DIR`;
  - bare-git workspace clones (`ensure_workspace_checkout()` rewrites their origin to
    the primary's origin, which is the bare repo);
  - existing tests that build real bare remotes, e.g.
    `tests/test_sdd_initialization.py`, `tests/test_bare_git_init*.py`,
    `tests/test_bare_git_workspace*.py`.

  If a fixture relied on a non-bare local origin, fix the fixture, not the contract.

- This is Python-only SDD-init logic. If you find an equivalent heuristic in the linked
  `sase-core` repo (open it with `sase repo open sase-core`), stop and record it as a
  follow-up instead of diverging silently.

### 5. Test-harness guards

#### 5a. Per-test prevention guard (autouse)

- Add an autouse fixture in `tests/_conftest_environment.py`, next to
  `_use_placeholder_directory_map_assets`.
- It monkeypatches `sase.sdd._commit_bare_git.is_local_bare_git_workspace` with a
  wrapper. `ensure_bare_git_sdd_initialized()` looks that name up as a module global at
  call time, so patching the module attribute covers every caller.
- When the resolved workspace is the suite's own checkout (`_REPO_ROOT`) or inside it,
  the wrapper:
  1. records the hit;
  2. returns `False`, so nothing is written or committed;
  3. fails the test after its teardown, naming the workspace.

  Otherwise it delegates to the real function.

- Reuse the existing teardown-check style (`check_tmp_env_leak_guard` and the
  `pytest_runtest_teardown` wrapper in `tests/conftest.py`) or a fixture-teardown
  `pytest.fail`, whichever fits the house style.
- After step 3, no test should trip it. Before step 3, the four tests above must trip
  it. Use that ordering to confirm the guard works.

#### 5b. Session tripwire for commits and `sdd/` writes in the real checkout

- Add `tests/_checkout_leak_guard.py` and wire it into `tests/conftest.py`'s
  `pytest_sessionstart`, `pytest_sessionfinish` and `pytest_terminal_summary`, beside
  the temp-dir leak guard.
- Mirror `tests/_tmp_leak_guard.py`'s structure, including how it handles the xdist
  controller vs. workers and nested `pytester` sessions.
- **At session start:** record `git rev-parse HEAD` and
  `git status --porcelain=v1 -z --untracked-files=all -- sdd` for `_REPO_ROOT`. If
  `_REPO_ROOT` is not a git checkout, disable the guard silently.
- **At session finish, it is a leak if either holds:**
  - HEAD moved and any commit in `<start>..HEAD` has an author or committer email of
    `sase-test@example.invalid` (the per-test hermetic identity set in
    `tests/conftest.py`) or `sase@localhost`;
  - any new `sdd` status entry appeared.
- **On a leak:**
  - make the session exit non-zero;
  - print the offending commits and paths;
  - print a recovery hint: `git reset --keep <start-sha>`, then delete the listed paths.
- New commits with other identities, such as a human committing during a run, are only a
  note in the summary, never a failure.
- Do not snapshot the whole worktree. That would false-positive on edits made while the
  suite runs.

### 6. Tests

- **Step 4, unit tests** on tmp repos:
  - `False` for a local **non-bare** checkout origin, and
    `ensure_bare_git_sdd_initialized()` writes nothing there;
  - `False` for an scp-style alias origin;
  - `True` for a `file://` bare origin;
  - `True` for a relative bare-path origin.

  Monkeypatch `detect_vcs` where needed to isolate the URL logic.

- **Step 5a:** a test proving the wrapper refuses `_REPO_ROOT` and fails the test.
  `pytester` or a direct call to the guard's check helper are both fine.
- **Step 5b:** tests for the pure helpers that classify start/finish snapshots, built on
  a tmp git repo. Cover:
  - a test-identity commit is a leak;
  - a foreign-identity commit is not;
  - a new untracked `sdd/README.md` is a leak;
  - a non-git root disables the guard.

## Verification

- `git ls-files sdd` prints nothing.
- Run the four formerly-leaky tests alone, then the full suite. Afterwards there is no
  `sdd/` change in `git status`, and HEAD is unchanged.
- `sase validate` still passes.
- Before finishing, read the `lint_and_test` reference memory with `/sase_memory_read`
  and follow it. `just check` is the agent verification recipe; per the
  `check-full-is-explicit` decision, do not run `just check-full` unless instructed.

## Follow-ups (out of scope)

File each as a task bead via `/sase_new_task` unless one is already tracked.

- Read-only bead paths auto-initialize stores. `get_read_view()` → `get_project()` →
  `init_beads()` does this, and attempts SDD-init commits in whatever repo the cwd
  resolves to, even for display-only lookups.
- Perf-check recipes and CI artifact uploads still write runtime JSON under legacy
  in-tree `sdd/plans/<YYYYMM>/perf_artifacts/` paths. These are gitignored but keep
  recreating a local `sdd/` tree.
