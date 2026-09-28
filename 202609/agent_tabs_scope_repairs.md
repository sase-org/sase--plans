---
tier: epic
title:
  "Agent tabs: repair the scope pipeline, tab switching, cross-tab jumps, and scope
  wording"
goal: "Fix the defects the sase-1bc.6.1 landing review found in the flagged agent-tabs
  feature before that epic closes. Worker-path status overrides and the tab-index memo
  work. A tab switch restores the target tab's own selection and focused panel. Emptied
  machine tabs never strand the user. Every cross-tab jump records the right back-anchor
  and restores the previous tab when its reveal fails. Bulk wording states the real
  scope. The epic's public symbols pass symvision. With the `agent_tabs` flag off, the
  TUI stays exactly as it was before agent tabs.

  "
phases:
  - id: switch-pipeline-repairs
    title: Scope pipeline, tab switch memory, catalog maintenance, and key yield fixes
    depends_on: []
    size: medium
    description:
      "switch-pipeline-repairs: apply worker-path status overrides over the
      tab-independent query result; make the tab-index memo hit on an unchanged roster
      without retaining old rosters; restore each tab's own row index, focused panel,
      and scroll anchor; latch emptied machine tabs instead of stranding the user; fix
      the doubled machine-gone toast glyph, the empty-first-catalog startup skip, and
      the attention-tab choice; unbind only the colliding bracket key in the legacy
      yield; hide the agent-tab help rows with the flag off."
  - id: cross-tab-jump-repairs
    title:
      Back-anchors, failed-reveal restore, and fold-aware reveal for every cross-tab
      jump
    depends_on:
      - switch-pipeline-repairs
    size: medium
    description:
      "cross-tab-jump-repairs: save the back-anchor before switching tabs in
      _try_reveal_agent_row; drop the notification pre-switch in favor of the Node
      Finder ladder; route the run-log, revive, Files, and link-trail jumps through the
      fold-expanding reveal with tab restore; restore the tab when a back-jump or a
      last-launch reveal fails; make the ,j off-tab path reveal-aware; show the off-tab
      chip on every off-tab Node Finder row; keep flag-off lookups unchanged; add tests
      for every entry point."
  - id: scope-wording-and-symbols
    title: Honest marked and custom scope wording, docs accuracy, and symvision cleanup
    depends_on:
      - switch-pipeline-repairs
      - cross-tab-jump-repairs
    size: medium
    description:
      'scope-wording-and-symbols: count the "N of M marked agents are on other tabs"
      line over one consistent set; make the custom cleanup header name the active tab;
      correct the docs/ace.md agent-tabs note and the flag table wording; resolve every
      symvision-flagged public symbol this epic added (privatize, delete, or re-key to a
      still-open sase-1bc phase), so no agent-tabs symbol remains in the symvision
      output.'
parent_bead: sase-1bc.6.1
proposed_by: bbugyi200.athena.sase-1bc.6.1.land
create_time: 2026-09-27 20:11:03
status: wip
---

- **PROMPT:**
  [prompts/202609/agent_tabs_scope_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_tabs_scope_repairs.md)
- **PARENT:**
  [202609/agent_tabs_scope.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_tabs_scope.md)

# Plan: repair the agent-tabs scope, switching, jumps, and wording

## Context

Epic `sase-1bc.6.1` (plan `plan:202609/agent_tabs_scope.md`) put agent tabs behind the
`agent_tabs` beta flag. It added a per-root tab index, an active-tab scope stage,
per-tab memory, `[`/`]` keys, a minimal strip, switch-then-reveal cross-tab jumps, and
tab-scoped bulk confirmations. All five phases closed, but the landing review found
defects that the epic caused. They are all flag-on only, except the help rows below.
This plan fixes them so the epic can close. Read that plan's "Rules that bind every
phase" and the parent plan's design contract (`plan:202609/agents_dynamic_tabs.md`)
first. Both still bind this work.

Rules for every phase:

- **Flag gating.** With `agent_tabs` off, every behavior stays byte-identical to the
  pre-epic TUI. Keep the Off branches explicit.
- **tui_perf.** Read `sase memory read tui_perf.md` before touching switch or refresh
  paths. A tab switch does no disk, network, or re-projection work. Re-read the active
  tab after every await.
- **Tests.** Cover both flag states, and prefer the real mixins over stubbed helpers.
  The existing suites to extend are:
  - `tests/ace/tui/test_agent_tab_scope.py`
  - `tests/ace/tui/test_agent_tab_switch.py`
  - `tests/ace/tui/test_agent_tab_cross_nav.py`
  - `tests/ace/tui/test_agent_tab_scope_honesty.py`
  - `tests/ace/tui/models/test_agent_tab_index.py`

  Outside pytest, the agent shell may export `SASE_FEATURE_FLAGS` with
  `"agent_tabs":false`, which beats `override_flags`. Unset it for ad-hoc scripts.

- **Verification.** Run `sase tool run check` and never `check-full`. Run golden checks
  through `/sase_monitor` with targeted selectors and inspect every change. The flag-off
  goldens must not change.
- **Known pre-existing failures.** These are tracked elsewhere and are not this plan's
  work:
  - `tests/ace/tui/test_agent_completion.py` and the two directive-completion absence
    tests (DISCOVERED ISSUE notes on `sase-1bc`, caused by `%tab`);
  - `test_expanded_overflowing_header_claims_half_page_scroll` (`sase-1b8`);
  - `test_no_system_clock_display_sites` (`sase-1bp`);
  - the `usage_windows.py` symvision pragmas (`sase-1bj`);
  - symvision unused-public symbols outside the agent-tabs files (`sase-1ay`).

Paths below are under `src/sase/ace/tui/` unless noted. Line numbers are approximate.

## Phase 1 — switch-pipeline-repairs

1. **Worker-path status overrides.**
   - **Bug.** `_apply_finalize_plan` (`actions/agents/_loading_finalize.py`, the "Apply
     status overrides" block) applies `plan.overrides.overrides_to_apply` only to rows
     in `app._agents`, which is already scoped to the active tab. The inline path
     applies overrides before scoping.
   - **Symptom.** A status override set for a row on tab A is lost when a worker reload
     lands while tab B is active. Switching back to A shows the stale on-disk status.
   - **Fix.** Apply the overrides over `app._agents_query_result`. The scoped rows are
     the same objects, so they pick the overrides up too.
   - **Same block.** Prune `_agent_content_search_cache` against the query result, not
     the scoped rows, so off-tab entries survive a switch. Match the inline path's
     behavior.
   - **Test.** Flag on, two tabs: an override for a tab-A row survives a worker apply
     while B is active and shows after switching to A.
2. **Tab-index memo.**
   - **Bug.** `refresh_agent_tab_index` (`actions/agents/_tab_scope.py`) passes a new
     `list(owner._agents_with_children)` copy on every call. `cached_agent_tab_index`
     (`models/agent_tab_index.py`) keys on `(id(roster), len(roster), token)` and checks
     the roster with `is`, so it never hits.
   - **Symptom.** Every inline finalize and every dismiss or kill re-walks the anchors
     and calls the Rust binding. The 16-entry cache also retains up to 16 stale rosters.
   - **Fix requirements:**
     - the memo hits on an unchanged roster;
     - any membership change, including in-place `append`/`remove`, misses;
     - retained rosters stay bounded to one or two.
   - **Tests.** Repeated refreshes on an unchanged roster build once, an in-place
     mutation rebuilds, and cache retention stays bounded. `test_memo_hit_and_miss` only
     passes the same list object today, so keep it and add these.
3. **Per-tab memory restore** (`_remember_active_tab_memory` / `_restore_tab_memory` in
   `actions/agents/_agent_tabs.py`).
   - **Wrong fallback index.** The nearest-row fallback clamps `_agents_last_idx`, which
     is the source tab's index. Store each tab's own row index in its memory and use
     that. A first visit to a tab with no memory selects row 0.
   - **Focused panel dropped.** The saved focused panel key is unpacked into
     `_panel_key` and discarded. Restore it when the panel still exists on the target
     tab.
   - **Scroll anchor.** Confirm the scroll anchor is restored, and restore it if it is
     not.
   - **Tests.** Use a multi-row source tab: a first visit lands on row 0, the fallback
     uses the target tab's own remembered index, and the focused panel is restored.
     `test_selection_falls_back_to_nearest_row` only passes today because its source tab
     has one row.
4. **Catalog maintenance** (`_reconcile_active_agent_tab`, `_machine_fallback`).
   - **Stranded machine tab.** When every root of the active machine tab disappears but
     the machine is still configured, the user is stuck:
     - machine keys are excluded from the latch, and `_machine_fallback` returns None;
     - so the tab stays active with 0 rows, the strip hides, and `]`, `[`, and the
       picker all do nothing.

     Fix: latch machine keys like any other key. Fall back to the default tab only when
     the machine left the configuration or machine mode is off.

   - **Doubled toast glyph.** The toast reads `agent tab ⌨ ⌨ apollo is gone`, because
     known labels come from the catalog as `⌨ apollo` and the toast prepends `⌨` again.
     Fix the toast text:
     - use the stored label as it is;
     - with no stored label, use the machine alias from the view config, not the
       installation id;
     - name the destination tab by its real label.

     `test_agent_tab_switch.py` injects a bare `apollo` label, which hides the bug.
     Switch it to a real catalog label.

   - **Startup selection skipped.** `_agent_tabs_reconciled_once` is set even when the
     first catalog is empty, so startup selection never runs for the first real catalog.
     Only settle startup selection on a non-empty catalog, and only while the user has
     not switched.
   - **Attention tab.** Startup step 3 should pick the tab holding the newest root that
     needs attention. It currently takes the first matching row in roster order, child
     rows included. Consider roots only and pick the newest.
   - **AcePage.** `agent_tabs` in `ace/testing/ace_page.py` should list the catalog view
     the strip shows, latched key included.
   - **Tests.** Machine-tab latch and navigating away, the toast text with a real label,
     an empty first catalog followed by a real catalog, and the newest-attention choice.

5. **Legacy bracket yield** (`keymaps/registry.py`, the loop after the legacy card-block
   warning).
   - **Current behavior.** Any bracketed card-block key unbinds both `next_agents_tab`
     and `prev_agents_tab`. "Explicitly bound" is detected as "differs from the
     default", so a user's explicit `next_agents_tab: "]"` is still unbound.
   - **Fix.**
     - Unbind only the tab action whose key actually collides with a card-block bracket
       binding.
     - Detect an explicit user binding by its presence in the user's keymap config.
     - Keep the no-double-binding guarantee.
   - **Tests.** Update `test_legacy_bracket_yield_keeps_explicit_tab_bindings`, which
     currently locks in `prev_agents_tab == "unbound"` while `prev_card_block` is `(`.
     Add a test for an explicit default-valued tab binding.
6. **Flag-off help rows.** `modals/help_modal/agents_bindings.py` shows the
   `Prev / next agent tab (beta)` and `Go to agent tab… (beta)` rows whenever keys are
   bound, even with the flag off. Gate them on `agent_tabs_enabled()` so the flag-off
   help is unchanged. Add a test for both flag states, and confirm that the flag-off
   help PNG goldens do not change.
7. **Retired sticky keys.**
   - **Bug.** With the flag on, `_prune_session_mounted_gone` returns scoped
     `(token, PanelKey)` tuples, although it is typed `set[PanelKey]`. They land in
     `panel_rebuild_keys` (`actions/agents/_display.py`), where they never match a
     widget.
   - **Fix.** Return plain `PanelKey`s for the active scope only, so keys retired on
     other tabs do not trigger refreshes.

**Done when:** `sase tool run check` passes, apart from the known pre-existing items
listed above, and the new tests cover both flag states.

## Phase 2 — cross-tab-jump-repairs

1. **Back-anchor saved after the switch.**
   - **Bug.** `_try_reveal_agent_row` (`actions/navigation/_member_jump.py`) calls
     `ensure_agent_tab_for_identity` before it captures `old_idx`, `old_agent`, and the
     anchor stacks, and before `_save_agents_jump_anchor()`. The anchor therefore
     records the destination tab's remembered row.
   - **Symptom.** `'` after any cross-tab Node Finder, notification, Procs, or relation
     jump lands on the wrong row and tab. `_arm_manual_unread_after_departure` also arms
     the wrong row.
   - **Fix.** Capture the source selection and stacks, and save the anchor with the
     source tab key, before switching. On failure, restore the stacks and the tab.
   - **Test.** A real cross-tab `_try_reveal_agent_row`, then a back-jump, returns to
     the original row and tab. The existing back/forward tests call
     `_ensure_agent_tab_for` directly and miss this.
2. **Notification pre-switch.**
   - **Bug.** `handle_jump_to_agent` (`actions/agents/_notification_handlers.py`) and
     `navigate_to_agent_tab` (`actions/agents/_notification_navigation.py`) switch tabs
     before `jump_to_loaded_agent`. That call goes through `_jump_to_node_identity` to
     `_try_reveal_agent_row`, which then sees no switch and cannot restore the tab when
     the reveal fails.
   - **Fix.** When the ladder (`_jump_to_node_identity`) exists, drop the pre-switch.
     The ladder already switches through item 1's path. Keep the helper only on the
     fallback scan path, and restore the tab when that scan fails.
   - **Tests.** Cross-tab `handle_jump_to_agent` through the real ladder, and a failed
     reveal restoring the tab. The current `navigate_to_agent_tab` test only exercises
     the fallback scan.
3. **Direct scanners without reveal or restore.** These callers switch tabs and then
   scan `_agents`:
   - the run-log modal `action_jump_to_agent_tab` (`modals/agent_run_log_modal.py`);
   - `_select_revived_agent` (`actions/agents/_revive_state.py`);
   - Files `_select_file_agent` (`actions/artifacts_files.py`);
   - link-trail restore (`actions/link_trail.py`).

   An off-tab target hidden under a fold or collapsed banner switches the tab, toasts
   "not found", and leaves the user on that tab. Route each caller through
   `_try_reveal_agent_row` (or the shared reveal with fold expansion), and restore the
   previous tab on failure.

4. **Failed back-jump leaves the tab switched.**
   - **Bug.** The `_pop_agents_jump_anchor` loop
     (`actions/navigation/_entry_jump_agents.py`) switches to each anchor's tab before
     validating it. When every anchor is stale, it returns without restoring the tab.
   - **Fix.** Restore the starting tab when no anchor validates.
   - **Also.** Treat anchors saved without a tab token (while the flag was off) as
     belonging to the current tab, instead of failing validation.
5. **`,j` off-tab path.**
   - **Bug.** `_jump_to_off_tab_timed_candidate`
     (`actions/agents/_unread_navigation.py`) builds candidates from
     `_agents_query_result`, which include rows hidden under a collapsed grouping banner
     on their own tab. Selecting one fails.
   - **Symptom.** The toast wrongly says "No unread completed agents". The saved anchor
     is never rolled back, so the next `,j` picks the same candidate again.
   - **Fix.** Reveal through the fold-expanding path, or apply the active-tab
     candidates' banner-hidden exclusion. Roll back the anchor and the tab on failure.
6. **Last-launch exception path.** `_reveal_last_launch_target`
   (`actions/agent_workflow/_kill_last_launch.py`) restores the previous tab when the
   reveal returns a failure, but not when it raises. Restore it in both cases.
7. **Node Finder chip.** `actions/agents/_node_finder_rows.py` adds the off-tab chip
   only when `reason_mask == 0`, so off-tab rows that are also folded or query-hidden
   get no chip. Show the chip on every off-tab row, and test the chip assignment and
   rendering.
8. **Flag-off lookups.** `_resolve_file_agent` (Files) and `_monitor_jump_agent` (Procs)
   now search `_agents_with_children` even with the flag off, which changes flag-off
   outcomes and toasts.
   - Keep the Files widening, which the original plan requires, now that item 3 routes
     it through the fold-expanding reveal. Confirm that the flag-off outcome is no worse
     than before.
   - Gate the Procs widening on `agent_tabs_enabled()` unless the flag-off result is
     provably identical.
9. **Tests.** For each entry point below, put the target on a non-active tab. With the
   flag on, assert that the tab switched and the row is selected. With the flag off,
   assert that behavior is unchanged. The entry points are:
   - `handle_jump_to_agent`;
   - link follow (`_reveal_loaded_agent`);
   - `_reveal_last_launch_target`;
   - the Procs monitor jump;
   - relation-jump activation;
   - the run-log modal;
   - Files open agent;
   - link-trail restore;
   - `_jump_to_node_identity` across tabs;
   - `,j` unread and its cache signature;
   - a back-jump after a real cross-tab jump.

**Done when:** `sase tool run check` passes, apart from the known pre-existing items,
and every entry point above has a flag-on and a flag-off test.

## Phase 3 — scope-wording-and-symbols

1. **Marked-agents line** (`actions/agents/_marking_kill.py`).
   - **Bug.** `marked_total` is `len(marked_agents)`, counted before clan expansion, but
     the off-tab count runs over the clan-expanded, local-only list.
   - **Symptoms.** A marked clan container on another tab yields "3 of 1 marked agents
     are on other tabs". Remote marked rows count toward M but never toward N.
   - **Fix.** Count N and M over the same set: the marks as the user placed them, where
     a clan container counts once and remote rows are included.
   - **Tests.** A marked off-tab clan container and marked remote rows.
2. **Custom cleanup header.** `_custom_cleanup_header`
   (`actions/agents/_kill_cleanup_selection.py`) reads "Custom selection across all
   tabs", but its candidates come from `_agents_in_focused_panel()` on the active tab.
   Use the regular bulk scope label: `on <tab>`, or `across all tabs` only at
   `ALL_AGENT_TABS`. Update its test.
3. **Docs.**
   - In `docs/ace.md`, the "Agent Tabs (beta)" note says remote roots land on a
     per-machine tab. That is only true in machine mode, so correct it. Move the note
     next to `### Machines`, as the original plan asked, if it reads well there.
   - Align the `agent_tabs` row in the `docs/configuration.md` flag table with the
     registry's flag wording.
4. **Symvision.** Read `sase memory read symvision.md` first; test references never
   count as consumers. Run the symvision gate (`just symvision`) and resolve every
   agent-tabs public symbol it flags. At landing review time these were:
   - `AgentTabCatalog` (`src/sase/core/agent_tab.py`);
   - `AgentTabStyle` and `agent_tabs_settings_for` (`agent_tabs_settings.py`);
   - `agent_tab_state_path` and `serialize_active_agent_tab`
     (`models/agent_tab_persistence.py`);
   - `catalog_view_for_owner` (`actions/agents/_agent_tabs.py`);
   - `clear_agent_tab_index_cache` (`models/agent_tab_index.py`);
   - `rescope_agents_to_active_tab`, `scoped_agents_for_owner`, `scoped_selection_key`,
     and `sticky_key_scope` (`actions/agents/_tab_scope.py`).

   Handle anything phases 1 and 2 added the same way. Follow the hierarchy:
   1. delete dead symbols;
   2. privatize in-file-only symbols (tests may import private names);
   3. add a non-test pragma only for a real consumer that symvision cannot see;
   4. add an `--epic-symbol '<bead>(<Symbol>)'` Justfile line only when a still-open
      `sase-1bc` phase bead will consume the symbol (for example `AgentTabStyle` for
      tab-strip styling), keyed to that open phase bead.

   Never key an entry to `sase-1bc.6.1` or to this plan's own beads. Confirm that no
   agent-tabs symbol remains in the symvision output. Other entries are tracked
   elsewhere.

**Done when:** `sase tool run check` passes, apart from the known pre-existing items;
the symvision output has no agent-tabs symbol; and the wording tests cover both flag
states.

## Out of scope

- Any feature from later `sase-1bc` phases: tab-strip, layout-ladder, machine-tabs,
  launch-view-ux, tab-moves, and finish.
- The pre-existing failures listed under Context.
- Closing `sase-1bc.6.1` itself. Its land agent resumes after this plan lands.
