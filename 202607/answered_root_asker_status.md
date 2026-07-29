---
tier: tale
title: Show ANSWERED on a rename-on-attach family root's own asker row
goal:
  The Agents-tab row for a non-plan family root's own main step shows ANSWERED (frozen at the answer time) once its
  question was answered and the work handed off to a later family member, instead of DONE.
create_time: 2026-07-29 06:47:38
status: wip
---

- **PROMPT:** [202607/prompts/answered_root_asker_status.md](prompts/answered_root_asker_status.md)

# Plan: Show ANSWERED on a rename-on-attach family root's own asker row

## Symptom

In `sase ace` → Agents tab, a family root's own row renders `DONE` when it should render `ANSWERED`.

Reproduced live against family `nr` (project `sase`, agent artifacts under `ace-run/202607/29/20260729062253`):

```
sase (RUNNING) ×7 -4 nr
  main (DONE) nr--0      <-- selected row; should be ANSWERED
  sase (RUNNING) nr--1
  diff (DONE) ▾#gh
```

`nr--0` paused on a `sase_questions` gate, the user answered it, and the answer handed off to the continuation `nr--1`.
`ANSWERED` is exactly the transient post-answer state that case is supposed to render (see
`src/sase/ace/tui/models/_agent_status_apply.py:266-271`).

## Root cause

The row that represents a family root's own work is the **concrete `main` workflow-step child**, not a family-member
child. `apply_workflow_child_identity_from_meta()` (`src/sase/ace/tui/models/_loaders/_meta_enrichment_common.py:262`)
stamps that step row with the root's family identity (`agent_name="nr--0"`, `agent_family="nr"`, `role_suffix="--0"`,
`agent_family_role="q"`), but its `child_linkage` stays `WORKFLOW_STEP` because it carries `parent_workflow`
(`src/sase/ace/tui/models/agent.py:370-396`).

There are two `ANSWERED` projections and neither reaches that row:

1. `is_answered_continuation_asker()` (`src/sase/ace/tui/models/_agent_status_family.py:415-428`) rejects it on
   `if not agent.parent_timestamp or not agent.is_family_member_child: return False`. **This single condition is the
   whole bug** — every other precondition already holds on the real row (verified against the live artifacts):
   - `status == "DONE"` ✓
   - `agent_family_role(row) == "q"` ✓ (`--0` parses as `kind="root_question"`, `role="q"` in `sase.plan_chain`; `--1`,
     `--2`, … are the same root-question sequence)
   - `questions_times == [2026-07-29 06:27:18.856220]` ✓
   - `question_response_path` set ✓ — `run_agent_exec_questions.py:165-185` records that field only after a real
     response arrives, so it is a trustworthy "answered" signal
   - `has_later_family_continuation()` ✓ (`nr--1` launched 06:30:58, after the step's 06:23:20)

2. `planner_child_status()` → `"ANSWERED"` (`src/sase/ace/tui/models/_agent_status_family.py:456-457`), reached through
   `sync_planner_child_from_parent()` at `_agent_status_apply.py:154-169`, is gated on `is_root_plan_workflow(parent)`.
   That is `False` here: the root is a _rename-on-attach_ root, not a plan-chain root. `promote_agent_to_family()`
   (`src/sase/agent/_family_promotion.py:136-157`) derives the `--0` generic suffix for an agent with no plan/approve
   metadata and explicitly pops `plan_chain_root`, and `canonical_plan_chain_suffix("--0") != "--plan"`.

So a rename-on-attach family root's asker row keeps the raw `DONE` the runner recorded when the question round finished.
A plan-chain (`--plan`) root never shows the bug: path 2 covers it, and when no concrete `main` step is loaded,
`ensure_synthetic_planner_children()` fabricates a _family-member_ row that path 1 then covers.

### Why the existing tests missed it

Every root in `tests/test_agent_loader_status_override_questions.py` sets `plan_chain_root=True`, and the generic-suffix
cases never supply a concrete `main` workflow-step row — so the row under test is always a family-member child and
always takes one of the two working paths.

## Change

### 1. `src/sase/ace/tui/models/_agent_status_family.py`

Add a sibling predicate next to `is_answered_continuation_asker()` (all imports it needs — `agent_family_role`,
`canonical_plan_chain_suffix`, `is_main_workflow_agent_step`, `is_root_plan_workflow`, `root_child_suffix`,
`has_later_family_continuation` — already exist in this module):

```python
def is_answered_root_asker_step(
    agent: Agent,
    parent_by_suffix: dict[str, Agent],
    children_by_parent: dict[str, list[Agent]],
) -> bool:
    """Return True for a family root's own step whose answer handed off.

    A rename-on-attach root's own work renders as its concrete ``main``
    workflow step, which is a workflow-step child rather than a family-member
    child, so :func:`is_answered_continuation_asker` skips it. Plan-chain roots
    are excluded because :func:`planner_child_status` already owns that row.
    """
    if agent.status != "DONE":
        return False
    if not agent.parent_timestamp or not agent.is_workflow_step_child:
        return False
    if not is_main_workflow_agent_step(agent):
        return False
    if agent_family_role(agent) != "q":
        return False
    if not agent.questions_times or not agent.question_response_path:
        return False
    parent = parent_by_suffix.get(agent.parent_timestamp)
    if parent is None or not parent.is_family_root_entry:
        return False
    if is_root_plan_workflow(parent):
        return False
    if canonical_plan_chain_suffix(agent.role_suffix) != root_child_suffix(parent):
        return False
    return has_later_family_continuation(agent, children_by_parent)
```

Notes on the guards:

- `parent.is_family_root_entry` (`agent.py:62-66`) keeps this to real family roots (`plan_chain_root` or
  `agent_family_role == "root"`); a plain agent that was never promoted cannot match.
- `is_root_plan_workflow(parent)` early-return is what preserves every existing plan-chain behavior —
  `planner_child_status()` stays authoritative for `--plan` roots, including the cases where it must return
  `PLAN APPROVED` / `TALE APPROVED` / `EPIC CREATED` rather than `ANSWERED`.
- `canonical_plan_chain_suffix(agent.role_suffix) == root_child_suffix(parent)` mirrors the guard already used at
  `_agent_status_apply.py:163-166`, so only the root's _own_ slot qualifies.

### 2. `src/sase/ace/tui/models/_agent_status_apply.py`

Import the new predicate and widen the existing `ANSWERED` loop (currently lines 266-271) — do not add a second loop, so
the `stop_time` freeze stays in one place:

```python
    for agent in all_agents:
        if is_answered_continuation_asker(
            agent, children_by_parent
        ) or is_answered_root_asker_step(
            agent, parent_by_suffix, children_by_parent
        ):
            agent.status = "ANSWERED"
            agent.stop_time = max(agent.questions_times)
```

Keep it at its current position in `apply_status_overrides()`: after the `QUESTION` catch-all (so an still-unanswered
row keeps `QUESTION`) and before the plan-handoff and root-mirroring passes.

## Invariants to preserve

These are load-bearing; the implementation must not disturb them.

- **`stop_time` must be set together with the status.** `agent_row_is_in_flight()` (`agent_family_members.py:27-29`) is
  `agent_is_active(status) and stop_time is None`, and `ANSWERED` is in `ACTIVE_AGENT_STATUSES`. Freezing `stop_time` at
  `max(questions_times)` is what keeps the row out of `_is_active_root_mirror_candidate()` (so the family root does not
  mirror a finished asker), keeps `_settled_member_bucket()` reporting `Done` for a non-final member, and gives the
  frozen runtime that `_leaf_runtime_interval()` renders instead of a live tick. This matches
  `_answered_asker_freeze_time()` and the existing `test_apply_status_overrides_...freezes...` coverage.
- **The family root row keeps its own status.** Existing tests assert `parent.status == "DONE"` for answered-question
  families; the root is a container whose visible status is mirrored from live children. Do not set `ANSWERED` on the
  root `Agent`.
- **`is_root_plan_workflow()` must not be broadened.** It gates `ensure_synthetic_planner_children()`,
  `active_approved_plan_handoff_status()`, `is_completed_epic_followup_child()`, and
  `is_completed_plan_handoff_child()`. Widening it to cover rename-on-attach roots would fabricate new synthetic rows
  and re-label members with `PLAN DONE` / `TALE DONE` / `WORKING PLAN` — far beyond this bug.
- **Unanswered questions still read `QUESTION`.** `has_unanswered_completed_question()` returns `False` as soon as
  `question_response_path` is set, and the new predicate _requires_ that field, so the two rules stay mutually
  exclusive.

## Tests

Add to `tests/test_agent_loader_status_override_questions.py` (follow the existing constructor style; a workflow-step
child is an `Agent` with `parent_workflow` + `step_type="agent"` + `parent_step_index=None`, a family-member child
carries only `parent_timestamp`):

1. `test_apply_status_overrides_rename_on_attach_root_step_is_answered` — the regression. Root (`AgentType.WORKFLOW`,
   `status="DONE"`, `raw_suffix="20260729062253"`, `role_suffix="--0"`, `agent_family="nr"`, `agent_family_role="root"`,
   `plan_chain_root=False`, `questions_times=[qt]`, `question_response_path=...`), its concrete `main` step child (same
   identity fields, `step_name="main"`, `parent_workflow` set, `run_start_time` before the continuation), and a running
   `nr--1` family-member continuation. Assert the step row is `ANSWERED` and `stop_time == qt`, and that the root and
   the continuation keep their own statuses.
2. `test_apply_status_overrides_rename_on_attach_root_step_without_response_is_question` — same shape, no
   `question_response_path` and no continuation: the step row must stay `QUESTION`.
3. `test_apply_status_overrides_rename_on_attach_root_step_without_continuation_stays_done` — answered but no later
   family member: stays `DONE` (nothing handed off).
4. `test_apply_status_overrides_plan_chain_root_step_projection_unchanged` — same fixture but `plan_chain_root=True` and
   `role_suffix="--plan"` on the root and step, with the continuation holding an approved plan (`plan_action="tale"`,
   `plan_times`): assert the plan-chain projection is untouched, proving the new rule does not shadow
   `planner_child_status()`.

## Verification

- `just install` first (ephemeral workspace), then `just check`.
- `.venv/bin/pytest tests/test_agent_loader_status_override_questions.py tests/ace/tui/models/ tests/test_agent_model.py tests/test_user_question_response.py tests/test_enrich_agent_pending_question.py`
  must pass — these hold the prior question/plan-chain status fixes.
- `just test-visual` — `ANSWERED` renders in bright azure (`_agent_list_render_agent.py:368`), so confirm no PNG golden
  shifts (and do not accept snapshot updates unless a golden genuinely covers this scenario).
- Manual: open `sase ace`, select the `main` row of a family whose root asked a question that was answered and
  continued; it must read `ANSWERED` with a runtime frozen at the answer time, and the family root row must keep
  mirroring its live child.

## Out of scope

- Rename-on-attach families also miss the plan-handoff labels (`PLAN DONE` / `TALE DONE` / `WORKING PLAN`) for the same
  `is_root_plan_workflow()` gate — in the live `nr` family, `nr--1` reads `DONE` and `nr--code` reads `RUNNING`. That is
  a separate, larger behavior question; do not fold it into this fix.
- The case where a rename-on-attach root has no concrete `main` step row loaded (the root row itself then stands in as
  the lane member). Unlike plan-chain roots there is no synthetic asker row to carry the label, and setting `ANSWERED`
  on the container row would conflict with root mirroring.
