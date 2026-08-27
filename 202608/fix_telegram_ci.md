---
tier: tale
title: Fix the two sase-telegram CI test failures
goal:
  sase-telegram CI is green on master because sase no longer requires the host-only
  branch_or_workspace_name helper to write a chat file, and the stale Telegram plan-gate
  fixture stubs the host plan archive the way sase's own tests do.
size: medium
proposed_by: bbugyi200.athena.0eo
create_time: 2026-08-27 08:39:11
status: wip
---

# Fix the two sase-telegram CI test failures

## Diagnosis

`actstat --repo sase-org/sase-telegram` reports the newest settled master commit
(`b37eb45 fix(gates): settle gate shells answered from Telegram`) as a `CI` failure. Run
<https://github.com/sase-org/sase-telegram/actions/runs/33034299329>, job
`check (3.13)`, step 11 (`Run tests`) ends with:

```
FAILED tests/test_custom_gates.py::test_tale_plan_pins_five_control_layout_and_submits_selected_options
  - FileNotFoundError: .../requests/plan/telegram-plan/response.json
FAILED tests/test_gate_shell_settlement.py::test_telegram_answer_settles_a_gate_shell
  - RuntimeError: Failed to get branch_or_workspace_name: /bin/sh: 1: branch_or_workspace_name: not found
2 failed, 582 passed
```

`check (3.12)` was cancelled by matrix fail-fast, so those two tests are the whole
failure set.

Neither failure was caused by a sase-telegram commit. `.github/workflows/ci.yml` in
sase-telegram checks out `sase-org/sase` at **master** and installs it into the test
venv, so every sase master change lands in sase-telegram CI immediately. Both failures
are sase-master-induced.

### Root cause 1 - sase shells out to a host-only helper (`branch_or_workspace_name`)

`sase.history.chat._get_branch_or_workspace_name()` runs the external command
`branch_or_workspace_name` and raises `RuntimeError` on any non-zero exit:

```python
result = run_shell_command("branch_or_workspace_name", capture_output=True)
if result.returncode != 0:
    raise RuntimeError(f"Failed to get branch_or_workspace_name: {result.stderr}")
return strip_reverted_suffix(result.stdout.strip())
```

That command is **not** shipped by sase; it lives in the project owner's personal
`~/bin` dotfiles. It is absent from every CI runner, so any code path that reaches
`save_chat_history()`/`generate_chat_filename()` without an explicit
`branch_or_workspace=` argument raises. Every sase test that hits it mocks it, which is
why the gap stayed invisible until a code path with no explicit value shipped.

`sase.gate_shell.settlement.settle_gate_shell()` is exactly that path. It calls
`_write_settlement_chat()`, which calls
`save_chat_history(..., branch_or_workspace=meta.get("cl_name"), ...)`. A gate-shell
member created from a `base_meta` without `cl_name` (the normal case for the synthetic
members these tests build, and a legitimate production case too) passes `None`, so
`generate_chat_filename()` falls back to `_get_branch_or_workspace_name()` and the whole
settlement blows up.

sase-telegram's new `test_telegram_answer_settles_a_gate_shell` calls
`inbound.resolve_gate_response()`, which calls `settle_gate_shell()` -- so it is the
first sase-telegram test to reach the landmine.

This is not a sase-telegram-only problem. The same root cause is currently red in sase's
own CI (<https://github.com/sase-org/sase/actions/runs/33069690753>) across
`tests/gate_shell/test_settlement_chat.py`,
`tests/gate_shell/test_settlement_followup.py`,
`tests/main/test_gate_shell_handler_cancel.py`,
`tests/gate_conformance/test_gate_shell_conformance.py`,
`tests/test_gate_cli_answer_detach.py`, and `tests/test_user_question_gates.py`. The fix
therefore belongs in sase, and it clears both repos at once.

Behavioural note for the fix: the host helper is a bash script that runs `branch_name`
first and `workspace_name` second. `branch_name` is not an executable on the owner's
`PATH`, so the helper _always_ falls through to `workspace_name`, which is
`pwd | dirname | basename | sd '_\d+$' ''`. The Python fallback must reproduce that and
nothing more, so no chat filename changes on the owner's machine.

### Root cause 2 - a stale sase-telegram plan-gate test fixture

`sase` commit `209375b22 fix(plan): publish archives before approval responses`
(2026-08-22) made the host-owned plan archive **mandatory and pre-terminal** for
commit-plan approvals: `PlanGateAdapter.prepare_terminal_response()` ->
`prepare_plan_terminal_response()` -> `_archive_plan_for_approval(..., required=True)`,
and any failure is re-raised as `GateError` _before_ `response.json` is written.

`archive_approved_plan()` starts by resolving the plan's project from the notification
action data and raises `_PlanArchiveProjectError` when it cannot:

```
no project could be resolved for the approved plan from action data keys
['bundle_path', 'original_plan_file', 'plan_tier', 'request_id', 'request_kind',
 'response_dir', 'session_id']
```

sase-telegram's `test_tale_plan_pins_five_control_layout_and_submits_selected_options`
builds its gate with `create_plan_approval_gate(plan_file, "telegram-plan")`, which
names no project (`agent_project_file`/`project_dir` are both absent) -- a synthetic
fixture, not a product defect. It submits with the `commit` checkbox still selected, so
the archive is required, raises, and `response.json` is never written; the test then
fails on `(bundle / "response.json").read_text()`.

The test's existing `patch("sase.plan_approval_actions.run_plan_side_effects")` is
ineffective here: the Telegram callback goes through `execute_gate_selection()`, which
uses the gate adapter's `prepare_terminal_response()` /
`apply_plan_post_terminal_side_effects()` hooks and never calls
`run_plan_side_effects()`. sase updated its own equivalent tests in the same commit by
stubbing `sase.plan_approval_actions._archive_plan_for_approval` (see
`tests/test_plan_gates_execution.py`); sase-telegram was never updated to match.

## Scope

Two repositories, one coherent change set:

- **sase** (this project's primary repo): remove the hard dependency on the host-only
  `branch_or_workspace_name` helper.
- **sase-telegram** (linked repo): refresh the stale plan-gate test fixture.

Open the linked repo with the `/sase_repo` skill
(`sase repo open sase-telegram -r "..."`) and use only the path it prints.

Out of scope: the other sase master-gate failures visible in `actstat`
(`tests/main/test_pager_command.py`, `tests/test_suite_gate_integration.py`,
`tests/ace/tui/test_artifacts_relation_collapse.py`). They are unrelated to
sase-telegram and must not be touched here; file task beads for them via
`/sase_new_task` if they are still red afterwards.

## Step 1 - sase: make `_get_branch_or_workspace_name()` degrade instead of raising

In `src/sase/history/chat.py`:

1. Keep calling `run_shell_command("branch_or_workspace_name", capture_output=True)`
   first, so a host that _does_ provide the helper keeps winning. Treat a non-zero exit
   **or** empty stdout as "unavailable" rather than fatal - drop the `RuntimeError`.
2. Add a module-private `_fallback_branch_or_workspace_name()` that reproduces the
   helper's effective behaviour in pure Python: the basename of `Path.cwd().parent`, or
   the literal `"unknown"` when that is empty (cwd at filesystem root). Do **not** add a
   git-branch probe: the host helper's `branch_name` half never resolves, so adding one
   would silently change every chat filename on the owner's machine.
3. Keep running the result through the existing `strip_reverted_suffix()`, which already
   performs the `_<N>` strip that `workspace_name` did with `sd`.
4. Give the fallback a docstring that states _why_ it exists: the helper is a host
   dotfile, not a sase-shipped binary, and a chat filename must never be able to abort a
   gate-shell settlement.

Do not add the new private helper to `__all__`.

## Step 2 - sase: update and extend the chat-path tests

In `tests/history/test_chat_paths.py`:

1. `test_get_branch_or_workspace_name_failure` currently asserts the `RuntimeError` that
   Step 1 removes. Rewrite it as a fallback test: mock `run_shell_command` to return
   `returncode=127` with `stderr="command not found"`, `monkeypatch.chdir()` into a temp
   directory whose parent name is known (e.g. `<tmp>/myproj_7/sub`), and assert the
   returned name is the `strip_reverted_suffix`-normalized parent name (`"myproj"`).
   Rename it to describe the new behaviour.
2. Add a case for a succeeding-but-empty helper (`returncode=0`, `stdout="\n"`) taking
   the same fallback path.
3. Keep `test_get_branch_or_workspace_name_strips_reverted_suffix` as is - the
   helper-present path is unchanged.

Add one regression test that pins the _actual_ CI failure rather than only the helper
unit: in `tests/gate_shell/test_settlement_chat.py`, assert that `settle_gate_shell()`
still writes `meta["chat_path"]` when the host helper is unavailable (patch
`sase.history.chat.run_shell_command` to return a 127 result for the duration of the
settlement). This is the test that would have caught the break.

## Step 3 - sase-telegram: fix the stale plan-gate fixture

In `tests/test_custom_gates.py`, inside
`test_tale_plan_pins_five_control_layout_and_submits_selected_options`:

1. Replace `patch("sase.plan_approval_actions.run_plan_side_effects")` with
   `patch("sase.plan_approval_actions._archive_plan_for_approval", return_value=str(gate_home / "archived-plan.md"))`,
   mirroring what sase's own `tests/test_plan_gates_execution.py` does.
2. Add a short comment recording _why_: this fixture's gate names no project, so the
   host-owned archive that sase requires for a commit-plan approval cannot run, and the
   test is about keyboard layout and submitted option ids - not archiving.
3. If dropping the `run_plan_side_effects` patch surfaces an unrelated side effect, keep
   both patches rather than weakening the assertions. Do **not** change what the test
   toggles (`x0` then `s0`) or its `response["selected_option_ids"] == ["commit"]`
   assertion - that selection is the behaviour under test.

Leave `tests/test_gate_shell_settlement.py` alone: Step 1 fixes it, and mocking the
helper there would re-hide the bug for the next surface that settles a gate shell.

## Verification

**sase** (from this workspace checkout):

```bash
just install                 # workspace venvs go stale; do this first
just check
```

Prove the CI condition specifically - run the gate-shell and chat suites with a `PATH`
that cannot see the host helper:

```bash
env PATH="/usr/local/bin:/usr/bin:/bin" .venv/bin/python -m pytest \
  tests/history/test_chat_paths.py tests/history/test_chat_save.py \
  tests/gate_shell/ tests/gate_conformance/ \
  tests/main/test_gate_shell_handler_cancel.py \
  tests/test_gate_cli_answer_detach.py tests/test_user_question_gates.py -q
```

Every one of those must pass. Confirm `branch_or_workspace_name` is genuinely invisible
to that `PATH` first --
`env PATH="/usr/local/bin:/usr/bin:/bin" sh -c 'command -v branch_or_workspace_name'`
must print nothing and exit 127.

Run `just check-full` through the `/sase_monitor` skill with the `TESTING`/`TESTED`
status pair before declaring done - this change touches a helper used by every
`save_chat_history()` call site, so the scoped lane is not sufficient.

**sase-telegram** (in the path printed by `sase repo open sase-telegram`):

`just install` there resolves the local sase checkout through
`linked_workspace_sase_source`, so the telegram venv picks up the Step 1 fix directly
and both failures can be reproduced and cleared end to end locally:

```bash
just install
just check
env PATH="/usr/local/bin:/usr/bin:/bin" .venv/bin/python -m pytest \
  tests/test_gate_shell_settlement.py \
  "tests/test_custom_gates.py::test_tale_plan_pins_five_control_layout_and_submits_selected_options" -q
```

Both previously-failing tests must pass, and `just check` must be clean on both
`ruff`/`mypy` and the full pytest run.

## Landing and follow-up

Both repos are part of this turn's final declaration and each needs its own `commit`
decision through `/sase_final`.

sase-telegram CI installs sase from **master**, so its master build only goes green
after the Step 1 commit lands on sase master. The Step 3 commit is independent and can
land at any time. Once both are on master, re-check with:

```bash
actstat --repo sase-org/sase-telegram -n 3
```

`CI` for the newest sase-telegram master commit must report success. If the sase-side
lint gate flags an unused-symbol or private-misuse error on the new helper, read
`sase memory read symvision:<keyword>` before adding any pragma.
