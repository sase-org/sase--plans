---
tier: tale
title: Family row mirrors QUEUED status and queue badge from its queued shell
goal:
  "A sequential agent family whose next shell is parked at the runner-slot admission
  gate displays QUEUED with that shell's queue position badge (e.g. QUEUED #1/2) on the
  family row, instead of a stale WAITING."
size: small
proposed_by: bbugyi200.athena.0l0
create_time: 2026-09-14 15:42:25
status: wip
---

# Family Row Shows WAITING While Its Queued Shell Shows QUEUED #n/m

## Problem

In the ACE Agents tab, a sequential agent family whose only pre-run shell is parked at
the runner-slot admission gate displays `WAITING` on the family row while the shell row
itself displays `QUEUED #1/2`. Observed 2026-09-14 with family `0kw`: `0kw--plan` was
`DONE`, `0kw--gate` was a pending `TALE` gate, and `0kw--code` was `QUEUED #1/2`, yet
the family row read `sase (WAITING)`. The family row should read `QUEUED #1/2`, and the
family should group under the Queued bucket, matching its queued shell.

## Root Cause (confirmed by code reading)

Status display is computed in two passes over the same mutable `Agent` rows:

1. **Root mirroring** — `apply_status_overrides` in
   `src/sase/ace/tui/models/_agent_status_apply.py` (waiting branch, ~lines 352-362)
   mirrors a sequential family root's status from its oldest child whose status is in
   `{WAITING, QUEUED}` and records that child as `parent.wait_display_source`. This pass
   runs during loader normalization, **before** any queue promotion, and `QUEUED` is
   never persisted (`runner_slot_display_status` in `src/sase/agent/status_buckets.py`
   ~lines 181-194: "Persisted and scan-level statuses remain WAITING"). So the root
   always mirrors `WAITING` here.
2. **Runner-slot pass** — `refresh_runner_slot_context` in
   `src/sase/ace/tui/models/agent_runner_slots.py` runs afterwards (see
   `src/sase/ace/tui/actions/agents/_loading_compute.py` ~line 208). Both its paths
   (`_apply_runner_capacity_snapshot` and `_refresh_runner_slot_context_fallback`)
   promote real slot waiters to `QUEUED` and assign
   `runner_slot_queue_position`/`runner_slot_queue_size`, then finalize every row:
   - clan containers are re-projected from their members via
     `apply_clan_container_status` — the comment at ~lines 154-158 records that this
     exists precisely to keep an all-queued clan from retaining its earlier `WAITING`
     status;
   - every other row, **including a sequential family root**, goes through
     `runner_slot_display_status(agent.status, slot_queued=<is the row itself a waiter>)`.
     The root is not itself a slot waiter, so `slot_queued=False` pins its mirrored
     status at `WAITING`.

   The clan fix was never given a sequential-family analogue. That is the bug.

Additionally, the `#pos/size` badge in
`src/sase/ace/tui/widgets/_agent_list_render_agent_status.py` (`QUEUED` branch, ~lines
103-111) reads `runner_slot_queue_position`/`runner_slot_queue_size` from the row
itself, and those fields are only populated on rows with their own `slot_requested_at`.
So even with the status fixed, the family row could not render `QUEUED #1/2`.

## Fix

All changes are TUI presentation state, so they belong in this repo (no Rust-core
change; the Rust admission snapshot already supplies queue order and positions).

### 1. Mirror queue promotion onto family roots in `agent_runner_slots.py`

In `src/sase/ace/tui/models/agent_runner_slots.py`, in **both** finalization loops —
`_apply_runner_capacity_snapshot` (the `else` branch, ~lines 328-332) and
`_refresh_runner_slot_context_fallback` (the `else` branch, ~lines 210-214) — treat a
row whose `wait_display_source` was promoted as queued itself:

```python
slot_queued = id(agent) in queue_positions
if not slot_queued:
    source = agent.wait_display_source
    slot_queued = source is not None and id(source) in queue_positions
agent.status = runner_slot_display_status(agent.status, slot_queued=slot_queued)
```

Notes for the implementer:

- In the fallback path the current check is `_is_live_slot_waiter(agent)`; replacing it
  with `id(agent) in queue_positions` is equivalent there because `queue_positions` is
  built from exactly the `_is_live_slot_waiter` rows. Using the same shape in both paths
  keeps them in lockstep.
- This is safe for non-family rows: `wait_display_source` is cleared at the start of
  every `apply_status_overrides` pass and only ever set on family roots whose display
  tracks a waiting member (sequential waiting branch and the legacy parallel-family
  branch — the latter benefits from the same fix; modern parallel families load as clan
  containers and keep the clan branch).
- `runner_slot_display_status` no-ops for statuses outside `{WAITING, QUEUED}`, so rows
  in any other status are unaffected.
- Promotion of waiters happens earlier in each function than the finalization loop, so
  the source row's promoted status is already visible; the derivation stays reversible
  and idempotent across repeated `refresh_runner_slot_context` calls (match the
  idempotency assertion style of
  `tests/ace/tui/test_agent_runner_slots_families.py::test_first_refresh_promotes_all_slot_waiters_and_clan_aggregate`).

### 2. Render the queue badge from the wait-display source

In `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py` (`QUEUED` branch), read
`runner_slot_queue_position` and `runner_slot_queue_size` from
`wait_display_agent(agent)` instead of `agent`. That branch already routes the
capacity/priority slot label through `wait_display_agent`, so this is consistent; for an
ordinary queued row `wait_display_agent` returns the row itself, leaving current
behavior unchanged. A family root whose wait source is queue position 1 of 2 then
renders `QUEUED #1/2`.

### Explicitly out of scope

The oldest-wins selection among multiple waiting children in `_agent_status_apply.py`
(~line 356, pinned by
`tests/test_agent_loader_status_override_followup_roots.py::test_apply_status_overrides_two_waiting_children_oldest_wins`)
stays as is. The reported family had exactly one pre-run shell, so selection is not part
of this bug; the family row showing the queue-front (oldest) waiter remains the intended
behavior when several members wait.

## Tests

1. **`tests/ace/tui/test_agent_runner_slots_families.py`** — add a sequential-family
   analogue of `test_first_refresh_promotes_all_slot_waiters_and_clan_aggregate`:
   - Build a family root (own terminal status, e.g. `DONE`, no `slot_requested_at`) plus
     a family member child (`parent_timestamp` set, live pid, `slot_requested_at` set,
     status `WAITING`) using the existing `_agent` helper from
     `tests/ace/tui/_agent_runner_slots_helpers.py` (extend it only if a needed field is
     missing).
   - Run `_apply_status_overrides([root, child])` (import as in
     `tests/test_agent_loader_status_override_followup_roots.py`) so
     `root.wait_display_source` is populated the way the real loader pipeline does, then
     `refresh_runner_slot_context`.
   - Cover **both** paths: with `effective_limit` (snapshot path) and without (fallback
     path).
   - Assert: the child is `QUEUED` with a queue position; the root's status is `QUEUED`;
     a second `refresh_runner_slot_context` call is idempotent (statuses and returned
     snapshot unchanged).
   - Negative case: a root whose waiting child is _not_ a slot waiter (e.g. a dependency
     wait with no `slot_requested_at`) keeps `WAITING`.
2. **Renderer test** (follow the existing style in
   `tests/ace/tui/widgets/test_agent_list_runner_slot_status.py`): a row with status
   `QUEUED` whose `wait_display_source` carries `runner_slot_queue_position=1` and
   `runner_slot_queue_size=2` renders `QUEUED #1/2`; a row with its own position still
   renders as before.

## Verification

- Run `just check` before finishing (run `just install` first if the workspace's
  virtualenv is stale). If it runs long, hand it to the `/sase_monitor` skill per the
  `lint_and_test.md` memory.
- Sanity-check that Agents-tab summary counts do not change shape: counting projects
  sequential family containers through `concrete_agent_statuses`
  (`src/sase/ace/tui/models/agent_family_members.py` ~line 457), i.e. from members, not
  the container's displayed status, so the root's new `QUEUED` display must not
  double-count. The existing count/grouping test suites
  (`tests/ace/tui/widgets/test_agent_list_grouping_buckets.py`, clan/count tests) are
  the backstop; investigate any failure there rather than adjusting expectations
  blindly.
