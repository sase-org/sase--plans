---
tier: tale
size: medium
title: Finish landing the Agents Archive view (sase-1jm.3.1.8) and close sase-1jm.3.1
  and sase-1jm.3
goal: The beta-flagged Agents Archive never acts on the parked Inbox, parks and restores
  decks, cards, and scroll per pane, reports exact-name arrivals honestly, adds no
  module to the TUI app import closure, and epics sase-1jm.3.1.8 and sase-1jm.3.1
  plus phase sase-1jm.3 are verified and closed.
proposed_by: bbugyi200.athena.sase-1jm.3.1.8.land
bead: sase-1jm.3.1.8
status: done
---

- **PARENT:**
  [202610/agents_archive_view_landing_fixes.md](https://github.com/sase-org/sase--plans/blob/main/202610/agents_archive_view_landing_fixes.md)
- **BEAD:**
  [sase-1jm.3.1.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jm/sase-1jm.3.1.8.md)

# Finish landing the Agents Archive view

This tale is the remaining work found while landing epic **sase-1jm.3.1.8** (_Finish the
read-only Agents Archive view before sase-1jm.3.1 lands_). All five of its phases are
closed, but land verification against master `791c1797b9` found real defects and plan
gaps in the shipped code. Each one below was confirmed by reading the code; several were
also reproduced in mounted-app probes. This tale fixes them, then performs the full
closeout chain: close `sase-1jm.3.1.8`, resume and close plan bead `sase-1jm.3.1`, and
close phase `sase-1jm.3`. Nothing resumes this landing after you finish, so the closeout
is part of this tale and happens in the same turn as the final code.

Governing plans (read with `sase artifact read`):

- `plan:202610/agents_archive_view_landing_fixes.md` (epic `sase-1jm.3.1.8`)
- `plan:202610/agents_archive_view_phase.md` (epic `sase-1jm.3.1`)
- `plan:202610/agents_archive_view.md` (epic `sase-1jm`)

Their final decisions still apply: `view_name = archive`, `toggle_filter = restore`,
`jump_opens_archive = yes`, `glossary_archive_term = no`, and
`glossary_strand_updates = no`. **Edit no memory files.** Keep the `agents_archive_view`
beta flag (flag bead `sase-1ju`). Do not implement other parent phases' work: no
restore/fork/copy chooser and no `⏎` restore (`sase-1jm.5`), no arrival routing or
toasts (`sase-1jm.6`), no shelf/bridge/Node Finder (`sase-1jm.7`), and no Record deck
(`sase-1jm.4` / `sase-1jm.4.1`).

The follow-up triage for this landing is already recorded on `sase-1jm.3.1.8` (the
`LAND TRIAGE` note). Do not redo it.

## Rules

- Before changing code, use `/sase_memory_read` to read `lint_and_test.md` and `tui.md`
  (and `tui_perf.md` before touching the `j`/`k` or refresh paths). Follow
  `src/sase/ace/CLAUDE.md`: footer rules, help rules, and the 57-column help box. Update
  `src/sase/default_config.yml` if you change any keymap.
- Reproduce each defect in a failing test before you fix it.
- Use a throwaway `SASE_HOME` for every mounted-app probe or test. Never let an `AceApp`
  write into the real `~/.sase`.
- Do not run `just check-full`. Run long commands in the foreground with a generous
  timeout, not through a monitor: the closeout must happen in this same turn.
- No manual commits, branches, or PRs. Finish with `/sase_final`.
- If you find new work that this epic did not cause, use `/sase_new_task`, name
  `sase-1jm.3.1.8` as the proposing bead, and record the outcome in your close note.

All TUI paths are under `src/sase/ace/tui/` unless a full path is shown. Resolve
line-number drift with `rg`.

## Fixes

### 1. False `missing` arrival from a coalesced idle request

- **The bug:** `_apply_archive_result` (`actions/agents/_archive_controller.py`) calls
  `_publish_arrival` on `kind == "unchanged"` results. Those results never run the
  lookup: `_build_archive_result` (`actions/agents/_archive_controller_build.py`)
  returns `{"kind": "unchanged"}` for any idle request whose tokens match.
- **How it happens:**
  1. An idle request is in flight and a second tick sets `idle_coalesced`.
  2. An exact-name open arrives while the UI is busy, so it is queued.
  3. `_maybe_coalesce_idle` starts a new idle request. That request captures
     `pending_exact_name` and comes back unchanged.
- **Result:** the controller publishes `missing` and clears the pending name, even
  though the row exists.
- **Fix:**
  - Skip the unchanged shortcut when `snapshot.pending_exact_name` is set.
  - Never publish an arrival outcome from a result that did not run the lookup.
  - Have `_maybe_coalesce_idle` drain `explicit_open_queued` as a VIEW request before it
    schedules another idle rebuild.
- **Test:** in `tests/ace/tui/test_agent_archive_controller.py`, set up an idle request
  in flight, a coalesced tick, and a queued exact-name open. The outcome must be `found`
  and never `missing`.

### 2. Beta-off palette leak

`is_command_available` (`commands/availability.py`) gates `agents.open_archive` and
`agents.back_to_inbox` on `agents_archive_view_enabled()`. It does not gate the
leader-mode palette entry `leader.toggle_agents_archive` ("Toggle Agents Archive /
Inbox", from `commands/_mode_commands.py`). With the beta off, that entry still shows,
and choosing it does nothing.

- Gate the entry on the beta, using a use-site import like its neighbors.
- Test that the entry is hidden with the beta off and shown with it on.
- Add a mounted test that `,a` with the beta off changes nothing. The gate exists in
  `actions/agent_workflow/_leader_mode.py` but has no test.

### 3. Inbox actions that still act on the parked Inbox while the Archive is open

These paths read `_get_selected_agent()` or `_marked_agents`, so they target the parked
Inbox selection or the Inbox marks while the Archive owns the panel:

- **`,x` kill_and_edit** (`actions/agent_workflow/_leader_mode.py`): calls
  `_bulk_kill_marked_agents_and_edit` when Inbox marks exist, otherwise
  `_kill_and_edit_agent`. This is a live mutation: guard it.
- **`,r` revert_agent** (`_start_revert_selected_agent` in `actions/agents/_revert.py`,
  including its marked bulk path). This is a live mutation: guard it.
- **`,<space>` agent_from_cl on the Agents tab** (`_start_agent_from_agent_quick` in
  `actions/agent_workflow/_entry_quick_launch.py`): launches with the parked Inbox
  agent's project. Guard it with the standard toast. Launching from an archived run
  belongs to `sase-1jm.5`.
- **`v` view_files** (`_view_agent_files_impl` in `actions/hints/_files.py`): this is a
  read.
  - Route it through `resolve_agents_read_agent` / the typed Archive selection. With no
    hydrated Archive target, do nothing.
  - If hint-mode file viewing cannot serve a read-only archived agent cleanly, guard it
    instead and say why in the close note.

Use the existing boundary for every guard: `guard_archive_live_action`,
`ARCHIVE_LIVE_ONLY_ACTIONS`, and `_ARCHIVE_GUARD_KEY_LABELS` in
`actions/agents/_archive_actions.py`. Leader actions do not go through
`check_agents_availability`, so put each guard in the handler itself.

Hide the matching palette entries (`leader.kill_and_edit`, `leader.revert_agent`,
`leader.agent_from_cl`) while the Archive is active. `commands/availability.py`
currently strips only `app.` ids.

Then audit the rest:

- `rg` every remaining `_get_selected_agent(` and `_marked_agents` reader that an
  Agents-tab key, leader key, or palette entry can reach.
- Classify each one as guarded, routed, or already safe (and say why).
- Record the audit in the `sase-1jm.3.1.8` close note.

Add mounted-app real-key tests for every newly guarded or routed action.

### 4. Metadata pager regression in the Inbox

8270bf768f changed `_agent_metadata_pager_selection_is_current`
(`actions/agents/_metadata_pager.py`) to call `_find_agent_by_identity`. For the Inbox,
that only checks that the agent is still in `_agents`. The old code compared against
`_get_selected_agent()`, so a `V` followed quickly by `j` now opens the pager for the
row the cursor already left.

- **Fix:** for the Inbox path, require that the Archive is inactive and that the
  selected Inbox agent's identity equals the requested identity. Keep the Archive-key
  path and the lookup inside `refresh_document_fn` as they are.
- **Test:** press `V` in the Inbox, then move the cursor before the document finishes
  building. No pager may open.

### 5. Per-panel deck and scroll parking

All of these are in `actions/agents/_archive_view.py`.

1. **Decks swap between panes.**
   - `_panel_deck_choices_from_snapshot` keys choices by the snapshot's
     `enumerate(panels)` reading-order index. `_restore_per_panel_deck_choices` and
     `_capture_detail_scrolls` treat the keys as real pane ids.
   - When pane ids are not in reading order, the Inbox decks come back swapped. The
     repro: split `|`, focus pane 0, then split `\`. Inbox `{1: final, 2: files}`
     returns as `{1: files, 2: final}`.
   - Fix: key capture and restore consistently by the real pane id.
2. **Sticky cards are wiped.**
   - Capture keeps only one card, falling back to the Main card. Restore then calls
     `deck_area.set_preferred_cards(pane_id, {deck: card})`, which replaces the panel's
     whole per-deck `preferred_cards` map.
   - The repro: pane 0 on TOOLS with `{main: context, final: instance:commit}` returns
     as `{tools: context}`.
   - Fix: park each panel's current deck and its full `preferred_cards` map, and restore
     them without dropping the other decks' cards.
3. **Archive list scroll is not restored.**
   - `_restore_archive_list_scroll` runs on re-entry, before the list has rows and
     before `_sync_archive_chrome`. In the repro, a parked offset of 10 comes back as 1.
   - Fix: restore after the list renders its rows (for example with `call_after_refresh`
     after the first applied snapshot), clamped to the maximum scroll.
   - Check the deck-panel scroll restore for the same timing problem.
4. **Collapsed-panel focus.** `_restore_parked_inbox` sets
   `_expanded_panel_focus = True` even when the focused panel is collapsed. Use the same
   `not panel_is_collapsed(...)` rule as `actions/agents/_folding_panels.py`.
5. **Remove the fake hook.** Restore currently prefers an
   `owner._restore_parked_deck_choices` hook that `AceApp` does not implement. Only the
   unit tests in `tests/ace/tui/test_agent_archive_parking.py` provide it, so the real
   restore path is untested. Remove the hook and make those tests use the real path.
6. **Mounted-`AceApp` tests:**
   - Pane ids out of reading order, with different decks and several sticky cards per
     panel, round-trip through the Archive in both directions.
   - The Archive list scroll round-trips.
   - Splits changed **inside** the Archive survive the return to the Inbox. The current
     `test_mounted_deck_splits_survive_archive_round_trip` in
     `tests/ace/tui/test_agent_archive_remaining_proof.py` splits before entering the
     Archive and never changes splits there, so it also passes on the old
     `_apply_agents_deck_snapshot` behavior. Strengthen it until it fails on that
     behavior.

### 6. Persistence after quitting from the Archive

`test_quitting_from_archive_persists_grouping_only` fakes a quit by calling
`init_agents_archive_view_state` again. Add a mounted test that:

- uses a throwaway `SASE_HOME`;
- quits `AceApp` while the Archive is active;
- asserts that the Inbox last-query file holds the parked Inbox query, not the Archive
  query;
- asserts that the Archive grouping persisted.

### 7. Amber `in:` scope chip

- **Where it is missing:** only the shared highlighter
  (`src/sase/ace/query/highlighting.py` and
  `src/sase/ace/query/profile_highlighting.py`) emits `scope_key` / `scope_value`. The
  Agents FilterBar editor renders `in:archive` unstyled: `AgentsFilterBar` has no
  resting display, and its `VimTextArea` editor (`widgets/filter_bar.py`) has no query
  highlighting.
- **Editor fix:** render `in:` and its value as bold `#d7af5f` in the Agents FilterBar
  editor, matching the info row's Archive chip.
  - Use the existing TextArea highlight machinery. See `widgets/_bullet_highlight.py`,
    `widgets/_todo_highlight.py`, and `LineRenderingMixin`.
  - Scope it to the Agents bar.
- **Highlighter fix:** 273d6f0d9b made `in:` a scope token on every query profile.
  Restrict that classification to the agents profiles, where `in:` is a host-owned scope
  token (`src/sase/ace/query/scope_token.py`). Other profiles must flag `in:` as
  `unknown_key` again, as they did before.
- **Tests:** cover both fixes.

### 8. Import budget: fold the leaf mixin back

273d6f0d9b moved `AgentArchiveColumnMixin` and the cheap Inbox probes into a new leaf
module, `actions/agents/_archive_column_mixin.py`. That adds one module to the
`sase.ace.tui.app` closure, although the plan required moving them into an
already-loaded module.

- **Fold the module in:**
  - Move the contents into modules that are already in the closure, without creating
    import cycles. For example, move the probe functions into
    `actions/agents/_display_helpers.py` (a leaf that `_display_incremental` and
    `_display_detail_render` already import) and the mixin class into
    `actions/agents/_core.py`.
  - Update every importer: `_display_refresh.py`, `_display_detail_render.py`,
    `_display_incremental.py`, `_panel_navigation.py`, `_folding.py`,
    `_agent_enter_action.py`, `navigation/_basic.py`, `_query_persistence.py`,
    `_archive_view.py`, and `tests/ace/tui/test_agent_archive_column_lazy.py`.
  - Delete the leaf module.
- **Confirm:** `.venv/bin/python tools/tui_import_closure --diff 69f647d912^` must show
  that this epic adds zero modules: not `_archive_view`, not `agent_archive_view`, and
  no replacement leaf. Keep Inbox `j`/`k` at one attribute read before any Archive
  import. `tests/ace/tui/test_agent_archive_column_lazy.py` must still pass.
- **Do not raise `_MAX_MODULE_COUNT`.** The remaining excess is tracked by `sase-1ic`: 2
  pre-epic modules plus 3 from 86a589e076's renderer split. So `test_app_import_budget`
  stays red, at about 3517 against `< 3513`, as a cited known failure.

### 9. Guard toasts name the configured toggle key

`_archive_guard_message` (`actions/agents/_archive_actions.py`) hardcodes `,a`. Read the
key from the configured `toggle_agents_archive` leader binding, the way
`widgets/agent_info_panel.py` does with `leader_key_display`, and fall back to `,a`.
Test it with a remapped key.

### 10. Coverage gaps (tests only)

- `h` on an expanded container member repaints the Archive detail
  (`actions/agents/_archive_column_nav.py`).
- The Archive footer refreshes on a cursor move, and `unmark (N)` disappears after `u`.
- `V`, `a` (artifact files), and the patch jump open for a loaded archived run. Today
  only the banner no-op case is tested.
- The palette hides every live-only action while the Archive is active, not just
  `app.kill_agent`.

## Verification

1. Run `just fix`, then `sase tool run check` on the final tree. These known failures
   are not caused by this epic:
   - `tests/ace/tui/test_app_import_budget.py` (`sase-1ic`)
   - `test_project_shown_when_no_patch` and
     `test_workflow_snapshot_render_reads_state_once_and_uses_parsed_data` (`sase-1k4`)
   - load flakes that pass in isolation (`sase-120`, `sase-1g8`, `sase-1k0` to
     `sase-1k3`, `sase-19c`)

   Treat any other failure as yours.

2. Run the Archive PNG module
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_archive.py` in check mode (the
   `just test-visual` / `fix_tui_screenshots --check` path) in the foreground. If a
   golden legitimately changes:
   - recapture it with a targeted `just fix-tui-screenshots -- <selectors>`;
   - inspect the run report, every created file, and each update group;
   - view each changed PNG;
   - rerun check mode until it is clean.
3. Run `just symvision`.

## Closeout (final step, in this same turn)

Do these in order. Never use `--force` merely to make a close succeed, and never use
`--force` to advance a successful nested landing.

1. **Close epic `sase-1jm.3.1.8`.**
   - Run `sase bead epic-symbols sase-1jm.3.1.8`. Resolve every entry it lists: wire the
     symbol up, privatize it, add a non-test pragma, or delete it. Re-key only to a
     still-open bead that really consumes the symbol.
   - Close it with `sase bead close sase-1jm.3.1.8 --note "<verification>"`. The note
     must cover:
     - each phase verified against the code;
     - the fixes from this tale;
     - the item 3 audit;
     - the check run id and its known failures;
     - the import-closure numbers;
     - the visual result.
   - Run `just symvision`.
   - Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN
     path `sase bead read sase-1jm.3.1.8 -r "<why>"` shows, ending in
     `202610/agents_archive_view_landing_fixes.md`.
2. **Resume and close plan bead `sase-1jm.3.1`.** Its `parent_bead` is phase
   `sase-1jm.3`.
   - Review its `LAND TRIAGE` note (note #1). Do not redo that triage.
   - Review all of its descendants and their notes, and the linked plan
     `plan:202610/agents_archive_view_phase.md`. Re-verify its scope (the "Verification
     and child landing" section) against the integrated tree: the beta-on Archive is
     usable and read-only, per-view state round-trips, beta-off Inbox behavior is
     unchanged, and no Archive work appears before the first complete Inbox load.
   - Confirm that every phase and child epic is closed.
   - Run `sase bead epic-symbols sase-1jm.3.1`. At planning time it listed only
     `--epic-symbol 'sase-1jm.3.1(archive_read_only_context)'`. That symbol is used only
     by tests today, and its planned consumer is the Record deck's Lifecycle and
     Provenance cards. So:
     - re-key the `Justfile` line to
       `--epic-symbol 'sase-1jm.4.1.3(archive_read_only_context)'` (`sase-1jm.4.1.3` is
       open);
     - add a `Justfile` comment, above `_lint-symvision` (never inside the continued
       command), that names that consumer.
   - Resolve any other entry the same way.
   - Close it with `sase bead close sase-1jm.3.1 --note "<what you rechecked>"`.
   - Run `just symvision`.
   - Set `status: done` in the frontmatter of the plan file ending in
     `202610/agents_archive_view_phase.md` (its PLAN path from `sase bead read`).
3. **Close phase `sase-1jm.3`, and stop there.**
   - Verify that the child plan completed the phase's work: the beta-flagged `,a` toggle
     and per-view parking, the Inbox pulse plus the day-grouped Archive list, the
     Archive info row, the `in:archive` filter bar, the grouping picker, read-only deck
     hydration from the bundle, the detail-ownership guard, live-only key toasts, the
     tab-label marker, the footer, and the help box.
   - Run `sase bead epic-symbols sase-1jm.3` and resolve any entry. At planning time,
     `archive_marked_run_keys` and `archive_read_target` already had real non-test
     callers, so they need no re-key.
   - Close it with
     `sase bead close sase-1jm.3 --note "<implemented behavior, verified checks, timings, and inspected PNG evidence>"`.
     For timings, use phase `sase-1jm.3.1.8.5`'s evidence: warm `,a` median 99.6 ms
     against the 100 ms budget, and Archive `j`/`k` p50 about 42 ms and p95 about 47-53
     ms against a same-host Inbox control of p50 17-56 ms and p95 45-65 ms on a loaded
     host.
   - Leave the glossary follow-ups (notes #1-#3 on `sase-1jm.3`) for `sase-1jm`'s land
     agent.
   - **Never close `sase-1jm` or any other ancestor.** If any parent turns out to be
     incomplete or ambiguous, stop there. Record a note on that parent describing the
     blocker, and report it in your final response.
