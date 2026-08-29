---
tier: tale
title: The gate shell node owns its decision status
goal:
  Selecting a gate option updates only the gate shell node's status; the agent shell
  that created the gate keeps its own terminal status, so a settled plan gate reads DONE
  on the planner row and TALE APPROVED on the gate row and the family node.
size: small
proposed_by: bbugyi200.athena.0g3
---

# Tale: The gate shell node owns its decision status, not the agent shell that created it

## Problem

When a gate option is selected, the decision status lands on **two** rows: the gate
shell node (correct) and the agent shell node of the sase agent that created the gate
(wrong).

Observed on a live plan family (`0g0.w0`) whose plan gate settled on `approve+commit`:

| Row            | Kind             | Status today    | Status it should have |
| -------------- | ---------------- | --------------- | --------------------- |
| `0g0.w0`       | family container | mirrors newest  | mirrors newest        |
| `0g0.w0--plan` | agent shell      | `TALE APPROVED` | `DONE`                |
| `0g0.w0--gate` | gate shell       | `TALE APPROVED` | `TALE APPROVED`       |

The same duplication also produces a transient _inconsistency_: the host writes
`plan_approved` / `plan_action` into the creator's `agent_meta.json` in
`prepare_plan_terminal_response` the moment the option is selected, while the gate
shell's own `gate_state` only flips to `answered` once the shell settles. In that window
the Agents tab shows `0g0.w0--plan (TALE APPROVED)` next to `0g0.w0--gate (TALE)`, which
reads as if the wrong node were being updated.

Because agent family nodes mirror their most recently run shell, fixing the agent shell
row also makes the family node read `TALE APPROVED` whenever the gate is the newest
shell, instead of borrowing the label from a stale planner row.

## Evidence

Measured against the live artifact corpus by loading the real Agents-tab model
(`sase.ace.tui.models.agent_loader.load_all_agents`) and bucketing every row whose
status is a plan/question decision label:

```
Counter({('TALE APPROVED', is_gate=False, has_gate_id=True):  10,   # <- the bug
         ('TALE APPROVED', is_gate=True,  has_gate_id=True):  10,   # correct gate rows
         ('TALE APPROVED', is_gate=False, has_gate_id=False): 10})  # legacy, pre-gate-shell
```

Every one of the 10 wrong rows is a `role=plan`, `role_suffix=--plan` workflow
agent-step row that carries a `gate_id` of its own. Every one of the 10 legacy rows is a
planner row from a family that never created a gate shell and therefore has no gate node
to carry the label.

## Root cause

`approved_followup_planner_status` in
`src/sase/ace/tui/models/_agent_status_family_policy.py` relabels any settled planner
row whose own metadata says the plan was approved:

```python
def approved_followup_planner_status(agent: Agent) -> str | None:
    """Return the sticky approved status for a concrete follow-up planner."""
    if (
        agent.parent_timestamp is None
        or agent.agent_family_parallel
        or agent_family_role(agent) not in PLANNER_FAMILY_ROLES
    ):
        return None
    if not agent.plan_times:
        return None
    if agent.plan_action not in APPROVED_PLANNER_ACTIONS:
        return None
    if agent.plan_action == "tale":
        return TALE_APPROVED_STATUS
    return PLAN_APPROVED_STATUS
```

It is applied from `apply_status_overrides` in
`src/sase/ace/tui/models/_agent_status_apply.py`:

```python
    for agent in all_agents:
        if agent.status in {"DONE", "QUESTION"}:
            approved_status = approved_followup_planner_status(agent)
            if approved_status is not None:
                agent.status = approved_status
```

This predates gate shells and is still correct for legacy families with no gate node. It
is wrong for gate-shell families, where `plan_gate_shell_block` in
`src/sase/plan_shell/create.py` already declares the full decision vocabulary
(`TALE APPROVED`, `PLAN APPROVED`, `PLAN COMMITTED`, `PLAN REJECTED`, `EPIC APPROVED`,
...) on the gate shell itself.

The guard suite that was supposed to pin this contract
(`tests/test_agent_loader_status_override_gate_shell_family.py`, whose module docstring
already promises "the planner member stays `DONE`") builds its planner fixture
**without** `plan_action` / `plan_times`, so `approved_followup_planner_status` returns
`None` there and the real production shape was never measured. That measurement gap is
why the epic `plan:202608/status_strip.md` recorded "planner row: `DONE`" for the
settled `approve+commit` case while production shows `TALE APPROVED`.

## The discriminating signal

An agent that hands a decision to a gate shell records the gate's id **in its own**
`agent_meta.json` before its turn ends:

- `src/sase/axe/run_agent_exec_gate.py` calls
  `update_meta_field(state.current_artifacts_dir, "gate_id", gate_id)` on the
  _creator's_ artifacts dir (and `gate_member_agent_name` alongside it).

That key reaches the row through both loader paths, because both flat-key projections
build a gate `family_shell` from any `gate_*` key:

- Python: `_family_shell_from_flat_keys` in
  `src/sase/core/agent_scan_wire_family_shell.py`.
- Rust: `family_shell_from_object` in `crates/sase_core/src/agent_scan/scanner.rs` of
  the linked `sase-core` repo.

So `agent.gate_id` is already populated on creator rows in the wire path, the snapshot
path, and the filesystem path — verified live: the `0g0.w0--plan` row carries
`gate_id=8b5addc2-...` while `agent.is_gate` is `False` (that property is role-based,
`is_real_gate_member`). No wire schema change and no `sase-core` change is needed.

`gate_id` is durable per row, so the predicate is stable under partial artifact-delta
loads. A sibling-scan over `children_by_parent` would not be:
`load_artifact_delta_agents` runs `apply_status_overrides` over a partial family, and
the relabel is sticky (the merged re-normalization only fires for rows still reading
`DONE` / `QUESTION`), so a delta that happened to exclude the gate row would permanently
strand the planner row on `TALE APPROVED`. Do not implement this with a sibling scan.

## Change

### 1. `src/sase/ace/tui/models/_agent_status_family_policy.py`

Add a named predicate and use it as the first guard in
`approved_followup_planner_status`:

```python
def decision_published_by_gate_shell(agent: Agent) -> bool:
    """Return True when a gate shell, not this row, publishes its decision.

    An agent that creates a gate shell records the gate's id in its own
    ``agent_meta.json`` before handing off, so a creator row carries
    ``gate_id`` without being the gate member (``is_gate`` is role-based).
    From that point the gate shell owns the decision status for its whole
    settled/pending pair, and the creating agent shell keeps its own
    terminal status. Legacy pre-gate-shell families record no ``gate_id``
    and keep the historical planner label, because they have no gate node
    to carry it.
    """
    return bool(agent.gate_id) and not agent.is_gate
```

`approved_followup_planner_status` returns `None` when the predicate holds. Keep every
other clause and the rest of the module unchanged:
`active_approved_plan_handoff_status`, `done_handoff_status`,
`is_completed_plan_handoff_child`, and `is_completed_epic_followup_child` label the
_coder_ rows the gate shell does not replace (`WORKING TALE`, `TALE DONE`,
`EPIC CREATED`) and must keep working.

Update `approved_followup_planner_status`'s docstring to say it is the legacy,
no-gate-shell fallback.

### 2. `src/sase/ace/tui/models/_agent_status_apply.py`

Extend the comment above the `approved_followup_planner_status` pass so it records that
gate-shell families are excluded and why. No behavioural change at the call site.

### 3. Export surface

Only `approved_followup_planner_status` itself consumes the new predicate, so it stays
inside `_agent_status_family_policy.py`. Re-export it through
`src/sase/ace/tui/models/_agent_status_family.py` **only** if a test needs it directly;
prefer driving tests through `_apply_status_overrides` so the contract, not the helper,
is what is pinned. If it is not exported, confirm `just lint`'s symvision gate is clean
for a module-private helper with one in-module consumer.

## Tests

### A. Close the measurement gap in the existing guard suite

`tests/test_agent_loader_status_override_gate_shell_family.py`

Its `_planner_step()` fixture is the shape that hid this bug. Give it the production
shape and add the settled cases:

- Add optional `plan_action` / `plan_times` / `gate_id` parameters to `_planner_step()`
  (default them to today's values so existing cases keep asserting what they assert).
- New: settled `approve+commit` gate whose planner step carries `plan_action="tale"`, a
  non-empty `plan_times`, and the same `gate_id` the gate member carries → assert
  `planner.status == "DONE"`, `gate.status == "TALE APPROVED"`,
  `root.status == "TALE APPROVED"`.
- New: settled `approve` (no commit) gate with `plan_action="approve"` → planner `DONE`,
  gate `PLAN APPROVED`.
- New: _pending_ tale gate whose planner already carries approval metadata (the
  transient window where the host has written `plan_approved` but the shell has not
  settled) → planner `DONE`, gate `TALE`, container `TALE`. This is the exact frame in
  the reported screenshot.
- New regression guard: a planner step with the same approval metadata and **no**
  `gate_id`, in a family with no gate member, still projects `TALE APPROVED` — the
  legacy fallback must not be collateral damage.

### B. End-to-end from artifact markers

New module `tests/test_agent_loader_gate_decision_row_ownership.py`, modelled on
`tests/test_agent_loader_pending_gate_shell.py` (same `_write_json` / `_artifact_dir` /
`_project_file` helper shape, same `load_all_agents(patch_snapshot=[])` entry point).

Write real markers so the assertion runs through the loader, not through hand-built
`Agent` objects:

- Root artifacts dir: `agent_meta.json` with `agent_family_role="root"`,
  `role_suffix="--plan"`, `plan_chain_root: true`, `plan: true`, `plan_submitted_at`,
  `plan_approved: true`, `plan_action: "tale"`, and `gate_id` — mirroring the real
  creator meta captured from a settled family. Plus `workflow_state.json`
  (`status: completed`, `appears_as_agent: true`), `prompt_step_main.json`
  (`step_type: "agent"`, `status: "completed"`, `parent_step_index: null`), and
  `done.json`.
- Gate member artifacts dir: `agent_meta.json` with `agent_family_role="gate"`,
  `role_suffix="--gate"`, `shell_kind: "gate"`, `parent_timestamp` pointing at the root,
  the same `gate_id`, `gate_state: "answered"`, `gate_start_status: "TALE"`,
  `gate_stop_status: "TALE APPROVED"`, plus `workflow_state.json`.

Assert:

- the `--plan` step row is `DONE`;
- the `--gate` row is `TALE APPROVED` with `gate_state == "answered"`;
- the family container row is `TALE APPROVED` (the gate is the newest shell), which is
  the "family node matches its most recently run shell" property from the report;
- with `gate_state: "pending"` instead, the `--plan` row is still `DONE`, and both the
  gate row and the container read `TALE`;
- with a coder member added after the settled gate, the container moves on to
  `WORKING TALE` and the `--plan` row is still `DONE` — proving the fix does not disturb
  the post-approval handoff labels.

Run `_apply_status_overrides` twice over the same rows in at least one case to pin
idempotency, matching the existing convention in
`tests/test_agent_loader_status_override_tale.py`.

## Verification

- `just check` before replying. If it has not been run in this workspace clone before,
  `just install` first (ephemeral workspace clones own an isolated virtualenv).
- Targeted first, for a fast signal:
  `pytest tests/test_agent_loader_status_override_gate_shell_family.py tests/test_agent_loader_gate_decision_row_ownership.py tests/test_agent_loader_status_override_tale.py tests/test_agent_loader_pending_gate_shell.py tests/ace/tui/models`
- `just test-visual`. The change alters one status label, not which rows a family emits,
  and no PNG fixture combines a planner row's `plan_action` / `plan_times` with a
  `gate_id` (only
  `tests/ace/tui/visual/_ace_agents_png_snapshot_family_panel_fixtures.py` sets
  `gate_id`, and only on gate member rows), so goldens are expected to be unchanged. If
  a golden does move, inspect `.pytest_cache/sase-visual/` before accepting anything.
- `just check-full` through the `/sase_monitor` skill with the `TESTING` / `TESTED`
  status pair, never inline. `sase-j0` is the known pre-existing
  `tools/check_test_cost_budgets` failure; confirm any red is that one before treating
  it as this tale's.

### Pre-measured blast radius

Simulating the exact fix (an out-of-tree pytest plugin monkeypatching
`approved_followup_planner_status` to return `None` when
`agent.gate_id and not agent.is_gate`) and running the affected suites produced **zero**
failures:

| Suite                                                                                               | Result                |
| --------------------------------------------------------------------------------------------------- | --------------------- |
| the four `test_agent_loader_*` status-override / gate-shell modules                                 | 37 passed             |
| `tests/ace/tui/models` + epic-created + status-buckets                                              | 772 passed            |
| `pytest tests/ -k "status_override or gate_shell or plan_family or family_members or agent_loader"` | 535 passed, 1 skipped |

Existing fixtures do not set `gate_id` on planner rows, which is exactly why they never
caught the bug — so the new tests in section A are what give the fix teeth. Do not treat
the green baseline as evidence the tests are sufficient.

## Out of scope

Record each of these as a `PROPOSED FOLLOW-UP:` note or, where the implementer judges it
worth tracking, file it through `/sase_new_task`. Do not fix them here.

1. **The question-gate analogue.** `is_answered_continuation_asker` /
   `is_answered_root_asker_step` in `_agent_status_family_policy.py` can label a settled
   asker row `ANSWERED`, which is the question gate shell's `gate_stop_status`. The live
   corpus currently contains **zero** question gate shells and zero `ANSWERED` rows, so
   there is no reproduction to fix against; `is_answered_root_asker_step` already
   excludes plan-chain roots for this reason. Fixing it blind would be a speculative
   change.
2. **Active creator rows under `%auto`.** `plan_enrichment_status`
   (`src/sase/ace/tui/models/_loaders/_meta_enrichment_common.py`) and its integration
   twin `_plan_status` (`src/sase/integrations/_agent_list_entry_builder.py`) both label
   an **active** creator row with the approved status. Under `%auto` the gate
   auto-settles and `GateShellCreation.should_handoff` is `False`, so the creator keeps
   running and does the work itself — a different lifecycle from the settled duplication
   reported here, and one with no wrong row in the current corpus.
3. **`gate_state` leaking onto creator rows.** Because `apply_gate_meta` defaults a
   missing state to `"pending"`, a creator row carrying only `gate_id` also ends up with
   `agent.gate_state == "pending"`. It is inert today — every consumer
   (`agent_row_is_in_flight`, `gate_row_is_settled`, `row_is_family_shell`,
   `agent_groups/_buckets.py`, `agent_time.py`) gates on the role-based `is_gate` first
   — but it is a trap for the next reader.
4. **Memory and decision records.** Do not edit anything under `sase/memory/`. "The gate
   shell owns the decision status" may deserve a decision record, but memory edits
   require the user's explicit approval and this plan does not carry it.

## Non-goals

- No change to `sase-core`. The signal already crosses the wire.
- No change to what the host persists. `plan_approved` / `plan_action` on the creator's
  `agent_meta.json` stay exactly as they are: they are legitimate metadata that
  `_family_progressed_past_plan`, `refresh_agent_plan_path`, the `WORKING TALE` vs
  `WORKING PLAN` choice, and `sase agent stats` all read. Only the **status projection**
  changes.
- No change to `concrete_agent_statuses`' `row_is_family_shell` filter. The container
  gets the gate's status through `_mirror_root_from_child`, not through that filter;
  `plan:202608/status_strip.md` measured this explicitly and warns against touching it.
- No new feature flag. This is a display correction with a legacy-preserving predicate,
  not user-reaching behaviour with an old branch worth keeping alive.
