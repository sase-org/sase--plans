---
tier: tale
size: medium
title: Teach the runner shutdown path that a gate handoff is not a failed run
goal:
  "Stop every gate-shell handoff from reporting its creator agent as a failed run:
  suppress the spurious `user-agent` completion notification, keep the gate handoff
  transcript, write a `completed` done marker, and leave the gate shell's workspace
  claim alone."
proposed_by: bbugyi200.athena.0fb
---

# Teach the runner shutdown path that a gate handoff is not a failed run

## Problem

When an agent creates a gate shell (`/sase_plan` + `sase plan propose`,
`/sase_questions`, `/sase_gate`, HITL, launch approval), the gate shell kills its
creator. The runner adopts the `.sase_gate_pending` marker,
`sase.axe.run_agent_exec_gate.handle_gate_marker` writes a "Gate handoff" transcript and
returns the loop outcome `"gated"`, and the run ends.

`"gated"` was added by `1cb772d9c feat(gate): add gate shell lifecycle` as a copy of the
older monitor handoff outcome `"monitored"` (`run_agent_exec_monitor.py`), but none of
the four downstream call sites that special-case `"monitored"` were taught about it.
Every one of them therefore treats a successful gate handoff as a failed agent run.

Observed on `0f9--plan` (project `gh_bobs-org__bob-cli`, artifacts
`.../ace-run/202608/27/20260827185959`), and reproduced by every `--plan` handoff on
this host today (`0ey.f2`, `0ez`, `0f0`, `0f1`, `0f2`, `0f3`, `0f4`, `0f9`):

1. **Spurious completion notification (the reported symptom).**
   `run_agent_runner_finalize.send_completion_notification` returns early only for
   `outcome in {"plan_rejected", "monitored"}`, so `"gated"` falls through and publishes
   a `sender="user-agent"` notification reading
   `CODEX(gpt-5.6-sol) @0f9 failed: ace(run)-260827_185959`, one second after the gate's
   own `sender="plan"` "Tale ready for review" notification. Two notifications for one
   decision, and the extra one says the agent failed.
2. **Empty transcript.** `run_agent_exec_finalize.finalize_loop` reuses the handoff
   transcript (`_last_saved_chat_path`) only for `"monitored"`. For `"gated"` it calls
   `save_chat_history` a second time with `response_content = ""`, overwriting the "Gate
   handoff" body `handle_gate_marker` just wrote. Every gated chat file on this host has
   an empty `## Response` section.
3. **Wrong done marker.** `finalize_loop`'s `{"completed", "monitored"}` branch writes
   `outcome: "completed"` plus the full artifact collection (step output, diff, markdown
   PDFs, images, retention sweep). `"gated"` misses it, so the creator writes
   `done.json` with `outcome: "gated"` and no artifacts. That collides with the gate
   shell **member's** own `outcome: "gated"` marker
   (`gate_shell/settlement.py::_done_marker`): ACE's done loaders and
   `core/dismissed_agent_completion.effective_done_outcome` read `"gated"` as "this
   artifact is a settled gate shell" and look for a `gate_state`. The creator has none,
   so `_effective_shell_outcome` fails closed to `FAILURE_OUTCOME`, making the creator
   look like a failed artifact to `%wait` dependency resolution.
4. **`FAILED` status and a bogus workspace-hold error.** `AgentExecResult.success` is
   `loop_outcome in {"completed", "epic_approved", "monitored"}`, so `"gated"` runs
   record `Agent completed with status: FAILED`. `finalize_runner_shutdown` then takes
   the visible-failure branch and calls `hold_workspace_claim` for a claim the gate
   shell already moved to workflow `ace-gate`, printing
   `Error holding workspace #10: workspace #10 claim for ace(run)-260827_185959/... was not found`.
   The monitor handoff is exempted from that whole block by
   `monitor_handoff_claim_transferred`; the gate handoff has no equivalent. This matters
   for correctness, not just noise: once `success` becomes `True` (item 4),
   `_should_hold_workspace` returns `False` and the _release_ branch would run
   `release_workspace` + `clear_occupant_record` on the workspace the pending gate shell
   owns. The exemption must land in the same change as the `success` fix.

The `.sase_gate_pending` SIGTERM path is already correct —
`run_agent_runner_signals._NON_MONITOR_HANDOFF_MARKERS` covers every non-monitor handoff
marker — so only the post-adoption shutdown path is wrong.

## Non-goals

- Changing what the gate shell **member** writes. `gate_shell/settlement.py` keeps
  `outcome: "gated"` with its `gate_state`; after this change that marker is the only
  producer of `"gated"` on disk, which is what ACE's loaders and
  `effective_done_outcome` already assume.
- Reconciling already-written artifacts. Existing `done.json` files and empty chat
  transcripts stay as they are.
- The separate ACE RUNNING-field loader defect that releases a pending gate shell's
  `ace-gate` workspace claim (`ace/tui/models/_loaders/_running_loaders.py`
  `_release_stale_running_claim`). That is a different root cause and is tracked
  separately.

## Approach

Give the two shell-handoff loop outcomes one shared name and use it at every site that
already special-cases `"monitored"`, then add the gate analogue of
`monitor_handoff_claim_transferred`.

### 1. Name the shell-handoff outcomes once

`sase.core.dismissed_agent_completion` already exports `MONITOR_OUTCOME = "monitored"`
and `GATE_OUTCOME = "gated"`. Reuse them rather than adding new literals: add a
`SHELL_HANDOFF_OUTCOMES = frozenset({MONITOR_OUTCOME, GATE_OUTCOME})` there and export
it from `__all__`. Every site below imports that constant instead of writing string sets
inline, so the next shell kind is a one-line change.

### 2. `src/sase/axe/run_agent_exec_finalize.py`

In `finalize_loop`:

- `if state.loop_outcome == "monitored":` (transcript agent name, ~line 142) →
  `if state.loop_outcome in SHELL_HANDOFF_OUTCOMES:`. `handle_gate_marker` promotes the
  starter suffix through `update_meta_suffix` exactly like `handle_monitor_marker`, so
  reading `name` back from the transcript meta is correct for both.
- `if state.loop_outcome == "monitored": saved_path = _last_saved_chat_path(state)`
  (~line 164) → `in SHELL_HANDOFF_OUTCOMES`. This is the fix for the empty transcript:
  `handle_gate_marker` already appended its chat path to `state.saved_chat_paths`.
- `if state.loop_outcome in {"completed", "monitored"}:` (~line 183) →
  `{"completed"} | SHELL_HANDOFF_OUTCOMES`. Leave the two inner
  `state.loop_outcome == "completed"` guards (`_link_saved_chats`, `_is_workflow_noop`)
  exactly as they are, so a gated run writes `completed_outcome = "completed"` and
  collects its default artifacts.
- `success=state.loop_outcome in {"completed", "epic_approved", "monitored"}`
  (~line 279) → `... or state.loop_outcome in SHELL_HANDOFF_OUTCOMES`.

### 3. Gate analogue of the monitor claim check

Add `src/sase/axe/run_agent_gate_handoff.py` with
`gate_handoff_claim_moved(project_file, workspace_num, *, cl_name=None, ...)`, mirroring
`run_agent_monitor_handoff.monitor_handoff_claim_transferred` in shape (injectable
`get_claimed_workspaces`, `except Exception: return False`).

The predicate differs from the monitor one and must not be copied verbatim: a monitor
claim is owned by a **live supervisor process with a different pid**, so the monitor
helper requires `claim.pid != runner_pid` and `process_running(claim.pid)`. A pending
gate shell has **no process at all** — `move_gate_shell_claim` deliberately transfers
the claim to `from_pid == to_pid == creator_pid` and only changes the workflow label
(see `gate_shell/start_claim.py`, and the same assumption documented in
`ace/scheduler/stale_running_cleanup._gate_claim_is_releasable`). So the gate predicate
is: a claim exists for `workspace_num` whose `workflow == GATE_WORKSPACE_CLAIM_WORKFLOW`
(`"ace-gate"`, from `sase.gate_shell.claims`) and whose `cl_name` matches when one was
given. No pid liveness check.

### 4. `src/sase/axe/run_agent_runner_lifecycle.py`

In `finalize_runner_shutdown`, generalize the existing `monitor_handoff_transferred`
local into `shell_handoff_transferred`:

```python
shell_handoff_transferred = (
    state.exec_outcome == MONITOR_OUTCOME
    and monitor_handoff_claim_transferred(...)
) or (
    state.exec_outcome == GATE_OUTCOME
    and gate_handoff_claim_moved(
        context.project_file, state.workspace_num, cl_name=context.cl_name
    )
)
if not context.is_home_mode and not shell_handoff_transferred:
    ...
```

so a gated run neither holds nor releases the workspace the gate shell now owns.

Also add `GATE_OUTCOME` to `_NON_HOLD_FAILURE_OUTCOMES` as a belt-and-braces guard for
the case where the claim probe fails closed (returns `False`) — a gated run must never
pin a workspace for "visible failure" reasons.

### 5. `src/sase/axe/run_agent_runner_finalize.py`

- `if outcome in {"plan_rejected", "monitored"}: return` →
  `if outcome == "plan_rejected" or outcome in SHELL_HANDOFF_OUTCOMES: return`. This is
  the reported bug's direct fix: the gate publishes its own actionable notification, so
  the creator's completion notification is pure duplication.
- Leave `classify_exec_success` alone; step 2 makes `success` already `True` for gated
  runs, and adding `"gated"` there too would be dead code. Confirm this with the runner
  test rather than by inspection.

### 6. Verify nothing depended on the creator writing `outcome: "gated"`

Before landing, grep for readers of a `"gated"` done marker and confirm each one wants
the **member**, not the creator:

```bash
rg -n '"gated"|GATE_OUTCOME' src tests
```

Known readers to re-check: `ace/tui/models/_loaders/_done_filesystem_loaders.py`,
`_done_snapshot_loaders.py`, `core/dismissed_agent_completion.effective_done_outcome`,
`core/wait_dependency_resolution/_artifact_state._SHELL_KIND_BY_OUTCOME`,
`integrations/_agent_list_entry_builder.py`. All of them pair `"gated"` with a
`gate_state` / `family_shell` field that only the member marker carries, so the change
should be a strict improvement. Existing fixtures that hand-write
`{"outcome": "gated", "gate_state": ...}`
(`tests/test_axe_chop_wait_checks_plan_families.py`,
`tests/test_core_agent_scan_wire_shells.py`) describe members and must keep passing
unchanged.

## Tests

Add focused regressions; do not weaken existing ones.

1. `tests/` — `finalize_loop` with `state.loop_outcome = "gated"`:
   - writes `done.json` with `outcome: "completed"`, not `"gated"`;
   - returns `AgentExecResult.success is True`;
   - reuses the chat path already in `state.saved_chat_paths` and does **not** call
     `save_chat_history` again (assert the handoff response text survives — this is the
     regression that would have caught the empty `## Response`). Mirror the existing
     monitor coverage in the same module so the two outcomes are asserted side by side.
2. `tests/` — `send_completion_notification(outcome="gated", ...)` appends no
   notification. Assert alongside the existing `"monitored"` / `"plan_rejected"` cases.
3. `tests/test_run_agent_runner_lifecycle.py` — `finalize_runner_shutdown` with
   `exec_outcome="gated"` and an `ace-gate` claim on the run's workspace calls neither
   `hold_workspace_claim` nor `release_workspace` / `clear_occupant_record`, and sends
   no completion notification. Add the negative case too: with **no** `ace-gate` claim
   present the probe fails closed and the run still must not pin the workspace
   (`_NON_HOLD_FAILURE_OUTCOMES`).
4. `tests/` — unit-test `gate_handoff_claim_moved` directly: true for an `ace-gate`
   claim on the workspace with a matching `cl_name`, true when the claim's pid is dead
   (the defining difference from the monitor helper), false for a different workspace,
   false for a non-gate workflow, false when `get_claimed_workspaces` raises.
5. Parity guard so the next shell kind cannot repeat this: a test asserting that every
   outcome returned by the shell marker handlers (`handle_monitor_marker`,
   `handle_gate_marker`) is a member of `SHELL_HANDOFF_OUTCOMES`, and that
   `SHELL_HANDOFF_OUTCOMES` is a subset of both the completion-notification suppression
   set and the `finalize_loop` completed-marker branch. Prefer asserting against the
   shared constant over re-listing strings.

## Verification

- `just check` for the scoped lanes.
- `just check-full` through `/sase_monitor` before landing — this touches the runner
  shutdown path that most agent tests exercise.
- Manual end-to-end confirmation on a real gate handoff: run an agent that proposes a
  tale, then confirm (a) exactly one notification is published for the decision, the
  gate's own `sender="plan"` one; (b) the creator's `done.json` has
  `outcome: "completed"`; (c) its chat file's `## Response` contains the "# Gate
  handoff" body; (d) the run log ends `Agent completed with status: SUCCESS` with no
  `Error holding workspace` line; (e) the `ace-gate` claim in the project spec's RUNNING
  field is untouched by the creator's shutdown. The workspace-claim ledger at
  `~/.sase/logs/workspace_claims.jsonl` records every RUNNING-field mutation with a
  `caller_tag` and is the fastest way to check (e).
