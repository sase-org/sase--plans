---
tier: tale
title: Fix sase-telegram CI test failures after sase gate changes
goal:
  sase-telegram CI passes again against sase master, with the two broken tests made
  hermetic and adapted to the new gate-shell and plan-credential preflight rules.
size: small
proposed_by: bbugyi200.athena.0pr
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0pr](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0pr.md)
- **COMMITS:**
  - [5deb9e0](https://github.com/sase-org/sase-telegram/commit/5deb9e00739e69ae62fdcc3be221d992fe366c37)
    — test(telegram): isolate git-remote probe and mark shell spec row-managed

# Fix sase-telegram CI: adapt two tests to recent sase gate changes

## Problem

GitHub Actions `CI` has failed on the last three sase-telegram master pushes (`fbe6d1a`,
`5b7f583`, `aff9a23`). In every run, `check (3.13)` fails at the **Run tests** step with
the same two failures (`2 failed, 673 passed`), and `check (3.12)` is cancelled by
fail-fast. Lint and mypy pass.

CI checks out **sase master** into `.sase-deps/sase` (see `.github/workflows/ci.yml`),
so sase-telegram's tests run against the latest sase. Two recent sase commits changed
gate behavior that these tests relied on. In both cases sase's behavior is correct and
intended. The sase-telegram tests need updating; sase itself does not change.

### Failure 1: `tests/test_custom_gates.py::test_tale_plan_pins_five_control_layout_and_submits_selected_options`

```
FileNotFoundError: .../requests/plan/telegram-plan/response.json
  (captured) sase._plan_approval_protocol.PlanApprovalActionError: the git remote
  rejected this host's SSH credential (`Permission denied (publickey)` ...)
```

Root cause: sase added `GateAdapter.preflight_decision` in
`sase/notification_gates/adapters.py`, which `execute_gate_selection` calls before it
accepts a decision. For `plan` gates it calls `preflight_plan_archive_credential`
(`sase/_plan_approval_side_effects.py`). When the selected options include `commit`,
that function runs `sase.service.ssh_agent.probe_git_remote_auth(os.environ)` (a real
`ssh -T git@github.com`). It refuses the answer with `git_credential_denied` when the
remote answers `denied`. On the GitHub runner there is no SSH key, so the probe returns
`denied`. The gate answer is refused and `response.json` is never written. On a
developer machine with a working GitHub key the probe answers `ok`, so the test passes
locally and hides the problem.

The test already patches `sase.plan_approval_actions._archive_plan_for_approval`, but
the preflight runs before that and is not covered. More broadly, no test in the suite
should make a real network SSH call. sase's own suite has an autouse fixture,
`_isolate_git_remote_probe` in its `tests/_conftest_runtime.py`, that stubs the probe to
answer `"unknown"`. `"unknown"` never refuses an answer.

### Failure 2: `tests/test_gate_shell_settlement.py::test_telegram_submits_a_shell_backed_gate`

```
GateError(missing_gate_shell_row): a custom gate declaring a shell block must be
created through sase.gate_shell.create_gate_shell ...
```

Root cause: sase commit `c6807d24c` ("refuse rowless shell-block custom gates") made
`validate_gate_spec` (`sase/notification_gates/validation.py`) reject a `custom` gate
with a `shell` block unless `spec.shell_row_managed` is true. Only the gate-shell
transaction sets that marker. This test calls `create_gate(_spec(..., shell=True))`
directly and then builds the gate-shell row itself with `create_gate_shell_member`. That
is the same pattern sase updated in its own tests
(`tests/test_gate_cli_answer_detach.py` and
`tests/gate_shell/_settlement_followup_helpers.py`). Its fix was to return
`replace(GateSpec.from_mapping(spec), shell_row_managed=True)` for the shell case, with
a comment explaining why.

## Changes (all in the linked `sase-telegram` repo)

Open the repo with `sase repo open sase-telegram` and work in the printed path. Read its
`AGENTS.md` first.

### 1. `tests/conftest.py`: stub the git-remote probe for the whole suite

Add an autouse fixture that mirrors sase's `_isolate_git_remote_probe`:

```python
@pytest.fixture(autouse=True)
def _isolate_git_remote_probe(monkeypatch: pytest.MonkeyPatch) -> None:
    """Keep the suite off the network: no test may run ``ssh -T git@github.com``.

    sase's plan-gate preflight (``preflight_plan_archive_credential``) asks the
    git remote whether this host's credential works before accepting a plan
    approval that selects ``commit``. On CI runners without an SSH key that
    answers ``denied`` and refuses the answer, while developer machines answer
    ``ok``. Consumers resolve ``probe_git_remote_auth`` through its module at
    call time, so answering ``unknown`` (never a refusal) here makes every test
    hermetic.
    """
    monkeypatch.setattr(
        "sase.service.ssh_agent.probe_git_remote_auth", lambda _env: "unknown"
    )
```

Match the existing fixture style in that file (docstring density, placement next to
`_no_service_owned_receiver`). Do not add a per-test patch in `test_custom_gates.py`.
The conftest fixture is the root-cause fix. The existing comment about
`_archive_plan_for_approval` in that test can stay as is.

### 2. `tests/test_gate_shell_settlement.py`: mark the shell spec as row-managed

Update `_spec` the way sase updated its equivalent helpers:

- Import `from dataclasses import replace` and
  `from sase.notification_gates.models import GateSpec` (keep the imports sorted so
  ruff/isort passes).
- Change the return annotation to `dict[str, Any] | GateSpec`.
- In the `if shell:` branch, after `spec["shell"] = {}`, return
  `replace(GateSpec.from_mapping(spec), shell_row_managed=True)` with a short comment.
  The test builds the gate-shell row itself with `_make_gate_shell_member`, so it marks
  the spec the way the production transaction does, and the shell-row guard accepts the
  setup.
- The non-shell path keeps returning the plain dict. `create_gate` accepts either type.

Do not switch this test to the real `create_gate_shell` transaction. That needs a
resolvable project and creator agent, which this test does not set up. The test covers
what Telegram submits, not how the shell row is created.

### 3. Check other callers

Search `tests/` for other `create_gate(` calls whose spec carries a `shell` block (for
example `"shell"` keys in spec dicts) and apply the same marker if there are any.
Currently only `test_gate_shell_settlement.py` fails.

## Verification

1. Install the dev env if needed (`just install`, which uses the sibling local sase and
   sase-core checkouts), then run the guarded check the way the repo's `AGENTS.md`
   requires: `sase tool run check` from the sase-telegram repo root. It must pass: ruff,
   mypy, and the full pytest suite.
2. Reproduce the CI condition to show failure 1 is fixed regardless of host credentials.
   For example, temporarily run the plan test with `GIT_SSH_COMMAND=false` or without an
   SSH agent, or add a local assertion that the stub is active. The test must pass on a
   machine with a working GitHub key and on one without.
3. After the change is committed and pushed, `actstat` should show the next
   sase-telegram `CI` run green for both `check (3.12)` and `check (3.13)`.

## Out of scope

- Do not change sase's preflight or gate-shell validation. Both are intended behavior.
- Do not pin CI to an older sase revision.
- `PTBDeprecationWarning` about `retry_after` in `tests/test_telegram_client.py` is only
  a warning and does not fail CI.
