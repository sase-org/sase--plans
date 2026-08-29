---
tier: tale
title: A pending gate shell is visible in ACE before anyone answers it
goal:
  A gate shell renders its own Agents-tab row from the moment it is created, so the
  family node shows the pending decision status (EPIC / TALE / PLAN / QUESTION / a
  custom pending_status) instead of falling back to the dead planner's DONE.
size: medium
proposed_by: bbugyi200.athena.0fu
---

# Plan: A pending gate shell is visible in ACE before anyone answers it

## The bug, and the exact reason for it

A `#plan` agent proposed an epic plan and its node immediately went `DONE` on the Agents
tab, with no gate row anywhere in the tree and no `⋔` chip on the family container. The
decision was pending for ~17 minutes with nothing in ACE saying so.

The cause is one line of list normalization:

```python
# src/sase/ace/tui/models/_agent_loader_normalization.py
def _filter_dead_pids(agents, *, is_process_running):
    """Filter out agents with dead PIDs while keeping completed agents."""
    verified_agents: list[Agent] = []
    completed_statuses = ("DONE", "FAILED")
    for agent in agents:
        if agent.status in completed_statuses:
            verified_agents.append(agent)
        elif agent.pid is not None:
            if is_process_running(agent.pid):   # <-- pending gate shell dies here
                verified_agents.append(agent)
        else:
            verified_agents.append(agent)
    return verified_agents
```

This encodes "a row whose status is not terminal must have a live process". That is
false for a gate shell. Per the `gates-never-block` decision record and
`plan:202608/gate_shells.md` §3, _"while pending, a gate shell has no process at all —
no pid, no supervisor"_, and its creator is killed the moment the descriptor prints. So
the pending row has:

- `status` = the gate's `pending_status` (`EPIC`, `TALE`, `PLAN`, `QUESTION`, or a
  custom label) — never `DONE` or `FAILED`; and
- `pid` = a **dead** pid it inherited from the creator.

Both branches miss, and the row is dropped on every refresh.

### Where the dead pid comes from

`shells/member.create_family_shell_member` →
`run_agent_helpers_artifacts .create_followup_artifacts` writes an initial
`workflow_state.json` containing `"pid": os.getpid()` and the creating process's
`process_identity`. Both `gate_shell/member.py` and `monitor/member.py` deliberately set
`"pid": None` in the member's `agent_meta.json` ("The selected agent's pid is not this
new proc shell's pid"), but nothing gives `workflow_state.json` the same treatment, and
the TUI's workflow row takes its `pid` from `WorkflowEntry.pid` → `wf_state.pid`.
Confirmed on the reported gate's real artifacts: its `workflow_state.json` carries
`"pid": 625201`, the planner's pid.

### Why monitors are not affected, and gates are

A monitor member survives because `monitor/start_claim.claim_monitor_workspace` moves
the workspace claim to the **live supervisor pid**, so
`_running_loaders.load_agents_from_running_field` emits a row for it (its own workflow
row is dropped by the same dead-pid filter, and `dedup_running_vs_workflow` never
notices). A gate shell has no supervisor: `gate_shell/start_claim.move_gate_shell_claim`
transfers the claim with `from_pid == to_pid == creator_pid` on purpose, and
`_running_loaders` therefore skips it with the comment _"A held pending gate-shell claim
still contributes no row: the gate-shell member renders from its own artifact record."_
That artifact record's row is exactly the one `_filter_dead_pids` throws away. Neither
source produces a row, so the shell is invisible until settlement writes `done.json` and
the done loader takes over.

### Verified reproduction

Building a two-directory artifact fixture (a plan-chain root with `done.json`, plus a
`agent_family_role: gate` / `gate_state: pending` member) and calling
`agent_loader.load_all_agents()` three times, varying only the gate member's
`workflow_state.json` pid:

| gate `workflow_state.json` pid | rows returned                                          |
| ------------------------------ | ------------------------------------------------------ | ---------------------------- |
| dead pid (production shape)    | root only, `0ft--plan                                  | DONE` — **the reported bug** |
| live pid                       | root `EPIC` + gate child `EPIC` (`gate_state=pending`) |
| pid absent                     | root `EPIC` + gate child `EPIC` (`gate_state=pending`) |

So everything downstream of the loader is already correct: `apply_gate_meta` sets
`status = pair.start` and `status_bucket = gate_state_bucket("pending") == "Stopped"`,
and `_agent_status_apply`'s `newest_pool` mirroring hands the family container the
gate's status exactly as `plan:202608/gate_shells.md` §8 promised. Only the row's
survival is broken.

### Why the epic missed it

`sase-ud.6` shipped the whole presentation layer and tested it by hand-building `Agent`
objects: `tests/test_agent_loader_status_override_gate_shell_family.py` constructs a
root, a planner step, and a gate member in memory and asserts
`_apply_status_overrides`'s output — its docstring even says _"purely additive; nothing
here should require a source change to pass"_. `tests/test_agent_loader.py`'s two gate
tests cover the workspace **claim** (held while pending, released once terminal) and
positively assert that the claim contributes no row — with nothing asserting that some
other source contributes one. No test starts from artifact markers and asks "does a
pending gate shell produce a row?", which is the only question that fails.

### Blast radius

Every gate shell kind, on every host, since the epic landed: epic/tale/plan gates
(`plan_shell/`), `/sase_questions` (`question_shell/`), HITL, launch approval, and
custom `/sase_gate` gates all create their member through `create_gate_shell_member`.
The user-visible losses are the pending row itself, the family node's decision status,
the `⋔N` pending lane chip on the container, the `Stopped` slice of the agent count
chip, and the `Stopped` bucket under `[group: by status]`.

## Scope decision: this stays in Python

Per `sase/memory/rust_core_backend_boundary.md`, backend rules that other frontends must
match belong in `../sase-core`. This one does not: the Rust agent catalog already
returns pending gate members (`sase agent search <family>` lists `<family>--gate` while
it is pending, and `index.rs::record_is_active` selects `has_done_marker = 0` rows), and
`is_runner_slot_occupying_record` already excludes gate members from runner occupancy.
`_filter_dead_pids` is ACE-TUI list normalization with no cross-frontend contract, so
the fix is Python-side only and no `sase-core` change is needed.

## Implementation

### 1. Stop using process liveness to decide whether a shell row survives

In `src/sase/ace/tui/models/_agent_loader_normalization.py`, teach `_filter_dead_pids`
that a durable family-shell member is not a process:

- Keep any row for which `agent_family_members.row_is_family_shell(row)` is true
  (`row.is_monitor or row.is_gate`) before the pid branch is reached.
- Replace the docstring's "keeping completed agents" framing with the real rule: OS
  process liveness gates _agent_ rows; a family shell's own `gate_state` /
  `monitor_state` gates shell rows, which is the same doctrine
  `ace/scheduler/stale_running_cleanup` and `gate_shell/claims.gate_claim_is_releasable`
  already encode for workspace claims.

Terminal shells stay correct for free: a settled gate is kept because it is a shell row,
and its `done.json` row already carries the settled status. A pending shell that is
never answered is bounded by `gate_timeout_seconds` and the registered
`gate_shell_reclaim` chop, so this cannot strand a row forever.

**Monitor regression risk, and how to close it.** Today a running monitor's workflow row
is dropped here and only the RUNNING-field claim row survives. After this change both
survive and must merge into one row. Verify `_dedup.dedup_running_vs_workflow` (and
`dedup_by_pid`) collapse them, and add a test that asserts a running monitor member
still yields exactly one row with its live supervisor pid. If they do not merge, fix the
dedup rather than narrowing the visibility rule back to gates only — the split doctrine
is what produced this bug.

### 2. Do not stamp a foreign pid onto a processless shell member

`create_family_shell_member` should not leave the creating process's `pid` /
`process_identity` in the member's initial `workflow_state.json`, for the same reason
`gate_shell/member.py` and `monitor/member.py` already null `pid` in `agent_meta.json`.
Either thread a flag through `create_followup_artifacts` or scrub the two keys in
`create_family_shell_member` after it returns; prefer whichever keeps
`create_followup_artifacts`'s ordinary follow-up behavior untouched.

This is belt-and-braces relative to step 1 — step 1 is still required, because gate
shells already on disk carry the bad pid — but it removes the derived failure described
next, and it makes the on-disk record honest.

### 3. Do not derive `FAILED` from a dead pid for a shell record

`_workflow_loaders.load_workflow_states` and
`_workflow_snapshot_loaders.load_workflow_states_from_snapshot` both flip a `running`
workflow state to `FAILED` when its recorded pid is dead. For a shell record that is
wrong regardless of step 2, and it is not harmless: `display_status == "FAILED"` also
drives `error_message`, `preferred_workflow_output_path`, and
`build_workflow_failure_fallback`, so a perfectly healthy pending gate would carry a
fabricated "workflow failed" error and output path that `apply_gate_meta` does not clear
(it only overwrites `status` and `status_bucket`).

Add a shell-aware skip in both loaders, mirroring the existing
`record.done.outcome == "monitored"` skip in `load_workflow_agents_from_snapshot`: when
the record is a real family-shell member, leave the status alone and let the member's
own metadata own it. Keep the two implementations in step with each other.

### 4. Confirm the downstream surfaces light up

These should all follow from step 1 with no further change; each needs a check, and a
fix only if it does not:

- The family container mirrors the gate's pending status (`_agent_status_apply`
  `newest_pool` / `_mirror_root_from_child`), so the reported family reads `EPIC`, not
  `DONE`.
- `agent_family_members.shell_lane_counts` puts a pending gate in the `gate.running`
  lane (`gate_row_is_settled` is false), so `_agent_list_render_agent` draws `⋔1` on the
  container.
- `agent_count_chip` counts the row under `stopped`, and `GroupingMode.BY_STATUS` files
  it under the `Stopped` L0 bucket.
- The row is _not_ counted as running work: `agent_row_is_in_flight` is false for
  `pending` (only `settling` is in flight), and runner-slot occupancy is record-based in
  Rust and already excludes gate members. Confirm the panel's `running` count does not
  move.
- The prompt panel's `GATE` section and the `⋔` row glyph render for a pending row that
  now exists.

If any PNG golden under `tests/ace/tui/visual/snapshots/png/` changes because a pending
gate row or `⋔` chip is newly drawn, rebaseline it deliberately and say so.

### 5. Tests — start from artifact markers, not hand-built rows

The whole defect lived in the space between "the loader produces a row" and "the
projection formats a row", so the new coverage must cross that seam.

Add loader-level tests (extend `tests/test_agent_loader.py`, or a sibling module) that
write real artifact directories and call the public loader:

- A pending gate member with a **dead** `workflow_state.json` pid yields exactly one
  gate row, with `status` equal to the gate's `pending_status`,
  `gate_state == "pending"`, `status_bucket == "Stopped"`, attached under its family
  container, and the container mirroring that status. This test fails on today's tree
  and is the regression guard.
- The same for a `question_shell` gate and a custom `/sase_gate` shell, so the fix is
  not accidentally plan-specific.
- A settled gate member still yields exactly one row with its stop status (no duplicate
  from the workflow record).
- A running monitor member still yields exactly one row (the dedup guard from step 1).
- A pending gate row carries no `error_message` and no failure `output_path` (step 3).

Add one conformance-style guard so the next shell kind cannot repeat this: assert that
no normalization step drops a row for which `row_is_family_shell` is true, keyed on the
shell's state rather than on `pid`.

Finally, add a line to `tests/test_agent_loader.py`'s existing pending-gate claim test —
or its docstring — pointing at the row-producing test, so the pair reads as "the claim
contributes no row _because_ the artifact record does".

## Verification

- `just fmt`, `just lint`.
- `just check`, escalating to the full suite as the lane requires.
- Targeted: `tests/test_agent_loader.py`, the new loader tests,
  `tests/test_agent_loader_status_override_gate_shell_family.py`, `tests/gate_shell/`,
  `tests/gate_conformance/`, `tests/test_stale_running_cleanup.py`, and the gate PNG
  snapshots.
- `just check-full` before landing, per `sase/memory/lint_and_test.md`'s two-speed rule.
- End to end on a real gate: create a shell gate, confirm `sase gate list` shows it
  pending **and** that ACE's Agents tab shows the `--gate` row with the pending status,
  the family node mirroring it, and the `⋔1` chip on the container — then answer it and
  confirm the settled row is unchanged from today.

## Out of scope, worth filing separately

`_workflow_loaders._iter_workflow_timestamp_dirs` only walks
`~/.sase/projects/*/artifacts/<workflow-dir>/<timestamp>/`; it does not understand the
sharded `<YYYYMM>/<DD>/<timestamp>` layout every real host now writes, so the Python
filesystem workflow loaders return nothing against production artifacts. ACE does not
depend on them (the tiered path uses the Rust scanner and its bounded fallback), so this
is not part of this fix, but the compatibility exports in `_done_loaders.py` /
`_workflow_loaders.py` advertise a fallback that cannot work on a sharded tree. File a
task bead rather than widening this plan.
