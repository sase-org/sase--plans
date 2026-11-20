---
tier: tale
title: Stop clan rows flashing FAILED while a failed monitor is still settling
goal:
  While a monitor supervisor is still settling a failed command (continuation not yet
  decided), the owning agent session, clan row, waves strip, wait counts, and tab
  counters read Running instead of Failed; a settled monitor with no follow-up, or one
  whose supervisor died mid-settle, still reads Failed.
size: medium
proposed_by: bbugyi200.athena.1em
proposed_with: claude/opus
create_time: 2026-11-20 11:20:01
status: wip
---

# Stop clan rows flashing FAILED while a failed monitor is still settling

## Problem

Agent clan nodes on the Agents tab sometimes read `FAILED` (with an `F1` chip, a `✗` on
the member in the CLAN waves strip and jump bar, a `✗1` wait-dependency count on the
`.land` member, the clan filed under the `Failed` status group, and `1 failed` in the
tab header) for several seconds up to about a minute, then go back to `RUNNING`. The
member that causes it renders as `(RUNNING)` in the same frame. The aggregates disagree
with the row because the member's `status` is `RUNNING` while its `status_bucket` is
`Failed`.

## Root cause (verified)

Observed on epic clan `sase-1tz.3.1`, member `sase-1tz.3.1.5` (screenshot taken
10:30:33):

- 09:34:01: monitor turn `sase-1tz.3.1.5--mon` starts `just check`.
- ~10:29:52: the command exits 1. `finish_monitor`
  (`src/sase/monitor/supervise_finish.py`) immediately persists `agent_meta.json` with
  `monitor_state: "failed"`. It deliberately pops `stopped_at` from this write. There is
  no `monitor_settled`, no `monitor_next_action`, no `monitor_followup_outcome`, and no
  `done.json` yet.
- It then runs `settle_claim_and_followup_under_lane_lock` (outcome policy resolution
  plus follow-up launch). The follow-up `--1` artifact dir and launch admission appear
  at 10:30:44. `wait_for_followup_started` follows. Only after that does it write
  `monitor_settled: true`, the `stopped_at` time,
  `monitor_followup_outcome: "launched"`, and `done.json`. Delivery finished at about
  10:32:01.

During that settle window, the TUI does the following:

1. `apply_monitor_meta` (`src/sase/ace/tui/models/_loaders/_meta_enrichment_monitor.py`)
   sets `status_bucket = monitor_state_bucket("failed") == "Failed"`. It rewrites
   `status` only when the state is `running`, so the row keeps the raw `RUNNING` status.
2. `monitor_row_is_settled` (`src/sase/ace/tui/models/agent_session_members.py`) treats
   any terminal `monitor_state` as settled.
3. `monitor_lane_status_bucket` (`src/sase/monitor_state.py`) receives no follow-up
   outcome and no next action. It therefore returns the monitor's own `Failed` bucket,
   the "real, actionable failure" branch.
4. `apply_status_overrides` (`src/sase/ace/tui/models/_agent_status_apply.py`) finds no
   active or waiting member, so the session root mirrors the newest turn, which is the
   monitor. `_mirror_root_from_child` copies `status = "RUNNING"` and sets
   `status_bucket = monitor_row_lane_bucket(child) == "Failed"`.
5. `apply_clan_container_status`, `clan_member_counts`, the waves strip, the wait maps,
   and `sase_agent_status_counts` all read `agent_status_bucket(member)`, so they report
   `Failed`.

This was reproduced in memory. A DONE session root, a monitor with `status="RUNNING"`,
`status_bucket="Failed"`, `monitor_state="failed"`, `stop_time=None`, and no follow-up
fields, plus one running sibling, gives a root of `RUNNING`/`Failed`, a clan status of
`FAILED`, and `clan_member_counts(...).failed == 1`.

The TUI breaks the documented doctrine in `docs/reference/automation/monitors.md`: "A
monitor is terminal only after it is settled: the command has exited..., and the
follow-up has launched or its disposition has been recorded." The existing sase-19o fix
(`tests/ace/tui/models/test_monitor_failed_count_leak.py`) covers only the steady state
after `monitor_followup_outcome` is recorded.

The lane-bucket logic is Python-only. sase-core only parses the fields, so no Rust or
wire change is needed. Note that the compact list wire used by the Agents tab
(`AgentArtifactListTurnWire`) carries neither `monitor_settled` nor `next_action`. It
does carry `pid`, `stopped_at`, and the supervisor `process_identity`, and those are
enough.

## Fix

Introduce a derived "settling" state for monitor turns. A monitor turn is settling when
all of the following hold:

- its command reached a terminal `monitor_state`;
- the supervisor has not stamped a stop time (`stop_time is None`, meaning no
  `stopped_at` and no done marker);
- the recorded supervisor process is provably still alive.

While a turn is settling, its continuation is not yet decided, so the agent-level lane
reads `Running`. The monitor's own row keeps its own `Failed` bucket, which is truthful
about the command.

The liveness guard is required. `should_reconcile_dead_supervisor`
(`src/sase/monitor/reconcile.py`) only reconciles `monitor_state == "running"`. A
supervisor that died mid-settle would therefore otherwise read as `Running` forever.
With the guard, a dead or unknown supervisor keeps today's `Failed` reading, so a stuck
lane still surfaces.

### 1. Shared lane rule: `src/sase/monitor_state.py`

- Add a keyword-only `settling: bool = False` to `monitor_lane_status_bucket`.
- The checks run in this order:
  1. A non-terminal state returns the own bucket.
  2. A non-empty `followup_error` returns the own bucket.
  3. Host-completed returns `Done`.
  4. A handoff outcome (`launched` / `launched-degraded`) returns `Running`.
  5. **New:** `settling`, with `followup_outcome is None` and the state not in
     `{"lost", "stopped"}`, returns `Running`.
  6. The existing next-action pending rule.
  7. Otherwise, the own bucket.
- Update the docstring bullets to say: "A monitor whose supervisor is still settling
  (continuation not yet decided) keeps the lane in progress (`Running`)."

### 2. TUI row field: `src/sase/ace/tui/models/_agent_state_ops.py`

- Next to the other monitor fields, add `monitor_settling: bool = False`. Use
  `compare=False` / `repr=False` if neighbouring derived fields do; check how
  `runner_is_live` is declared in `_agent_state_core.py` and follow that pattern.
- Add a comment: it is a load-time derived fact, set only by loader normalization after
  a bounded supervisor-liveness probe; never persisted; defaults to `False` so rows from
  any other path (fleet/remote rows, tests, restores) keep the current behaviour.
- Make sure dedup and copy helpers don't need it. Normalization stamps it after
  `_deduplicate`, so no `_dedup.py` change is expected; confirm this.

### 3. Pure helpers: `src/sase/ace/tui/models/agent_session_members.py`

- Add `monitor_row_is_settle_candidate(row) -> bool`. It returns
  `row.is_monitor and monitor_state_is_terminal(row.monitor_state) and row.stop_time is None and row.pid is not None`.
- In `monitor_row_lane_bucket`, pass `settling=row.monitor_settling` to
  `monitor_lane_status_bucket`. Update its docstring.
- Do **not** change `monitor_row_is_settled`, `agent_row_is_in_flight`, or
  `_monitor_roster_bucket`. The monitor's own row, the `⚙N` gear lanes, and the roster
  styling stay as they are.

### 4. Liveness stamp: `src/sase/ace/tui/models/_agent_loader_normalization.py`

- Add `_mark_settling_monitor_turns(agents, *, is_process_running)`. For each row where
  `monitor_row_is_settle_candidate(row)` holds, set
  `row.monitor_settling = is_process_running(pid) and process_identity_matches(pid, row.process_identity)`.
  Memoize per pid within one call. For every other row, set
  `row.monitor_settling = False` so a reloaded or merged row can't keep a stale `True`.
- Call it after `_deduplicate(...)` and before `apply_status_overrides(...)` in both
  `normalize_loaded_agents` and `normalize_live_plan_agents`.
- The probe is bounded: at most one `kill(0)` plus one identity read per unsettled
  terminal monitor, which is usually zero or one row. It runs on the loader path, which
  is already off the UI thread (it already probes agent pids in `_filter_dead_pids`).
  Keep it out of any render, keystroke, or UI-thread path, per the TUI perf rules.
- `_loading_compute_merge._normalize_relationships_after_merge` re-runs
  `apply_status_overrides` on already-stamped rows. Rows only gain the flag through
  normalization, and fresh incoming rows replace cached ones. Confirm by reading that no
  extra stamping is needed there.

### 5. Tests

- `tests/test_monitor_lane_status_bucket.py`: add parametrized cases:
  - `failed` + `settling=True` with no outcome → `Running`;
  - `failed` + `settling=True` + `followup_error` → `Failed`;
  - `failed` + `settling=True` + `followup_outcome="not-launchable"` → `Failed`;
  - `lost` + `settling=True` → own bucket;
  - `completed` + `settling=True` → `Running`;
  - the existing `settling=False` case with no next action stays `Failed`.
- `tests/ace/tui/models/test_monitor_failed_count_leak.py`: add the screenshot
  regression.
  - Setup: a session root `.5` (DONE root) whose only turn is a settle-window monitor
    (`status="RUNNING"`, `status_bucket="Failed"`, `monitor_state="failed"`,
    `stop_time=None`, a pid, no follow-up fields, `monitor_settling=True`), plus a
    running sibling member.
  - After `apply_status_overrides` + `project_clan_tree`, assert:
    - `agent_status_bucket(root) == "Running"`;
    - the clan container status is not `FAILED`;
    - `clan_member_counts(container).failed == 0`;
    - `sase_agent_status_counts(projected, ()).failed == 0`;
    - `agent_panel_counts(projected, set()).failed == 0`;
    - the monitor row's own bucket is still `Failed`.
  - Also add the negative twin, the same setup with `monitor_settling=False` (dead or
    unknown supervisor). It must still report `failed == 1`.
- `tests/test_agent_loader_pid_liveness_stamp.py`, or a sibling new test file: drive
  `normalize_loaded_agents` with an injected `is_process_running`. Assert that
  `monitor_settling` is `True` only for a terminal, stop-time-less monitor with a live,
  identity-matching pid. It must be `False` when:
  - the pid is dead;
  - the identity mismatches (monkeypatch `process_identity_matches` or use the existing
    test seam);
  - `stop_time` is set;
  - the state is `running`;
  - the row is not a monitor. Also assert the end-to-end effect on the mirrored session
    root bucket.
- Existing suites must keep passing unchanged, notably:
  - `tests/test_agent_loader_status_override_monitor_agent_session.py`, including
    `test_nested_terminal_failed_monitor_root_mirrors_failed_bucket` (it has a
    `stop_time`, so not settling);
  - `tests/ace/tui/models/test_monitor_*`;
  - `test_agent_tree_clan_status_monitor.py`;
  - `test_agent_session_monitor_lanes.py`;
  - `tests/_agent_compact_parity_helpers.py` users. If a compact/full parity test
    enumerates `Agent` fields, add `monitor_settling` to its derived or ignored set
    rather than to the parity set.

### 6. Docs: `docs/reference/automation/monitors.md`

Follow `docs_writing` routing: this is the behaviour of an existing feature, so it goes
in its Reference page. After the paragraph "A monitor is terminal only after it is
settled: …", add one sentence. It should say that while a monitor's supervisor is still
settling, the Agents tab keeps the owning session and clan rows in progress instead of
counting the command's failure against them. The failure then surfaces only if the
settled monitor records no follow-up, or if its supervisor dies before settling. Keep it
to one or two sentences; no new section.

## Verification

- Read the `lint_and_test` memory before finishing and run the verification it
  prescribes (`just check` via the sanctioned tool-run path). Do not run
  `just check-full`.
- Re-run the in-memory reproduction from the Root cause section as a quick sanity check.
  It should now show the root `RUNNING`/`Running`, the clan `RUNNING`, and `failed == 0`
  when `monitor_settling=True`.

## Out of scope

- No change to the monitor supervisor's write ordering (`finish_monitor`, the
  proc-adapter settle in `src/sase/monitor/_proc_adapter_followup.py`, or reconcile),
  and no sase-core wire change.
- No change to the monitor turn row's own label during the settle window. It still shows
  the raw `RUNNING` with its own `Failed` bucket and `✗ <code>` until `done.json` lands.
  The session container keeps reading `RUNNING`, now with a consistent `Running` bucket.
- Fleet/remote rows (`src/sase/ace/tui/models/_fleet_agents_nodes.py`) can't be
  liveness-probed locally. They keep the current behaviour.
- Possible pre-existing backend gap: a monitor supervisor that dies after persisting a
  terminal `monitor_state` but before `monitor_settled` appears never to be reconciled,
  because `should_reconcile_dead_supervisor` requires `monitor_state == "running"`. If
  you confirm this while implementing, file it through `/sase_new_task` rather than
  fixing it here.
