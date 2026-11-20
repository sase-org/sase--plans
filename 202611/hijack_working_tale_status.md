---
tier: tale
title: Let ,k hijack running WORKING TALE and WORKING PLAN agents
goal:
  Pressing ,k on a live session member that the Agents tab labels WORKING TALE, WORKING
  PLAN, EPIC APPROVED, or PLAN COMMITTED reaches the same hijack decision as sase agent
  hijack, instead of refusing with ineligible_status.
size: small
decisions:
  alias_scope:
    ask:
      Should hijack also treat a live EPIC APPROVED or PLAN COMMITTED member as RUNNING?
    choices:
      all:
        Also map EPIC APPROVED and PLAN COMMITTED to RUNNING when the member's process
        is alive
      working:
        Map only WORKING TALE/PLAN; running epic- and commit-role members stay refused
    default: all
    why:
      All four labels come from the same RUNNING rewrite, so fixing two leaves the bug
      on other roles
    answer: all
proposed_by: bbugyi200.athena.1ep
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 18:05:25
status: wip
---

# Fix `,k` hijack refusing running WORKING TALE / WORKING PLAN agents

## Problem

In the Agents tab the user selected the running tale coder `bob-cli-o6.4.1.2`. Its row
showed `WORKING TALE`, and the detail panel showed the session turns
`--plan ✓ · --gate ✓ · --code ▶`. The user pressed `,k` (hijack). The TUI refused with
this toast:

```
ineligible_status: agent is not RUNNING, STARTING, or RETRYING · wait for the turn to run, then hijack
```

The agent was actively running its code turn, so hijack should have been offered and
started like it is for any `RUNNING` row. `sase agent hijack NAME` is not affected: the
CLI resolves members from backend scan rows, and those report `RUNNING`.

## Root cause

1. The TUI status pass in `src/sase/ace/tui/models/_agent_status_apply.py` runs the
   block commented "Active agent session code handoff rows display the coder-specific
   working state". It calls `active_approved_plan_handoff_status` in
   `src/sase/ace/tui/models/_agent_status_agent_session_policy.py`. For a sequential
   session member child whose status is `RUNNING`, that function overwrites
   `agent.status` with a display label:
   - `WORKING TALE` or `WORKING PLAN` for role `code`
   - `EPIC APPROVED` for role `epic`
   - `PLAN COMMITTED` for role `commit`

   The session root row then mirrors that label.

2. The TUI computes the hijack decision from the row object itself. The call chain is
   `compute_hijack_decision(agent, deps=hijack_row_local_deps(app, agent))`, then
   `resolve_and_decide(agent)`, then `resolve_hijack_subject` in
   `src/sase/agent/hijack/_facts_resolve.py`. `_running_rows` puts the input row first,
   and `_row_for_member` matches it by artifacts dir. `_status(row, process_alive)` then
   returns the row's `status` verbatim, upper-cased. The only exception is that a live
   status with a dead process becomes `LOST`.
3. As a result, the selected member's `status` fact is `"WORKING TALE"`. The Rust v2
   decision `agent_hijack_decide` (sase-core
   `crates/sase_core/src/agent_hijack/eligibility.rs`) only accepts
   `RUNNING | STARTING | RETRYING` and returns `ineligible_status`. In that function the
   `has_numbered_claim` check runs before the status check. The refusal therefore
   confirms that the status fact was the only blocker.

In short, a TUI presentation label leaks into a backend hijack fact. The same label also
keeps the member out of `_LIVE_STATUSES`, so its status alone never counts it as live.

## Fix (Python facts adapter only; no sase-core change)

The Rust decision is right to require a canonical runner status. The bug is that the
Python facts adapter forwards a display label, so the fix normalizes the label there.
This needs no Rust wire change and no `sase-core-revision.txt` bump.

### 1. Name the alias set in `src/sase/agent/status_buckets.py`

Define the set after `WORKING_PLAN_STATUSES`, `EPIC_APPROVED_STATUS`, and
`PLAN_COMMITTED_STATUS`:

```python
#: Display labels the Agents tab paints over a live session member whose runner
#: status is ``RUNNING`` (see ``active_approved_plan_handoff_status``).
#: ``EPIC APPROVED`` and ``PLAN COMMITTED`` also label settled rows, so a label
#: here is evidence of ``RUNNING`` only together with a live process.
RUNNING_DISPLAY_ALIAS_STATUSES: frozenset[str] = WORKING_PLAN_STATUSES | frozenset(
    {EPIC_APPROVED_STATUS, PLAN_COMMITTED_STATUS}
)
```

> [!decision] alias_scope = working Define
> `RUNNING_DISPLAY_ALIAS_STATUSES = WORKING_PLAN_STATUSES` and drop the comment's second
> sentence. Every test below then covers only `WORKING TALE` and `WORKING PLAN`.

### 2. Normalize in `_status` in `src/sase/agent/hijack/_facts_resolve.py`

After a row status is upper-cased, return `"RUNNING"` when
`status in RUNNING_DISPLAY_ALIAS_STATUSES and process_alive`. Keep every other branch
exactly as it is:

- A live status with a dead process still becomes `LOST`.
- Any other row status is returned verbatim.
- With no row, the result is still `RUNNING` or `DONE` by liveness.

An alias row whose process is dead keeps its label, so nothing changes for dead rows.
Import the constant at module top: `sase.agent.status_buckets` is a light leaf module.
Confirm that the import adds no cycle.

This turns the member's `MemberFacts.status` into `RUNNING`, which has three effects:

- `in_flight` is true by status.
- The occupant-selection request carries `RUNNING`.
- `_build_hijack_facts` sends `status: "RUNNING"`.

The decision then becomes `immediate`, or `latched` while finalizing, exactly as on the
CLI path. The TUI footer chip and the kill-modal `h` both read the same cached decision,
so the fix covers them too. Confirming the hijack runs `submit_agent_hijack`, which
re-resolves by name and was never affected.

### 3. Leave these deliberately unchanged

- **`_row_status` in `src/sase/agent/hijack/_stop_target.py`:** No decision reads
  `HijackTarget.status`. Before skipping it, check with
  `grep -rn "\.status" src/sase/agent/hijack`. If any decision reader exists, apply the
  same liveness-gated mapping there, using `process_is_running(pid)`.
- **The `list_running` filter in
  `src/sase/ace/tui/actions/agents/_hijack_roster_snapshot.py`:** A live member that is
  not in that list already falls back to `_status(None, process_alive)`, which yields
  `RUNNING`. Widening the filter would turn dead alias rows from `DONE` into their
  label.
- **The Rust `eligibility.rs`:** It keeps the canonical status vocabulary.
- **The display labels and their rendering.**

## Tests

1. **`_status` unit test.** Add a parametrized test to
   `tests/test_agent_hijack_facts.py`, next to
   `test_stale_live_row_is_not_an_occupying_member`. It imports
   `sase.agent.hijack._facts_resolve._status` and covers:
   - each label in `RUNNING_DISPLAY_ALIAS_STATUSES` with `process_alive=True` gives
     `"RUNNING"`
   - each such label with `process_alive=False` keeps the label
   - `RUNNING` with a dead process gives `LOST` (existing behavior)
   - `DONE` with a live process gives `DONE`
   - `_status(None, process_alive=True)` gives `RUNNING`
2. **TUI/CLI parity regression.** Add a test to
   `tests/ace/tui/test_agent_hijack_tui_parity.py`, reusing that file's helpers:
   `_write_member`, `_install_session_lookups`, `_tui_agent`, `_named`, `_decision_key`,
   `create_project_file_with_running`, and `WorkspaceClaim`. Build a plan-chain session
   with two members:
   - a settled root `--plan` member with a done outcome
   - a live member with `agent_session_role` set to the role that produces the label
     (`code`, `epic`, or `commit`), `lifecycle_phase: "provider_running"`,
     `runner_capabilities: ["hijack"]`, and an `llm_provider` and `model`. Its `pid`
     matches a `WorkspaceClaim` (workspace 4) whose `artifacts_timestamp` is the
     member's timestamp.

   Parametrize the member's TUI row status over the alias labels: `WORKING TALE` and
   `WORKING PLAN`, plus `EPIC APPROVED` and `PLAN COMMITTED` under the default decision.
   Use `StopDeps` with `process_alive=lambda *_: True`, and assert:
   - `compute_hijack_decision(tui_row, deps=tui_deps)` returns mode `immediate`, a code
     other than `ineligible_status`, and the live member as occupant
   - its `_decision_key` equals the key from
     `resolve_and_decide(member_name, deps=cli_deps)`, where `cli_deps.list_running`
     returns a raw `status="RUNNING"` row for the member. This is the
     `sase agent hijack --explain` parity.
   - with `process_alive=lambda *_: False`, the TUI decision is still `refused`, so dead
     rows do not regress

3. **Drift guard.** Add a test to
   `tests/test_agent_loader_status_override_gate_turn_agent_session.py`, which already
   has `_root` and `_coder` fixtures and `test_running_coder_still_shows_working_tale`.
   Assert that every non-`None` status `active_approved_plan_handoff_status` returns for
   a `RUNNING` sequential session child is in `RUNNING_DISPLAY_ALIAS_STATUSES`. Cover
   role `code` with a tale parent, role `code` with a plan parent, role `epic`, and role
   `commit`. This keeps a future running-member display label from silently breaking
   hijack again. Under `alias_scope = working`, assert this only for the two `code`
   cases.

## Verification

- While iterating, run
  `just test tests/test_agent_hijack_facts.py tests/ace/tui/test_agent_hijack_tui_parity.py tests/test_agent_loader_status_override_gate_turn_agent_session.py`.
- Finish with `sase tool run check`, the guarded `just check` recipe. Do not run
  `just check-full`.
- Optional manual check: in `sase ace`, select a running `WORKING TALE` or
  `WORKING PLAN` row and press `,k`. Hijack should start, or open its confirmation,
  instead of showing the `ineligible_status` toast. `sase agent hijack -e NAME` should
  report the same mode.
