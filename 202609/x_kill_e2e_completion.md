---
tier: epic
title: Complete Agents-tab x end-to-end regression coverage
goal: Exercise the sase-18d kill and dismissal contract through the real Agents tab,
  durable cleanup, disk reload, and process trees.
parent_bead: sase-18d
phases:
- id: clan_race
  title: Pilot harness and clan removal race
  depends_on: []
  description: 'clan_race: Drive the mounted Agents tab with on-disk agents through
    a clan x, an in-flight load, and fleet reprojection.'
  size: medium
- id: row_lifecycle
  title: Live row, process tree, and restart scenarios
  depends_on:
  - clan_race
  description: 'row_lifecycle: Complete the pilot scenarios for FAILED, DONE, immediate
    exit, and restart; repair any epic-caused defect exposed.'
  size: medium
proposed_by: bbugyi200.athena.sase-18d.land
create_time: 2026-09-24 22:00:12
status: done
bead_id: sase-18d.7
---

- **PROMPT:** [prompts/202609/x_kill_e2e_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/x_kill_e2e_completion.md)
- **PARENT:** [202609/x_kill_removal_reliability.md](https://github.com/sase-org/sase--plans/blob/main/202609/x_kill_removal_reliability.md)
- **BEAD:** [sase-18d.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18d/sase-18d.7.md)

# Complete the missing sase-18d regression phase

The parent epic's phase `sase-18d.6` closed after verifying existing component tests,
but its plan requires a Textual-pilot integration module with on-disk fixture agents and
real process trees. The current coverage is split among
`tests/test_agents_tab_removal_tombstones.py` (fake app and in-memory load),
`tests/test_agent_terminate_processes.py` and `tests/test_kill_durable_termination.py`
(real process or durable transaction), and
`tests/ace/tui/test_agent_member_scope_kill.py` (fake Agents mixin). No test drives the
complete `x` flow across those boundaries. This child plan addresses that gap only. The
parent land agent resumes its close after this child lands through `parent_bead`.

## Phase `clan_race`

Add a Textual `run_test()` module for the mounted Agents tab. Use an isolated temporary
`SASE_HOME`, on-disk agent fixtures, and the real process-tree fixture in
`tests/_agent_process_tree_helpers.py`; reap any live fixture pids during teardown.
Drive the user `x` action and confirmation through the pilot. Let the durable cleanup
payload execute through `apply_cleanup_payload_for_result` while retaining the real UI
tombstone and row publication path. Avoid unbounded sleeps; wait on observable load and
process conditions.

Test a clan container with running members while a load has prepared its roster but has
not applied it. Press `x`, confirm, then let that load apply and trigger fleet
reprojection. Assert the clan and every removed member stay absent after each stage,
including a forced complete-history reload. Assert the durable stage kills and verifies
every targeted fixture process. Reuse established pilot helpers where practical, but do
not substitute a fake mixin app for the mounted Agents tab. Keep heavy disk and process
work off Textual's event loop.

## Phase `row_lifecycle`

Using the same pilot harness, cover the remaining acceptance cases from
`plan:202609/x_kill_removal_reliability.md`:

- A live FAILED runner in retry backoff offers kill confirmation, disappears, has its
  complete process tree terminated, and stays hidden after refresh.
- A live DONE finalizing runner is dismissed and stays hidden without receiving a
  signal.
- Pressing `x` on a running agent and immediately exiting the app still leaves the
  durable payload to verify all fixture processes dead before releasing the workspace
  claim.
- A fresh app instance against the same on-disk state does not show any removed row.

Run the tests against current master behavior, including the newer finished-agent fork
and agent-session query changes. If a test exposes an epic-caused defect, repair it in
this phase and retain the regression. Keep the existing component tests. Run focused
tests and the repository's recorded `sase tool run check` gate; do not run
`just check-full`.

This plan does not include parent bead closure, Symvision whitelist retirement, or
parent plan status updates. Those remain the parent land agent's duties.
