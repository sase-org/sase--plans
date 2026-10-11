---
tier: epic
title: Finish the read-only Agents Archive view before sase-1jm.3.1 lands
goal:
  The beta-flagged Agents Archive is correct and complete. Detail ownership never
  strands the Inbox or the Archive, live-only keys explain themselves, read callers
  never fall back to the parked Inbox, every promised parking and parent seam exists,
  project banners show names, the TUI import closure is back to its pre-epic size, and
  the Archive goldens are stable and current.
phases:
  - id: detail-ownership
    title: Detail ownership, stuck placeholders, and reader guards
    size: medium
    depends_on: []
    description:
      "detail-ownership: record a fresh owner token at every debounced Inbox detail
      schedule so Inbox detail repaints after an Archive round trip. Keep hidden Inbox
      paths from cancelling or replacing the Archive's pending hydration on the shared
      debouncer. Repaint Archive detail when the controller moves the selection. Clear
      the stuck titled loading card when the cursor leaves a run, and drop hydration
      while the pulse is focused. Stop FINAL and tool-run live ticks for archived
      agents, and label partial replies as partial."
  - id: controller-seams
    title: Controller robustness, parent seams, and project display names
    size: medium
    depends_on:
      - detail-ownership
    description:
      "controller-seams: stop off-thread probe errors from wedging the Archive
      controller. Drain a queued explicit open on idle, and coalesce overlapping idle
      rebuilds. Publish a typed found/owner/missing arrival outcome and clear unmatched
      exact names. Add the restored-row overlay seam with invalidation, and keep list
      option IDs unique. Show PROJECT_NAME on project banners through the project
      display snapshot."
  - id: action-boundary-fixes
    title: Live-key toasts, read isolation, marks, footer, help, and info row
    size: medium
    depends_on:
      - controller-seams
    description:
      "action-boundary-fixes: make live-only keys reach their Archive guards so they
      toast instead of being silently swallowed. Stop copy readers from falling back to
      the parked Inbox agent, and guard or route the remaining unaudited Inbox actions.
      Render Archive marks and refresh the footer after mark and cursor changes. Remove
      the nonfunctional Enter read hint, fix the overflowing help row, and gate help and
      leader chrome on the beta. Fix the info row's color, key label, and resize
      alignment (the cause of the flaky missing-index golden), and the exception chip
      drop order."
  - id: parking-and-import
    title: Complete per-view parking, scope chip, and import deferral
    size: medium
    depends_on:
      - action-boundary-fixes
    description:
      "parking-and-import: restore deck card choices without reverting the shared deck
      splits, and park the Archive's per-panel deck, card, and scroll. Close any open
      FilterBar session before a view change, render the amber in: scope chip, and fix
      early-toggle query parking. Add the missing round-trip, tab-marker, and
      quit-persistence tests. Defer the two Archive modules out of the TUI app import
      closure."
  - id: remaining-proof
    title: Regression proof, golden recapture, and performance evidence
    size: medium
    depends_on:
      - parking-and-import
    description:
      "remaining-proof: add mounted-app regressions for every defect fixed by this epic
      in both flag states. Recapture the affected Archive PNG goldens through sase
      monitor, inspect each one, and prove the missing-index golden is stable.
      Re-measure Archive j/k and warm open against a same-host control. Run sase tool
      run check, and record the evidence for this child epic's land agent."
proposed_by: bbugyi200.athena.sase-1jm.3.1.land
parent_bead: sase-1jm.3.1
create_time: 2026-10-11 19:18:48
status: wip
---

- **PROMPT:**
  [prompts/202610/agents_archive_view_landing_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/agents_archive_view_landing_fixes.md)
- **PARENT:**
  [202610/agents_archive_view_phase.md](https://github.com/sase-org/sase--plans/blob/main/202610/agents_archive_view_phase.md)

# Finish the read-only Agents Archive view

This child epic holds the remaining work found while landing epic **sase-1jm.3.1**
(_Implement the read-only Agents Archive view for sase-1jm.3_). All seven of that epic's
phases are closed. But land verification found real defects and plan-scope gaps in the
shipped code, and an interaction with work that landed concurrently. That epic stays
open until this child epic lands. This child epic's land agent then resumes the
interrupted landing, as described under [Child landing](#child-landing).

The governing designs are `plan:202610/agents_archive_view_phase.md` (the sase-1jm.3.1
child plan) and `plan:202610/agents_archive_view.md` (parent epic sase-1jm). Read them
with `sase artifact read`. Their final decisions still apply:

- `view_name = archive`
- `toggle_filter = restore`: `,a` is a pure toggle and records no history.
- `jump_opens_archive = yes`
- `glossary_archive_term = no` and `glossary_strand_updates = no`: **edit no memory
  files.**

Keep the `agents_archive_view` beta flag (bead `sase-1ju`). The parent's retire-pane
phase removes it.

Do not implement other parent phases' work:

- the restore/fork/copy chooser and the `⏎` restore (`sase-1jm.5`)
- arrival routing and toasts (`sase-1jm.6`)
- the shelf, the bridge, and Node Finder (`sase-1jm.7`)
- the Record deck (`sase-1jm.4` / `sase-1jm.4.1`)

Provide only the seams named below.

All TUI paths are under `src/sase/ace/tui/` unless a full path is shown. Line numbers
were measured at master `d289b664a6`; resolve drift with `rg`. Each defect below was
confirmed by reading the code, and most also by a throwaway reproduction during land
verification. Every phase must still reproduce each defect in a failing test before
fixing it.

## Phase details

### detail-ownership

1. **Inbox detail stops repainting after an Archive visit.**
   - Several sites schedule `_fire_debounced_detail_update` without
     `remember_agents_detail_owner`:
     - `_refresh_agents_display_debounced` (`actions/agents/_display_refresh.py` ≈205)
     - the deferred branch of the full refresh (≈155-185)
     - the prompt-active branch of `_refresh_tribe_summary_only`
       (`actions/agents/_display_detail_render.py`)
     - the defer path in `actions/agents/_display_incremental.py` (≈375-391)
   - `invalidate_agents_detail_ownership` deliberately leaves the old token in
     `_agents_detail_scheduled_owner`. So after one open/close, every Inbox debounced
     update is dropped as stale until something calls `_refresh_agent_focus_detail`.
     Decks and the footer go stale on Inbox `j`/`k`, finalize, and enrichment.
   - Fix: route every schedule of `_fire_debounced_detail_update` through one helper
     that records a fresh token. Find every site with `rg`.
2. **Hidden Inbox paths clobber the Archive's pending hydration.** The Archive's
   hydration callback shares `_agent_detail_debouncer` with the Inbox.
   - The `_display_incremental.py` defer branch has no Archive guard. Finalize always
     defers, so any roster change within 150 ms of an Archive `j`/`k` replaces the
     callback and leaves the detail on loading. The same branch can also run
     `_sync_agents_onboarding` over the Archive panel.
   - `_refresh_agent_focus_detail` (`_display_detail_render.py` ≈184-210) cancels the
     debouncer after an immediate paint and replaces the callback when
     `render_immediate=False`. It is reached from `_fleet_projection.py`,
     `_startup_loads_tags.py`, `_startup_prompt_catalog_semantics.py`, and
     `watch_theme`.
   - Fix: gate every one of these paths on `agents_inbox_detail_allowed`.
3. **Controller-moved selections never repaint.**
   - `_install_model` (`actions/agents/_archive_column.py` ≈392-402) compares
     `archive.selected_key` before and after normalization. But `_apply_selection`
     (`actions/agents/_archive_controller.py` ≈496-525) has already moved it for:
     - exact-name reveals
     - drift fallbacks
     - committed-query changes
   - `h` on an expanded container member (`_archive_column_nav.py` ≈205-207) has the
     same problem.
   - Fix: track the key the detail was last scheduled for and schedule whenever the
     normalized key differs from it.
4. **Stuck "Loading archived run…" card** (sase-1jm.3.1.7 follow-up #1).
   - How it gets stuck:
     1. `paint_archive_detail_placeholder` (`actions/agents/_archive_reader.py`
        ≈150-172) shows the titled card through `AgentDetail.show_tab_empty_state`
        (`widgets/_agent_detail_state.py` ≈64-88). That pushes the card straight into
        the Main panels.
     2. Moving to a banner, tail, or the pulse calls `show_empty()` → `_deck_show_empty`
        (`widgets/_agent_detail_deck_refresh.py` ≈303-308). That writes the same "No
        agent selected" text into the prompt source.
     3. `_apply_section_content` (`widgets/prompt_panel/_section_view.py` ≈72-80) sees
        an unchanged digest, so the Main sink never fires.
   - Fix: in `_deck_show_empty`, after `source.show_empty()`, push an empty Main
     document unconditionally (for example `self._on_main_document("", False, None)`).
     This also fixes the same latent Inbox case.
5. **Hydration after the pulse takes focus.** `K` moves focus to the pulse while
   `selected_key` is unchanged, so a pending hydration still paints. The reader's
   post-await acceptance check must also require the list section to be focused.
6. **Live ticks run for archived agents.** The FINAL deck's 1 Hz tick
   (`widgets/decks/final/view.py` ≈316-333) and the tool-runs tick
   (`widgets/decks/tool_runs/widget_live.py` ≈100-117) must never start or keep polling
   for an agent with `_archive_read_only`.
7. **Partial replies.** The archive-reader plan requires partial replies to be shown as
   partial. Today `widgets/prompt_panel/_agent_display_render.py` (≈587-593) renders a
   retained partial reply with no marker. Mark it partial in the Reply card (title or
   header chip), using the facts the reader already probes.
8. **Epic symbols.** `agents_view_is_archive`, `capture_agents_detail_owner`, and
   `invalidate_agents_detail_ownership` have only in-file callers. After the fixes
   above:
   - privatize them, or give them a real non-test consumer;
   - update the tests that import them;
   - delete their `--epic-symbol 'sase-1jm.3.1(...)'` lines from the `Justfile`.

### controller-seams

1. **Off-thread errors wedge the controller.**
   - `_probe_tokens`, `_resolved_profile`, and `_load_facets`
     (`actions/agents/_archive_controller_build.py` ≈68, 83, 148, 201-203) can raise.
     When they do, `_apply_archive_result` never runs and `refreshing` stays true.
   - The idle tick then returns early forever (`_archive_controller.py` ≈239), and the
     column shows "Loading Archive…" instead of the honest unavailable state.
   - Fix: catch errors in the worker, publish the honest status, and always clear the
     current generation's in-flight state.
2. **Queued explicit opens never drain.**
   - `request_archive_view_load` queues an explicit open when the UI is not idle
     (≈267-269). Only a complete Inbox apply drains that queue (≈196-199), and the
     column has already recorded the request signature (`_archive_column.py` ≈201). So
     `,a` within 250 ms of navigation sits on loading until the next complete roster
     apply.
   - Fix: drain the queued open on the idle tick.
   - Also reset `inbox_load_scheduled` when the gate's Inbox load settles or fails.
3. **Overlapping idle rebuilds.** Warm idle requests (≈379-382) never mark themselves in
   flight. Under sustained drift with a compile slower than the tick, every tick
   supersedes the last, nothing applies, and worker threads pile up. Allow at most one
   in-flight idle rebuild and coalesce the rest.
4. **Typed arrival outcome** (a promised seam for `sase-1jm.6`).
   - Today `open_agents_archive_view_with_query` always returns `opened` synchronously.
     A lookup miss leaves `pending_exact_name` set forever.
   - `_lookup_pending` reads that live state from the worker thread. Capture the name in
     the immutable request snapshot instead.
   - When the lookup resolves, publish a typed outcome:
     - `found`: the row's name equals the requested name.
     - `owner`: `AgentArchiveCorpus.lookup` returned the owner of a workflow child.
     - `missing`: no row.
   - Store the outcome on `ArchiveViewState` and deliver it through an optional
     callback, so the routing phase owns its toasts and trail hop.
   - Clear the pending name on a miss.
   - `request_archive_lookup` has no caller: fold it into this API or delete it, and
     drop its `Justfile` epic-symbol line.
5. **Restored-row overlay seam** (for `sase-1jm.5`).
   - Add a presentation overlay on `ArchiveViewState`, keyed by stable archive key. The
     restore phase writes it.
   - Rows render `⌂ restored` from it.
   - Clear it on a committed query change and on Archive exit. It never touches the
     corpus or counts.
   - Whitelist any writer API that has no caller yet with
     `--epic-symbol 'sase-1jm.5(<name>)'`, and add a `Justfile` comment naming the
     consumer.
6. **Duplicate option IDs.**
   - `archive_row_key`'s name fallback can give two legacy rows (no archive key) the
     same option ID. `refresh_column` swallows the resulting `DuplicateID`, which
     freezes the column.
   - Fix: make the IDs unique, and test it with legacy rows.
7. **Project display names.** This integrates df61663204 and the accepted decision
   `decisions:project-labels-never-keys`.
   - `_plain_group_label` (`models/agent_archive_list.py` ≈185-186) paints the raw
     directory key on `project` banners.
   - Use the project display snapshot helper from df61663204
     (`project_display_name_for`; confirm with `rg`). Keep banner keys on the raw key.
   - Warm the snapshot off-thread in `build_archive_result`. If you do, add that module
     to the allowlists in `tests/test_project_display_presentation_audit.py`.
   - Add a test where the directory key and `PROJECT_NAME` differ.

### action-boundary-fixes

1. **Live-only keys are swallowed silently.**
   - `check_agents_availability` (`_app_action_availability_agents.py` ≈191-207) returns
     False for `ARCHIVE_LIVE_ONLY_ACTIONS`. Textual's `App.run_action` never dispatches
     an action whose `check_action` is falsy, so `guard_archive_live_action` never
     toasts. This is exactly the failure the action-boundary plan warned about.
   - Fix: let key dispatch reach the guarded methods while the Archive owns the panel.
     Keep these actions hidden in the command palette through its own availability path
     (`commands/availability.py`), and verify they stay hidden.
   - Test real key presses in a mounted app for every live-only family:
     `x X s S w W A R n N U [ ]` and the tab picker.
2. **Copy readers fall back to the parked Inbox agent.** `resolve_agents_read_agent`
   (`actions/agents/_archive_actions.py` ≈197-211) falls back to `_get_selected_agent()`
   when there is no hydrated Archive target. That happens on a banner, tail, or pulse,
   during loading, and for an unreadable bundle. While the Archive is active, never fall
   back; with no target, do nothing.
3. **Unaudited Inbox actions.** These still act on the parked Inbox selection:
   - `edit_hooks` → fork (`hints/_hooks.py` ≈20-22, `actions/agents/_fork_actions.py`
     ≈114, 125)
   - `view_agent_metadata` (`_metadata_pager.py` ≈73)
   - `open_tmux`
   - `jump_to_agent_patch`
   - `open_artifact_files`
   - `edit_spec`

   Classify each one. Guard live mutations through `guard_archive_live_action` and
   `ARCHIVE_LIVE_ONLY_ACTIONS`. Route reads (for example metadata) through the typed
   selection. Record the extended audit in your phase note.

4. **Help row overflow.** In `modals/help_modal/agents_archive_sections.py`, the key
   label `x s w R (Inbox only)` is wider than the key column, so its row renders 61
   columns wide. Fix it, and add a test that every Archive help row fits the 57-column
   box with descriptions of at most 32 characters.
5. **Nonfunctional Enter read hint.**
   - Enter on an Archive run row does nothing (`_archive_column_nav.py` `_activate`
     ≈214-229), yet the footer (`widgets/_keybinding_bindings_agents.py`) and help
     advertise `⏎ read`.
   - Remove the hint from both. Enter on run rows stays reserved for the parent restore
     chooser, so do not add an Enter no-op that would shadow it.
6. **Marks are never drawn.** Nothing under `widgets/agents_archive/` renders
   `ArchiveViewState.marks`. Draw marked runs and containers with the Inbox mark style,
   and refresh row rendering on `m`/`u`.
7. **Footer staleness.** Sync the Archive footer after `m`/`u` and after Archive cursor
   moves, so `unmark (N)` never lags.
8. **Beta-off leakage.** These appear with the beta off:
   - the Archive help box (`modals/help_modal/agents_bindings.py` ≈34)
   - the `,a` leader-mode entry and its label

   Gate them on `agents_archive_view_enabled()` and test with the beta off.

9. **Info row.** In `widgets/agent_info_panel.py` (≈465-500):
   - Use amber `#d7af5f` for the Archive chip instead of `#FFAF87`.
   - Read the toggle key from the configured `toggle_agents_archive` binding instead of
     hardcoding `a`.
   - Re-render the Archive row on resize. Its right-aligned `,a Inbox` padding comes
     from `self.size.width` at build time and falls back to two spaces before layout
     settles. That is the root cause of the flaky
     `test_archive_missing_index_png_snapshot` golden (sase-1jm.3.1.7 follow-up #4).
10. **Exception chip.** The plan drops MODEL and then RUNTIME as width shrinks, but
    `widgets/agents_archive/_rows.py` (≈170) drops the exception chip first. Keep the
    exception chip.
11. **Grouping picker.** Choosing the current grouping again in the `o` picker currently
    resets folds and selection. Make it a no-op.

### parking-and-import

1. **Deck splits revert on return.** `_restore_parked_inbox` calls
   `_apply_agents_deck_snapshot` (`actions/agents/_deck_persistence.py` ≈99-113), which
   re-applies layout, ratio, and panels. Splits are shared between views: restore only
   each panel's deck and card.
2. **Archive per-panel parking.**
   - `ArchiveViewState.deck_choices` and `.scroll` are never written or read.
   - Capture each panel's deck, card, and scroll when leaving the Archive, and restore
     them on re-entry.
   - Either restore the Inbox `focused_panel` and `panel_layout` that `ParkedInboxState`
     already captures, or delete those unused fields.
3. **Open editor across a view change.** `,a`, the palette entries, and the pulse return
   can switch views while a FilterBar session or preview is open
   (`_close_agents_filter_session` runs only on submit or dismiss). Close or cancel the
   session first, and make sure a late preview task can never install the wrong query.
4. **Amber `in:` scope chip** (query-parking step 4). Render the `in:` token in the
   FilterBar input as the amber chip, consistent with the info row. Verify first whether
   any existing highlighting already does this.
5. **Early toggle.** `,a` pressed before the async Inbox last-query restore finishes
   parks an empty query. Returning then overwrites the restored query for the session.
   Park after the restore, or reconcile.
6. **Stale preview facade.** Committing a saved slot outside an editor session copies a
   stale preview facade into the committed facade
   (`actions/agents/_filter_bar_session.py` ≈222). Stop copying it.
7. **Missing tests from the query-parking plan:**
   - tab-marker hit ranges and cached cell width (`widgets/tab_bar.py`)
   - query, selection, fold, tab, deck, and scroll round trips
   - persistence after quitting from the Archive
8. **Import budget.** `tools/tui_import_closure --diff 69f647d912^` shows this epic
   added `sase.ace.tui.actions.agents._archive_view` and
   `sase.ace.tui.models.agent_archive_view` to the `sase.ace.tui.app` closure (3514 →
   3516, against a `<3513` cap).
   - Defer both, using use-site imports behind a cheap attribute check on the Inbox
     `j`/`k` path.
   - `AgentArchiveColumnMixin` is imported by `actions/agents/_core.py`. Move its
     handlers into an already-loaded module, with use-site imports.
   - Other module-level edges to remove: `_panel_navigation.py`,
     `_display_incremental.py`, `_agent_enter_action.py`, `navigation/_basic.py`,
     `widgets/prompt_panel/_agent_display_render.py`, and the detail-render modules.
   - Confirm with the diff tool that neither module remains in the closure.
   - Do not raise the cap. Task `sase-1ic` tracks the remaining pre-epic excess.
   - Keep `tests/ace/tui/test_agent_archive_column_lazy.py` passing.

### remaining-proof

1. **Regressions.** Add mounted-`AceApp` regressions, for both flag states, for every
   defect the earlier phases fixed that does not already have a mounted test. At
   minimum:
   - an Inbox repaint after an Archive round trip
   - hidden Inbox refresh during Archive hydration
   - a live-only key press that toasts
   - copy on a banner
   - the stuck loading card
   - marks rendering
   - deck splits surviving a round trip
2. **Goldens.**
   - Run targeted `just fix-tui-screenshots -- <archive selectors>` through
     `/sase_monitor`. Expect changes to the help, footer, info-row, list, and marked
     goldens.
   - Add a marked-row golden and a restored-overlay row golden.
   - Inspect the retained manifest, every created file, and each update group
     (representative plus members). Resolve `partial` skips.
   - Prove the missing-index golden is stable: three consecutive check-mode runs of the
     Archive visual module all pass.
3. **Performance.** Re-run `bench_tui_jk_archive` and the existing axe `bench_tui_jk`
   back to back on the same host, as a control. Record Archive `j`/`k` p50 and p95
   against the control's, and the warm `,a` first-row median against the 100 ms budget
   (sase-1jm.3.1.7 follow-up #2).
4. **Record evidence.** Run `sase tool run check` on the final tree. Record the evidence
   on this phase bead for the child land agent.

## Rules for every phase

- Before finishing, read `lint_and_test.md` with `/sase_memory_read`. Run `just fix` and
  `sase tool run check`; do not run `just check-full`. Use `/sase_monitor` for genuinely
  long commands, and `/sase_final` for completion. Make no manual commits, branches, or
  PRs.
- Before changing TUI code, read `tui.md` with `/sase_memory_read`. Follow
  `src/sase/ace/CLAUDE.md`'s footer, help, and 57-column rules. Update
  `src/sase/default_config.yml` for any keymap change.
- Phase workers never create beads. Record unrelated discoveries as
  `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own phase bead.
- **Epic symbols.** Key a new temporary public seam only to a still-open consuming bead
  (this child epic, one of its phases, or the named parent consumer). Before closing
  your phase, run `sase bead epic-symbols <your-phase-id>` and resolve every entry.
- The known red `just check` items not caused by this epic are tracked elsewhere:
  - the import-budget pre-epic excess: `sase-1ic`
  - the marker-audit `_scoped_project_rows` site: note on `sase-1jy`
  - the stale `sase-1jx.6(classify_project_candidates)` Symvision entry: note on
    `sase-1jx`
  - the load flakes: `sase-1k0`…`sase-1k3`, `sase-120`, `sase-19c`

  Cite these; do not fix them here.

## Child landing

This child epic's land agent:

1. Verifies every phase above against the code. Runs `sase tool run check` on the
   integrated tree. Resolves this child epic's own epic-symbol entries, then closes the
   child epic.
2. Resumes the interrupted landing of plan bead **sase-1jm.3.1**:
   - Re-verify its scope against the integrated tree. Its follow-up triage is already
     recorded in its land-triage note; do not redo it.
   - Resolve every `sase bead epic-symbols sase-1jm.3.1` entry. Recommended outcomes:
     - Leftover `agents_view_is_archive`, `capture_agents_detail_owner`,
       `invalidate_agents_detail_ownership`, `request_archive_lookup`: privatize or
       delete them.
     - `open_agents_archive_view`: privatize it if its only callers are still the
       in-file actions. Otherwise re-key it to `sase-1jm.6`, if the arrival routes (for
       example `!R` "Browse the Archive") need the typed pure open.
     - `archive_read_only_context`: re-key it to the still-open Record consumer
       (`sase-1jm.4.1.3`, Lifecycle and Provenance cards for both views, or the open
       `sase-1jm.4.1` epic), with a `Justfile` comment.
   - Close it with `sase bead close sase-1jm.3.1 --note "<verification>"`.
   - Run `just symvision`.
   - Set `status: done` in `202610/agents_archive_view_phase.md`.
3. Handles that plan's parent phase **sase-1jm.3** as its landing section requires:
   - Run `sase bead epic-symbols sase-1jm.3`. Re-key `archive_marked_run_keys` and
     `archive_read_target` to `sase-1jm.5` (Restore, fork, and copy), their real
     consumer, and update the `Justfile` comment.
   - Close `sase-1jm.3` with a note covering the implemented behavior, checks, timings,
     and inspected PNG evidence.
   - Never close `sase-1jm` or any other ancestor.

Never use `--force` merely to make a close succeed.
