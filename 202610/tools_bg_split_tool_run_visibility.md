---
tier: epic
title: 'tools: / bg: top-bar split and ⚒ tool-run visibility in tribes and clans'
goal: 'The top bar tells the truth: a `tools:` group shows every live `sase tool`
  run on the machine with the ⚒ identity (ledger-sourced, fresh on every tab, never
  a false zero), and `bg:` shows only the TUI''s own background procs. Each run is
  drawn exactly once. Every Agents-tab surface (concrete rows, session containers,
  local clan rows, tribe titles including rail and collapsed titles, and selection
  headers) shows ⚒ where the work is happening.

  '
phases:
- id: tool-proc-facts
  title: Tool-run proc facts (session stamping and join tags)
  depends_on: []
  size: small
  description: 'tool-proc-facts: stamp a tool-run proc''s session only when a live
    TUI submitted it; tag join monitors `tool-run-join:<id>` and teach the Procs pane
    to follow that tag.'
- id: core-glance-join
  title: Join fact on the ToolRun live glance (sase-core plus mirror)
  depends_on: []
  size: medium
  description: 'core-glance-join: add optional `join_kind`/`join_id` to the sase-core
    live-glance wire (batch-loaded), cover it with binding tests, and mirror it on
    Python `ToolRunGlance`. Commit both repos in one declaration so the pin moves.'
- id: top-bar-tools-bg
  title: Top-bar tools and bg split with tab-independent glance refresh
  depends_on:
  - tool-proc-facts
  size: medium
  description: 'top-bar-tools-bg: carrier classifier and the bg/tool/monitor/update
    partition; new ToolsIndicator (live, silent, bare-monitor chips, tooltip, all-projects
    click); ProcIndicator becomes `bg:`; the glance probe runs on every tab with stale
    and fallback states; Procs header ⚒ chip; top-bar docs, tests and goldens.'
- id: agents-tool-attribution
  title: Agents-tab attribution index and container rollups
  depends_on:
  - core-glance-join
  - top-bar-tools-bg
  size: medium
  description: 'agents-tool-attribution: build the run-to-node index once per generation
    (owner, joiner, starter, reused-name guard); roll runs up into session-container
    and local clan rows in one container badge slot; patch changed container rows;
    attribution matrix tests and row goldens.'
- id: tribe-titles-headers
  title: Tribe titles, rail titles, clan and tribe headers, docs and help
  depends_on:
  - agents-tool-attribution
  size: medium
  description: 'tribe-titles-headers: ⚒N/⚒⚠M on tribe border, collapsed and rail titles
    with targeted title refresh; clan header chip and Tool runs field through a union
    selector; tribe detail header and Summary card line; Agents-tab docs, help glyph
    rows and goldens; record follow-ups.'
proposed_by: bbugyi200.athena.research.43.linker.w0
create_time: 2026-10-08 18:31:51
status: wip
bead_id: sase-1ih
---

- **PROMPT:** [prompts/202610/tools_bg_split_tool_run_visibility.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/tools_bg_split_tool_run_visibility.md)
- **BEAD:** [sase-1ih](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ih/README.md)

# Plan: `tools:` / `bg:` split and ⚒ tool-run visibility

## Context

Agents run `sase tool run …`, and those runs often execute in procs
(`origin="tool-run"`, detached by inline escalation) or monitors. These procs are not
tied to the TUI session. They don't block restarts, and `collect_restart_blockers`
already treats them as non-blockers. Yet today they are counted as blue gears under the
top bar's `procs:` group, the lane that claims to mean "the TUI's own background work".
On athena, 47 of 84 retained non-service, non-monitor proc rows were agent tool runs.

The research report
`research:202610/tools_bg_split_and_tool_run_visibility/tools_bg_split_and_tool_run_visibility.md`
(read it with `sase artifact read <ref> "<why>"`) verified the code paths and live
state. It also found four bugs that the literal request would ship. **The user agreed
with every requirement in that report (A1–A8)**, and this plan adopts all of them.
Section [Lead-design refinements](#lead-design-refinements) lists where this plan is
more precise than the report, after re-checking the code.

Prior design this builds on is `plan:202609/tool_runs_tui_surfaces.md`, specifically D1
(`⚒` means ToolRun everywhere), D3, D4, and D15 (no chips on remote rows). This epic
intentionally reopens two parts of it: §3.3 (clan and tribe chips) and §6 (a top-bar ⚒).

## Design contract (every phase implements against this)

### One glyph per noun

| Glyph | Style                        | Means                                                                                       | Where                                                                     |
| ----- | ---------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `⚒`   | `TOOL_RUN_ACCENT` `#87D7FF`  | live tool run                                                                               | top bar `tools:`, tribe titles, container rows, row chips, headers, cards |
| `⚒⚠`  | `#FF5F5F` (`FAILURE_COLOR`)  | silent live run (no activity ≥ `silent_after_s`, 60 s)                                      | same places                                                               |
| `⚙`   | `MONITOR_GEAR_HUE` `#FFAF5F` | top bar: a monitor turn **not** carrying a run; rows and titles: a monitor node (unchanged) | `tools:`, monitor rows and titles                                         |
| `⚙`   | `PROC_GEAR_HUE` `#48CAE4`    | TUI background work                                                                         | `bg:`, Procs header                                                       |

Chips keep the existing `icon_count_chip` grammar: `{glyph} {n}` in bold `#1a1a1a` on
the hue, hidden at zero. **Counts are disjoint everywhere.** `⚒N` counts healthy live
runs and `⚒⚠M` counts silent ones, so the total is N+M. Do not add hues, an animation, a
status word such as `TOOLING`, or a border recolor.

### "⚒ beats ⚙": an execution is drawn once

A proc that carries a live ToolRun is in the **tool** lane and is never drawn as a gear.
Carriers are:

- a `tool-run` proc;
- an adopt monitor (tags `tool-run`, `tool-run:<id>`);
- a join monitor (new tag `tool-run-join:<id>`);
- an `ace` proc that is the `owner_id` of a live glance run with `owner_kind == "proc"`
  (the `: tool run` Command Line case).

Classify only by `origin` and the exact tags. Never classify by `session_id`, labels
such as `tool:`, or command prefixes. In the top bar, the ⚒ count comes from the ledger
glance (runs). The proc partition only decides what is _not_ a gear.

### Top bar (row 2, right-aligned cluster)

```text
full     tools: [⚒ 2][⚒⚠ 1][⚙ 1] · bg: [⚙ 1] · updates: … · stash: [≡ 2] · inbox: ⚑1 ✉18
common   tools: [⚒ 3] · stash: [≡ 2] · inbox: ⚑1 ✉18
compact  [⚒ 2][⚒⚠ 1][⚙ 1] · [⚙ 1] · [≡ 2] · ⚑1 ✉18
```

- **Order.** `tools` comes first, then `bg`, then the existing groups unchanged. Each
  chip hides at zero, and a group hides only when all of its chips are zero. Use the
  existing full/compact density machinery: no new tier and no wrapping.
- **`tools:` scope.** Every ledger run in `created` or `running` on this machine, in
  every mode (escalated, `-d`, `-H`, catalog `r`, `: tool run`, monitor adopt/join,
  foreground). A live child folds into its live parent. The scope ignores the project
  filter, tribe selection and session.
- **`tools:` tooltip.**

  ```text
  3 live tool runs on <hostname>
    ⚒⚠ test   0y9--mon                    silent 4m
    ⚒  check  sase-1id.5--1               7/11 · 20m
    ⚒  check  toobig-7f.plan_decisions.0  9/11 · 31m
  1 monitor not running a tool: acme--mon
  Click to open Admin Center › Tools (all projects)
  ```

  It lists at most 5 runs, silent first and then oldest. Each line reuses the existing
  chip vocabulary (`row_chip_text`, `BUCKET_STYLES`) and is minute-quantized.

- **`bg:` tooltip.** A headline `N TUI background procs`, then up to 5 labels (oldest
  first). A row whose origin is not `ace` shows its origin in parentheses. The last line
  is `Click to open the Procs tab`. The tooltip uses the words "TUI background", because
  the glossary's "background command" (`!` oneshots) means service rows, which are never
  in this lane.
- **Clicks.** `tools:` opens Admin Center › Tools › Runs with **all projects**, ordered
  silent → live → settled (the pane already orders this way), so the pane never shows
  fewer runs than the chip claims. `bg:` keeps opening the Procs tab. There is no keymap
  change, so `src/sase/default_config.yml` is untouched.
- **Never a false zero.**
  - Before the first glance load, the ⚒ chips are hidden. The orange chip may still
    show.
  - If loads keep failing for ≥10 s while the binding exists, keep the last snapshot,
    render the ⚒ chips dim with a trailing `?`, and say so in the tooltip.
  - If the binding is missing (ToolRun surfaces disabled), show a dim ⚒ lower bound that
    counts distinct run ids from carrier tags, with the tooltip
    `ledger unavailable — counting tool procs`.
  - A `truncated` glance renders `⚒ 200+`.

### Procs-tab header moves in lockstep

`Procs · this session  [⚙ bg][⚒ n][⚙ mon][⚙ upd]` is one partition of the same rows the
top bar classifies.

- Blue equals top-bar `bg:`. Keep however the header treats update rows today, but the
  blue/`bg:` equality must hold.
- `⚒` counts tool-lane **procs**. It is an inventory chip, so it shows a dim `⚒ 0` when
  empty, like blue and orange do today. It can differ from the top-bar run count:
  foreground runs have no proc, and a joined run has two procs. Docs explain this.
- Orange equals the top-bar orange (bare monitors).
- The Procs tab keeps listing tool procs with their existing `⚒ <label>` marker. Only
  the summaries change.

### Agents tab: attribution

Build a run → node assignment **once per (glance generation, node-directory revision)**
and reuse it for rows, container rows, titles and headers. A render path never scans the
runs per row. The node directory is built from the loaded Agents model, unfiltered and
including collapsed children. Attribution therefore never depends on collapse, filter or
selection. It holds:

- monitor ids;
- named-proc ids;
- concrete agent names, each with its node start times;
- remote flags.

The rules are exclusive and applied in order to each live run:

1. **Owner node.** If `owner_kind=monitor` and the owner id is a monitor node, or
   `owner_kind=proc` and the owner id is a named-proc node, the run goes to that node.
   This is the existing behavior.
2. **Joiner node.** If `join_kind=monitor` and the joiner is a monitor node, the run
   goes to that monitor. There is no `since_ts` guard, because the run predates its
   monitor.
3. **Starter.** If `agent` names a local concrete node, the run goes to that row. With
   reused names, pick the node with the latest start at or before `created_ts + 60 s`,
   falling back to the newest. This fixes every escalated or `-d` run, which today lands
   on no row, and covers owners or joiners without a visible node.
4. Otherwise the run is unattributed. The top bar still counts it.

**Containers.**

- A session container is the union of its members' assignments, walked the way
  `turn_lane_counts` walks members, so it includes member monitors' owned and joined
  runs.
- A local clan is the union over `clan_members(agent)` (the current generation) and
  their descendants. Never use a name-prefix guess. Never use `status_display_source`
  either: it points at the member that owns the status word, not at the members running
  tools.
- Rollups union run ids, fold live children, and never sum rendered badges.
- Collapse never changes a count. An explicit filter changes the represented membership.
- Remote rows and remote clans get nothing (D15). A mixed local/remote clan shows its
  local subset only.

### Agents tab: presentation

- **Concrete rows.** The grammar is unchanged (`cdx (RUNNING) ⚒ check 7/11`, with `+N`,
  `starting`, `stopping` and `⚒⚠ … silent 4m`), and the chip now appears for escalated
  runs. Status stays a lifecycle word. When a row is narrow, truncate the label before
  dropping the `⚒N`.
- **Container rows (clan and sequential session containers)** use one slot after the
  count chip or fold annotation (`×N [R… W…]`) and before the monitor `⚙N` and gate
  badges:
  - 0 runs: nothing.
  - 1 run: the full chip, `⚒ check 9/11` (or the silent variant).
  - 2 or more: `⚒N` in bold `#87D7FF`, plus `⚒⚠M` in bold `#FF5F5F` when any run is
    silent.

  Session-container rows move their chip from after `)` into this slot, so there is one
  container grammar. Clan status precedence is untouched, so `▸ research.43 (ASKING) ⚒2`
  is correct. The badge shows whether the row is collapsed or expanded.

- **Tribe titles** (border, collapsed): `⌂ @research · 5 [R3 W2] ⚒3 ⚒⚠1 ⚙1`.
  - `⚒N` and `⚒⚠M` go right after the metric chip, before named-proc `⚙`, monitor `⚙`
    and gate badges.
  - They stay out of `metric_items`, so status counts still sum to lanes.
  - Titles keep counting monitor **nodes**: titles count nodes, the top bar counts
    executions.
  - Merged tribe panels dedupe run ids.
- **Rail titles** add one compact token, `⚒N` (the total), in sky blue, or red when any
  run is silent. Over budget, pieces drop in this order: mark → icon → `×N` → ⚒ token.
  The hint, the label (minimum 3 cells) and the urgency roll-up are never dropped.
- **Headers.**
  - A local clan gets the same header chip as a node: the full chip for one run,
    `⚒N`/`⚒⚠M` for several.
  - It also gets a `Tool runs` field, through a union selector.
  - The tribe detail header appends `⚒N`/`⚒⚠M` after its count chip.
  - Activating a clan's header chip opens Admin Center › Tools › Runs (all projects)
    focused on the primary run. It never reveals a blank clan deck.
  - The clan Runs card waits.

### Refresh and performance (per the `tui_perf` memory)

- The stat-only drift probe runs on **every tab**, at most once every 2 s, piggybacking
  on the existing countdown tick. It is still skipped while navigating or while the
  prompt is active.
- The first glance no longer waits for the first Agents load.
- Loads stay `spawn_pump_free_task` + `asyncio.to_thread` with the 250 ms busy timeout
  and the `NavigationGate` deferral. Do not add a new timer or loop.
- Publishing a glance generation does the following:
  - It updates the top bar on all tabs.
  - On Agents, it patches only changed rows and container rows, plus titles whose ⚒ pair
    changed.
  - Returning to Agents with a newer generation runs one reconcile.
  - When a badge appears or disappears, it invalidates width caches and preserves
    selection and scroll.
- Silence can begin without a store write (a dead executor). Re-evaluate the top-bar
  model on the same ≤2 s probe cadence, and repaint only on change.
- No render or navigation path stats or opens the ToolRun store.
- **Target:** a start, stop or settle shows up everywhere within about 2 s.

### Non-goals

- Restart policy is unchanged. `collect_restart_blockers` must not start using the new
  classifier.
- No Agents-tab nodes for `tool-run` procs.
- No new Tools-pane run-id filter.
- No clan Runs card.
- No `⚒ N/cap` gauge.
- No backend reconcile of silent runs. The TUI never reconciles (D4).
- **No feature flag.** Phase `top-bar-tools-bg` replaces a mislabeled count with a
  correct one, and its Off branch would be the bug. The later phases are additive. Each
  phase lands a complete surface, so no landed phase exposes half a feature
  (`sase_flags` memory).

## Lead-design refinements

These are where this plan deliberately goes further than the research report, after
re-checking the code:

1. **The silent chip ships without a measurement gate.** Core derives `last_activity_ts`
   from the executor's 10 s resource samples, and the threshold is "six missed samples"
   (`TOOL_RUN_SILENT_AFTER_SECONDS`). A healthy long stage therefore never goes silent.
   Only a stuck or dead executor does.
2. **The join fact is a single `join_kind`/`join_id` pair, not a `joined_monitor_ids`
   list.** The store holds one `join_json` record per run, and a second joiner is
   refused (`JoinedElsewhere`).
3. **Starter attribution is generalized.** Any run without a visible owner or joiner
   node falls back to its `agent`'s row. There is no `launch_mode == handoff` test, so
   `-d`, escalation, adopt and future modes all work.
4. **Subtracting the `: tool run` ace owner is required, not optional.** "An execution
   is drawn once" is the core promise.
5. **One partition function** feeds the top bar and the Procs header, so they cannot
   drift apart.
6. **One container grammar.** Session containers adopt the clan badge slot.

## Phase `tool-proc-facts` (small)

**Session stamping (P0a).** In `src/sase/tool/handoff_launch.py::submit_handoff_run`,
stop stamping `resolve_session_ref(None)`. Stamp the session only when the submitting
process is **itself** a live registered session: `current_session_id()` is among
`live_sessions()`. That holds for the TUI catalog `r` path, where `launch_catalog_tool`
calls `execute_handoff` in the TUI process. In every other case, stamp
`session_id=None`: agent inline escalation, `sase tool run -d`, and a human shell `-H`.

- If no helper exists in `sase.sessions`, add a small one, for example
  `own_live_session_id()`.
- Do **not** change `resolve_session_ref` or `_resolve_current`. Telegram-approved epics
  rely on their latest-session fallback.

Tests cover three cases:

- An agent-like process with another live TUI registered stamps `None`.
- A process whose own session is registered and live is stamped.
- A shell with no live sessions stamps `None`.

**Join tags (P0b).**

- In `src/sase/monitor/start_launch.py`'s join branch, which currently leaves
  `proc_tags = []`, tag the monitor proc `tool-run-join:<run_id>`.
- Define the prefix and a `join_tags(run_id)` helper next to `owner_tags` in
  `src/sase/tool/handoff.py`. The prefix is distinct so that `tool-run:<id>` keeps
  meaning "owner".
- Teach `tool_run_id_for_task` (`src/sase/ace/tui/modals/procs_pane_render.py`) to fall
  back to the join tag, so the owner tag still wins and Enter on a join monitor opens
  its run.
- Check that `procs_pane_agent_jump.py` stays correct.

Tests:

- The join branch carries exactly the join tag, and the adopt branch is unchanged.
- The Procs-pane run-id lookup works for owner, join, and both kinds of tag.

## Phase `core-glance-join` (medium)

This phase crosses the Rust boundary (`rust_core_backend_boundary`). Open the linked
repo with `sase repo open sase-core -r "<why>"`, read its `AGENTS.md`, and work in the
printed path.

- **Wire.** `ToolRunGlanceWire` (`crates/sase_core/src/tool_run/projection/wire.rs`)
  gains `join_kind: Option<String>` and `join_id: Option<String>`, with
  `#[serde(default, skip_serializing_if = "Option::is_none")]`.
  - Populate them from the run's `join_json` (`ToolRunJoinRecordWire`) in the glance
    builder (`projection/glance.rs` and `projection/shared.rs::glance_for_row`).
  - Batch-load joins for the glance's run ids in one statement (for example through
    `BatchContext`). Do not add a query per row.
  - `release_join` clears the record, so the glance shows `None` again.
- **Tests (core).** Cover these glance cases: unjoined, joined, and released. Update any
  wire fixtures and schema snapshots. Add or extend the `sase_core_py` binding
  round-trip test.
  - Run `sase tool run check` inside the sase-core checkout. A targeted `-p sase_core`
    run is not enough.
- **Mirror (sase).** `ToolRunGlance` (`src/sase/core/tool_run_views.py`) gains
  `join_kind`/`join_id` through `_optional_str` in `from_wire`.
  - Python tests cover both cases: fields present, and fields absent (older wheel),
    which yields `None`.
  - Rebuild the local extension (`just rust-install`, which is long, so use
    `/sase_monitor` if needed) before any sase test that exercises the real binding.
- **Pin.** Commit both repos in the same declaration. The host commits the sase-core
  sibling first and writes its SHA into `sase-core-revision.txt` (`docs/rust_backend.md`
  › The CI source revision pin).
- This phase has no TUI behavior change. Consumers land in `agents-tool-attribution`.

## Phase `top-bar-tools-bg` (medium)

This phase lands atomically: the split, the fresh-on-every-tab refresh, and the Procs
header all arrive together.

1. **Partition** (`src/sase/ace/tui/_proc_observer_models.py`).
   - Add `is_tool_run_carrier(row)`. It is true when `origin == "tool-run"`, or the tags
     contain `tool-run`, `tool-run:<id>` or `tool-run-join:<id>`.
   - Add `tool_run_ids_for_row(row)`, which reads ids from the owner and join tags.
   - Rename the lane literal: `GearLane = Literal["bg", "tool", "monitor", "update"]`.
     Renaming `"proc"` to `"bg"` forces every consumer to be revisited.
   - `proc_gear_lane(row, *, tool_run_owner_proc_ids=frozenset())` applies this
     precedence:
     1. a carrier monitor → `tool`;
     2. a bare monitor → `monitor`;
     3. a service row → `None`;
     4. an update row → `update`;
     5. a carrier, or a row whose `durable_proc_id`/`proc_id` is in the owner set →
        `tool`;
     6. otherwise → `bg`.
   - `ProcGearLanes` exposes `bg`, `bg_rows` (oldest first), `tool_procs`, `monitors`
     (bare only), `update_rows`, and `tool_run_ids` (distinct run ids from carrier
     tags). It is computed in one pass.
   - `proc_gear_lanes(...)` threads the owner set through.
   - Update the `is_gear_eligible_row` docstring.
   - Keep `action_open_update_procs` working.
2. **Pure top-bar model**, in a new `src/sase/ace/tui/tool_runs/top_bar.py`.
   - Write a pure function of these inputs:
     - the glance snapshot (or `None`);
     - `tool_runs_disabled_reason()`;
     - the lanes;
     - `now`;
     - the load-failure state.
   - It returns a frozen model with these fields:
     - live and silent counts (disjoint, children folded, deduped);
     - `truncated`;
     - `stale`;
     - `fallback`;
     - the bare-monitor count and names;
     - tooltip text.
   - Also add `tool_run_owner_proc_ids(snapshot)`, which returns the `owner_id` of live
     runs with `owner_kind == "proc"`.
   - Reuse `is_silent`, `row_chip_text` and the existing fold helper. Do not
     re-implement them.
3. **Widgets.**
   - Add `ToolsIndicator(TopBarGroup)` in a new
     `src/sase/ace/tui/widgets/tools_indicator.py`:
     - `GROUP_LABEL = "tools"`;
     - `CLICK_ACTION = "open_live_tool_runs"`;
     - body: `⚒ N` on `TOOL_RUN_ACCENT`, then `⚒⚠ M` on `#FF5F5F`, then orange `⚙ K`;
     - `set_model(...)` is a no-op when nothing changed.
   - Extend `icon_count_chip` with optional display text, for `200+` and the dim stale
     `?` variant.
   - `ProcIndicator` becomes the `bg:` group:
     - `GROUP_LABEL = "bg"`, blue chip only, with the new tooltip;
     - keep the class name and the `#proc-indicator` id to limit churn, but fix the
       docstrings.
   - In `widgets/top_bar.py`, compose `ToolsIndicator(id="tools-indicator")` first and
     update `_TOP_BAR_GROUP_IDS`. Change the "five"/"four" docstrings to "six"/"five".
   - Add the lazy export in `widgets/__init__.py` and `__init__.pyi`.
4. **Wiring.**
   - `_proc_action_observer._update_proc_indicator` (keep the name, since many tests
     stub it) computes the lanes with the snapshot's owner set. It then updates
     `#tools-indicator`, `#proc-indicator` and the updates indicator.
   - In `tool_runs/loader.py`:
     - after `apply_loaded_snapshot`, call it on **every** tab;
     - track `failing_since_mono` in `ToolRunsLoadState`: set it when a load returns
       `None` while the binding exists, and clear it on success;
     - drop the `current_tab == "agents"` and `_agents_first_load_done` gates from the
       probe and scheduler;
     - keep the `_agents_loading` deferral;
     - patch rows only on the Agents tab;
     - on the Agents tab-switch hook, call `_apply_tool_runs_snapshot` once when the
       generation is newer than the last one applied;
     - re-evaluate the top-bar model inside the ≤2 s probe.
   - In `actions/_event_countdown.py`, move the probe call out of the agents-only
     branch, still behind the navigation and prompt gates.
5. **Click.** Add `action_open_live_tool_runs` in `actions/_base_admin.py`. It sets the
   Tools session state to `active_view="runs"` and `all_projects=True`, then calls
   `_open_config_center("tools")`. Leave `open_tool_runs_panel` unchanged.
6. **Procs header** (`modals/procs_pane_selection.py::_title_text`): render
   `[⚙ bg][⚒ n][⚙ mon][⚙ upd]` from the same partition and owner set. `⚒` uses
   `TOOL_RUN_GLYPH`/`TOOL_RUN_ACCENT` and shows a dim `⚒ 0` when empty.
7. **Docs** (`docs/ace.md`):
   - Rewrite "Top-Bar Indicators" to give the real group order and click targets.
   - Rename "Proc Indicator" to "Tools and Background Indicators" and rewrite it.
   - Update these Procs Tab subsections: Durability and Scope (header chips and the
     `bg:` semantics), Proc Status Icons, and Monitors on this tab.
   - In "Tools Tab", note that the top-bar click opens all projects.
   - Fix the top-bar sentence at `docs/integrations.md` (around line 248).
8. **Tests.**
   - Update these pinned tests on purpose:
     - `tests/ace/tui/test_top_bar_indicators.py` (`procs:` → `bg:`/`tools:`, click
       targets);
     - `tests/ace/tui/widgets/test_proc_indicator.py`;
     - `tests/ace/tui/widgets/test_top_bar_group.py`;
     - `tests/ace/tui/test_top_bar_order.py`;
     - `tests/ace/tui/test_proc_gear_lanes.py`, including `test_lane_totals_preserved`;
     - `tests/ace/tui/test_procs_pane_header_counts.py`;
     - `tests/ace/tui/test_services_phase_closure.py`;
     - `tests/ace/tui/test_proc_actions_session_workers_indicators.py`;
     - every `_update_proc_indicator` stub.
   - Add a classification matrix test:
     - escalated, `-d`, `-H`/catalog and `: tool run` rows are never `bg`;
     - adopt and join monitors are never orange while they carry a run;
     - a bare monitor is orange;
     - update and service precedence holds.
   - Add these model and refresh tests:
     - nested, joined and parallel runs count correctly;
     - a live child with a settled parent stays visible;
     - pre-load is hidden;
     - stale shows `?`;
     - the disabled fallback shows a dim lower bound;
     - truncated shows `200+`;
     - a run started while on Services appears without visiting Agents;
     - the count survives a TUI restart, because the ledger is session-free.
9. **Goldens.**
   - Top-bar PNGs at 80, 120 and 160 columns, with `tools` (live, silent, bare monitor),
     `bg`, `updates`, `stash` and `inbox` all populated.
   - Update `test_ace_png_snapshots_top_bar_indicators.py` and the neighbors test in
     `test_ace_png_snapshots_updates_indicator.py`.
   - Make the startup golden stub the glance (`seed_tool_run_surfaces`, or a reset), so
     no stray chip appears.
   - Inspect ⚒ sky against ⚙ cyan. If they blur in the PNG, darken only the top-bar ⚒
     fill one step (`#5FA8D3`). Never add a hue family.

## Phase `agents-tool-attribution` (medium)

1. **Index** (`src/sase/ace/tui/tool_runs/attribution.py`). Implement the rules in
   [Agents tab: attribution](#agents-tab-attribution) as a frozen index built from the
   snapshot plus a node directory.
   - Expose a module-level accessor in the style of `snapshot.get_snapshot()`, for
     example `current_attribution()`. It rebuilds lazily when the glance generation or
     the directory revision changes. Bump that revision at the single point where the
     loaded Agents model is replaced (see `actions/agents/_loading_apply.py`), and add a
     revision counter if none exists.
   - The API includes `runs_for_node(agent)`: an O(1) lookup for concrete nodes and a
     memoized union for containers.
   - Move every current caller of `select_live_runs` onto the index:
     - `_agent_list_widget.py::_row_has_live_tool_run`;
     - `_agent_list_render_agent_status.py::_append_tool_run_chip`;
     - `_agent_list_render_cache.py::_tool_run_chip_token`;
     - `models/agent_time.py::_tool_run_chip_ticks`;
     - `loader.py::_apply_tool_runs_snapshot`.
   - Rules 2 and 3 tolerate glances without `join_kind`/`join_id`, such as an older
     wheel. Join attribution then degrades to the starter's row.
2. **Container badge.**
   - Add one helper that appends the container badge in `format_agent_option`'s
     container slot, after the clan count chip, unknown-wait and fold annotation, and
     before the monitor and gate lanes. It covers both clans and sequential session
     containers.
   - Remove the session container's chip from after `)`.
   - `_tool_run_chip_token` returns the container rollup token. It contains the run ids,
     each run's progress and silent flag, and the minute bucket (only when one run shows
     a full chip). It feeds both `agent_render_key` and the `cached_format_agent_option`
     key.
   - Containers with a single full chip join the minute-tick set.
3. **Patching.** `_apply_tool_runs_snapshot` compares per-identity chip tokens with the
   last applied set (not mere presence), and patches changed concrete **and container**
   rows. Keep the existing rebuild escalation. Invalidate width caches when a badge
   appears or disappears, and preserve selection and scroll.
4. **Headers for concrete nodes.** `summaries.node_live_runs` and the monitor selector
   include runs the monitor joined, without the `since_ts` guard, so a monitor's header
   agrees with its row.
5. **Tests.**
   - An attribution matrix that covers these cases:
     - an escalated run on its starter;
     - `-d`;
     - adopt;
     - join with and without join fields;
     - a hidden owner falling back to the starter;
     - a session member;
     - a session member monitor's owned run;
     - a clan with 1 run and with N runs;
     - an ASKING clan with tools;
     - a remote clan;
     - a mixed local/remote clan;
     - reused names;
     - collapse invariance.
   - Flip `test_remote_and_clan_rows_never_match`
     (`tests/ace/tui/test_tool_runs_glance.py`) so that remotes still never match and
     local clans do.
   - A guard test that the index builds once per generation and revision, not once per
     row.
6. **Goldens.** Add PNG tests next to `test_ace_png_snapshots_tool_runs_rows.py` and
   `_tool_runs_session.py`, using `seed_tool_run_surfaces`. Cover:
   - an escalated concrete row;
   - a session container with runs;
   - clan rows, collapsed and expanded, with 1 run, with N runs, and with silent runs.

   Inspect every change. Do not batch-accept.

## Phase `tribe-titles-headers` (medium)

1. **Titles** (`actions/agents/_display_panel_titles.py`).
   - `AgentPanelCounts` gains `tool_runs_live` and `tool_runs_silent`. They are excluded
     from `metric_items`.
   - `agent_panel_counts` unions the index's runs over the panel's nodes and dedupes by
     run id.
   - `agent_panel_border_title` renders `⚒N` and `⚒⚠M` after the metric chip, before the
     named-proc, monitor and gate badges. This covers collapsed titles too.
2. **Rail** (`widgets/_agent_list_render_rail.py::rail_panel_title`). Add the compact
   `⚒N` token, using the drop order in the design contract, and update the docstring's
   piece list.
3. **Title refresh.** After a published glance on the Agents tab, compare each panel's
   (live, silent) pair with the last pair rendered. Call `_refresh_agent_panel_titles()`
   once if any pair changed.
4. **Clan and tribe headers.**
   - In `tool_runs/summaries.py::selector_for_agent`, a local clan gets a union
     selector:
     - agents ∪ owners ∪ joined monitors of its members;
     - `since_ts` = the earliest member start − 60 s.
   - Remote clans stay `None`. A mixed clan shows its local subset with the copy "local
     runs only — ToolRun history lives on `<machine>`".
   - In `tool_runs/header_chip.py`, replace the `is_clan_container` early returns in
     `header_chip_for_node` and `tool_runs_field_entries` with the clan path.
   - Header activation (`header_run_id_for_agent` / `%r`) on a clan opens Admin Center ›
     Tools › Runs (all projects), focused on the primary run through
     `tool_run_focus_target`.
   - `probe_tool_runs_card` keeps clans card-less.
   - The tribe detail header (`widgets/prompt_panel/_agent_display_tribe_header.py`)
     appends `⚒N`/`⚒⚠M` after its count chip.
   - Tribe and clan Summary cards add a `⚒ N live tool runs` line when N > 0.
5. **Help and docs.**
   - Add `⚒`, `⚒N` and `⚒⚠N` rows to the help modal's "Agent Row Glyphs" box
     (`help_modal/agents_reference_sections.py`). Follow the 57-character box rules in
     `src/sase/ace/CLAUDE.md`.
   - Update these `docs/ace.md` sections:
     - Agents Tab Tool Runs (starter attribution, containers, clan header);
     - Group Banners and Folding (the clan badge slot);
     - Agent Row Glyphs;
     - Tribe Side Panels (the title badge);
     - Agent Data Decks and Cards.
   - Fix the stale `▣N` named-proc title line: the code renders `⚙`.
   - Update `docs/tool.md` › In the TUI.
6. **Tests.**
   - Split `test_remote_and_clan_have_no_chip_or_field`
     (`tests/ace/tui/test_tool_runs_header_chip.py`): remotes stay `None`, and local
     clans get the chip and the field.
   - Extend the title tests (`test_agent_panel_titles.py`,
     `test_agent_panel_title_counts.py`) and the rail-title tests, including budget
     drops.
   - Add a merged-panel dedupe test.
   - Add PNG goldens for a tribe title with live and silent runs (border, collapsed and
     rail) and for a clan header chip. Inspect each one.
7. **Follow-ups.** Record `PROPOSED FOLLOW-UP:` notes on this phase's bead for:
   - a backend reconcile job for silent runs;
   - a `⚒ N/cap` gauge once tool admission lands.

## Verification (every phase)

- Read the `lint_and_test` memory, plus the `tui`, `tui_perf` and `tui_screenshot`
  memories for TUI phases.
- Run `sase tool run check` in sase, and in sase-core for `core-glance-join`. Do not run
  `just check-full`.
- Update PNG goldens with `just fix-tui-screenshots -- <selectors>` through
  `/sase_monitor`. Inspect every created and updated golden before finalizing, because
  generation is not approval.
- For each TUI phase, take one live `sase screenshot` to confirm the visible result. Use
  a real `sase tool run` if one is available, but treat the live PNG as a check, not as
  a golden.
