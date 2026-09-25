---
tier: tale
title: Stop settled verification monitors from counting as failed agents
goal:
  A failed `just check` monitor whose continuation is launched or pending never makes
  its session, clan, tribe panel, global header, wait badge, clan roster, or tribe
  summary report a failed agent, while real agent failures and failed continuation
  launches still surface.
size: medium
proposed_by: bbugyi200.athena.0sh
create_time: 2026-09-25 18:29:55
status: wip
---

# Stop settled verification monitors from counting as failed agents

## Diagnosis

The `@epic` panel showed `106 [R5 Q3 W15 F1 D82]`. It also showed a **Failed** group
holding clan `sase-19o (RUNNING) [W1 F1 D1]`, a global header with `1 failed`,
`sase-19o.land (WAITING ✗1)`, and a JUMP bar with `.2 ✗ .3 ✗`. None of this was a real
agent failure.

- Live data check: every one of the 27 rows in the `@epic` tribe whose bucket is
  `Failed` is a settled `TESTED` monitor shell (`just check`). No agent row has failed.
- `sase-19o.3` timeline:
  - The plan agent ran from 15:59:12. At 16:12:08 it started a `just check` monitor.
  - At 16:12:14 the monitor exited 1. Its `done.json` persisted `monitor_state=failed`,
    status `TESTED`, and `status_bucket=Failed`.
  - The agent's turn ended at 16:12:31.
  - The monitor continuation launched `sase-19o.3--1` at 16:13:27
    (`monitor_followup_outcome=launched`). That agent fixed the problem and finished
    `DONE`.
- `sase-19o.2` followed the same pattern. `sase-19p.2` reproduced it live between 16:42
  and 16:44.

### Root cause

A monitor row's own `status_bucket` comes from `MONITOR_STATE_BUCKETS` in
`src/sase/monitor_state.py`:

- `failed`, `timeout`, and `lost` map to `Failed`.
- `completed` and `stopped` map to `Done`.

That bucket describes the monitor's _command_, not an agent. Project doctrine says
monitors are not agents: see the `AgentPanelCounts` docstring,
`is_agents_tab_agent_node`, and `concrete_agent_statuses`, which all drop shells. Three
agent-level aggregations still consume a settled monitor's bucket.

1. **Session-root mirroring** — `_mirror_root_from_child` in
   `src/sase/ace/tui/models/_agent_status_apply.py`, reached from
   `apply_status_overrides`.
   - When a session's newest row is a settled monitor, the root copies
     `child.status_bucket = "Failed"`. This window runs from monitor settlement until
     the continuation agent is attached and loaded, which is about a minute or longer.
   - The root is an agent node, so the false failure spreads through everything that
     reads its bucket:
     - `sase_agent_status_counts`: global header, panel title `F1`, group banners.
     - `clan_member_counts`: clan chip `F1`.
     - `apply_clan_container_status`: the clan aggregate becomes Failed, so the clan is
       filed under the Failed group.
     - The wait-dependency buckets in `src/sase/ace/tui/_agent_completion_wait.py`: the
       `.land (WAITING ✗1)` badge.
   - The screenshot's `(RUNNING)` label on a Failed-bucketed row is a label/bucket
     mismatch. A `Failed` bucket can only reach a session root through this mirror, so
     removing it at the source also removes the mismatch.
   - The exact path that re-set the label to RUNNING was not pinned down. Do not spend
     time hunting it; the regression tests below cover the observable outcome.
2. **Clan roster / JUMP bar** — `clan_roster_entries` in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_roster.py`.
   - It aggregates each session's rows (`agent_session_rows(...)`, which include monitor
     shells) with `aggregate_agent_group_effective_status` and
     `aggregate_agent_group_bucket`.
   - `Failed` wins that aggregation. So every session that ever had a failing check
     shows `✗ FAILED` forever. Today `.2` and `.3` both read FAILED although both are
     DONE.
3. **Tribe summary** — `build_agent_tribe_summary_snapshot` in
   `src/sase/ace/tui/models/agent_tribe_summary.py`.
   - `aggregate_entries` is built from `tribe_unit_real_rows`. For clans that is
     `clan_section_member_rows`, which includes monitor session children.
   - Result: the `@epic` tribe status is FAILED right now, while its own counts report
     `failed=0`. It is rendered by `build_tribe_compact_lines`.

## Design

A settled monitor keeps its own bucket **on its own row**, so its roster child line
still shows `✗ TESTED` as accurate history. Agent-level aggregates, however, read a
monitor only as the _current state of its lane_, with handoff semantics:

- If a continuation was launched or is pending, the lane is still in progress
  (`Running`).
- If the continuation failed to launch, or none exists, the monitor's own bucket stands.
  That is a real, actionable failure.

Existing precedents to align with:

- Core wait resolution:
  `src/sase/core/wait_dependency_resolution/_artifact_state.py::shell_followup_handoff_agent`
  (`SUCCESSFUL_SHELL_FOLLOWUP_OUTCOMES = {"launched", "launched-degraded"}`). It also
  treats a terminal, non-lost/stopped shell that has a non-blank `next_action` and no
  outcome yet as a pending follow-up.
- Gate handoffs: `HANDOFF_SETTLED_STATUSES` in `src/sase/agent/status_buckets.py` keeps
  settled gates `Running` "while the follow-up agent is still coming up".

Boundary note: session-root mirroring and roster/tribe aggregation live only in this
repo's Python projection today, so no `sase-core` change is needed. Put the pure rule in
`src/sase/monitor_state.py` next to `MONITOR_STATE_BUCKETS` so other frontends can reuse
it.

## Steps

### 1. Pure lane-bucket rule — `src/sase/monitor_state.py`

Add module constants, defined as local literals:

- `MONITOR_HANDOFF_FOLLOWUP_OUTCOMES = frozenset({"launched", "launched-degraded"})`
- `MONITOR_HOST_COMPLETED_OUTCOME = "host-completed"`
- `MONITOR_HOST_COMPLETED_STATUS = "completed_by_host"`

Do **not** import `sase.monitor.*`. That package's `__init__` pulls in stores and
launchers, and TUI models import `sase.monitor_state` on hot paths.
`src/sase/ace/tui/widgets/_agent_list_styling.py` already duplicates
`"launched-degraded"` locally the same way.

Add:

```python
def monitor_lane_status_bucket(
    monitor_state: str | None,
    own_bucket: str,
    *,
    followup_outcome: str | None = None,
    followup_error: str | None = None,
    next_action: str | None = None,
    host_completion_status: str | None = None,
) -> str:
```

Apply these rules in order, and document them in the docstring with the rationale above:

1. `not monitor_state_is_terminal(monitor_state)` → `own_bucket`.
2. Non-blank `followup_error` → `own_bucket`. The continuation failed to launch.
3. `followup_outcome == MONITOR_HOST_COMPLETED_OUTCOME` or
   `host_completion_status == MONITOR_HOST_COMPLETED_STATUS` → `"Done"`.
4. `followup_outcome in MONITOR_HANDOFF_FOLLOWUP_OUTCOMES` → `"Running"`.
5. `monitor_state not in {"lost", "stopped"}` and `followup_outcome` is None and
   `next_action` is non-blank → `"Running"`. The continuation is pending.
6. Otherwise → `own_bucket`. This covers legacy monitors without continuation metadata,
   and lost or stopped monitors without a follow-up.

Export the new names in the module's `__all__` if it has one.

### 2. Agent adapter and lane entries — `src/sase/ace/tui/models/agent_session_members.py`

Add `monitor_row_lane_bucket(row: Agent) -> str`:

- Compute `own = agent_status_bucket(row)`.
- Return `own` unless `row.is_monitor and monitor_row_is_settled(row)`.
- Otherwise return
  `monitor_lane_status_bucket(row.monitor_state, own, followup_outcome=row.monitor_followup_outcome, followup_error=row.monitor_followup_error, next_action=row.monitor_next_action, host_completion_status=row.monitor_host_completion_status)`.

Add
`agent_session_lane_status_entries(rows: Sequence[Agent]) -> tuple[tuple[str, str], ...]`:

- Start from `buckets = agent_session_member_status_buckets(rows)`.
- For each `(row, bucket)`, when `row.is_monitor and monitor_row_is_settled(row)`:
  - Skip the row if it is **not** the final row. Its result was handed to a later row.
  - If it is the final row, use `monitor_row_lane_bucket(row)` as its bucket.
- Every other row keeps its bucket.
- Return `(row.status, bucket)` pairs.

Add both functions to `__all__`.

### 3. Session-root mirror — `src/sase/ace/tui/models/_agent_status_apply.py`

In `_mirror_root_from_child`, inside the `if child.is_monitor:` branch, set
`parent.status_bucket = monitor_row_lane_bucket(child)`.

- Keep the status label and the monitor presentation fields exactly as today.
- `copy_missing_display_metadata` fills `status_bucket` only when it is None, so the
  order stays safe.
- Update the `apply_status_overrides` docstring and inline comment: a mirrored settled
  monitor keeps its label but contributes its lane (handoff) bucket.

### 4. Clan roster aggregate — `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_roster.py`

In `clan_roster_entries`, compute the session aggregate status and bucket from
`agent_session_lane_status_entries(rows)` instead of zipping `rows` with
`agent_session_buckets`.

- Keep `agent_session_buckets` for each `MemberRosterChild.effective_bucket`, so the
  `--mon` child line still shows its own `Failed`.
- Keep the `or member.display_status` fallback.

### 5. Tribe summary aggregate — `src/sase/ace/tui/models/agent_tribe_summary.py`

In `build_agent_tribe_summary_snapshot`, drop settled monitor rows from
`aggregate_entries`, i.e. skip rows where
`row.is_monitor and monitor_row_is_settled(row)`.

Each session's lane state is already represented by its session root row in `real_rows`,
and Step 3 makes that root's bucket handoff-aware. `_unit_snapshot` already uses
shell-free concrete projections, so leave it alone.

Before finishing, grep `src/sase/ace/tui` for other session- or tribe-level aggregates
that feed `aggregate_agent_group_*` or `agent_session_member_status_buckets` from row
lists that include monitor shells. Apply the same helper if you find one. The grep done
while planning found only the three sites above.

### 6. Tests

1. **Pure rule.** Unit-test every rule of `monitor_lane_status_bucket`. Extend an
   existing monitor-state test module if one covers `monitor_state_bucket` (search
   `tests/` for `monitor_state_bucket`); otherwise add
   `tests/test_monitor_lane_status_bucket.py`. Cases:
   - running → own bucket
   - failed + launched → Running
   - failed + `launched-degraded` → Running
   - failed + pending (`next_action` set, no outcome) → Running
   - failed + `followup_error` → Failed
   - completed + host-completed → Done
   - completed + launched → Running
   - lost + `next_action` → Failed
   - stopped + `next_action` → Done
   - legacy failed with no metadata → Failed
2. **Root mirror.** In
   `tests/test_agent_loader_status_override_monitor_agent_session.py`, extend the
   `_nested_monitor` helper with the optional fields `monitor_followup_outcome`,
   `monitor_followup_agent`, `monitor_followup_error`, `monitor_next_action`, and
   `monitor_host_completion_status`. Add these tests:
   - A failed nested monitor with a launched follow-up (not yet loaded) makes the root
     mirror the stop label with `status_bucket == "Running"` and
     `agent_status_bucket(root) == "Running"`. The monitor row itself keeps `"Failed"`.
   - A pending follow-up (only `monitor_next_action` set) → Running.
   - `monitor_followup_error` set → root stays `Failed` (a real failure is preserved).
   - Completed + host-completed → Done.
   - The same launched case on a plain-agent session root (`_plain_agent_session_root`)
     → Running.
   - Leave `test_nested_terminal_failed_monitor_root_mirrors_failed_bucket` and
     `test_nested_terminal_successful_monitor_root_mirrors_stop_label` unchanged. Their
     fixtures carry no continuation metadata, so they must still pass.
3. **Lane entries.** In `tests/ace/tui/models/test_agent_session_members.py`, test
   `agent_session_lane_status_entries`:
   - `[root DONE, failed monitor (launched), --1 DONE]` has no `Failed` entry.
   - A final failed monitor with a launched follow-up yields a `Running` entry.
   - A final failed monitor with `followup_error` yields a `Failed` entry.
4. **Screenshot scenario.** Build a clan `sase-19o` with three members:
   - `.2`: a session with a DONE plan root, a failed monitor that has a launched
     follow-up, and a `--1` that is DONE.
   - `.3`: a session with a DONE plan step and a settled failed monitor whose follow-up
     is launched but not yet loaded.
   - `.land`: WAITING.

   Run it through `apply_status_overrides` and `project_clan_tree`. Reuse helpers from
   `tests/ace/tui/widgets/_agent_display_clan_helpers.py` or the monitor test fixtures
   where they fit. Put it in `tests/ace/tui/widgets/test_agent_display_clan_roster.py`
   or a new focused model test. Assert:
   - `clan_member_counts(container).failed == 0`, and the container's
     `agent_status_bucket` is not `"Failed"`.
   - `sase_agent_status_counts(top_level_rows, ()).failed == 0` and
     `agent_panel_counts(rows, set()).failed == 0`. `agent_panel_counts` is in
     `sase.ace.tui.actions.agents._display_panel_titles`; see
     `tests/ace/tui/test_agent_panel_title_counts.py` for usage.
   - The `clan_roster_entries` session entries for `.2` and `.3` have an
     `effective_bucket` other than `"Failed"`, while their `--mon` child lines keep
     `"Failed"`.
   - Negative control: give `.3`'s monitor a `monitor_followup_error` and assert the
     failed counts become 1.

5. **Tribe summary.** In `tests/ace/tui/models/test_agent_tribe_summary.py`:
   - A tribe whose clan has a DONE session with a settled failed monitor in its history
     has `snapshot.status != "FAILED"` and `snapshot.status_bucket != "Failed"`.
   - Negative control: a real `FAILED` agent row still yields a FAILED tribe status.

### 7. Verification

- Run `just check` through `sase tool run`, following the `lint_and_test` memory.
- Do not run `just check-full`.
- PNG goldens are not expected to change. The failed monitor in
  `tests/ace/tui/visual/_ace_agents_png_snapshot_agent_session_fixtures.py` carries no
  continuation metadata. If a visual test does fail, run targeted
  `just fix-tui-screenshots -- <selectors>` and inspect every diff before accepting it.

## Acceptance

- When a `just check` monitor fails and its continuation is launched or pending, the
  following are all clear: the session row bucket, the clan chip, the clan's group
  placement, the tribe panel `F` count, the global header "failed" count, the
  wait-dependency `✗N` badge, the clan roster/JUMP glyph, and the tribe summary status.
  None of them reports a failure.
- The monitor's own row and roster child line still show its failed check (`✗ TESTED`).
- A continuation that fails to launch (`monitor_followup_error`), a lost monitor with no
  follow-up, and real `FAILED` agent rows still surface as failures.
