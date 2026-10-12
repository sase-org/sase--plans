---
tier: epic
title: Finish the Agents Archive epic before sase-1jm lands
goal: 'Every promise of plan:202610/agents_archive_view.md holds on master with no
  flag residue. The retired Artifacts strip, the saved-query migration, the Archive
  chooser, restore, link arrivals, the shelf and bridge, and light-row Record behave
  as the plan says, with tests, goldens, and benches as proof, so the sase-1jm land
  agent can close the epic.

  '
phases:
- id: flag-residue
  title: Flag residue, dead code, symvision, and lazy startup
  depends_on: []
  size: medium
  description: 'flag-residue: strip every override_flags(agents_archive_view=...)
    and flag-era duplicate test, delete stale beta comments, vestigial parameters,
    and the dead custom-search revival path, resolve all six epic-caused symvision
    symbols plus the Justfile epic-symbol row, and stop reading the Archive grouping
    file in AceApp.__init__.'
- id: retire-pane-finish
  title: Retired Artifacts strip, migration correctness, docs, and goldens
  depends_on:
  - flag-residue
  size: medium
  description: 'retire-pane-finish: make the test and visual startup fixtures follow
    the production artifacts_agent_pane_retired state, fix the Help Guide digits,
    pane show -j, global show_artifacts_agents redirect, and the toast marker. Validate
    migrated saved queries against the archive profile, sweep the stale docs, and
    capture the renumbered-strip and toast goldens.'
- id: archive-chooser-copy
  title: Archive chooser sections, container restore, and copy keys
  depends_on:
  - flag-residue
  size: medium
  description: 'archive-chooser-copy: add the READ section, open the chooser on container
    rows with Restore clan, keep marked restore reachable, gate Fork on a retained
    chat, hydrate off the event loop, add the dedicated agents_copy_reference y action,
    fix the doubled %l prefix, make %j copy the bundle JSON, update help and docs,
    and capture chooser goldens.'
- id: arrivals-finish
  title: Lookup-first link arrivals and selection that survives clearing
  depends_on:
  - flag-residue
  size: medium
  description: 'arrivals-finish: look up before opening the Archive so a miss keeps
    context, reveal filter-hidden agents across agent tabs and folds, keep the Archive
    selection when name: is cleared, point target_for_ref_kind agent at the Agents
    tab, test ,a from another tab, and fix the link-follow docs.'
- id: archive-restore-run
  title: Restore execution, single toast, and restore-and-show reveal
  depends_on:
  - archive-chooser-copy
  - arrivals-finish
  size: medium
  description: 'archive-restore-run: emit one toast with failure reasons, keep the
    parked Inbox untouched by a plain restore, dim restored rows, bump the pulse at
    once, make Restore and show wait for the revive refresh and apply the filtered-
    target rule, move submit-time re-checks off the loop, and prove it through the
    real revive path plus a restored-row golden.'
- id: bridges-finish
  title: Shelf, zero-result bridge, and Node Finder correctness
  depends_on:
  - flag-residue
  size: medium
  description: 'bridges-finish: use the restore-decision shelf hint and consistent
    plurals, stop recount flicker, keep the lit state and focus flag current, match
    in:inbox queries, show the bridge only for true zero results with time columns,
    keep the Node Finder open on a cold Archive, and add the plain-shelf golden and
    the missing bridge and navigation tests.'
- id: record-light-rows
  title: Light rows carry bundle path and times, and labels show names
  depends_on:
  - flag-residue
  size: medium
  description: 'record-light-rows: carry bundle_path, stop_time, and dismissed_at
    on the sase-core archive light row, move the core revision pin, let bundle resolution
    and light-row Record use them, and show project display names instead of directory
    keys on light-row surfaces.'
- id: land-proof
  title: Help goldens, visual sweep, benches, and acceptance evidence
  depends_on:
  - retire-pane-finish
  - archive-restore-run
  - bridges-finish
  - record-light-rows
  size: medium
  description: 'land-proof: fix the Archive help PNG scroll stop, regenerate help-modal
    goldens once, run the targeted Archive, Record, Artifacts, saved-group, and help
    visual checks, measure the Archive and Inbox j/k and warm ,a benches back to back,
    walk every epic acceptance gate, and record evidence for the sase-1jm land agent.'
proposed_by: bbugyi200.athena.sase-1jm.land
parent_bead: sase-1jm
create_time: 2026-10-12 14:50:47
status: wip
bead_id: sase-1jm.9
---

- **PROMPT:** [prompts/202610/agents_archive_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/agents_archive_completion.md)
- **PARENT:** [202610/agents_archive_view.md](https://github.com/sase-org/sase--plans/blob/main/202610/agents_archive_view.md)
- **BEAD:** [sase-1jm.9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jm/sase-1jm.9.md)

# Plan: Finish the Agents Archive epic before sase-1jm lands

## Background

Epic `sase-1jm` (plan `plan:202610/agents_archive_view.md`) has closed all eight phases.
Phase `.8` (commit `101c853939`) removed the `agents_archive_view` beta flag and hid
Artifacts ▸ Agent behind the `artifacts_agent_pane_retired` sunset flag (bead
`sase-1kb`). The sase-1jm land agent checked the source against the plan and found that
phases `.5`–`.8`, none of which had a land agent of its own, left real gaps. Several
`DISCOVERED ISSUE` notes on `sase-1jm` are also still open. This epic finishes that
work. It does not close `sase-1jm`; that epic's land agent resumes the landing after
this epic lands.

The parent epic's final decisions bind every phase here. Do not re-ask them:

- `view_name = archive`: user-facing strings say Archive and the token is `in:archive`.
- `toggle_filter = restore`: `,a` is a pure toggle. Only the shelf's and bridge's `⏎`
  carry the Inbox filter. Do not implement the plan's
  `[!decision] toggle_filter = carry` blocks.
- `jump_opens_archive = yes`: a dismissed Jump-panel target opens read-only in the
  Archive.
- No memory note may be edited. The glossary follow-up is task `sase-1kc`.

Shared rules for every phase:

- Follow `tui.md` and the perf rules in the parent plan's Architecture section: no sync
  I/O on handlers or the pump, re-capture the selection after every await, and never
  paint-path stats.
- New and changed keys go in `src/sase/default_config.yml`, the keymap registries, the
  `?` help modal, and the footer metadata, following `src/sase/ace/CLAUDE.md`.
- Do not regenerate help-modal goldens (`test_ace_png_snapshots_help_panel.py`, or the
  Archive help capture in `test_ace_png_snapshots_agents_archive.py`) outside
  `land-proof`. Several phases change help text, and `land-proof` recaptures those
  goldens once. Capture other goldens with targeted
  `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`, and inspect the
  report before finalizing.
- Run `sase tool run check` before finishing. When reporting known failures, cite the
  existing owners: `sase-1ic` for the import budget, `sase-1k4` for the Project:
  fake/tmp header tests, `sase-1jx` for the New Project symvision symbols, and
  `sase-1kd` for the plan decision-card symvision symbols.

## Flag residue, dead code, symvision, and lazy startup

Clear the dead branches and leftovers of the beta flag first, so later phases edit clean
files.

- **Tests.** `grep -rln "agents_archive_view=" tests` lists 33 files with about 206
  `override_flags(agents_archive_view=True)` calls. The resolver ignores the unknown key
  (`feature_flags/resolver.py`), so these calls are dead. Remove them, along with any
  `override_flags()` context left empty. Merge the tests that now duplicate each other:
  - `test_dismissed_neighbor_digit_opens_archive_instead_of_reviving` and its
    `…_when_enabled` twin;
  - `test_files_open_agent_routes_dismissed_match_to_archive` and its `…_when_enabled`
    twin.

  Rename `test_custom_search_row_becomes_archive_browse_when_flagged`. Delete any test
  that only asserted the deleted flag-off behavior.

- **Stale comments and docstrings.**
  - `src/sase/ace/tui/styles.tcss` near line 4134 ("beta agents_archive_view");
  - `_archive_controller.py:6-7` ("With the beta off…");
  - `_display_helpers.py:148-150`;
  - `_app_action_availability_agents.py:185`;
  - `_link_follow_targets.py:342` ("falls through to Artifacts ▸ Agent");
  - the docstring at `actions/artifacts_files.py:268` ("reviving a dismissed match").
- **Vestigial parameters.**
  - Drop `enabled=` from `AgentArchiveShelf.set_snapshot`
    (`widgets/agent_archive_shelf.py`; its caller is `_archive_shelf.py`).
  - Drop `show_archive=` from `NodeFinderModal` (`_node_finder_modal_archive.py`;
    `_node_jump.py:71`) and make the Archive section unconditional.
  - Drop the unused `owner` argument from `archive_dismissal_toast_suffix`.
- **Dead custom-search revival path.** Delete `_show_dismissed_agents_for_custom_search`
  and `_dismissed_agent_rows` (`actions/agents/_revive_archive.py`), and their test in
  `tests/test_agent_group_revival_routing.py`. Then grep for other production
  constructors of `DismissedAgentSelectModal` (`modals/revive_agent_modal.py`). If none
  remain, delete the modal, its CSS, its `modals/__init__.py`/`.pyi` exports, and its
  tests. Fix the stale copy "custom revival search" in
  `saved_agent_group_revival_modal.py:171` and "before the custom search row" in the
  revival rendering module (`rendering.py:209`).
- **Symvision.** The six unused public symbols this epic caused fail `just symvision`.
  Resolve each one per `symvision.md`:
  - `archive_corpus_is_warm` (`_archive_controller.py`): delete it and its `__all__`
    entry, then switch `tests/ace/tui/test_agent_archive_controller_prewarm.py` and
    `tests/ace/tui/bench_tui_jk_archive.py` to `archive_corpus_snapshot(owner).warm`.
    Remove the `--epic-symbol 'sase-1jm(archive_corpus_is_warm)'` line from the
    `_lint-symvision` recipe in `Justfile`, keeping the continued command valid.
  - `archive_name_query` (`_archive_arrival.py`), `load_archive_name_candidates`
    (`_node_finder_archive.py`), `sync_archive_shelf` (`_archive_shelf.py`),
    `convert_agents_pane_query` (`src/sase/ace/_agents_pane_query_migration.py`), and
    `ArtifactLinkGroup` (`widgets/decks/record/relations.py`): privatize each one that
    is only used in its own file, and update its tests. Delete anything that is truly
    dead. `bridges-finish` may later make `sync_archive_shelf` public again with a real
    production caller.

  Afterwards, `just symvision` must list no symbol from these files. The remaining
  `sase-1jx` and `sase-1kd` items are owned elsewhere.

- **Lazy grouping preference (DISCOVERED ISSUE #2 on sase-1jm).**
  `init_agents_archive_view_state` (`_archive_view_state.py:46-60`, called from
  `actions/_state_init_runtime.py`) reads `~/.sase/agent_archive_grouping.txt` with
  `Path.read_text` inside `AceApp.__init__`. Defer that read to the first Archive open,
  or to the existing off-thread prewarm. It must not slow the `,a` open path, which has
  a warm budget under 100 ms. Keep the cold controller install, which probes nothing.
  Add a test that startup performs no Archive file reads.

## Retired Artifacts strip, migration correctness, docs, and goldens

Phase `.8` hid the pane, but tests and goldens still mount the old strip. Several
surfaces also ignore the retirement.

- **Fixtures follow production.**
  - `_fast_artifacts_subtabs` (`src/sase/ace/testing/_startup.py:87-106`, patched in at
    `:270`) and `_visual_artifacts_subtabs`
    (`tests/ace/tui/visual/_ace_png_snapshot_startup.py:82-84`, patched at `:252-262`)
    must include `fixed_descriptor("agents")` only when `artifacts_agent_pane_retired()`
    is off. That matches `_build_artifacts_subtabs` (`artifact_tabs.py:120-122`).
  - Fix the local override in `test_ace_png_snapshots_artifacts_descriptions.py:181` the
    same way.
  - `artifacts_contract/test_agents_pane_conformance.py:73-95` should compare both flag
    states.
  - Pin `test_ace_png_snapshots_artifacts_agents.py` to the flag off.
- **Help ▸ Guide digits.** `_build_tabs_card` (`widgets/patch_onboarding.py:174-184`)
  loops over the static `ARTIFACTS_SUBTAB_ORDER`. It must use
  `resolve_artifacts_subtabs()` order and digits, so the retired strip reads "1
  Stitch…".
- **Legacy mapping.** Delete the redundant branch at `artifact_tabs.py:348-351`, which
  returns `DEFAULT_ARTIFACTS_SUBTAB` either way. Then make one flag-aware mapping send a
  persisted or legacy `agents` sub-tab to `stitches` when retired, and leave it
  unchanged when not. Test both states.
- **CLI.** `sase artifact pane show agents -j` (`artifact_cli/pane.py:57-71`) must emit
  JSON for the moved-pane explanation instead of Rich text, per `cli_rules.md`.
- **Explicit-command redirect.** `show_artifacts_agents` is available only on the
  Artifacts tab (`_app_action_availability_artifacts.py:134-145`). When the pane is
  retired, it must be available from every tab and open Agents ▸ Archive, so a user
  keymap or palette entry works anywhere. Keep today's behavior with the flag off.
- **One-time toast.** `_startup_loads_maintenance.py:61-63` writes the "seen" marker
  before it checks the flag. Write the marker only when the toast is actually shown.
- **Help modal.** Test that the Agent Pane section appears with the flag off and
  disappears with it on.
- **Saved-query migration** (`src/sase/ace/_agents_pane_query_migration.py`).
  - Run every converted text through `canonical_agents_tab_query` (the agents-archive
    profile).
  - If validation fails, leave the entry in the old `agents` namespace, as the plan
    requires for unconvertible entries. Examples are `revivable:true`, `state:running`,
    `parent:…`, `hidden:true`, and `tribe:` values that are not valid `clan_tribe:`
    values.
  - Store the canonical text that validation returns, not the raw text.
  - Treat a negated `-dismissed:true` or `-state:dismissed` as unconvertible instead of
    dropping it, because dropping it reverses its meaning.
  - Fix the test fixture `tribe:core`.
  - Keep the marker and the free-slot rules.
- **Docs.**
  - `docs/configuration.md`:
    - `:5198` must list `artifacts_agent_pane_retired` and how to opt out;
    - fix the `sase agent search` "Artifacts → Agent Boolean dialect" line (`:7555`);
    - fix the "Artifacts Agents pane" key-sharing rows (`:1519-1520`);
    - document the `,a` / `toggle_agents_archive` keymap.
  - `docs/artifacts_pane_contract.md`.
  - `docs/artifacts_pane_visual_grammar.md` (`:3`, `:113`).
  - `docs/ace.md`: the Artifacts headings and lists at `:319`, `:390`, `:426`, `:440`,
    `:876`, `:2868`, `:5153`, and the query-key and save-query rows at `:3503`/`:3516`.
    `arrivals-finish` owns the link-follow section at `:245-262`, and
    `archive-chooser-copy` owns the copy-key docs.
  - `docs/query_language.md:151`.
- **Goldens.** Add one golden for the renumbered Artifacts strip (Stitch on `1`, no
  Agent) using the production resolver.
  - To get the real strip, call `artifact_tabs.reset_artifacts_subtabs_cache()` and
    monkeypatch `load_project_provider_records`, as the conformance test does.
  - Add one golden for the one-time toast, following
    `test_ace_png_snapshots_post_update_toast.py`: point
    `_artifacts_agent_pane_notice`'s marker at a tmp path, call
    `_maybe_show_artifacts_agent_pane_toast()`, then wait for the toast.
  - Regenerate every non-help Artifacts golden that changes now that fixtures follow the
    retired strip. Inspect each group in the report.

## Archive chooser sections, container restore, and copy keys

The plan's chooser and copy keys are only partly built. Abbreviations below: `entry` is
`actions/agents/_archive_restore_entry.py` and `keys` is `_archive_restore_keys.py`.

- **READ section.** Add `"read"` to the section literal in
  `modals/agent_action_chooser_modal.py:17`, and put `e` Open chat, `y` Copy `@agent`
  reference, and `%` More copy targets under READ. Restore choices stay under ARCHIVE
  (`entry:145-234`).
- **Containers.** Today `⏎` on a container row only toggles expansion
  (`_archive_column_nav.py:229-232`). It must open the chooser with
  `Restore clan <label> (N members)` (or the session wording) as the default, which
  makes the container branch in `keys:73-105` reachable. `l`/`h` keep expanding and
  collapsing containers.
- **Marks.** Offer `Restore N marked` even when the cursor row cannot be restored, and
  open the chooser when marks exist even if the cursor row's bundle is unreadable
  (`entry:134-136`, `:145`). N counts the restore targets, so a marked container counts
  its matching members.
- **Fork.** Offer Fork only when the hydrated bundle `Agent` has a retained chat that
  the existing fork path accepts. Today it only checks for a name (`entry:33-52`).
- **No sync I/O, no detail race.** `hydrate_archive_candidate` (`keys:122-140`,
  `_archive_reader.py:303-335`) reads the bundle and probes files on the event loop.
  - Hydrate in a thread worker before showing the chooser, re-capturing the selection
    after the await.
  - Make Fork, Open chat, and Copy use the agent that the chooser hydrated, not
    `archive_read_target`, which depends on the detail panel's async load
    (`entry:267-292`, `_archive_actions.py:216-235`).
- **`y`.** Replace the Agents-tab branch inside `action_artifacts_copy_reference`
  (`clipboard/_artifacts.py:123-125`) with a dedicated `agents_copy_reference` action
  bound to `y` on the Agents tab, in both views.
  - Add it to `default_config.yml`, the keymap registries, availability (split from the
    global `artifacts_copy_reference`), and palette metadata
    (`commands/_app_metadata_actions.py:125`).
  - Test both views, and test that `y` on the Artifacts tab is unchanged.
- **`%l`.** `reference_for_agent_row` already returns `agent:<name>`, so
  `clipboard/_agents.py:223` and the palette preview (`clipboard/_palette.py:169`)
  produce `agent:agent:…`. Use the reference itself as the link target (see
  `format_markdown_link`), and test with a real reference rather than a mock.
- **`%j`.** For an archived run, copy the bundle JSON, read off-thread
  (`clipboard/_agents.py:231-266`). Inbox agents keep the metadata record.
- **Help, footer, and docs.**
  - Add `%!`, `%l`, `%j`, `%d`, and `y` to the Inbox help Copy Mode box
    (`help_modal/agents_mode_sections.py:244-259`), keeping the 57-character box rules.
  - Update footer metadata if any of these keys are conditional.
  - Document `y` and the `%` targets in `docs/ace.md` and `docs/configuration.md`.
- **Tests and goldens.**
  - Chooser composition: restorable, not restorable, container, marks with an
    unrestorable cursor, and Fork with and without a retained chat.
  - PNG goldens for the Archive chooser with and without marks.

## Lookup-first link arrivals and selection that survives clearing

- **Miss keeps context.** Today `_archive_arrival.py:49-128` opens the Archive before
  the exact lookup, so a miss flashes Agents ▸ Archive and replaces the parked Archive
  query, selection, and pages. Run the off-thread exact lookup first:
  - On a hit, open the Archive with `in:archive name:"<canonical>"` and select the row,
    recording one Ctrl+O hop.
  - On a miss, toast `agent:… has no record on this machine` and change nothing.
- **Filter-hidden reveal.** `_reveal_filtered_loaded_agent` (`_link_follow_targets.py`)
  only searches the active agent tab's `_agents`.
  - Match `_reveal_loaded_agent`: switch agent tabs and expand folds as needed.
  - Do not replace the Inbox query until the target is confirmed loaded, so a
    fallthrough never leaves a rewritten query behind.
  - Export the reveal as one helper that `archive-restore-run` reuses.
- **Selection survives clearing `name:`.** Inside the Archive, a commit through
  `_filter_bar_session.py:203-206` → `open_agents_archive_view_with_query` resets
  `selected_key`, `reveal_row`, and `loaded_pages`
  (`_archive_view_lifecycle.py:141-150`). When the Archive is already active, keep the
  selected identity across the new query and reveal it in its banner if it still
  matches. Test the link-arrival → clear `name:` flow.
- **Ref-kind target.** `target_for_ref_kind("agent")`
  (`src/sase/ace/tui/relations/artifact_links.py:267-268`) still returns
  `ArtifactEntryTarget("agents", …)`, the retired pane. Audit its callers, then point
  agent refs at the Agents tab (Inbox, or Archive by lookup). The flag-off pane path
  stays reachable only where the pane itself is shown.
- **`,a` from another tab.** Add a mounted test showing that `,a` on the Artifacts or
  Beads tab opens Agents ▸ Archive.
- **Docs.** `docs/ace.md:245-262` still describes the deleted "lands in Artifacts ▸
  Agent" fallback and its toast line. Rewrite it to describe the four-step ladder.

## Restore execution, single toast, and restore-and-show reveal

Abbreviation: `run` is `actions/agents/_archive_restore_run.py`.

- **One toast.** `_do_revive_agents` already toasts `Revived N agent(s)` or
  `Failed to revive N agents` (`_revive_execution.py:459`, `:565`), so every Archive
  restore shows two toasts. Add a quiet mode for Archive callers. The Archive toast then
  reports restored, skipped, and failed counts, with skip and failure reasons
  (`run:295-325`).
- **Parked Inbox untouched.** `_on_revive_loaded` (`_revive_execution.py:591-597`) runs
  `_try_reveal_agent_row` whenever the Agents tab is current. That saves a jump anchor
  and may switch the parked Inbox agent tab (`navigation/_member_jump.py:442-444`).
  While the Archive is active, a plain Restore must not move the parked Inbox selection,
  agent tab, folds, or jump anchors.
- **Row and pulse.**
  - Render restored rows dimmed, with the existing `⌂ restored` chip
    (`widgets/agents_archive/_rows.py:173,192-193`).
  - Repaint the Inbox pulse as soon as the revive lands, without waiting for the next
    full reload.
- **Restore and show.** Today `_show_restored_in_inbox` (`run:328-350`) closes the
  Archive and tries a synchronous select. Instead:
  - Wait for the revive delta refresh, then switch to the Inbox and reveal the agent.
  - If the parked Inbox filter hides it, apply the same filtered-target rule as link
    follow, using the helper from `arrivals-finish`: commit `name:"<canonical>"`, push
    the old query to `^`, and toast.
- **Off-loop re-check.** Run the submit-time capability re-check (`keys:109-140`,
  `run:229-237`) in a thread worker.
- **Proof.**
  - Mounted tests through the real `_do_revive_agents` under a temp `SASE_HOME`, not a
    stub: single restore, container, marked, a skip with reasons, a failure with
    reasons, parked Inbox unchanged, pulse count, and restore-and-show both with and
    without a hiding filter.
  - A golden of the restored row with its toast.

## Shelf, zero-result bridge, and Node Finder correctness

- **Shelf copy.**
  - `widgets/agent_archive_shelf.py:62` renders `⏎ / ,a search`, the hint from the
    rejected `carry` design. Render `⏎ search`, and update the widget test.
  - Fix the inverted plurals ("1 match", "7 matches") so they read
    `1 matches this filter` and `N match this filter`, and make the bridge use the same
    wording.
- **No flicker.** A recount, after a commit or an index drift, must keep the last
  snapshot visible until the new count lands. Today it sets `loading`, which hides the
  shelf and reflows the panel heights twice.
- **Lit state.** Whenever the Inbox result's emptiness changes, the amber state must
  update, even if neither the query nor the index changed. Give `sync_archive_shelf` (or
  its replacement) a production caller in the Inbox apply path, or delete it.
- **Focus flag.** Clear `_agent_archive_shelf_focused` on any blur, click elsewhere, or
  tab switch, so `J` is never swallowed.
- **`in:inbox`.** `_compute_shelf_snapshot` strips the scope token before storing the
  query, so a committed `in:inbox …` never matches. As a result the bridge stays on
  "Checking…" and the shelf won't carry. Compare scope-stripped forms on both sides.
- **Bridge.**
  - Show the bridge only when the filter matches nothing on every agent tab. Never show
    it over the "No feed" state, and delete the unreachable `if empty_state is None`
    branch (`_agent_tabs_switch_strip.py:398-415`).
  - Bridge rows show time, outcome, and name.
- **Node Finder.**
  - With no jumpable rows and a cold Archive, `_node_jump.py:26-31` shows "No nodes to
    jump to". Open the finder with `Archive warming…` instead.
  - Cache the Archive name candidates per corpus generation, so each open does not
    reload every corpus row.
- **Tests and goldens.** Add tests for:
  - bridge states (counting, matches, Inbox-only fields, zero);
  - `J`/`K` navigation order with the shelf;
  - the carry on `⏎`;
  - no count per keystroke;
  - a cold-corpus Node Finder.

  Add the missing plain (no-filter) shelf golden.

## Light rows carry bundle path and times, and labels show names

DISCOVERED ISSUE #7 on sase-1jm is still open. `resolve_archive_bundle_path`
(`src/sase/ace/tui/models/agent_archive_reader.py:158`) and the light-row Record subject
(`widgets/decks/record/model.py`, the light-row builder near `:350-370`) read
`bundle_path`, `stop_time`, and `dismissed_at`. But the Rust `AgentArchiveLightRowWire`
(sase-core `crates/sase_core/src/agent_archive/corpus/wire.rs:210-226`, with
`deny_unknown_fields`) carries none of them. The compiled corpus already selects all
three (`corpus/compile.rs:88-108`, `:388-422`).

- **Core.** Open sase-core with `sase repo open sase-core`.
  - Add optional `bundle_path`, `stop_time`, and `dismissed_at` to the light row, filled
    from the compiled corpus. Container rows take their newest member's values or
    `None`.
  - Update the wire tests and bindings per its `AGENTS.md`.
  - Run `sase tool run check` there.
  - Move this repo's `sase-core-revision.txt` past that commit (`docs/rust_backend.md`).
- **Python.**
  - `resolve_archive_bundle_path` uses the row's real `bundle_path` first and falls back
    to the root scan.
  - Light-row Record shows the Dismissed and Ended facts, plus the Bundle provenance
    path, for unreadable entries.
  - Drop reads of keys the row will never carry, such as `raw_suffix`.
  - Check the CLI `sase agent search 'in:archive …' -j` output and the parity oracle
    with the new fields.
- **Project names, not keys.** Light-row Record uses `row.get("project")`, the directory
  key. Map it through the Archive controller's `display_snapshot`, the same path the
  Archive banners use, so every light-row surface shows `PROJECT_NAME`. This follows the
  `project-labels-never-keys` decision and `df61663204`. Test with a project whose name
  differs from its key.

## Help goldens, visual sweep, benches, and acceptance evidence

- **Archive help golden (DISCOVERED ISSUE #5).**
  `test_ace_png_snapshots_agents_archive.py::test_archive_help_png_snapshot` scrolls
  until "Live actions stay in Inbox" paints. By then the box title "Archive (read-only)"
  has scrolled off, a bisected regression from `ebd17bf5bf`. Fix the stop condition so
  it stops once the title row is visible, or scrolls the box into view. Then rebaseline.
- **Help goldens.** Regenerate the help-modal goldens once, covering every help change
  made by the earlier phases, and inspect each group.
- **Visual sweep.** Run a targeted strict check through `/sase_monitor` over these
  `tests/ace/tui/visual/` modules:
  - `test_ace_png_snapshots_agents_archive.py`
  - `…_agents_archive_entry_points.py`
  - `…_agents_record.py`
  - `…_artifacts_*.py`
  - `…_saved_groups.py`
  - `…_help_panel.py`
  - `…_post_update_toast.py`

  Every result must be created, updated, or unchanged, with no skips. Record the run ID.

- **Benches (DISCOVERED ISSUE #3).** Run `tests/ace/tui/bench_tui_jk_archive.py` and the
  Inbox control `bench_tui_jk.py` back to back, and run the warm `,a` first-rows bench.
  Record `uptime` load with each run.
  - Target: Archive j/k p95 under 16 ms and warm `,a` under 100 ms.
  - On a loaded host, an Archive p95 within the same-host Inbox control band counts as
    load, not an Archive regression. Record both distributions.
  - An Archive p95 clearly above the control band is a defect to fix in this phase.
- **Acceptance walk.** Check each of the parent plan's "Acceptance gates (whole epic)"
  against the tree and tests, then record pass/fail evidence as a note on this phase's
  bead.
- **Final check.** Run `sase tool run check`, and confirm `just symvision` lists nothing
  from this epic's files.
