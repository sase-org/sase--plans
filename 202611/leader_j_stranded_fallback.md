---
tier: tale
title: Make ,j fall back to the most recently stranded agent
goal:
  When no unread completed or stopped agent exists, Agents-tab ,j jumps to the most
  recently stranded agent node before toasting.
size: small
decisions:
  repeat_press:
    ask:
      Should repeated `,j` presses in the new stranded tier cycle on to older stranded
      nodes?
    choices:
      cycle:
        Each press moves to the next older stranded node and wraps, like the unread and
        stopped tiers
      most_recent:
        Every press lands on the single most recently stranded node; repeat presses stay
        put
    default: cycle
    why:
      The first press still lands on the newest stranded node; cycling matches the other
      two tiers
    answer: cycle
proposed_by: bbugyi200.athena.19z
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-08 09:06:53
status: wip
---

# Plan: `,j` final fallback to the most recently stranded agent

## Goal

On the Agents tab, `,j` (`leader_mode.keys.jump_to_next_unread_done_agent`) currently
tries the next unread completed agent and then falls back to `,J` (next stopped agent),
toasting `No unread completed or stopped agents` on a double miss. Add a third and final
tier. When neither an unread completed agent nor a stopped agent can be reached, `,j`
jumps to the **most recently stranded** agent node. It toasts only when all three tiers
miss.

`,J` (`jump_to_next_stopped_agent`) does not change. No new keymap is added.

## Definitions (use these exactly)

- **Stranded node:** a real Agents-tab agent row (not a clan container) whose own bead's
  cached stranded-work verdict has `state == "stranded"`. These are the rows that render
  `STRANDED ⚐` (Done bucket) or a leaf `⚐` (other buckets). They are also exactly the
  rows the agents-live `stranded:true` query field matches
  (`src/sase/ace/tui/models/agent_live_query.py`). Some rows do **not** qualify:
  - rows whose verdict is `blocked` (the dim breadcrumb `⚐`);
  - waiters whose only connection is a stranded wait target;
  - rows with no verdict.

  Clan containers are excluded, as they are for every timed jump. Their stranded
  _members_ are still reachable through the shared candidate pipeline, which covers
  members inside collapsed clans and rows on other agent tabs.

- **Stranding time:** `StrandedWorkVerdict.since`, the epoch seconds at which the work
  became stranded. The same value drives the `⚐ 31h` age label and the resolve modal's
  "ended at". Convert it to a naive configured-timezone datetime so it sorts with model
  datetimes such as `Agent.stop_time`: `to_local(datetime.fromtimestamp(since, tz=UTC))`
  (`sase.core.time.to_local`). When `since` is `None`, fall back to
  `agent.stop_time or agent.start_time`. That is the default jump time of the shared
  candidate builder, so no tier mixes naive and aware values.
- **Data source:** only the in-memory `_STRANDED_REPORT_CACHE` in
  `src/sase/ace/tui/models/agent_stranded.py`, which the Agents deferred bead warmup
  keeps warm. The keypress path must do no disk I/O, stat, or report resolution (TUI
  perf rules 1 and 8). A cold or empty cache simply yields no stranded candidates, and
  the press falls through to the toast.

## Implementation

### 1. One shared stranded-row helper — `src/sase/ace/tui/models/agent_stranded.py`

Add a public `stranded_leaf_verdict(agent: Agent) -> StrandedWorkVerdict | None`. It
returns `_stranded_verdict_for_agent(agent)` when that verdict's `state == "stranded"`,
otherwise `None`. Export it in `__all__`. It is a memory-only dict lookup: it reuses
`_agent_bead_id_for_stranded`, which already rejects non-agent nodes, and the cached
`_verdict_index`.

Route the agents-live `stranded` field in `src/sase/ace/tui/models/agent_live_query.py`
through `stranded_leaf_verdict(agent) is not None` in place of building the whole
`agent_stranded_view(agent)`. This is behavior-identical, because the field only reads
`.verdict`. It keeps `stranded:true` and the new `,j` tier on one definition.

### 2. The stranded jump — `src/sase/ace/tui/actions/agents/_unread_navigation.py`

- Add a module-level `_stranded_agent_jump_time(agent) -> datetime | None` that applies
  the stranding-time rule above. It sits next to `_stopped_agent_jump_time`.
- Add `AgentUnreadNavigationMixin._jump_to_next_stranded_agent(self) -> bool`. It calls
  `self._jump_to_next_matching_agent_by_time(...)` with these arguments:
  - `predicate=lambda agent: stranded_leaf_verdict(agent) is not None`
  - `after_select=None`
  - `time_for_agent=_stranded_agent_jump_time`
  - `include_collapsed_clan_members=True`

  Reusing that method gives the existing behavior for free:
  - newest-first ordering;
  - the jump anchor (`_save_agents_jump_anchor`);
  - collapsed tribe-panel and clan-fold reveal;
  - the off-tab reveal with rollback;
  - the debounced detail (`defer_detail=True`).

  Do **not** call `land_stopped_jump_on_main_context`. That Context landing exists for
  stopped rows awaiting a plan or question decision. A stranded row is resolved with
  `Enter`, which already opens the stranded resolve modal.

- Import `stranded_leaf_verdict` with the module's other `...models` imports. If that
  creates an import cycle, import it lazily inside the method.
- Update the module docstring to "Time-ordered unread, stopped, and stranded-agent
  navigation helpers."

> [!decision] repeat_press = cycle No change to `_jump_to_next_matching_agent_by_time`.
> When the focused row is already a stranded candidate, the next press advances to the
> next older one and wraps, exactly like the stopped tier.

> [!decision] repeat_press = most_recent Add a keyword-only `cycle: bool = True`
> parameter to `_jump_to_next_matching_agent_by_time`. When `cycle` is false, skip the
> "advance past the current candidate" loop so `target_pos` stays `0`.
> `_jump_to_next_stranded_agent` passes `cycle=False`. If the focused row already is the
> newest stranded node, the call returns `True` without moving focus (no anchor save, no
> refresh), so it does not toast. Every other caller keeps the default.

### 3. Leader dispatch — `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`

In the `jump_to_next_unread_done_agent` branch, extend the existing fallback inside the
same `leader.unread_jump` span:

```python
hit = self._jump_to_next_unread_done_agent()
fell_back = not hit
stranded_fallback = False
if fell_back:
    hit = self._jump_to_next_stopped_agent()
    if not hit:
        stranded_fallback = True
        hit = self._jump_to_next_stranded_agent()
trace["fallback"] = fell_back
trace["stranded_fallback"] = stranded_fallback
```

Change the miss toast to exactly `No unread completed, stopped, or stranded agents`.
Keep the "no trailing refresh on hit or miss" fast path and its comment, and update the
comment's "Both jumps" wording to cover all three. Leave the `,J` branch untouched.

### 4. Command-palette availability

- `src/sase/ace/tui/commands/types.py`: add `stranded_agent_count: int = 0` to
  `CommandContext`, next to `stopped_agent_count`.
- `src/sase/ace/tui/commands/context.py`: add `_stranded_agent_count(app)`. It counts
  `app._agents` rows that are not clan containers and for which
  `stranded_leaf_verdict(a) is not None`. This mirrors `_stopped_agent_count`, which
  also scans only the loaded rows. Compute it only when `tab == "agents"`, otherwise use
  `0`, and pass `stranded_agent_count=` into the `CommandContext`.
- `src/sase/ace/tui/commands/_availability_agents.py`: the
  `leader.jump_to_next_unread_done_agent` entry becomes available when
  `unread_completed_agent_count > 0 or stopped_agent_count > 0 or stranded_agent_count > 0`.
  Leave `leader.jump_to_next_stopped_agent` unchanged.

Leave the leader-mode footer (`src/sase/ace/tui/widgets/_keybinding_modes.py`)
unchanged. The `,j next unread done` hint stays tied to unread rows, which is how the
stopped fallback is treated today.

### 5. Labels, config comment, generated catalog

- `src/sase/ace/tui/commands/_mode_commands.py` `_LEADER_LABELS`:
  `"Jump to next unread completed agent (else stopped, else stranded)"`.
- `src/sase/ace/tui/modals/help_modal/agents_mode_sections.py`:
  `"Jump to next unread done agent (else stopped, else stranded)"`.
- `src/sase/default_config.yml`: change the `jump_to_next_unread_done_agent` comment to
  `# falls back to jump_to_next_stopped_agent, then the most recently stranded agent, before toasting`.
  Key values do not change.
- Regenerate `src/sase/footprint/catalog.jsonl` with `just sync-footprint-catalog`.
  Never hand-edit it.
- `docs/reference/tui/keymaps.md` is generated, and its `,j` label is unaffected. Do not
  edit it by hand.

### 6. Docs (Reference and runbook only)

- `docs/reference/tui/agents-tab-modes.md`, Leader Mode paragraph: after the `,J`
  fallback sentence, say that when no stopped agent exists either, `,j` jumps to
  stranded rows, most recently stranded first. Those are the owner rows that show
  `STRANDED ⚐` or a leaf `⚐` and that `stranded:true` matches. Link
  `../work-units/bead-lifecycles.md#stranded-beads`, and say it toasts only when all
  three tiers miss. Under `repeat_press = most_recent`, say that it always lands on the
  newest one. Stay within the page budget.
- `docs/contribute/perf-runbook.md`: extend the `fallback: true` sentence. When the
  stopped jump also misses, the same span records `stranded_fallback: true`, and its
  duration includes the stranded jump.

No memory note states the old two-tier behavior, so no memory change is needed.

## Tests

Keep `tests/ace/tui/test_leader_keymap_dispatch.py` under the `toobig` 700-line tier by
putting the new navigation cases in a new file.

Update existing tests:

- `tests/ace/tui/_leader_keymap_helpers.py`: give the fake app `jump_stranded_count`,
  `jump_stranded_result = True`, and a `_jump_to_next_stranded_agent` stub.
- `tests/ace/tui/test_leader_keymap_dispatch.py`:
  - The double-miss test becomes a triple miss. Assert one call per tier and the new
    toast.
  - The stopped-hit test asserts `jump_stranded_count == 0`.
  - The non-Agents-tab and `,J` tests assert the stranded stub is never called.
- `tests/ace/tui/test_unread_trace_spans.py`: assert `stranded_fallback` is `False` on
  an unread hit and on a stopped hit, and `True` after a stopped miss, both with and
  without a stranded hit.
- `tests/ace/tui/test_agent_unread_jump_fast_path.py`: update the expected toast
  strings.
- `tests/test_command_availability_agents_navigation.py`:
  - Assert that `stranded_agent_count=1` alone enables `,j` but not `,J`.
  - Assert that all three counts at `0` disable it.
  - Assert that it stays Agents-tab only.
- `tests/test_command_context_extraction.py`: cover `_stranded_agent_count` against a
  seeded cache. Count a stranded row; skip a blocked row, a row with no verdict, and a
  clan container.
- `tests/test_command_catalog_build.py` and `tests/test_keymaps_display_help_panels.py`:
  update the label assertions.
- `tests/ace/tui/models/test_agent_stranded.py`: test `stranded_leaf_verdict`. It
  returns the verdict for `stranded`, and `None` for `blocked`, a missing verdict, and a
  cold cache.

Add `tests/ace/tui/test_agent_stranded_navigation.py`:

- Use an autouse fixture that clears `agent_stranded._STRANDED_REPORT_CACHE` before and
  after each test.
- Seed reports through
  `_STRANDED_REPORT_CACHE.set("demo", StrandedReport(...), "test")`. `make_agent` rows
  live in project key `demo`. Attach beads with
  `dataclasses.replace(make_agent(..., status="DONE"), phase_bead_id=...)`, as in
  `tests/ace/tui/test_agent_enter_targets_stranded.py`.
- Drive the cases through `LeaderUnreadJumpApp` / `UnreadJumpApp` from
  `tests/ace/tui/_agent_unread_navigation_helpers.py`, using the fixtures from
  `tests/ace/tui/test_agent_stopped_navigation.py`.

Cases to cover:

1. With no unread or stopped rows, `,j` selects the row whose verdict has the largest
   `since`, even when another stranded row has a later `stop_time` or comes first in the
   list.
2. A second `,j`:
   - under `cycle`, moves to the next older stranded row and wraps after the oldest;
   - under `most_recent`, stays on the newest row and does not toast.
3. An unread completed row, or a stopped row, still wins over a newer stranded row.
4. Rows with a `blocked` verdict and rows with no verdict are skipped. With only those,
   the press toasts exactly `No unread completed, stopped, or stranded agents`. An empty
   cache gives the same toast.
5. A stranded row with `since=None` orders by its `stop_time` among rows that do carry
   `since`, without a naive/aware `TypeError`.
6. A stranded member hidden by a collapsed clan is revealed and selected. Mirror the
   collapsed-clan setup in `tests/ace/tui/test_agent_stopped_navigation_clans.py`.
7. A stranded hit does not request the main-context landing, and the display refresh
   keeps `defer_detail=True`.

## Verification

1. `just fmt`, then `just sync-footprint-catalog`.
2. Run the touched and new test files with targeted `pytest`, then run `just check`
   (agent default; do not run `just check-full` unless instructed).
3. `just check` excludes the PNG visual suite. The previous `,j` label change
   (`c29f67a1e7`) needed no golden updates. If a help-panel PNG golden
   (`tests/ace/tui/visual/test_ace_png_snapshots_help_panel.py`) now differs because of
   the new label, regenerate it with `just fix-tui-screenshots` and review the diff.

## Acceptance criteria

- With unread completed rows, `,j` behaves as before. With none but stopped rows, it
  behaves exactly like `,J`. With neither, it lands on the stranded row whose
  `verdict.since` is newest, across tabs and collapsed clans. It toasts
  `No unread completed, stopped, or stranded agents` only when all three tiers miss.
- The keypress path reads only the in-memory stranded cache.
- The command palette offers `,j` when only stranded rows exist. The help modal, palette
  label, config comment, footprint catalog, and both docs pages describe the three-tier
  fallback.
- `just check` passes.
