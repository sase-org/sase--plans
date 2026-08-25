---
tier: tale
title:
  Repair the eight just test failures in the agent-catalog row factories and the Agent
  pane mount test
goal:
  just test and just check-full pass again on master, and adding a field to
  AgentCatalogRow can no longer silently break unrelated test modules.
size: medium
proposed_by: bbugyi200.athena.bngrde806zge.f0
---

# Plan: Repair the eight `just test` failures in the agent-catalog row factories and the Agent pane mount test

## 1. Situation

The `just check-full` proc record that triggered this work ran against a tree that
predates commit `2fa772b9` ("feat(ace-tui): add agents pane with detail view and
artifact relations"). **All five failures it reported already pass on current master**
and must not be "fixed" again:

| Reported failure                                                                                                    | Status on master                         |
| ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| `tests/ace/tui/modals/test_saved_query_picker.py::test_picker_is_pr_only_and_bare_digits_only_switch_artifacts`     | passes                                   |
| `tests/ace/tui/test_artifacts_scaffold.py::test_subtab_strip_labels_and_accents_cover_all_panes`                    | passes (assertion updated by `2fa772b9`) |
| `tests/test_command_availability_changespecs.py::test_saved_query_picker_and_slots_are_pr_only`                     | passes                                   |
| `tests/test_command_availability_changespecs.py::test_saved_query_slot_mode_command_is_pr_only`                     | passes                                   |
| `tests/ace/tui/test_patch_query_pane_isolation.py::test_beads_pane_saved_and_history_keys_do_not_touch_patch_query` | passes                                   |

`just test` on current master nonetheless fails. Two independent full runs produced the
identical set of **eight** failures:

```
FAILED tests/test_agent_search_cli.py::test_agent_search_json_filters_with_shared_profile
FAILED tests/test_agent_search_cli.py::test_agent_search_default_scope_excludes_hidden_and_workflow_children
FAILED tests/test_agent_search_cli.py::test_agent_search_pretty_output_smoke
FAILED tests/ace/tui/test_agents_pane_mount.py::test_agents_pane_mounts_activates_and_loads
FAILED tests/ace/tui/test_artifacts_agents_revival.py::test_single_revivable_selected_row_revives_directly
FAILED tests/ace/tui/test_artifacts_agents_revival.py::test_family_row_with_many_revivable_members_seeds_narrow_query
FAILED tests/ace/tui/test_artifacts_agents_revival.py::test_marked_rows_revive_only_revivable_visible_targets
FAILED tests/ace/tui/test_artifacts_agents_revival.py::test_seed_query_filters_revivable_dismissed_rows
```

Those eight are this plan's scope. There are two distinct root causes.

## 2. Root cause A — three required `AgentCatalogRow` fields never reached two test factories

Seven of the eight failures are the same `TypeError`:

```
TypeError: AgentCatalogRow.__init__() missing 3 required positional arguments:
'retry_of_timestamp', 'retried_as_timestamp', and 'retry_chain_root_timestamp'
```

`AgentCatalogRow` (`src/sase/agents/catalog/_models.py`) is a
`@dataclass(frozen=True, slots=True)` whose ~30 fields are all required with no defaults
— deliberately, so every producer must decide every field. Commit `2fa772b9` added
`retry_of_timestamp`, `retried_as_timestamp`, and `retry_chain_root_timestamp` to it and
updated the one production constructor
(`src/sase/agents/catalog/_build.py::_build_row`).

Two later commits — `85e2f768` (`tests/test_agent_search_cli.py`) and `a1e029c6`
(`tests/ace/tui/test_artifacts_agents_revival.py`) — each added a _new_ local
`AgentCatalogRow` factory that was written against the pre-`2fa772b9` field list. They
landed after the field addition, so nothing rebased them and no gate caught it.

There are now **four** hand-maintained copies of essentially the same factory:

| Module                                               | Factory                                                          | Has the three new fields? |
| ---------------------------------------------------- | ---------------------------------------------------------------- | ------------------------- |
| `tests/test_agent_search_cli.py`                     | `_row(name, **overrides)`                                        | **no — broken**           |
| `tests/ace/tui/test_artifacts_agents_revival.py`     | `_row(name, *, kind, state, family, clan, dismissed, revivable)` | **no — broken**           |
| `tests/ace/tui/test_artifacts_pane_state.py`         | `_agent_row(name="0b4--0", **overrides)`                         | yes                       |
| `tests/ace/tui/test_agents_pane_detail_relations.py` | `_agent_row(name, **overrides)`                                  | yes                       |

Adding the missing three keys to the two broken factories fixes the seven failures, but
leaves four copies that will diverge again on the next field. The duplication is the
defect worth removing.

## 3. Root cause B — the Agent pane mount test asserts a load has finished after two fixed pauses

`tests/ace/tui/test_agents_pane_mount.py::test_agents_pane_mounts_activates_and_loads`
fails with:

```
AssertionError: assert <ArtifactsPaneState.LOADING: 'loading'> in
{<ArtifactsPaneState.EMPTY: 'empty'>, <ArtifactsPaneState.RESULTS: 'results'>}
```

This test deliberately uses `startup_policy="real"` so the live, flag-gated
`resolve_artifacts_subtabs()` runs instead of `AcePage`'s fast-startup stub, which means
the Agent pane performs a genuine asynchronous catalog load. The test then asserts the
terminal pane state after exactly two `await page.pause()` calls:

```python
await page.press(page.artifacts_digit("agents"))
await page.pause()
pane = page.query_one_widget("#artifacts-agents-pane", ArtifactsAgentsPane)
assert pane.artifacts_active is True
assert pane.first_activation_count == 1
await page.pause()
assert pane.pane_state() in {ArtifactsPaneState.RESULTS, ArtifactsPaneState.EMPTY}
assert pane.snapshot is not None
```

Two pauses are enough on an idle machine — the test passes standalone and passed in a
`-n 14` run of `tests/ace/tui` alone — but not under the full suite, where it failed in
both full runs. A fixed pause count is a speed assertion, not a correctness assertion;
the pane state is the thing under test, so the test must wait for it.

The repository already owns the idiom for exactly this:
`page.wait_for(predicate, timeout=...)` (`src/sase/ace/testing/ace_page.py`) combined
with `LOAD_TOLERANT_TIMEOUT` from `tests/_load_tolerant.py`, whose docstring states
these waits are deadlock detectors rather than speed assertions. See
`tests/ace/tui/test_artifacts_files_loading.py` and
`tests/ace/tui/test_residual_freeze_soak.py` for existing users.

## 4. Work

### Step 1 — add one shared catalog-row factory

Create `tests/_agent_catalog_helpers.py`, following the existing
`tests/_agent_revive_helpers.py::make_agent` shape (module docstring,
`from __future__ import annotations`, a `make_*(**overrides)` function over a `defaults`
dict).

Export `make_agent_catalog_row(name: str, **overrides: Any) -> AgentCatalogRow` that
builds a `defaults` dict listing **every** `AgentCatalogRow` field explicitly, applies
`defaults.update(overrides)`, and returns `AgentCatalogRow(**defaults)`.

Pick neutral base defaults (the concrete per-module values are supplied by the wrappers
in Step 2), including the three fields this plan is about:

```python
"retry_attempt": None,
"retry_of_timestamp": None,
"retried_as_timestamp": None,
"retry_chain_root_timestamp": None,
```

This module is the single place a future `AgentCatalogRow` field must be added.

### Step 2 — route all four existing factories through it

Keep each module's _public_ factory name and signature exactly as they are today, so no
call site changes and no test semantics move. Each becomes a thin wrapper that passes
its module-specific defaults as overrides:

- `tests/test_agent_search_cli.py::_row` — preserve
  `canonical_global_name=f"bbugyi200.athena.{name}"`, `kind=("agent",)`,
  `project="gh_sase-org__sase"`, `state="active"`, `raw_suffix="20260801000000"`,
  `artifacts_dir=f"/artifacts/{name}"`, `model="gpt-5"`, `llm_provider="codex"`,
  `status="DONE"`, `started_at="2026-08-01T00:00:00+00:00"`, `from_artifact_index=True`,
  and its `**overrides` pass-through. **Fixes 3 failures.**
- `tests/ace/tui/test_artifacts_agents_revival.py::_row` — preserve its keyword-only
  signature (`kind`, `state`, `family`, `clan`, `dismissed`, `revivable`) and every
  derived value: `canonical_global_name=name`, `project="sase"`,
  `raw_suffix=f"{name}-suffix"`, `artifacts_dir=f"/tmp/{name}"`,
  `bundle_path=f"/tmp/{name}.json" if dismissed else None`,
  `status="DONE" if dismissed else None`, `from_artifact_index=not dismissed`,
  `from_dismissed_archive=dismissed`. **Fixes 4 failures.**
- `tests/ace/tui/test_artifacts_pane_state.py::_agent_row` — preserve `name="0b4--0"`
  default, `kind=("member",)`, `project="alpha"`, `state="active"`, `family="0b4"`,
  `role="code"`, `status="RUNNING"`.
- `tests/ace/tui/test_agents_pane_detail_relations.py::_agent_row` — preserve
  `kind=("member",)`, `project="alpha"`, `state="active"`, `status="RUNNING"`, and its
  existing explicit `retry_of_timestamp` call-site overrides.

The last two already pass; touching them is what removes the recurrence risk, and their
tests must still pass unchanged afterwards.

### Step 3 — make the Agent pane mount assertion wait for the load

In `tests/ace/tui/test_agents_pane_mount.py`, replace the second bare
`await page.pause()` with an explicit wait on the state under test, then keep the
existing assertions:

```python
from tests._load_tolerant import LOAD_TOLERANT_TIMEOUT
...
await page.wait_for(
    lambda _state: pane.pane_state() is not ArtifactsPaneState.LOADING,
    timeout=LOAD_TOLERANT_TIMEOUT,
)
assert pane.pane_state() in {ArtifactsPaneState.RESULTS, ArtifactsPaneState.EMPTY}
assert pane.snapshot is not None
```

Note that `page.wait_for`'s predicate receives the page state dict; closing over `pane`
and ignoring the argument is the intended usage. Do **not** weaken the assertions to
accept `LOADING`, and do not delete the `startup_policy="real"` setting — the point of
this test is that the real, flag-gated pane mounts and completes a real catalog load.
Update the test's docstring if the wait changes how it reads.

## 5. Verification

1. Targeted, before anything else, to confirm each root cause is gone:
   ```bash
   .venv/bin/python -m pytest -q \
     tests/test_agent_search_cli.py \
     tests/ace/tui/test_artifacts_agents_revival.py \
     tests/ace/tui/test_artifacts_pane_state.py \
     tests/ace/tui/test_agents_pane_detail_relations.py \
     tests/ace/tui/test_agents_pane_mount.py
   ```
2. The mount fix must be verified **under load**, because that is the only condition it
   fails in. Run the mount test concurrently with a parallel run of another large
   subtree (for example `-n 14 --dist=worksteal` over `tests/ace/tui`), or simply rely
   on step 3 below and confirm the mount test is absent from the summary.
3. Full gate. `just check-full` outruns a single agent turn, so hand it to a monitor
   rather than running it inline:
   ```bash
   sase monitor start --command 'just check-full' \
     --start-status TESTING --stop-status TESTED --next '...'
   ```
   Run `just install` first if the workspace has been idle. Expect `0 failed` and no new
   lint findings. `just lint` was verified clean on master before this plan, so any lint
   finding is caused by this change.

Every one of the eight listed failures must be gone, and the five stale failures from
the original report must still pass.

## 6. Out of scope

- Do not change `AgentCatalogRow` itself. Its fields are intentionally required with no
  defaults so every producer must decide each one; adding `= None` defaults would hide
  the next mismatch instead of surfacing it. The shared factory in Step 1 gives the
  tests one place to update without weakening the model.
- Do not add the three retry-chain fields to `sase agent search`'s JSON output or to
  `_JSON_KEYS` in `tests/test_agent_search_cli.py`. Whether the search command should
  surface retry-chain provenance is a product question, not part of this repair.
- Do not touch the saved-query availability rules in
  `src/sase/ace/tui/commands/availability.py` or
  `src/sase/ace/tui/_app_action_availability.py`. The original report's saved-query
  failures came from a pre-`2fa772b9` tree and do not reproduce.
