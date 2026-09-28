---
tier: tale
size: medium
title:
  Finish landing sase-1bt - wire every ToolRun jump, fix glance-apply rebuilds, and
  close the epic
goal:
  Every ToolRun surface that promises to open a run actually lands on that run's ⚒ Runs
  block, the glance snapshot apply never forces a full agent-list rebuild for rows the
  user cannot see and clears a settled chip on the next apply, and epic sase-1bt is
  closed with a clean Symvision whitelist.
proposed_by: bbugyi200.athena.sase-1bt.land
bead: sase-1bt
create_time: 2026-09-28 14:37:36
status: wip
---

- **PARENT:**
  [202609/tool_runs_tui_surfaces.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_runs_tui_surfaces.md)
- **BEAD:**
  [sase-1bt](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1bt/README.md)

# Finish landing sase-1bt (ToolRun TUI surfaces)

## Context

Epic `sase-1bt` (plan `plan:202609/tool_runs_tui_surfaces.md`) shipped the ToolRun TUI
surfaces. All 13 phase beads are closed. Its land agent verified the work and found
epic-caused gaps that must be finished before the epic can close. This tale does that
work and then performs the epic closeout itself. **Nothing resumes the landing after
this tale**, so the final section is mandatory.

Already done by the land agent (verify, don't redo):

- `src/sase/config/sase.schema.json` got `stop_run` and `run_tool` entries in the
  `ace.keymaps.tool_runs` block (sase-1bt.11 added the keys without schema). If they are
  missing from your checkout, add them, placing them between `copy_run_id` and `reload`.
  The descriptions are "Stop the selected live tool run" and "Run the selected catalog
  tool as a hand-off". `tests/test_config_schema.py` must pass.
- The flag bead `sase-1bv` (`ace_tool_runs`) is closed.
- Follow-up triage is recorded on `sase-1bt`: tasks `sase-1c8` and `sase-1c9` were
  filed, and `sase-1a7` got a +1. Do not re-triage.

Read before editing: `sase memory read tui.md -r "<why>"` and
`sase memory read symvision.md -r "<why>"`. The sase-core step also needs the linked
checkout's `AGENTS.md` (open it with `sase repo open sase-core -r "<why>"` and work only
in the printed path).

## Verified gaps (all caused by the epic)

1. **Admin Center → Tools `a` never works.**
   - Where: `src/sase/ace/tui/modals/tool_runs_pane_tool_actions.py`
     `action_jump_to_agent` (epic note #2).
   - It passes the brief's plain agent-name string to `app._reveal_agent_row`. That
     method expects a `MemberIdentity` tuple, which `prepare_agent_navigation_target`
     compares with `agent.identity ==`. So every press closes the modal and toasts a
     failure.
   - It also never shows Tools or selects the run's block, which the plan requires:
     "close the modal, set the tab, then `_reveal_agent_row`, and select the run's
     block".
   - No test drives the real path.
2. **Failures view `enter`/`a` are stubs.**
   - `enter` (`tool_runs_pane_events.py`) only re-renders counts. `a` in Failures looks
     up `"failure-N"` as a run id and warns "No owning agent".
   - The plan requires `enter` to list the affected runs and agents in the detail
     region, and `a` to jump to one ("Failures reaches affected agents" is an epic done
     criterion).
   - The core `tool_run_failures` wire (`ToolRunFailuresGroupWire`) only carries counts
     plus `last_run_id`/`newest_owners`, so the per-run list needs a small opt-in
     sase-core extension.
3. **Run-link jumps are not wired** (sase-1bt.9's PROPOSED FOLLOW-UP):
   - `src/sase/ace/tui/tool_runs/links_jumps.py` (`tool_run_jump_target`,
     `run_id_from_jump_target`, `run_jump_hint_label`, `visible_tool_run_jump_targets`)
     has no non-test consumer.
   - The LLM Calls suffix (`_llm_calls_panel_timeline.py` `_append_run_link_suffix` →
     `links_suffixes.suffix_text_with_jump`) and the monitor/named-proc Context
     `Tool run` row (`_agent_monitor_section.py`, `_agent_named_proc_section.py` via
     `context_tool_run_line`) are rendered but not clickable.
   - `links_suffixes._stamp_jump_meta` stamps `DECK_BLOCK_META_KEY`
     (`"sase_card_block"`, the block-anchor key). That collides with real block headers
     and makes `_section_navigation.segment_section_identity` create bogus
     `block:<run_id>` anchors in the prompt panel.
   - `v` hint mode (`actions/hints/_files.py` `_append_tool_run_log_hint_targets`)
     offers only `⚒ run log` targets, never `⚒ run <8hex>` jumps.
     `actions/hints/_view_processing.py` would treat an unknown `toolrun-jump:` target
     as a file path.
4. **OpenToolRun stops short.**
   - `actions/agents/_notification_handlers.py` `handle_open_tool_run` jumps to the node
     and shows the Tools deck, but never activates the ⚒ Runs card or selects the run's
     block. The plan says "select that node and its Runs block (`p t`)".
5. **Focus target on an already-loaded pane.**
   - `modals/tool_runs_pane_shell.py` (~line 181) `focus_tool_run`: once `_loaded_once`,
     it calls `_select_run()` before switching to the Runs view. A Procs `⚒` Enter or
     OpenToolRun fallback that arrives while the pane is on Failures or Catalog
     therefore selects nothing visible.
6. **Glance apply rebuild storm** (epic note #3).
   - `src/sase/ace/tui/tool_runs/loader.py` `_apply_tool_runs_snapshot` walks
     `_agents_with_children` and calls `_try_patch_agent_row` for every row with a live
     run.
   - That call returns False for rows not in the rendered `_agents` list, and whenever
     `current_tab != "agents"`. So `needs_rebuild` forces
     `_refresh_agents_display(list_changed=True)` on every ~2 s drift apply while any
     live run belongs to an off-tab, hidden, or collapsed row, or while the user is on
     another top-level tab.
   - Separately, rows whose chip disappeared on settle are never patched ("rely on the
     next auto-refresh"). The plan says a settled run leaves the row on the next glance.

## Implementation

### A. One shared "reveal a run's block" helper

- Add one helper, for example in `src/sase/ace/tui/actions/agents/_tool_run_actions.py`
  or a sibling module under 500 lines. Given `run_id` plus optional owner facts
  (`agent`, `owner_kind`, `owner_id`):
  1. resolves the owning local row with owner-first matching. Reuse
     `_tool_run_node_matches` from `_notification_handlers.py`: move it somewhere shared
     if needed, and don't copy it. Resolve with `resolve_loaded_agent`, then navigate
     with `jump_to_loaded_agent` (`_notification_navigation.py`). Both already handle
     agent tabs, hidden rows, and filter clearing.
  2. shows the Tools deck (`_show_tools_deck_best_effort` / `action_show_tool_runs_card`
     logic).
  3. activates the `runs` card (`TOOLS_RUNS_CARD_ID`, `DeckPanel._show_tools_card`).
  4. selects the block via `DeckPanel.select_block(run_id)` /
     `select_document_block(DeckId.TOOLS, run_id)`.
- Selecting a block needs a pending selection. The Tools document for a newly selected
  node loads on a worker (`document.partial`, `ToolRunsDeckLoaded`), so
  `select_document_block` returns False until the load lands.
  - Store a pending `run_id` on the Tools host, keyed to the node identity.
  - Apply it once, when that node's Runs document lands.
  - Drop it on node change.
  - Follow the "pending until first load, then select" pattern the Admin pane uses for
    `ToolRunFocusTarget`.
  - A single-run Runs document is block-less (no rail), so treat "the only run is shown"
    as success.
- When the run belongs to the currently selected node (LLM Calls, Context, and `v`
  jumps), skip the row navigation and do steps 2–4 only.
- The helper must do no SQLite, stat, or log I/O on the keystroke path; owner facts come
  from in-memory briefs or summaries. `reconcile_unsettled_tool_runs` must never be
  reachable.
- Toast `No agent row for <name>` (or a similar message) when no local row matches.

### B. Admin Center → Tools `a` (Runs and Failures)

- Runs `a`: close the modal and call the helper from `call_after_refresh`, passing the
  brief's `run_id`, `agent`, `owner_kind`, and `owner_id`. The helper resolves the row
  and passes `row.identity` (never a name string) to the reveal path. The procs pane's
  `_monitor_jump_agent` (`modals/procs_pane_agent_jump.py`) shows the row-resolution
  idiom if you need it.
- Failures `enter`: list the group's affected runs in the detail region, newest first
  and bounded. Show 8-hex run id, agent, time, and class, plus the distinct agent set.
- Failures `a`: jump to the newest affected run whose owner row resolves locally; toast
  when none does.
- Data: see step C. When the loaded core lacks the new field, fall back to `last_run_id`
  plus `newest_owners`. The pane must stay useful and must never crash.

### C. sase-core: opt-in affected runs on `tool_run_failures`

Work in the `sase-core` checkout from `sase repo open sase-core` and follow its
`AGENTS.md`.

- `crates/sase_core/src/tool_run/triage/failures.rs`:
  - Add an opt-in request field to `ToolRunFailuresRequestWire`, for example
    `#[serde(default)] runs_limit: u32` (0 = off, capped at 50).
  - Add a `#[serde(default, skip_serializing_if = "Vec::is_empty")] affected_runs` list
    to `ToolRunFailuresGroupWire`. Each entry is
    `{run_id, agent?, workspace?, created_ts, class?, owner_kind?, owner_id?}`, newest
    first.
- `crates/sase_core/src/tool_run/store/failures.rs`:
  - The grouping loop already holds every row's `run_id`, agent, workspace,
    `created_ts`, and class. Add the run's `owner_kind`/`owner_id` to the query only if
    they are not already selected.
- Default output must stay byte-identical, so `sase tool failures` CLI and JSON output
  do not change.
- Add Rust tests: off by default, bounded, newest first, and distinct runs.
- Register nothing new: the binding name is unchanged.
- Pass `sase tool run check` inside that checkout.
- The sase side must work against the currently pinned core, which lacks the field. The
  pin moves later through the core-pin ratchet workflow; do not hand-edit
  `sase-core-revision.txt` to an uncommitted SHA. The sase tests for the Failures detail
  use fixture dicts, with and without `affected_runs`, not the live binding's new field.
- The Tools pane passes `runs_limit` (e.g. 20) in its `_load_failures` request. It stays
  a pure read and never calls `src/sase/tool/failures.py`, which reconciles.

### D. Run-link click targets and `v` jumps

- Stop stamping `DECK_BLOCK_META_KEY` in `links_suffixes._stamp_jump_meta`. Stamp a
  dedicated meta key instead, e.g. `sase_toolrun_jump` holding the run id, defined next
  to `TOOLRUN_JUMP_TARGET_PREFIX`.
- Add `on_click` handling to `AgentLLMCallsPanel` (`widgets/llm_calls_panel.py`) and to
  the prompt-panel widget that renders the monitor/named-proc Context sections. The
  handler reads `event.style.meta` for that key and calls the helper from A (same-node
  path).
- Model the dispatch on `BlockRail.on_click` → `BlockRailSelected` →
  `DeckPanel._on_block_rail_selected` (post a message rather than reaching into the
  app).
- Keep the markdown twins text-only.
- `v` hint mode:
  - Add one `⚒ run <8hex>` jump target (`visible_tool_run_jump_targets`) per run linked
    from a visible LLM Calls row or Context `Tool run` row. Also add one per visible
    Runs block, alongside the existing `⚒ run log` targets.
  - Add a `run_id_from_jump_target` branch in `_view_processing.py`, before the
    file-path fallback, that calls the helper.
- Every `links_jumps` public symbol must end up with a real non-test consumer. Otherwise
  privatize or delete it per `symvision.md`.

### E. OpenToolRun and the focus target

- `handle_open_tool_run`: after `jump_to_loaded_agent`, use the helper to activate the
  Runs card and select the run's block. Keep the Admin Center fallback unchanged.
- `focus_tool_run` (`tool_runs_pane_shell.py`): always switch to the Runs view first,
  then select (loaded) or hold pending (not loaded). Clear the modal's stale
  `_tool_run_focus_target` after delivery if it is re-read on later tab switches.

### F. Glance apply

In `_apply_tool_runs_snapshot`:

- return early when `current_tab != "agents"`;
- patch only rows in the rendered `_agents` list (build the visible identity set from
  `_agents`), and skip others, which repaint on their next normal rebuild;
- remember which visible identities carried a chip on the previous apply, and patch
  those whose chip is now gone, so a settled run leaves its row on the next glance;
- escalate to `_refresh_agents_display(list_changed=True, defer_detail=True)` only when
  a patch of a _visible_ row fails.

Keep the render path pure.

### Tests

Add them next to the existing suites: `tests/ace/tui/test_tool_runs_pane.py`,
`test_tool_runs_run_links.py`, `test_tool_runs_glance.py`, `test_tool_run_actions.py`,
and `test_tool_runs_deck_cards.py`.

- Admin `a` drives the **real** `_reveal_agent_row` / `jump_to_loaded_agent` path, not a
  monkeypatch of it. Cover an inline agent run, a monitor-owned run (monitor row, not
  its starter), agent tabs on and off, and a miss that toasts.
- Failures `enter` renders affected runs and agents from a fixture group, with and
  without `affected_runs`. Failures `a` jumps to the newest resolvable run.
- The pending Runs-block selection applies after the Tools document loads, is dropped on
  node change, and treats a single-run document as success.
- An LLM Calls suffix click and a Context-row click select the right block. `v` lists
  `⚒ run <8hex>` targets and opening one selects the block. A `toolrun-jump:` target
  never reaches the file opener. The prompt panel no longer gets `block:<run_id>`
  anchors from Context rows.
- OpenToolRun on a visible node ends with the Runs card active and the run's block
  selected.
- A focus target delivered while the pane is on Failures or Catalog switches to Runs and
  selects the run.
- Glance apply:
  - a live run on an off-tab, hidden, or collapsed row triggers no full rebuild;
  - `current_tab != "agents"` is a no-op;
  - a settled run's chip is patched off on the next apply;
  - a progress change is still a patch, not a rebuild.
- No reconcile: the existing "patched reconcile raises" guard stays green across the new
  paths.

### Verification

- Run targeted pytest for the touched suites.
- Run `just fix`, then `sase tool run check` in sase, and in sase-core for step C. Use
  `/sase_monitor` if the full check would exceed the turn. Do not run `just check-full`.
- Do a live `sase screenshot` inspection of:
  - Admin Tools `a` landing on the run's block;
  - an LLM Calls suffix jump;
  - Failures `enter` detail.
- If a golden legitimately changes, rebaseline it only with targeted
  `just fix-tui-screenshots -- <selectors>` and inspect every changed PNG.
- Update `docs/ace.md`'s Tools-tab and run-link rows if the key behavior text changes
  (e.g. Failures `enter`/`a`).

## Epic closeout (mandatory final step - do this last, in order)

1. `sase bead epic-symbols sase-1bt` currently lists four entries in the `Justfile`
   `_lint-symvision` recipe:
   - `sase-1bt(RowIdentity)`;
   - `sase-1bt(ToolRunGlanceSnapshot)`;
   - `sase-1bt(ToolRunLogTail)`;
   - `sase-1bt(ToolRunStateStyle)`.

   Each is a public type used only inside its own module:
   - `RowIdentity` in `src/sase/ace/tui/tool_runs/attribution.py` (its only other import
     is a test);
   - `ToolRunGlanceSnapshot` in `tool_runs/snapshot.py`;
   - `ToolRunLogTail` in `src/sase/tool/logs.py`;
   - `ToolRunStateStyle` in `src/sase/tool/view_vocabulary.py`.

   Resolve every one per the `symvision.md` hierarchy: give it a real non-test consumer
   that genuinely needs the type (for example a type annotation in a module that already
   handles those values), or make it private and update in-file uses, `__all__`, and
   test imports. Then delete its `--epic-symbol` line.

   No still-open bead needs these exemptions, so do not re-key them. Rerun
   `sase bead epic-symbols sase-1bt` and confirm it lists nothing. Also confirm
   `just _lint-symvision` passes.

2. Close `sase-189` ("Give the tool-run settlement notification a real action that opens
   the settled ToolRun"): `sase bead close sase-189 --note "<what you verified>"`.
   sase-1bt.11 implemented OpenToolRun, and this tale made it select the run's Runs
   block.
3. Close the epic: `sase bead close sase-1bt --note "<verification>"`. Summarize:
   - the land agent's verification of all 13 phases and its triage note;
   - the gaps fixed here (A–F and the schema fix);
   - the test and check results.
   - If the close is rejected for leftover epic symbols, finish step 1 and close again.
   - Never use `--force` merely to make the close succeed.
4. Run `just symvision` and confirm the whitelist is clean.
5. Set `status: done` in the YAML frontmatter of the epic's plan file (the PLAN path
   `sase bead read sase-1bt -r "<why>"` prints: `202609/tool_runs_tui_surfaces.md` in
   the plans repo).
6. `sase-1bt` has no `parent_bead`; nothing further to close.
