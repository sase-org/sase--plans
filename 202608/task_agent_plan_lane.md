---
tier: tale
size: medium
title: Show both BEAD and PLAN lanes for task-bead agents that authored a plan
goal:
  Give a task-bead agent that authored a plan both a BEAD lane and a PLAN lane in the
  Agents metadata SASE CONTEXT section, by resolving its authored plan the same way an
  epic phase row already does instead of hard-coding the task role's associated plan to
  None.
proposed_by: bbugyi200.athena.x1
create_time: 2026-08-10 09:59:06
status: wip
---

# Show both BEAD and PLAN lanes for task-bead agents

## Problem

In the ACE Agents tab, the metadata panel's `SASE CONTEXT` section renders only a `BEAD`
lane for an agent launched by `sase bead work` against a **task** bead, even when that
agent authored and submitted a plan (`#plan`) in the same run. The `PLAN` lane is
missing, so the plan the agent produced is invisible on its own row.

Reproduced live on a task-bead agent whose `agent_meta.json` contains all of:

- `bead_id: <task_id>` (an `IssueType.TASK` bead; no `epic_bead_id` / `phase_bead_id`)
- `plan_path` (the machine-local plan archive under `~/.sase/plans/<yyyymm>/`)
- `sdd_plan_path` (the committed copy in the `plans` sidecar)
- `plan_submitted_at`, `plan_approved: true`, `plan_action: "tale"`,
  `plan_committed: true`
- `agent_family_role: "root"`, `role_suffix: "--plan"`

Its panel shows `SASE CONTEXT` with `BEAD · ◆ task <task_id>` and no `PLAN` lane.

## Diagnosis

### The rendering layer is not the cause

The natural suspicion — that `SASE CONTEXT` cannot show `BEAD` and `PLAN` at the same
time — is wrong. Both lanes are independent throughout the render path:

- `src/sase/ace/tui/widgets/prompt_panel/_agent_context.py:108` iterates
  `CONTEXT_LANE_ORDER` (`BEAD`, `PLAN`, `ARTIFACTS`, `MEMORY`, `SKILLS`, `WORKSPACES`)
  and appends every lane whose renderer produced text. Nothing suppresses one when the
  other is present.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py:270-284` builds
  `ResponsiveBeadSection` and `ResponsivePlanSection` independently from
  `summary.bead_summary` and `summary.associated_plan`, and
  `_agent_display_header.py:376-380` registers both responsive ranges.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_sase_context.py:358`
  (`test_agents_phase_family_bead_and_plan_context_png_snapshot`) already pins a golden
  where an epic **phase** `--plan` root agent renders `BEAD` **and** `PLAN` together.

### The real cause: the task role hard-codes `associated_plan=None`

Plan/bead enrichment is resolved once in
`src/sase/ace/tui/models/agent_associated_plan.py::resolve_agent_plan_enrichment`, which
returns an
`AgentPlanEnrichment(role, phase_bead, associated_plan, resolved_plan_paths)`. Its own
docstring in `src/sase/ace/tui/models/_agent_associated_plan_types.py:67-75` states that
BEAD and PLAN are independent relationships and a row may carry both.

Only the phase branch honors that. `_agent_associated_plan_phase.py`'s
`resolve_phase_plan_enrichment` resolves the parent-epic BEAD and then calls
`_resolve_phase_authored_plan` (`_agent_associated_plan_phase.py:141-223`) to attach a
distinct authored PLAN.

The task branch never does. A task worker named `<task_id>` is classified by
`_initial_agent_plan_role` (`agent_associated_plan.py:249-269`) as:

- `"land"` when the derived bead id has no dot (`sase-cj`), or
- `"ambiguous"` when it is dotted (`sase-cj.4`),

and `_is_explicit_land_role` is `False` (no `epic_bead_id`, name does not end in
`.land`). Both branches therefore run the bounded bead lookup, see `IssueType.TASK`, and
return early:

- `agent_associated_plan.py:119-125` —
  `return _AgentPlanEnrichment("task", association.bead_summary, None, ())`
- `agent_associated_plan.py:137-143` — the identical early return for dotted task ids

Both hard-code `associated_plan=None` and `resolved_plan_paths=()`, before
`_direct_plan_reference(agent)` (which reads `plan_path` / `archived_plan_path` /
`sdd_plan_path` / `plan_action` / `plan_committed`) is ever consulted. The agent's
authored plan is therefore dropped by the model, not by the renderer.

Confirmed by direct execution against `resolve_agent_plan_enrichment` with the live
agent's field values and a stubbed task issue:

```
role: task
bead: ('sase-cj', 'task')
plan: None
resolved_plan_paths: ()
```

and by calling the phase authored-plan resolver on the same agent with
`parent_path=None`, which produces exactly the wanted lane:

```
plan: ('<plan title>', 'tale', 'sase/repos/plans/<yyyymm>/<plan>.md')
```

### Two consequences of the same defect

1. **Missing PLAN lane** (the reported symptom).
2. **`resolved_plan_paths=()`** — `_agent_display_header_summary.py:200-209` uses that
   tuple to remove plan files from the generic `ARTIFACTS` lane. With it empty, a task
   agent's plan can only ever appear as an undifferentiated artifact, never as a plan.

### Related-but-separate behavior that must be preserved

`tests/ace/tui/models/test_agent_associated_plan_roles.py:145-206`
(`test_task_worker_resolves_to_plan_free_task_bead_lane`) asserts that a task worker
with only an ambient `sdd_plan_path` (no `plan_action`, no `plan_committed`) resolves
**no** plan and reads **no** plan file. That contract is correct and must survive: a
task bead has no plan of its own, and the task BEAD lane deliberately renders no plan
row (`_agent_bead_section.py:101-116`). Only a plan the agent itself authored may
appear.

`_resolve_phase_authored_plan` already encodes precisely that rule through its
`has_handoff_evidence` gate (`_agent_associated_plan_phase.py:171-183`): an authored
plan counts only when there is a distinct archived plan path, a `plan_action`, or a
committed SDD handoff from a `code`/`plan`/`feedback` family role. Reusing it keeps the
existing test green while fixing the reported case.

## Approach

Reuse, do not duplicate: promote the phase-only authored-plan resolver into a shared,
role-neutral helper and call it from the task branch with `parent_path=None`. Keep the
task BEAD summary exactly as it is today.

Do **not** widen this to the `land`/`author`/`ordinary` roles: those already resolve a
plan through `_direct_plan_reference`, and changing their bead behavior is out of scope.

## Implementation

### 1. Extract a shared authored-plan resolver

Symvision forbids importing a `_private` symbol across files, so the shared helper must
be public.

In `src/sase/ace/tui/models/_agent_associated_plan_summary.py`, move these three
functions from `src/sase/ace/tui/models/_agent_associated_plan_phase.py` verbatim
(behavior unchanged):

- `_resolve_phase_authored_plan` → rename to public `resolve_authored_plan`
- `_resolve_distinct_authored_reference` (stays private, moves with its only caller)
- `_same_plan_path` (stays private, moves with its callers)

`resolve_authored_plan` keeps the same signature and return type:

```python
def resolve_authored_plan(
    agent: Agent,
    *,
    parent_path: Path | None,
    load_plan_metadata: PlanMetadataLoader,
    resolve_plan_reference: PlanReferenceResolver,
) -> tuple[AssociatedPlanSummary | None, Path | None]:
```

Move the `PlanMetadataLoader` / `PlanReferenceResolver` type aliases (currently defined
in `_agent_associated_plan_phase.py:31-37`) alongside it, or re-declare them where they
are needed; `_agent_associated_plan_phase.py` must keep exporting the alias names it
still uses in its own signatures. Update its docstring so it no longer claims to be
phase-specific: it returns a plan only when the agent has a distinct authored artifact.

`_agent_associated_plan_phase.py` then imports `resolve_authored_plan` from
`._agent_associated_plan_summary` (it already imports `build_associated_plan_summary`,
`direct_plan_reference`, and `effective_commit_state` from that module, so no import
cycle is introduced) and calls it at `_agent_associated_plan_phase.py:127-132` with
`parent_path=parent_path`. Keep `_resolved_plan_paths` in the phase module — it is still
only used there.

### 2. Resolve the authored plan for the task role

In `src/sase/ace/tui/models/agent_associated_plan.py`, add a module-private helper:

```python
def _task_plan_enrichment(
    agent: Agent,
    association: _ResolvedPlanAssociation,
) -> _AgentPlanEnrichment:
    """Pair a task BEAD with a plan the task agent itself authored.

    A task bead never owns a plan, so the only PLAN a task row may show is the
    handoff artifact this agent produced. ``parent_path=None`` keeps the shared
    resolver's evidence gate intact: without a distinct archive, a plan action,
    or a committed SDD handoff, no plan file is read and no lane appears.
    """
    summary, path = resolve_authored_plan(
        agent,
        parent_path=None,
        load_plan_metadata=_load_plan_metadata,
        resolve_plan_reference=_resolve_cached_reference,
    )
    return _AgentPlanEnrichment(
        "task",
        association.bead_summary,
        summary,
        () if path is None else (str(path),),
    )
```

Replace both hard-coded task early returns with a call to it:

- `agent_associated_plan.py:119-125` (name-inferred `land` branch)
- `agent_associated_plan.py:137-143` (`ambiguous` branch)

### 3. Keep a task's bead `design` out of the PLAN lane

In the `source == "bead"` branch of `resolve_agent_plan_enrichment`
(`agent_associated_plan.py:169-186`), return through `_task_plan_enrichment` as soon as
the resolved association reports `role == "task"`, before `association.path` is used.

This is defensive today (the two early returns above catch every task path currently
reachable), but it makes the invariant local and obvious, and it prevents the existing
fall-through at `agent_associated_plan.py:231-246` from ever rendering a task bead's
`design` reference as that task's PLAN while silently dropping its BEAD summary.

Delete `agent_associated_plan.py:194-195`
(`if role == "task": return _AgentPlanEnrichment(role, phase_bead, None, ())`) only if
it becomes unreachable after this change; otherwise route it through
`_task_plan_enrichment` too. Do not leave two different ways for a task to build its
enrichment.

### 4. Cache checks (verify, expect no change)

- `associated_plan_cache_key` (`agent_associated_plan.py:64-80`) already includes
  `plan_path`, `archived_plan_path`, `sdd_plan_path`, `plan_committed`, and
  `plan_action`, so the detail-header summary is invalidated when a task agent's plan
  metadata appears mid-run. Confirm, and only touch it if a new input is introduced.
- `_PLAN_ASSOCIATION_CACHE` entries are keyed by
  `(source, value, project, workspace, workspace_num)`
  (`_agent_associated_plan_paths.py:19-30`). The new call sites reuse the existing
  `authored-archive` / `authored-sdd` sources, which are keyed by plan reference, so no
  cache-key change is needed.

## Tests

All new/updated tests live under `tests/`.

1. `tests/ace/tui/models/test_agent_associated_plan_roles.py`
   - **Keep** `test_task_worker_resolves_to_plan_free_task_bead_lane` unchanged and
     passing: it now pins the "no handoff evidence → no plan read" half of the contract.
     Extend its docstring/comment to say so.
   - **Add** `test_task_worker_with_authored_plan_shows_bead_and_plan`, parametrized
     over `("sase-task", "sase-task.4")` so both the `land`-inferred and `ambiguous`
     early returns are covered. Build a `tale` plan file on disk, an agent with
     `archived_plan_path` + `sdd_plan_path` + `plan_committed=True` +
     `plan_action="tale"`, and a stubbed `IssueType.TASK` issue. Assert
     `role == "task"`, `bead_summary.bead_type == "task"`, `associated_plan is not None`
     with the plan's title/goal and `effective_tier == "tale"`, and
     `resolved_plan_paths == (str(selected_plan),)`.
   - **Add** a pending-approval variant: only `archived_plan_path` set (no
     `sdd_plan_path`, no `plan_committed`, `plan_action=None`), asserting the plan is
     still surfaced and `effective_tier` is `"plan"` (uncommitted archive).
   - **Add** a case proving a task bead's `design` alone never becomes a PLAN: stub an
     `IssueType.TASK` issue whose `design` points at a real plan file, give the agent no
     authored-plan metadata, and assert `associated_plan is None` and
     `resolved_plan_paths == ()` while the BEAD summary is still present.

2. `tests/ace/tui/models/test_agent_associated_plan_phase.py` (and the other phase
   modules) must keep passing unmodified — they are the regression net for the extracted
   helper. Do not change assertions there; if one breaks, the extraction changed
   behavior.

3. Widget-level coverage in `tests/ace/tui/widgets/` (`test_agent_context.py` is the
   right home; follow its existing fixtures): render the header for a task agent with an
   authored plan and assert the `SASE CONTEXT` body contains a `BEAD` lane and a `PLAN`
   lane, in that order.

4. PNG snapshots: no golden change is expected.
   `test_agents_task_bead_notes_png_snapshot`
   (`tests/ace/tui/visual/test_ace_png_snapshots_agents_sase_context.py:282`) stubs the
   enrichment as `_AgentPlanEnrichment("task", bead, None, ())`, so it is unaffected.
   Adding a new task BEAD+PLAN golden is optional; only do it if `just test-visual`
   remains green and the new golden is committed with its source.

## Documentation

`docs/ace.md` currently states the opposite of the fixed behavior in two places; both
must be updated in the same change:

- The `SASE CONTEXT` paragraph near `docs/ace.md:1038-1042` ("When ACE knows a
  planner/author or epic lander's associated plan…"): add that a task worker also shows
  a `PLAN` lane beside its `BEAD` lane when it authored a plan in that run.
- The `**SASE CONTEXT / PLAN**` bullet near `docs/ace.md:3288` ("Shown for the
  epic-authoring planner and epic lander when…"): extend the list of plan-bearing roles
  to include task workers with an authored plan, and state explicitly that a task bead's
  own `design` field is never rendered as a `PLAN` lane.
- While there, correct the `**SASE CONTEXT / BEAD**` bullet near `docs/ace.md:3279`,
  which says the lane is "Shown for an epic phase worker" — it is also shown for task
  workers, with the task field set (`Task Title`, `Description`, optional `Notes`,
  `Size`, optional `+1 Reports` / `+1 Evidence`, `Created`).

Do not edit `CHANGELOG.md` (release-please generates it) and do not edit any file under
`sase/memory/`, `AGENTS.md`, or the generated provider instruction shims.

## Verification

```bash
just install
just check
```

`just check` must pass with no new failures. Additionally run the directly affected
suites explicitly, since the scoped lane is a heuristic:

```bash
.venv/bin/python -m pytest \
  tests/ace/tui/models/test_agent_associated_plan_roles.py \
  tests/ace/tui/models/test_agent_associated_plan_phase.py \
  tests/ace/tui/models/test_agent_associated_plan_phase_metadata.py \
  tests/ace/tui/models/test_agent_associated_plan.py \
  tests/ace/tui/widgets/test_agent_context.py \
  tests/ace/tui/widgets/test_agent_display_bead_section.py \
  tests/ace/tui/widgets/test_agent_display_plan_section.py
```

If `docs/ace.md` or any rendering-affecting default changed, also run `just test-visual`
and confirm no unintended golden drift.

## Out of scope

- Showing a `BEAD` lane for `land`/`author` (epic) rows, which currently show only
  `PLAN`. That is a separate behavior question about epic rows, not a defect in the
  reported case; file a task bead if it should change.
- Any change to how phase rows resolve their parent epic or authored plan.
- Any change to bead storage, `sase bead work`, or agent launch metadata. The fix is
  read-side only: every field it needs is already persisted in `agent_meta.json`.
