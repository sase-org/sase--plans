---
tier: tale
title: Make sase-telegram's test suite hermetic against ambient SASE agent env
goal:
  A sase-telegram pytest run inside a live SASE agent never registers real gate shells,
  notifications, or feature-flag state on the host.
size: small
proposed_by: bbugyi200.athena.0kw
create_time: 2026-09-14 15:19:38
status: wip
---

# Make sase-telegram's Test Suite Hermetic Against Ambient SASE Agent Env

## Problem

On 2026-09-14 the `0kv` agent ran sase-telegram's full test suite (twice) while
implementing an approved plan. Each run leaked one **real** launch-approval gate shell
into the live project artifact store (`0kv--gate`, `0kv--gate-0`), attributed to the
running agent family. Both later settled as `lost` / "gate bundle unreachable" and
surfaced as an apparent `0kv` failure, even though the agent's actual work succeeded.

Root cause chain:

1. `tests/test_custom_gates.py::test_launch_approval_uses_the_same_singleton_renderer`
   (sase-telegram) calls
   `sase.agent.launch_request.create_launch_approval_request(...)`.
2. That function branches on `running_agent_context_requires_launch_approval()`, which
   is `bool(os.environ.get("SASE_AGENT"))`. A pytest process spawned by a live SASE
   agent inherits `SASE_AGENT` plus the whole agent-context/claim environment.
3. Under that env the call takes the `create_gate_shell(...)` branch instead of the
   plain `create_gate(...)` branch. The test's `gate_home` fixture monkeypatches the
   gate bundle and notification store module attributes into `tmp_path` (so the bundle
   landed in a pytest tmpdir), but gate-shell registration derives its target from the
   ambient claim env vars, not those module attributes — so a real gate-shell record and
   notification id were written under the live project's artifacts store.
4. When pytest's tmpdir was cleaned up, the bundle path in the real record dangled. The
   host reclaim chop (`sase.gate_shell.reclaim._reclaim_one`) found `bundle.is_dir()`
   false and settled both shells `gate_state="lost"`, reason "gate bundle unreachable".

This is the second inherited-agent-env test bug in this file in one day: commit
`0ec0562` ("test: isolate sudo gate flag override from env", sase-telegram) added a
one-off `monkeypatch.delenv(SASE_FEATURE_FLAGS_ENV, ...)` because inherited
`SASE_FEATURE_FLAGS` shadowed `override_flags` in
`test_registry_declared_generic_forms_render_keyboards`.

The main sase repo already solves this class of bug wholesale: its autouse
`_clear_agent_env_vars` fixture in `tests/_conftest_environment.py` scrubs the ambient
agent/launch/proc/chop env before every test. sase-telegram has **no**
`tests/conftest.py` at all, so its suite is only hermetic when run outside an agent
(which is why CI never caught this).

## Goal

Any sase-telegram pytest run — including one spawned inside a live SASE agent — must
never read the surrounding agent context and never mutate real host state (gate-shell
records, notifications, feature-flag resolution).

## Changes

All changes live in the linked repo `sase-telegram`. Open it with
`sase repo open sase-telegram -r "<reason>"` and work only under the printed path.

### 1. Add `tests/conftest.py` with an autouse agent-env scrub fixture

Create `tests/conftest.py` containing an autouse fixture modeled on the sase repo's
`_clear_agent_env_vars` (see `tests/_conftest_environment.py` in the main sase repo for
the reference implementation and docstring rationale). It must `monkeypatch.delenv`
(raising=False):

- every env var with these prefixes: `SASE_AGENT_`, `SASE_LINKED_REPO_`,
  `SASE_MONITOR_DELIVERY_`, `SASE_PROC_`, `SASE_SIBLING_REPO_`
- these exact names: `SASE_AGENT`, `SASE_ARTIFACTS_DIR`, `SASE_BEAD_ID`,
  `SASE_CHOP_LUMBERJACK`, `SASE_CHOP_NAME`, `SASE_CHOP_PROMPT_HASH`, `SASE_CHOP_RUN_ID`,
  `SASE_LINKED_REPOS_JSON`, `SASE_MONITOR_CONTINUATION`, `SASE_SIBLING_REPOS_JSON`,
  `SASE_MODEL_ALIAS_OVERRIDES`, `TMUX_PANE`
- the feature-flag transport var via the canonical constant:
  `from sase.feature_flags import SASE_FEATURE_FLAGS_ENV`
- the workspace pin vars via the canonical constant:
  `from sase.env_contracts import WORKSPACE_PIN_ENV_VARS`

Give the fixture a docstring explaining the concrete failure this prevents (agent-run
suites registering real gate shells / notifications and inheriting feature-flag
overrides). A test that intentionally exercises agent-context behavior can still
`monkeypatch.setenv("SASE_AGENT", ...)` in its own body, since autouse fixtures run
first.

### 2. Remove the now-redundant one-off delenv

In `tests/test_custom_gates.py`, delete the
`monkeypatch.delenv(SASE_FEATURE_FLAGS_ENV, raising=False)` line inside
`test_registry_declared_generic_forms_render_keyboards` (added by `0ec0562`), and drop
the `SASE_FEATURE_FLAGS_ENV` import and the test's `monkeypatch` parameter if they
become unused. The conftest scrub now covers it.

### 3. Regression assertion on the launch-approval test

In `test_launch_approval_uses_the_same_singleton_renderer`, immediately after the
`create_launch_approval_request(...)` call, assert `result.gate_shell_creation is None`.
`LaunchRequestCreationResult.gate_shell_creation` is only populated on the agent-context
`create_gate_shell` branch, so this assertion fails loudly if ambient agent env ever
reaches this test again instead of silently writing real gate-shell records.

### 4. Environment canary test

Add a small `tests/test_conftest_env.py` (or extend an existing suitable module) with a
test asserting the scrub is active, e.g. `assert "SASE_AGENT" not in os.environ` and
`assert SASE_FEATURE_FLAGS_ENV not in os.environ`. This documents the hermeticity
contract and catches accidental removal of the conftest fixture.

## Non-Goals / Notes

- No sase (core) repo changes. The env-keyed gate-shell branch is intended behavior for
  real agent turns; the defect is purely test hermeticity in the plugin repo.
- A shared, importable scrub fixture exported from the `sase` package (so plugin repos
  don't hand-copy the env list) was considered and deliberately deferred: sase-telegram
  is currently the only plugin repo whose tests exercise gate-creating sase code paths
  (sase-github's tests do not), so there is no second consumer yet.
- No cleanup of the two settled `0kv--gate*` records is needed; they are terminal
  history and harmless.

## Verification

From the opened sase-telegram checkout:

1. `just test tests/test_custom_gates.py tests/test_conftest_env.py` — focused pass.
2. `just check` — the repo's required full gate (ruff, mypy, full pytest) must pass.
3. Hermeticity spot-check: run the launch-approval test with the agent env simulated,
   e.g.
   `SASE_AGENT=1 just test tests/test_custom_gates.py::test_launch_approval_uses_the_same_singleton_renderer`,
   and confirm it passes and that no new gate-shell artifact directory appears in the
   live project artifacts store during the run.
