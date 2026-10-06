---
tier: tale
title: Fix TUI runner load over-count from stale-PID ghost claims
goal:
  The Agents-tab load gauge counts only verified-live runner claims, matching launcher
  admission, and its tooltip names every claim holder so by-design hidden holders such
  as session monitors are explainable.
size: medium
proposed_by: bbugyi200.athena.0xg
create_time: 2026-10-06 14:36:57
status: wip
---

# Plan: Fix the TUI runner load gauge over-count caused by stale-PID ghost claims

## Problem

The Agents-tab `load: <load>/<capacity>` gauge showed `5.25/8`. At that moment the user
could see seven running agents: five `%q(w=0.25)` research agents and two normal-weight
(`1.0`) epic phase agents, for an expected `3.25`. The extra `2.0` came from two
capacity claims that do not appear anywhere in the Running bucket:

1. **A ghost claim (bug, about 1.0).** `0x9--1`, a bob-cli planner member whose runner
   process died hours earlier. It has no `done.json`, and its `agent_meta.json` still
   records the dead `pid`. The launcher's own admission snapshot (the disk scan with a
   real PID probe) does not count it. Only the TUI's capacity projection does, so the
   gauge, queue blockers, and free-capacity numbers in the TUI disagree with real
   admission. The ghost has inflated the gauge by 1.0 all day.
2. **A legitimate but invisible claim (by design, 1.0).** Session `0xe` was still
   finishing its coder turn and had just started a `sase monitor`. Per
   `docs/monitors.md` ("Runner slots"), a monitor keeps the session's single weighted
   claim for its whole lifetime. However, the session row was filed under **Done**
   (`TALE DONE`), so nothing on screen explained that 1.0.

The incident decomposes exactly. Ghost `1.0` plus `bob-cli-4p.2` `1.0`, five research
agents at `1.25`, `sase-1h3.2--1` `1.0`, and the `0xe` claim `1.0` total `5.25`.
Replaying the TUI projection (`load_tiered_agents()` followed by
`capacity_record_from_agent` and the Rust snapshot) still listed the dead `0x9--1`
claim. The disk/launcher snapshot taken at the same time did not.

## Root cause of the ghost

The capacity projection's contract (`refresh_runner_slot_context` docstring) assumes
"the loader has already PID-filtered active rows", and `_capacity_record_is_live()`
(`src/sase/ace/tui/models/_agent_runner_slot_capacity.py`) equates
`agent.pid is not None` with a live runner. That assumption breaks like this:

1. `_filter_dead_pids()` (`src/sase/ace/tui/models/_agent_loader_normalization.py`)
   probes PIDs only for rows it might drop. Rows whose **loaded** status is `DONE` /
   `FAILED`, and session-turn rows (monitors and gates), are kept **without probing**,
   and their stale `pid` stays on the row. `0x9--1` loads as `DONE` because its
   `workflow_state.json` says `completed`.
2. `apply_status_overrides()` then relabels that row with the sticky handoff status
   `TALE APPROVED` (`plan_approved=true`, `plan_action=tale`).
3. `capacity_record_from_agent()` emulates the disk `has_done_marker` from the
   **post-override** display status. It checks
   `stop_time is not None or status in {DONE, FAILED, FAILED (RETRIED)}`, and
   `TALE APPROVED` is not in that set. It sets
   `live = runner_is_live or pid is not None`, which is True for the stale PID, and
   `run_started_at` from `run_start_time`. The Rust engine therefore sees a live,
   started, not-done user-agent record, and it becomes an occupying `serial_session`
   claim at the default weight `1.0`.

The disk path is correct because the launcher feeds real liveness
(`record_liveness_probe()`). The TUI adapter is the only adapter that trusts an
unverified PID. The fix belongs in this repo's thin TUI adapter and loader. No Rust
change is needed, because `sase_core`'s capacity math is correct for truthful inputs.

Do **not** "fix" this by treating `TALE APPROVED` (or other sticky statuses) as done. A
`%auto` synchronous continuation, or a runner still finalizing after
`workflow_state.json` says `completed`, can legitimately hold a live PID behind such a
label, and the launcher counts it. Liveness is the only correct lever.

## Changes

### 1. Stamp rows whose PID the loader never verified

- `src/sase/ace/tui/models/_agent_state_core.py`: next to `runner_is_live`, add a
  runtime-only field
  `pid_liveness_unverified: bool = field(default=False, compare=False, repr=False)`. Its
  comment should say that the loader kept the row without probing its PID (a terminal
  loaded status or a session-turn row), so `pid` alone is not proof of a live runner.
- `src/sase/ace/tui/models/agent_bundle.py`: add `"pid_liveness_unverified"` to
  `_RUNTIME_ONLY_BUNDLE_FIELDS` so the field is never persisted into dismissal bundles
  or the cleanup archive DTO.
- `src/sase/ace/tui/models/_agent_loader_normalization.py` `_filter_dead_pids()`:
  - In the session-turn branch and the completed-status branch, set
    `agent.pid_liveness_unverified = True` when
    `agent.pid is not None and not agent.runner_is_live`.
  - In the branch where `is_process_running(agent.pid)` succeeded, set it to `False`.
  - Leave the filtering decisions themselves unchanged, and update the docstring to
    describe the stamp.
- Dedup: `_merge_agent_fields()` in `src/sase/ace/tui/models/_dedup.py` already
  propagates `runner_is_live`. Because the adapter below checks `runner_is_live` first,
  a stamped WORKFLOW row merged with a live RUNNING claim stays live, so no dedup change
  is needed. Confirm this with a quick read, and add the propagation only if a path
  clears `runner_is_live`.

### 2. Make the TUI capacity adapter use verified liveness

In `src/sase/ace/tui/models/_agent_runner_slot_capacity.py`:

- Give `capacity_record_from_agent()` a keyword-only
  `is_pid_live: Callable[[int], bool] | None = None`.
- Compute `has_done_marker` first, then derive `live` with:
  - `runner_is_live` → True;
  - `pid is None` → False;
  - row not stamped → True. This is today's behavior for rows the loader verified, and
    for directly constructed test rows.
  - stamped and already done-marked → False, without probing. Every Rust consumer of
    `live` (`is_occupying_record`, `is_waiting_record`, `better_priority_agent_pending`)
    first requires `is_user_agent_record`, which rejects done-marked records, so this
    changes nothing observable and bounds the probe count.
  - stamped and not done-marked → `is_pid_live(pid)`.
- In `src/sase/ace/tui/models/agent_runner_slots.py` `refresh_runner_slot_context()`:
  - Add a keyword-only `is_pid_live: Callable[[int], bool] | None = None`. When it is
    `None`, build a per-call memoized wrapper around
    `sase.ace.hooks.processes.is_process_running` (the probe the loader already uses).
  - Pass it to every `capacity_record_from_agent` call. Probes happen only for the
    handful of stamped, not-done rows, typically zero to a few. This path already runs
    off the event loop on async loads and in `_run_agents_capacity_refresh_from_roster`
    (`asyncio.to_thread`), and each probe is a `kill(pid, 0)` plus a `/proc` read, so
    the cost is negligible.
  - Update the docstring: the loader verifies the PIDs it filters on, and the adapter
    verifies the rest.

### 3. Explain claim holders in the load gauge tooltip

After step 2, a running monitor can still make `load` exceed the visible running count,
and that is by design. Make the gauge self-explaining:

- `src/sase/ace/tui/models/_agent_runner_slot_types.py`:
  - Add a frozen `RunnerCapacityHolder(label: str, weight: float, kind: str | None)`.
  - Add `holders: tuple[RunnerCapacityHolder, ...] = field(default=(), compare=False)`
    to `RunnerCapacitySnapshot`. Use `compare=False` because existing tests assert
    snapshot equality against literal `RunnerCapacitySnapshot(...)` values.
- `src/sase/ace/tui/models/agent_runner_slots.py` `_apply_runner_capacity_snapshot()`:
  - Build `holders` from `raw_snapshot["claims"]`. For each claim, resolve the occupier
    rows through the existing `source_agent_by_artifact_dir` map. Set
    `label = presented_agent_name or agent_name or cl_name` of the first resolved
    occupier, `weight = claim["occupied_capacity"]`, and `kind = "monitor"` if any
    occupier `is_monitor`, `"gate"` if any `is_gate`, else `None`.
  - Skip claims with no resolvable occupier, and sort by weight descending, then label.
  - The fallback (no-limit) path keeps `holders=()`.
- `src/sase/ace/tui/widgets/agent_load_indicator.py`:
  - Change the signature to `update_load(limit, occupied, holders=())`. Store the
    holders, include them in the no-op equality check, and pass them to
    `_agent_load_tooltip`.
  - When holders exist, the tooltip appends a `Held by:` block with one line per holder,
    for example `• 0xe--mon-1 (monitor): 1` and `• research.3r.cdx: 0.25`. Format
    weights with `_format_load_value`, and cap the list at 12 lines plus `… +N more`.
  - The gauge text and its width are unchanged.
- `src/sase/ace/tui/actions/agents/_display_detail_info.py`: pass
  `getattr(runner_capacity, "holders", ())` to `update_load`.
- `docs/ace.md`, in the runner-load paragraph ("Hovering the gauge spells out..."): add
  one sentence saying that the tooltip lists each claim holder and its weight, and that
  a session's running monitor holds its session's claim even when the session row is
  under Done. Link the "Runner slots" section of `docs/monitors.md`.

## Tests

Use injected probes. Never rely on whether a real PID such as `100` exists on the host.

- **Loader stamp**, in a new test or next to
  `tests/test_agent_loader_pending_gate_turn.py`, calling
  `normalize_loaded_agents(..., is_process_running=fake)`:
  - a `DONE` row with a dead PID is kept and stamped;
  - a monitor session-turn row with a PID is kept and stamped;
  - a `RUNNING` row whose PID the fake reports alive is kept and not stamped;
  - a `RUNNING` row with a dead PID is still dropped.
- **Adapter**, in `tests/ace/tui/test_agent_runner_slots_capacity.py`, using
  `_agent_runner_slots_helpers._agent`:
  - The incident shape: five running `w=0.25` rows, two running `1.0` rows, and a
    stamped `TALE APPROVED` session member with `run_start_time` set, no `stop_time`,
    and `is_pid_live` returning False. Assert `occupied_capacity == 3.25` and
    `slots_in_use == 7`.
  - The same stamped row with `is_pid_live` returning True is counted (`4.25`). This
    protects the `%auto` and finalizing-runner cases.
  - Unstamped rows never call the probe: use a probe that raises.
  - A stamped `DONE` row never calls the probe.
- **Holders**: a snapshot with a monitor-held session claim and a `0.25` claim yields
  holders ordered by weight, with `kind == "monitor"` on the monitor claim.
- **Tooltip**: `_agent_load_tooltip` (or the widget through `update_load`) renders the
  `Held by:` block and the `+N more` cap. The existing gauge-text tests stay green.
- If `tests/test_capacity_snapshot_parity.py` can express it cleanly, add a parity
  entity for the stale planner: the disk record is not live and has no done marker, and
  the TUI row loaded `DONE`, was overridden to `TALE APPROVED`, and is stamped with a
  dead PID. Assert that the CLI, admission, and TUI views agree. Skip it if the harness
  would need contortions; the targeted tests above are required.

## Verification

- Read the `lint_and_test` reference memory before finishing, and run the gate it
  prescribes (`sase tool run check` / `just check`).
- Read the `tui` and `tui_perf` reference memories before editing TUI code. Probes must
  stay bounded and must never move onto a render or keystroke path.
- Manual sanity check, optional but useful: run a short script that calls
  `load_tiered_agents()` and then
  `refresh_runner_slot_context(..., effective_limit=get_max_running_agents())`. Confirm
  that no claim lists a dead-PID artifact dir, and that `occupied_capacity` matches the
  launcher-side
  `runner_capacity_snapshot(scan_runner_slot_records(), record_liveness_probe(), ...)`
  up to transient start or finish races.
