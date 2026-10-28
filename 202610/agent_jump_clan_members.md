---
tier: tale
title: Make every Agents-tab jump reach agents inside clans
goal:
  Every Agents-tab jump reveals and selects its target agent even when that agent is a
  member of a collapsed clan. This starts with the capacity inspector jump that failed
  with "Agent 'sase-1ou.land' not found".
size: medium
proposed_by: bbugyi200.athena.15d
proposed_with: claude/opus
create_time: 2026-10-28 16:00:00
status: wip
---

# Plan: Make every Agents-tab jump reach agents that live inside clans

## Problem

A user opened the capacity inspector (`,l` or the `cap` chip) and pressed Enter on its
first "Using capacity" row, `sase-1ou.land`. That row is the running land agent and a
member of the `sase-1ou` clan, which shows in the `@epic` panel. The TUI showed the
toast `Agent 'sase-1ou.land' not found` and did not move. The reference docs promise
"Press Enter on an agent row to jump to it"
(`docs/reference/tui/agents-tab-grouping.md`, Capacity inspector section).

### Root cause (confirmed)

- `AxeServiceMixin.action_show_capacity_inspector` in
  `src/sase/ace/tui/actions/axe_service.py` calls
  `navigate_to_agent_tab(self, result.name)` from its `on_dismiss`.
- `CapacityInspectorSelection` (`src/sase/ace/tui/modals/capacity_inspector_screen.py`)
  carries:
  - `name`: the agent name. `_display_name` in
    `src/sase/core/runner_slots/_capacity_view_rows.py` returns `agent_name`, or the
    artifact-dir basename. For monitors, `_fallback_name` returns the `monitor_id`.
  - `artifact_dir`: the exact artifact directory.
- `navigate_to_agent_tab`
  (`src/sase/ace/tui/actions/agents/_notification_navigation.py`) treats its second
  argument as a **`cl_name`** (the Patch/project key) and matches
  `agent.cl_name == cl_name`. The real `sase-1ou.land` row has
  `cl_name='gh_sase-org__sase'` and `agent_name='sase-1ou.land'` (verified from its
  `agent_meta.json`), so `resolve_loaded_agent` returns `None` and the toast fires
  before any fold or clan reveal is tried. `artifact_dir` is thrown away. The same
  failure hits every capacity row (e.g. `15c`, whose `cl_name` is
  `gh_bobs-org__bob-cli`). It shows up most for clan members, which are the common
  capacity holders.
- `tests/ace/tui/test_capacity_inspector_openers.py::test_inspector_selection_jumps_to_the_named_agent`
  mocks `navigate_to_agent_tab`, so the field mismatch was never exercised.

### Second, smaller gap: the Runners modal

`RunnerJumpTarget` (`src/sase/ace/tui/modals/runners_modal.py`) carries `raw_suffix`.
The runners `on_dismiss` in `axe_service.py` drops it and calls
`navigate_to_agent_tab(self, result.cl_name, result.pid)`. The pid match is used first,
but if the pid is missing or stale, the `cl_name` fallback picks the first row of the
same project. For a clan member, that row is in the visible tier, which is usually a
sibling, so the jump lands on the wrong row.

### Audit of the other Agents-tab jump entry points

These entry points go through the shared reveal ladder. That ladder is
`prepare_agent_navigation_target` / `reveal_agent_navigation_target` in
`src/sase/ace/tui/actions/navigation/_agent_reveal.py`, reached through
`_try_reveal_agent_row` / `_reveal_agent_row` in `navigation/_member_jump.py` or
`_jump_to_node_identity` in `navigation/_node_jump.py`. It resolves against the complete
`_agents_with_children` roster and expands the member's `clan:` ancestor fold, enclosing
grouping banners, and a collapsed tribe panel. It also switches agent tabs, and the Node
Finder rungs handle `I`-hidden and query-filtered rows. Existing collapsed-clan test
coverage is noted for each:

| Entry point                                 | Code                                                                   | Collapsed-clan test today                         |
| ------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------- |
| Node Finder `"`                             | `_jump_to_node_identity`                                               | yes (`test_node_jump_ladder.py`)                  |
| `JumpToAgent` notification                  | `handle_jump_to_agent` → `jump_to_loaded_agent`                        | yes (`test_notification_jump_to_agent_reveal.py`) |
| Runners modal (pid)                         | `navigate_to_agent_tab`                                                | yes (pid only)                                    |
| `,j` / `,J`                                 | `_unread_navigation.py`                                                | yes (recent fix)                                  |
| Member digit jumps                          | `_member_jump.py`                                                      | yes                                               |
| Procs-pane monitor jump, run-log modal jump | `procs_pane_agent_jump.py`, `agent_run_log_modal.py`                   | yes (`test_agent_tab_cross_nav_modals.py`)        |
| Off-tab timed candidates                    | `_jump_to_off_tab_timed_candidate`                                     | yes (`test_agent_tab_cross_nav.py`)               |
| Artifacts Files "jump to producing agent"   | `_select_file_agent` in `actions/artifacts_files.py`                   | **no**                                            |
| Link-trail hop restore                      | `_restore_agents_link_trail_hop` in `actions/link_trail.py`            | **no**                                            |
| `agent:` link follow                        | `_reveal_loaded_agent` in `actions/_link_follow_targets.py`            | **unverified**                                    |
| `,X` last-launch reveal                     | `_reveal_last_launch_target` in `agent_workflow/_kill_last_launch.py`  | **no**                                            |
| ToolRun owner jump / `OpenToolRun`          | `tool_runs/reveal.py`, `handle_open_tool_run` → `jump_to_loaded_agent` | **unverified**                                    |
| Capacity inspector                          | `navigate_to_agent_tab(self, result.name)`                             | **broken** (above)                                |

## Changes

### 1. Resolve capacity selections by artifact dir, then by name

In `src/sase/ace/tui/actions/agents/_notification_navigation.py`, add
`navigate_to_named_agent(app, name: str, *, artifact_dir: str | None = None) -> bool`.
Place it next to `navigate_to_agent_tab`, re-export it from `_notification_actions.py`,
and add it to that module's `__all__`.

- Call `enter_agents_tab(app)` first, as `navigate_to_agent_tab` does.
- Resolve with `resolve_loaded_agent`, in strict precedence order, running each tier as
  its own `resolve_loaded_agent` call so an exact match on a later roster tier beats a
  looser match on an earlier one. `resolve_loaded_agent` keeps the visible → complete
  roster → `_hideable_agents` tiers and skips clan containers.
  1. **Artifact dir:** compare normalized `agent.artifacts_dir` with normalized
     `artifact_dir`, using pure string normalization
     (`os.path.normpath(os.path.expanduser(...))`). Do **not** call
     `Agent.get_artifacts_dir()` or `resolve_agent_artifact_path()` here. They stat the
     disk or cross into Rust path resolution, and this runs on the UI thread (tui_perf
     rules 1 and 8).
  2. **Agent name:** `agent.agent_name == name`.
  3. **Monitor id:** `agent.monitor_id == name`, because `_fallback_name` uses
     `monitor_id` for monitor records.
- If nothing matches, warn with the existing toast text, `Agent '<name>' not found`, and
  return `False`.
- If a row matches, return `jump_to_loaded_agent(app, target)`. That uses the Node
  Finder ladder, so collapsed clans, collapsed tribe panels, collapsed status banners,
  `I`-hidden rows, query-hidden rows, and other agent tabs are all handled. Its failure
  toasts stay as they are.

In `axe_service.py`, change the capacity `on_dismiss` to
`navigate_to_named_agent(self, result.name, artifact_dir=result.artifact_dir)`.

### 2. Make the Runners-modal fallback exact

- Extend the signature to
  `navigate_to_agent_tab(app, cl_name, pid=None, *, raw_suffix=None)`. Keep the
  positional arguments backward compatible.
- Match order:
  1. pid, as today.
  2. If `raw_suffix` is given:
     `agent.raw_suffix == raw_suffix and agent.cl_name == cl_name`.
  3. The existing `cl_name` fallback.
- The runners `on_dismiss` in `axe_service.py` passes `raw_suffix=result.raw_suffix`.

### 3. Close the collapsed-clan coverage gaps (tests, fix only if red)

Add a regression test for each entry point marked **no** or **unverified** in the audit
table. Each test hides the target only by a collapsed clan fold, runs the entry point,
and asserts two things:

- the selected row (`_agents[current_idx]`) is the target identity;
- the clan fold is no longer `FoldLevel.COLLAPSED`.

If any entry point fails, route it through the shared reveal ladder
(`_try_reveal_agent_row` or `prepare_/reveal_agent_navigation_target`). Do not add a
bespoke clan expander.

### 4. Tests for the new behavior

Put shared fixtures in a public test helper so tests do not import another test module's
private names, which symvision flags as private misuse. Either extend
`tests/ace/tui/_member_jump_navigation_helpers.py`, or move `_Harness` /
`_collapsed_clan` out of `test_notification_jump_to_agent_reveal.py` into a new public
helper such as `tests/ace/tui/_agent_jump_clan_helpers.py`. Give fixture members
realistic identity fields:

- a project-key `cl_name` that differs from `agent_name` (e.g.
  `cl_name="gh_sase-org__sase"`, `agent_name="sase-1ou.land"`, `agent_clan="sase-1ou"`);
- a non-empty `artifacts_dir`.

New or updated tests:

- `navigate_to_named_agent` reveals a collapsed-clan member found by `artifact_dir` and
  selects it. The clan fold is expanded, and only that clan.
- When `artifact_dir` is `None` or matches nothing, the name tier still reveals the
  member. A monitor row resolves through `monitor_id`.
- Two loaded rows share `agent_name` (e.g. a visible sibling or retry with the same
  name): the row whose `artifacts_dir` matches wins over the visible-tier name match.
- A sibling with the same project `cl_name` never wins a capacity jump. This is the
  regression for the original bug.
- An unknown name shows the warning toast `Agent '<name>' not found` and returns
  `False`.
- Runners: pid is `None`, the hidden clan member's `raw_suffix` is given, and a visible
  sibling shares its `cl_name`. The hidden member is selected.
- Update `test_inspector_selection_jumps_to_the_named_agent` in
  `tests/ace/tui/test_capacity_inspector_openers.py`. It should patch
  `sase.ace.tui.actions.agents._notification_actions.navigate_to_named_agent` and assert
  `navigate.assert_called_once_with(app, "worker-a", artifact_dir="/worker-a")`.
- Add one real-app (`AcePage`) test, modeled on
  `test_real_app_notification_jump_expands_collapsed_clan`:
  1. Load clan members with distinct `cl_name` and `agent_name`.
  2. Collapse the clan and switch to another tab.
  3. Call
     `navigate_to_named_agent(app, <member agent_name>, artifact_dir=<member dir>)`.
  4. Assert the Agents tab is current, the member is selected, and the clan fold is
     expanded.

### 5. Docs (Reference tier only; no keymap or config changes)

- `docs/reference/tui/agents-tab-grouping.md`, Capacity inspector section: change "Press
  Enter on an agent row to jump to it." to say that Enter jumps to the agent's
  Agents-tab row, expanding the collapsed clan, tribe panel, or status group that hides
  it.
- `docs/reference/agents/agent-clans.md`: the paragraph that now names only `,j` / `,J`
  should say that every Agents-tab jump also reaches a member row hidden only by a
  collapsed clan, and that the jump expands only the folds on the member's path. List
  the Node Finder `"`, notification jumps, the capacity inspector, the runners modal,
  `agent:` links, and `,j` / `,J` as examples.

## Out of scope

- No Rust core (`sase-core`) changes. Matching a capacity row or a runner row to a TUI
  row is presentation-only navigation glue.
- No keybinding, `default_config.yml`, help-modal, or rendering changes, so no PNG
  snapshot updates.

## Verification

1. Run `just fix`, then `sase tool run check` (the agent-default `just check`). Do not
   run `just check-full`.
2. Manual check in a running TUI:
   1. Collapse a clan that has a running member.
   2. Press `,l` and Enter on that member: the clan opens and the member is selected.
   3. From the Runners modal (`,R`), jump to a clan member: the same member is selected.
