---
tier: tale
title: Fix sase-github CI test drift against sase master
goal:
  sase-github's CI test matrix passes against sase master again. Its tests satisfy
  sase's push-identity guard and stage real handoffs, and they stay green on PyPI sase
  0.18.1.
size: small
proposed_by: bbugyi200.athena.1gd
proposed_with: claude/opus
create_time: 2026-11-26 05:49:34
status: wip
---

# Fix sase-github CI: align plugin tests with sase's push-identity guard and staged handoffs

## Goal

Make `sase-org/sase-github` CI green again. The work is test-only and lives entirely in
the linked **sase-github** repo. No sase, sase-core, or sase-github `src/` changes are
needed.

Open the repo with `sase repo open sase-github -r "<reason>"` and work only in the
printed path. Read its `CLAUDE.md` first.

## Diagnosis (already done; do not re-derive)

`actstat` shows `sase-org/sase-github` CI run #163 (push of `5f85ceb` to `master`):

- `test (3.12)` failed at "Run tests".
- `test (3.13)` was cancelled by fail-fast.
- `lint` passed.

The open release-please PR #33 (`chore(master): release 0.2.25`) is red for the same
reason. The scheduled "Release dependency floor" runs are green and unrelated.

CI checks out **sase `master` and sase-core `master`** and builds them from source
(`SASE_PYTHON_PATH` / `SASE_RUST_CORE_PATH`). The suite therefore runs against
unreleased sase, and two intentional sase behavior changes broke 7 tests:

- **Reproduced locally:** against current sase master the result is exactly
  `7 failed, 348 passed`.
- **Older sase still passes:** against PyPI sase 0.18.1 the suite passes 355/355.
- **Lint is clean:** `ruff check src/ tests/` and `mypy` pass against sase master.

### Root cause 1: sase's push-identity guard (6 tests in `tests/test_github_plugin.py`)

sase commit `a14bffcdbf` ("fix(workspaces): guard Git mutations and pushes by checkout
identity", first released in sase v0.19.0) changed `CommandRunner._run`. It now calls
`sase.workspace_provider.push_guard.require_push_identity(cwd)` before every `git push`.
That guard probes the checkout:

1. `git rev-parse --show-toplevel` must print the cwd itself.
2. `git remote get-url origin` must print an origin URL.

Both probes call `subprocess.run`, and the tests patch that same global attribute
(`_MOCK_TARGET = "sase.vcs_provider._command_runner.subprocess.run"`). The tests' fakes
either:

- are positional `side_effect` lists (push, pr view, pr create) that the probes consume
  first, or
- return empty stdout for `rev-parse`.

So the guard raises `Refusing Git mutation: /workspace is not its own Git worktree`.
`_run` turns that into a failed `CommandOutput`, and the plugin returns
`(False, "git push failed: ...")`.

Failing tests:

- `test_plugin_mail_creates_pr`
- `test_plugin_mail_existing_pr`
- `test_direct_mail_push_and_create_pr`
- `test_direct_mail_push_existing_pr`
- `test_vcs_create_pull_request_success`
- `test_vcs_create_pull_request_with_pr_title_prefix`

The guard also makes the tests non-hermetic. When no linked-repo metadata or checkout
marker matches, `require_push_identity` calls
`collect_repo_inventory(include_disabled=True)`. That walks the host's real SASE
inventory under `SASE_HOME` and issues `git remote` calls for every registered repo. On
a dev box this means dozens of extra mocked calls and about 0.4 s per test. With
`SASE_HOME` pointed at an empty temp directory, only the two probes remain.

**Production code is correct.** `GitHubPlugin.vcs_mail` does not retry pushes, so a
guard refusal already surfaces as `git push failed: ... Refusing Git mutation ...`.
sase's own `"Refusing Git"` short-circuit in `_push_with_retry` has no counterpart to
add here.

### Root cause 2: retired pending-handoff markers (`tests/test_gh_workspace_claims.py`)

sase commit `d3518e567d` ("feat(axe): retire the staged-handoff kill protocol", v0.19.0)
changed how `sase.workspace_provider.vcs_release.release_vcs_workspace` detects a
handoff:

- **Before:** `has_pending_handoff()`, which looked for leftover `.sase_monitor_pending`
  and similar markers.
- **Now:** `read_staged_handoff(artifacts_dir)`, which looks for the
  `.sase_handoff_staged.json` record.

sase's own `tests/workspace_provider/test_vcs_release.py` now asserts that leftover
markers do **not** skip release (`test_leftover_pending_marker_does_not_skip_release`).
It stages a real handoff with `write_staged_handoff(build_plan_staged_record(...))`
under `SASE_AGENT=1`.

`test_gh_release_step_skips_on_handoff` still writes the retired `.sase_monitor_pending`
marker. The release therefore proceeds and prints `released=true`, so
`assert "released=false" in out` fails. Nothing else in sase-github references the
retired markers.

### Tests that pass for the wrong reason (fix them in the same pass)

- **`test_direct_mail_push_fails`:** its fake returns `returncode=1` for every command.
  It now fails at the guard's `rev-parse` probe, and the guard error still contains
  `"git push failed"`. The push-failure path is never exercised.
- **`test_vcs_create_pull_request_pr_create_fails`:** its positional list makes
  `git diff --cached --quiet` return 0, so the flow stops at
  `"No staged changes to commit"`. It never reaches push or `gh pr create`, and it only
  asserts `ok is False`. This was already the case before the guard.

### Version window

sase v0.19.0 is tagged (2026-11-11), but PyPI's latest sase is still **0.18.1**. A plain
`just install-venv` (no source-override env vars) installs PyPI sase. That is also what
`sase tool run check` uses when it first creates sase-github's `.venv`. In 0.18.1 there
is no push guard, and `sase.agent.staged_handoff` does not exist. The fix must keep the
suite green on both sase master (CI) and PyPI 0.18.1 (default local venv):

- **Mail and PR fakes:** answer the probes when they arrive, but never assert that they
  ran. On 0.18.1 the probes never happen.
- **Handoff test:** import the staged-handoff helpers inside the test with
  `pytest.importorskip`. The module still imports on 0.18.1, and only that one test
  skips there.

## Changes (all in the sase-github checkout)

### 1. `tests/test_github_plugin.py`

1. Add a module-level `autouse` fixture. It uses `monkeypatch` to:
   - set `SASE_HOME` to a `tmp_path` subdirectory, which empties the repo inventory;
   - unset `SASE_LINKED_REPOS_JSON` (use `raising=False`).

   With this, `require_push_identity` never reads host state.

2. Add small module-level helpers.
   - **Origin constant:** `_ORIGIN_URL = "git@github.com:user/repo.git"`.
   - **Probe answerer:** `_identity_probe(cmd, kwargs) -> MagicMock | None` answers
     sase's pre-push identity probes:
     - `["git", "rev-parse", "--show-toplevel"]` returns `returncode=0` and
       `stdout=f"{kwargs['cwd']}\n"`. The guard passes `cwd` as a resolved `Path`, and
       `/workspace` and `/ws` resolve to themselves on Linux.
     - Any command starting `["git", "remote", "get-url"]` returns `returncode=0` and
       `stdout=f"{_ORIGIN_URL}\n"`.
     - Anything else returns `None`.

     Give it a one-line docstring naming `require_push_identity`.

   - **Command filter:** `_plugin_cmds(mock_run) -> list[list[str]]` returns the
     recorded `cmd` lists, minus anything `_identity_probe` would answer. Assertions
     then check the plugin's own command sequence and do not depend on how sase probes.
   - **Mail fake:** a mail fake builder, for example
     `_mail_fake(*, pr_exists: bool, push: MagicMock | None = None)`. Its
     `side_effect(cmd, **kwargs)` dispatches in this order:
     1. the identity probes;
     2. `git push` returns `push` or success;
     3. `gh pr view` returns `"42\n"` with rc 0 when `pr_exists`, else rc 1 with stderr
        `"no PR"`;
     4. `gh pr create` returns success;
     5. any other command returns `returncode=99` with stderr
        `f"unexpected command: {cmd}"`.

     Do not raise in that last branch: `_run` swallows exceptions.

3. Convert the four mail tests from positional `side_effect` lists to `_mail_fake`:
   - `test_plugin_mail_creates_pr`
   - `test_plugin_mail_existing_pr`
   - `test_direct_mail_push_and_create_pr`
   - `test_direct_mail_push_existing_pr`

   Replace the `call_count` and `call_args_list[i]` assertions with
   `_plugin_cmds(mock_run) == [...]`:
   - Create-PR cases:
     `[["git", "push", "-u", "origin", "feature-branch"], ["gh", "pr", "view", "--json", "number", "-q", ".number"], ["gh", "pr", "create", "--fill"]]`.
   - Existing-PR cases: only the first two entries.

   Keep `success is True` and `error is None`.

4. In `test_vcs_create_pull_request_success` and
   `test_vcs_create_pull_request_with_pr_title_prefix`, make the existing `_side_effect`
   return `_identity_probe(cmd, kwargs)` first when it is not `None`. Keep the existing
   assertions, but take the first and last command from `_plugin_cmds(mock_run)`.
5. Strengthen `test_direct_mail_push_fails`. Use `_mail_fake` with a push result of
   `returncode=1, stderr="permission denied"`, then assert:
   - `success is False`;
   - `"git push failed" in error` and `"permission denied" in error`;
   - no `gh` command appears in `_plugin_cmds(mock_run)`.
6. Strengthen `test_vcs_create_pull_request_pr_create_fails`. Use a dispatching fake
   like the success test's: identity probes first, `git diff --cached --quiet` returns
   rc 1, `gh pr create` returns rc 1 with stderr `"error creating PR"`, everything else
   succeeds. Then assert:
   - `ok is False`;
   - `err` contains both `"gh pr create failed"` and `"error creating PR"`.

Leave the other tests in the file unchanged. They already pass for the right reasons;
for example, `test_vcs_create_commit_success` asserts its real result.

### 2. `tests/test_gh_workspace_claims.py`

1. Change the signature of `test_gh_release_step_skips_on_handoff` to also take
   `monkeypatch: pytest.MonkeyPatch`. At the top of the body, add:
   `staged_handoff = pytest.importorskip("sase.agent.staged_handoff")`. This keeps the
   module importable on PyPI sase 0.18.1, which lacks the module.
2. Replace the
   `(artifacts / ".sase_monitor_pending").write_text("{}", encoding="utf-8")` line with
   a real staged handoff, mirroring sase's
   `test_staged_handoff_skips_release_and_occupant`:
   - call `monkeypatch.setenv("SASE_AGENT", "1")`, because stage admission evaluates
     `is_agent` and CI has no `SASE_AGENT`;
   - then call
     `staged_handoff.write_staged_handoff(staged_handoff.build_plan_staged_record(plan_file="/tmp/plan.md", original_file="/tmp/plan.md", dirty_obligations_at_stage=[], created_at="2026-10-28T12:00:00Z"), artifacts_dir=str(artifacts))`.

   Keep the `SASE_ARTIFACTS_DIR` patch and all existing assertions:
   - `released=false`;
   - `skip_reason=handoff`;
   - the claim for workspace 23 is kept;
   - the occupant record is kept, with the runner pid.

   This was verified by probe: with a staged record in place, `gh_release.main` prints
   `released=false` and `skip_reason=handoff` against sase master.

3. Do not add a leftover-marker counterpart test. That behavior belongs to sase and is
   already covered by sase's own suite. `gh_release` is a thin wrapper.

## Verification

Follow `decisions:guarded-recipes`: run sase-github's guarded `check` recipe only
through `sase tool run`, never as raw `just check`. Per
`decisions:check-full-is-explicit`, no `check-full` lane is involved.

1. **Against sase master (what CI runs; the step that proves the fix).** From the
   sase-github checkout, run the suite with the `.venv` at the root of the sase
   workspace that hosts this linked checkout. If the checkout sits at
   `sase/repos/linked/sase-github`, that venv is `../../../../.venv`. It has sase master
   installed editable, plus `sase_core_rs`.

   Run: `../../../../.venv/bin/python -m pytest -q -p no:cacheprovider`

   Expected: 0 failures and 355 passed. The two strengthened tests are rewritten, not
   added, so the count stays the same. The project's `pythonpath = ["src", "tests"]`
   makes the checkout's `src/sase_github` win over any installed copy.

   Fallback if that venv is missing or lacks `sase_core_rs`: build a CI-faithful
   sase-github `.venv` with
   `SASE_PYTHON_PATH=<sase checkout> SASE_RUST_CORE_PATH=<sase-core checkout> just install-venv`
   (open sase-core with `sase repo open sase-core`), then use `just test`. This is
   slower because maturin does a release build.

2. **Lint and the default dev venv.** From the sase-github checkout, run
   `sase tool run check`, which runs ruff, mypy, and pytest in sase-github's own
   `.venv`. If that venv does not exist yet, `_setup` creates it from PyPI sase,
   currently 0.18.1.

   Expected:
   - lint is clean;
   - all tests pass;
   - on sase < 0.19, `test_gh_release_step_skips_on_handoff` reports SKIPPED;
   - if PyPI has sase ≥ 0.19 by then, nothing is skipped.

3. Confirm `git status` in sase-github shows only the two test files modified.

## Landing

Commits are host-owned (`decisions:host-owned-completion`). Do not commit yourself.
Declare the sase-github repository obligation as a `commit` through `/sase_final`.

Suggested commit subject:
`test: align plugin tests with sase push-identity guard and staged handoffs`. A `test:`
commit does not trigger a release-please bump.

After it lands on sase-github `master`:

- the CI push run should go green;
- release-please will refresh PR #33, re-running its CI.

Neither can be observed inside the implementing turn. The user can re-check with
`actstat`.

## Out of scope

- Any change to sase's guard or handoff code. Both changes are intentional and covered
  by sase's own tests.
- Any change to sase-github `src/`.
- Bumping the `sase>=0.18.0` dependency floor. Runtime code uses no 0.19-only API.
- Publishing sase v0.19.0 to PyPI.
