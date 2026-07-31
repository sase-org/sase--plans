---
tier: tale
title: TaskTriage gate kind end to end
goal: Ready task beads can be launched or canceled through one trusted human triage gate on ACE and mobile surfaces.
bead: sase-bg.8
create_time: 2026-07-30 21:09:02
status: wip
---

- **PROMPT:** [202607/prompts/task_triage_gate.md](prompts/task_triage_gate.md)
- **PARENT:** [202607/task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202607/task_beads.md)
- **BEAD:** [sase-bg.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bg/sase-bg.8.md)

# TaskTriage gate kind end to end

## Goal

Implement phase bead `sase-bg.8` from the task-beads epic: add a trusted first-class `task_triage` notification gate
whose default `launch` branch starts the selected task bead through the detached task-launch service (forwarding
optional feedback), whose `close` branch cancels the bead with the required feedback as its close reason, and whose
notification is actionable in ACE and mobile clients.

## Context and constraints

- The gate is privileged and human-only: register kind `task_triage`, action `TaskTriage`, sender `bead-task-triage`,
  pending-action kind `task_triage`, and `auto_policy="forbidden"`.
- The canonical query is `launch OR close`, the primary/default branch is `("launch",)`, launch feedback is optional,
  and close feedback is required.
- The payload owns the task bead identity (`bead_id`, project key, and title). The bundle owns command shims plus a
  Markdown preview containing the task description and notes. Commands only validate their input and return typed JSON;
  host side effects run after the durable response is written.
- Launch must reuse `sase.bead.task_launch`: resolve the payload project to its canonical primary checkout, map the gate
  response source to a detached-task origin, submit/reuse `sase bead work <id> --yes-to-all`, forward optional feedback,
  and persist the returned background task id into `response.json`.
- Close must be in-process and use the same locked mutation, commit, and push semantics as `sase bead close`, with
  `resolution=canceled` and the required feedback as `reason`. Project resolution must be explicit and must not mutate
  process-global cwd.
- ACE must keep all bundle loading and verification off the Textual event loop by reusing the current pump-free generic
  gate modal path. `TaskTriage` needs its actionable/dismissal handling, badge/icon, HITL tab membership, and debug
  projection. Mobile must admit it through the common neutral-gate executor.
- Preserve current behavior for plan, epic, question, launch, HITL, and custom gates, including custom gates'
  neutral-only presentation rule.

## Implementation

1. Add `src/sase/bead/task_gate.py` as the canonical TaskTriage contract. Define stable option/resource constants; build
   and create the gate spec; render the bead-detail preview; generate hashed Python command shims; validate command
   stdin and return typed launch/close results; translate a persisted response into its bead/project/action/feedback
   data; resolve the project key through its ProjectSpec; and expose launch and close host helpers. Make the close
   helper use the existing bead mutation API with the canonical close commit message.

2. Extend the bead mutation context in `src/sase/bead/cli_common.py` only as needed to accept an explicit workspace/cwd
   for store resolution, committing, and pushing. Keep the current no-argument CLI behavior unchanged. This gives the
   TaskTriage close side effect a reusable, lock-safe cross-project path without shelling out or changing global cwd.

3. Register and enforce the privileged gate kind in `src/sase/notification_gates/adapters.py`,
   `src/sase/notification_gates/kind_validation.py`, and `src/sase/notification_gates/validation.py`. Validate the exact
   query, primary branch, payload, preview/command resources, feedback modes, and adapter-owned command content. In
   `GateAdapter.apply_side_effects`, dispatch TaskTriage responses through `task_gate.py`; write `task_launch_task_id`
   atomically for a launch, or perform the typed canceled close for a close.

4. Wire the action through ACE without adding synchronous I/O to handlers:
   `src/sase/ace/tui/actions/agents/_notification_modal_flow.py` should keep the row unread/pending and dispatch
   `TaskTriage` to `handle_custom_gate`; `_notification_custom_gate.py` should accept and verify the registered
   TaskTriage bundle, then derive controls from its branch structure exactly as it does for custom gates. Add TaskTriage
   to pending-action dismissal confirmation, the HITL tab, badge/icon tables, debug kind projection, and shared priority
   classification where user-attention gate actions are enumerated.

5. Add `TaskTriage` to `src/sase/integrations/_mobile_notification_actions.py` with action kind `task_triage`,
   preserving the generic selected-options/feedback executor path.

## Tests and verification

- Add focused TaskTriage tests covering the canonical spec and preview, adapter registration and automatic-resolution
  rejection, kind validation against forged commands/payload/branches, real command execution and response translation,
  detached launch submission with feedback/origin/cwd plus response task-id persistence, and canceled close with the
  exact feedback reason and canonical commit request.
- Extend ACE tests to prove `TaskTriage` loads in the generic modal off-pump, dispatches from the notification modal,
  remains protected as a pending action, appears in the HITL tab with the intended badge/icon, and retains the existing
  custom-gate behavior.
- Extend mobile and notification-priority tests to prove TaskTriage is selectable, reports `action_kind="task_triage"`,
  and is surfaced as requiring user attention.
- Run the focused gate, ACE, mobile, and bead mutation tests first. Then run `just install` (required for this ephemeral
  workspace) followed by `just check`. Re-run any affected focused tests after fixes and confirm the worktree contains
  only the intended implementation and test changes before closing `sase-bg.8` with a note naming the successful
  verification.
