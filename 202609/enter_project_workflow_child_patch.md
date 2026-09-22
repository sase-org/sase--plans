---
tier: tale
title: Stop project-level workflow children from resolving to a phantom Patch
goal:
  Enter on a project-level plan-family container with a pending gate opens that gate
  directly, because workflow step children of project-level workflows no longer resolve
  the project name as a Patch.
size: small
proposed_by: bbugyi200.athena.0pm
create_time: 2026-09-22 18:27:06
status: wip
---

# Plan: Fix phantom "Go to Patch" target on plan-family Enter for project-level agents

## Problem

On the Agents tab, pressing `<enter>` (`act_on_agent`) on a plan-family container such
as `0pk.f0` (a project-level `#plan` agent with a pending TALE plan-approval gate) opens
the "Act on 0pk.f0" chooser with two targets:

- GATE: `Review tale plan` (correct)
- PATCH: `Go to Patch` / `sase` (wrong: this agent has no Patch)

Because two targets resolve, Enter shows the chooser instead of opening the gate
directly. The expected behavior is a single gate target, so Enter opens the gate
immediately.

## Root cause (reproduced against live agent data)

1. The container row `0pk.f0` is a _project agent_: its `cl_name` equals the project
   name (`gh_sase-org__sase`), so `Agent.is_project_agent` is true.
   `AgentPatchNavigationMixin._resolve_agent_cl_name` handles that correctly for the
   container row itself by consulting `get_meta_patch_name(agent)`, which returns `None`
   (no `meta_patch` in its step output).
2. `resolve_agent_enter_targets` (container scope, in
   `src/sase/ace/tui/actions/agents/_agent_enter_resolver.py`) then falls back to
   `_newest_member_patch(roster, patch_name_for)` over
   `concrete_family_shell_rows(container)`.
3. The first roster row is the plan workflow's concrete planner step (`0pk.f0--plan`,
   `parent_workflow=tmp_…`, `parent_timestamp` = container `raw_suffix`). It is a
   workflow step child, so `_resolve_agent_cl_name` takes the workflow-child branch:
   `_resolve_workflow_child_cl_name` finds the parent workflow row (the container
   itself) and returns the parent's raw `cl_name`, which is the **project name**
   `gh_sase-org__sase`. That branch never applies the project-agent rule that the
   non-child branch does.
4. `_newest_member_patch` accepts that name, and `patch_target` humanizes it to `sase`,
   which produces the phantom `Go to Patch · sase` choice.

The same bug also affects every other caller of `_resolve_agent_cl_name` for a workflow
step child of a project-level workflow: the jump-to-Patch action
(`action_jump_to_agent_patch`), the footer `can_jump` hint in
`_display_detail_footer.py`, and the command-palette `_can_jump_to_patch` in
`src/sase/ace/tui/commands/context.py`. All of them treat the project name as a Patch
name.

The `workflow_state.json` fallback (`_read_workflow_state_cl_name`, used when the parent
row is not loaded) has the same gap: for a project-level workflow its `context.cl_name`
is the project name, and it is returned unchanged.

## Fix

Keep the fix in `_resolve_agent_cl_name`, so every caller benefits, rather than
special-casing the Enter resolver. The rule is that a workflow step child resolves to
the same Patch its parent workflow row would resolve to.

In `src/sase/ace/tui/actions/agents/_patch_navigation.py`:

1. Change `_resolve_workflow_child_cl_name` so that, when it finds the parent workflow
   row in `self._agents_with_children` (same matching as today: not a workflow child,
   `raw_suffix == agent.parent_timestamp`, `workflow == agent.parent_workflow`):
   - if `candidate.is_project_agent`, return `get_meta_patch_name(candidate)`. The
     workflow loader merges step `meta_*` output into the parent row's `step_output`
     (see `_workflow_loaders.py`), so a project-level workflow that did create a Patch
     still resolves to it;
   - otherwise return `candidate.cl_name` as today.
2. In the `workflow_state.json` fallback path (parent row not loaded), return `None`
   when the read `cl_name` equals the child's project name
   (`Path(agent.project_file).parent.name`, the same definition `Agent.is_project_agent`
   uses), because that is a project-level workflow, not a Patch. Guard against an empty
   or missing `project_file`.
3. Keep the existing `None` normalization in `_resolve_agent_cl_name` for
   empty/`unknown`/`~` results. Import `get_meta_patch_name` the same way the existing
   project-agent branch does (lazy import from `._notification_actions`). Update the
   method docstrings to describe the project-level parent rule.

No change is needed in `_agent_enter_resolver.py`. With the resolver fixed, the planner
step resolves to `None`, `_newest_member_patch` finds no Patch, and the container
resolves to just the pending gate target. One target means
`_dispatch_agent_enter_resolution` runs it directly and the footer hint becomes the gate
label instead of `choose action`. For Patch-scoped workflows, the child still resolves
to the parent's `cl_name`, so existing Enter and jump behavior is unchanged.

Boundary note: `_resolve_agent_cl_name` is existing TUI-side Python glue with no
`sase_core` counterpart today. This is a targeted bug fix in place. Moving agent→Patch
resolution into `sase-core` is out of scope.

## Tests

Add regression coverage. Use existing fixtures and helpers where possible.

1. `tests/ace/tui/test_jump_to_changespec.py`, in `TestWorkflowChildResolution` (uses
   `_make_agent` with `project_file="/tmp/projects/myproj/myproj.sase"`):
   - A child whose loaded parent is a project-level workflow (`cl_name="myproj"`, no
     `step_output`) resolves to `None`.
   - Same shape, but the parent has `step_output={"meta_patch": "new_feature"}`: the
     child resolves to `"new_feature"`.
   - The `workflow_state.json` fallback (parent not loaded, mirroring
     `test_child_falls_back_to_workflow_state_json`) whose `context.cl_name` equals the
     project name (`"myproj"`) resolves to `None`.
   - `action_jump_to_agent_patch` with such a child selected emits the `No Patch`
     warning instead of navigating.
   - The existing Patch-scoped tests (`test_child_resolves_to_parent_cl_name`,
     `test_can_jump_true_for_workflow_child_valid_parent`, and the rest) must keep
     passing unchanged.
2. `tests/ace/tui/test_agent_enter_targets.py`: an end-to-end resolver regression for
   the reported scenario, using the **real** mixin resolver (a small
   `AgentPatchNavigationMixin` subclass like `_PatchApp` in
   `test_patch_mixin_rejects_running_marker`, with `_agents_with_children` populated) as
   `patch_name_for` instead of the module's `_patch_name_for` stub:
   - Build a project-level family container: `cl_name` equal to its project directory
     name, `workflow`/`raw_suffix` set, and family-root markers as in the existing
     `_family` helper. Give it a workflow step child member whose
     `parent_workflow`/`parent_timestamp` point at the container, and a pending gate
     member (`_gate_row`).
   - Assert `resolve_agent_enter_targets(container, …)` yields exactly the gate target,
     with no `kind == "patch"` target, and that
     `enter_action_label_for_targets(resolution.targets)` is not `"choose action"`.
   - Preferably model the member as the concrete planner step (a `runtime_children`
     workflow step child with `step_type="agent"` on a plan family root), which is the
     real roster shape. If that fixture proves awkward, the regression is still valid as
     long as the member reaches the roster and goes through the workflow-child branch of
     `_resolve_agent_cl_name`. Confirm the new test fails before the fix and passes
     after.

## Verification

- Run the targeted tests:
  `pytest tests/ace/tui/test_jump_to_changespec.py tests/ace/tui/test_agent_enter_targets.py tests/ace/tui/test_agent_enter_wire.py tests/test_command_context_extraction.py`.
- Run `just fix`, then `sase tool run check`. Do not run `check-full`.
- Rendered TUI output does not change for the snapshot fixtures, so no PNG snapshot run
  is needed unless `just check` indicates otherwise.
