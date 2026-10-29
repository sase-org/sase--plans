---
tier: tale
title: Light jump targets for agent sessions running a monitor
goal:
  Agent relation jump targets for agent sessions (and the clans or tribe units
  containing them) whose current turn is a running monitor render lit as in flight,
  exactly like targets with a running agent turn.
size: small
proposed_by: bbugyi200.athena.15p
proposed_with: claude/opus
create_time: 2026-10-28 22:09:38
status: wip
---

# Plan: Light jump targets for agent sessions running a monitor

## Problem

The Agents-tab jump panel lights (gold-on-olive pill, `▶` glyph) every agent relation
jump target whose status bucket is `Running`/`Starting`. A target for an agent session
whose current turn is a **running monitor** is not lit; it renders as a dim `✓` Done
target instead.

Reproduction (from the user's screenshot): clan `bob-cli-du` is selected. Member `.9` is
an agent session whose list row reads `(BUILDING) … ⚙1` with a live `2m03s / 19m50s`
runtime, and the clan header counts it as running (`[R1 W1 D8]`). The WAVES ladder
lights `.9 ▶`, but the JUMP legend shows `08 .9 ✓`, unlit.

## Root cause (verified with a scratch reproduction)

A monitor turn is parented to the agent turn that started it (for example `.9` → `.9--2`
→ `.9--mon`), not to the session container. See
`tests/ace/tui/models/test_monitor_agent_session_root_projection.py::test_normalize_loaded_agents_nests_screenshot_monitor_at_depth_three`.
The session container row itself stays live because `_agent_status_apply` mirrors the
running monitor onto it (`status == "BUILDING"`, bucket `Running`). Two roster builders
recompute a session's bucket from its **agent turns only**, so they drop the nested
monitor:

1. **CLAN MEMBERS** — `clan_roster_entries` in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_roster.py` aggregates
   `agent_session_lane_status_entries(agent_session_rows(member, children))`. Here
   `children` holds only the direct session-member children (the agent turns), so a
   monitor nested under `--2` never appears. Every agent turn is Done, so the entry is
   `DONE`/`Done`. A monitor parented directly to the session root _is_ a direct child
   and already works; only the nested shape, which is the real-world shape, fails.
2. **TRIBE MEMBERS** — `_unit_snapshot` in
   `src/sase/ace/tui/models/agent_tribe_summary.py` builds `unit_bucket` from
   `agent_status_projections((unit,))`, which projects concrete agent rows only
   (`concrete_agent_statuses` excludes monitor turns). The mirrored root is non-final,
   so `agent_session_member_status_buckets` settles it to Done. A session unit therefore
   reads `BUILDING` with bucket `Done`. A clan unit whose only live lane is a monitor
   reads `Waiting`/`Done` even though its own row shows `BUILDING`.

The scratch reproduction gave these results. The session row and clan row are
`BUILDING`/`Running`. The clan roster entry `.9` is `DONE`/`Done` with children
`['--2']`. The tribe clan unit is `BUILDING`/`Done`, and the tribe solo-session unit is
`BUILDING`/`Done`.

The other rosters already light correctly, so leave them as they are. NEIGHBORS and CLAN
NEIGHBORS use `agent_status_bucket(row)`, which reads the mirrored container bucket.
SESSION TURNS lights the monitor turn's own row through `_monitor_roster_bucket`.

## Design

Treat a sequential agent session as in flight while its current turn is a running
monitor. The rule lives in one shared model helper, and both broken roster builders feed
it into their existing aggregation as one extra lane entry. Aggregation keeps its
precedence ladder, so `QUESTION`, pending plan review, and `Failed` still outrank a
running monitor.

This is TUI presentation over existing Python TUI models (`agent_session_members.py`
already owns session-lane semantics), so there is no `sase-core` or Rust change.

### 1. Shared helper — `src/sase/ace/tui/models/agent_session_members.py`

Add the helper next to `current_agent_session_turn_row` and export it in `__all__`:

```python
def agent_session_running_monitor_row(agent: Agent) -> Agent | None:
    """Return the session's current turn when it is a running monitor turn.

    A monitor nests under the agent turn that started it, so aggregates built
    from a session's agent turns settle a lane whose only live work is that
    monitor. Callers add this row to their lane entries to keep it in flight.
    """
    current = current_agent_session_turn_row(agent)
    if current is not None and current.is_monitor:
        return current
    return None
```

Reuse `current_agent_session_turn_row` rather than adding a new walk. That function
already drives the agent list's live `2m03s / 19m50s` runtime for the same row, so a
target lights exactly when its list row shows a running monitor turn. It returns `None`
for clan containers and non-session rows, and it only returns monitors whose
`monitor_state == "running"` and have no `stop_time`.

Both call sites append the returned row as the lane entry `(monitor.status, "Running")`.
Use `"Running"` explicitly. It matches `_monitor_roster_bucket` in
`_agent_display_agent_session.py`, which lights the same monitor's own SESSION TURNS
row. A duplicate entry, for example a root-parented monitor that is already among the
rows, cannot change the any/all aggregate ladder, so callers do not dedupe.

### 2. CLAN MEMBERS — `clan_roster_entries`

In the session branch:

- Keep `rows`, `agent_session_buckets`, and `roster_children` unchanged. The expanded
  unnumbered child rows still list agent turns only, so no monitor child rows appear.
- Compute the base entries with `agent_session_lane_status_entries(rows)` as today. Call
  `monitor = agent_session_running_monitor_row(member)` once. When it returns a row,
  append `(monitor.status, "Running")` to get the status entries, then derive
  `agent_session_status` and `agent_session_bucket` from those entries.
- Status label: when the monitor entry is what made the lane in flight, show the
  monitor's own label. The rule is: the base aggregate bucket is not
  `Running`/`Starting`, the final bucket is `Running`, and a running monitor row exists.
  In that case set `agent_session_status = monitor.status` (for example `BUILDING`). The
  CLAN MEMBERS row then matches the agent-list row and the TRIBE MEMBERS row, which both
  show `BUILDING`. In every other case keep the aggregate label exactly as today, so
  existing running-agent-turn rows and goldens don't change.

### 3. TRIBE MEMBERS — `_unit_snapshot` in `agent_tribe_summary.py`

Build the existing `(row.status, bucket)` entries into a tuple, then append running
monitor entries before aggregating `unit_bucket`:

- Lanes: use `clan_members(unit)` for a clan unit (import it from `._agent_clan` if it
  isn't already imported), otherwise `(unit,)`. For each lane where
  `agent_session_running_monitor_row(lane)` returns a monitor, append
  `(monitor.status, "Running")`. Non-session units and plain agents are no-ops, because
  the helper returns `None` for them.

Keep `status=unit.display_status` unchanged; it already reads `BUILDING`. `unit_bucket`
also feeds the tribe `attention` list (`Failed`/`Stopped`/`Waiting` units). A clan that
is actually running a monitor correctly leaves that list. This matches its own row
status, so don't add anything to compensate.

Out of scope: the tribe-level `status`/`status_bucket` aggregate in
`build_agent_tribe_summary_snapshot` (`aggregate_entries`), and gate turns. A pending
gate is a human wait, and the SESSION TURNS roster renders it `Stopped`, not lit. None
of these are agent relation jump targets.

### 4. Docs — `docs/reference/tui/agents-tab.md`

Before editing, read the `docs_writing.md` memory with `/sase_memory_read`. In the
paragraph that starts "Lit `▶`/`◐` targets get a softly tinted…", add one sentence. It
should say that a session target counts as in flight while its current turn is a running
monitor, not only a running agent turn. It should also say that the same applies to a
clan or tribe-member target whose only live lane is such a session. Run `just fmt`
afterward. The agents-main-deck jump-panel paragraph only says "in-flight"; leave it
alone unless the docs-writing guidance says otherwise.

## Tests

Add `tests/ace/tui/widgets/test_agent_jump_monitor_session_in_flight.py` (new file; keep
modules small for the `toobig` gate). Build the real nested shape with
`normalize_loaded_agents` in the same way as
`test_normalize_loaded_agents_nests_screenshot_monitor_at_depth_three`. The clan
generation members are: a Done member `<clan>.1`, and a session `<clan>.9` with
`agent_session_role="root"`, a Done `--2` turn, and a running monitor `<clan>.9--mon`
(`agent_session_role="monitor"`, `monitor_id`, `monitor_state="running"`,
`status="BUILDING"`, `status_bucket="Running"`, no `stop_time`) whose `parent_timestamp`
is `--2`'s `raw_suffix`. Cover:

1. **Clan roster lights the session.**
   `clan_roster_entries(clan, ordered_clan_members(clan), …)` gives `.9` the bucket
   `effective_bucket == "Running"`, the status `"BUILDING"`, and children labels that
   are still `['--2']`. Then pass the entries through `append_member_roster` and assert
   that the `.9` jump target has `status_bucket == "Running"` and
   `is_in_flight_jump_bucket(...)` is true. Render
   `JumpLegendRenderable(jump_map, mode="collapsed")` at a wide width and assert that
   the `.9` label carries the `IN_FLIGHT_RUNNING_TINT` background. Follow
   `tests/ace/tui/widgets/test_agent_jump_legend_in_flight.py` for span and style
   assertions.
2. **Settled monitor does not light.** The same shape with `monitor_state="completed"`
   and a `stop_time` set keeps `.9` at `Done`/`DONE`.
3. **Root-parented monitor still works.** A monitor parented directly to the session
   root stays `Running`. This guards against regressions in the shape that already
   worked.
4. **Tribe roster.**
   `build_agent_tribe_summary_snapshot(None, rows, panel_collapsed=False, now=…)`: the
   clan unit and a standalone session unit (same nested shape, no clan) both get
   `status_bucket == "Running"`. With a settled monitor, both drop back to their
   previous buckets.
5. **Helper unit tests.** Add these to
   `tests/ace/tui/models/test_agent_session_members_current.py`.
   `agent_session_running_monitor_row` returns the nested running monitor. It returns
   `None` when the current turn is an agent turn, when the monitor is settled, for a
   non-session row, and for a clan container.

Existing tests to keep green include
`tests/ace/tui/models/test_monitor_failed_count_leak.py` (settled failed monitor lanes),
`tests/ace/tui/models/test_agent_tribe_summary.py`, and
`tests/ace/tui/widgets/test_agent_jump_legend*.py`.

## Verification

- Read the `lint_and_test.md` memory, then run `sase tool run check`.
- Visual goldens: run the targeted Agents jump-panel, clan, and tribe PNG snapshot tests
  (`tests/ace/tui/visual/test_ace_png_snapshots_agents_jump_panel.py`,
  `…_agents_agent_session_panel_monitor.py`, `…_agents_tribe_prompts.py`). If a golden
  changes, it should only be because a session that is running a monitor now renders
  lit. In that case, read `tui_screenshot.md` and update just that golden. Treat any
  other diff as a bug.
- Performance: the helper adds one `current_agent_session_turn_row` walk per clan member
  and tribe unit per detail render. The list row already pays that cost per row. Skim
  `tui_perf.md` and confirm nothing new runs on the navigation hot path beyond that
  walk.
- Optional live check: `sase screenshot` of an Agents tab with a clan member session
  running a monitor shows its JUMP cell lit with `▶`.
