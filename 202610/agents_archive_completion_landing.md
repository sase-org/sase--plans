---
tier: tale
size: medium
title: Finish the Agents Archive completion epic and land sase-1jm.9 and sase-1jm
goal:
  The tests sase-1jm.9 broke pass. Container and marked restore match the chooser. The
  shelf, the Node Finder, the goldens, and the docs keep the Archive plan's promises.
  sase-1jm.9 is closed, and sase-1jm is closed once it rechecks complete.
proposed_by: bbugyi200.athena.sase-1jm.9.land
bead: sase-1jm.9
create_time: 2026-10-13 00:19:50
status: done
---

- **PARENT:**
  [202610/agents_archive_completion.md](https://github.com/sase-org/sase--plans/blob/main/202610/agents_archive_completion.md)
- **BEAD:**
  [sase-1jm.9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jm/sase-1jm.9.md)

# Plan: Finish the Agents Archive completion epic and land sase-1jm.9 and sase-1jm

## Background

Epic `sase-1jm.9` (plan `plan:202610/agents_archive_completion.md`) closed all eight
phases. Its commits are `ba89d2a6db`, `48b31c4e43`, `9fa479245c`, `95430b58aa`,
`e243ce730f`, `a27c6252f3`, `be3a9a5d1d`, and `1230bb32c1`. The land agent checked the
tree at `1230bb32c1` against that plan and found:

- five failing tests the epic caused;
- a container and marks restore bug;
- several plan promises that are only partly met.

This tale finishes that work. It then closes `sase-1jm.9`, and after that its parent
epic `sase-1jm`.

The parent epic's decisions still bind this work. Do not re-ask them:

- `view_name = archive`;
- `toggle_filter = restore`: `,a` is a pure toggle, and only the shelf's and bridge's
  `⏎` carry the Inbox filter;
- `jump_opens_archive = yes`;
- No memory note may be edited. The glossary follow-up is task `sase-1kc`.

Follow `tui.md` and the perf rules: no sync I/O in handlers or the pump, re-capture the
selection after every await, and never compute stats on the paint path.

Follow-up triage is already done and recorded on `sase-1jm.9`. Do not create task beads
for the failures below; they have owners:

- `test_tui_app_import_stays_under_startup_budget` belongs to `sase-1ic`. This epic adds
  no modules to the import closure.
- `test_project_shown_when_no_patch` and
  `test_workflow_snapshot_render_reads_state_once_and_uses_parsed_data` belong to
  `sase-1k4`.
- `test_ace_page_restores_fast_startup_overrides_after_success` is flake `sase-1kf`.

Paths below are relative to `src/sase/ace/tui/` unless they start with `src/`, `tests/`,
`docs/`, or `Justfile`.

## 1. Failing tests the epic caused

All of these fail at `1230bb32c1`.

- **Stale path-label exemption.**
  `tests/test_project_display_presentation_audit.py::test_path_label_exemptions_name_existing_sites_and_rationales`
  fails because `tests/_project_display_presentation_audit_path_labels.py` (about
  line 165) still exempts `src/sase/ace/tui/modals/revive_agent_modal.py` /
  `_agent_filter_label`. `ba89d2a6db` deleted that file. Remove the exemption entry.
- **Node Finder rebuilds twice per keystroke.** This is a real perf bug.
  - `ba89d2a6db` made the Archive section unconditional. `_apply_refilter`
    (`modals/_node_finder_modal_filtering.py`) calls `_schedule_archive_search(query)`.
    That method always ends in `_refresh_archive_projection()`, which composes a view
    and calls `_rebuild_options` (`modals/_node_finder_modal_archive.py`).
    `_apply_refilter` then composes again and calls `_rebuild_options` a second time,
    with a fresh view id, so the identical-rebuild skip never fires.
  - Give `_schedule_archive_search` a keyword such as `project: bool = True`. Pass
    `project=False` from `_apply_refilter`, which does its own single compose, rebuild,
    and paint.
  - Keep the projection refresh for callers that need it: load finish, search finish,
    and the failure paths.
  - This must make
    `test_node_finder_modal.py::test_rapid_keystrokes_coalesce_to_final_query` pass
    without editing it.
- **Node Finder gc test baseline.**
  `test_node_finder_modal.py::test_dismissed_modal_releases_snapshot_rows_without_gc`
  assumes `_view is _live_view` after construction. With the Archive section always
  composed, they are separate objects. Both are released, and one empty view is
  installed.
  - Take the `NodeFinderView` baseline before constructing the modal.
  - Assert that no view holding rows survives, for example `base_views + 1` live views
    with empty `rows`.
  - Keep the snapshot-row and row-count assertions strict.
- **Lazy grouping read.**
  `tests/ace/tui/test_agent_archive_parking.py::test_quitting_from_archive_persists_grouping_only`
  still expects `init_agents_archive_view_state` to load the persisted grouping
  synchronously. `ba89d2a6db` deliberately made that load lazy
  (`request_archive_grouping_preference_load` in
  `actions/agents/_archive_view_state.py`).
  - Rewrite the test to the lazy contract. Re-init resets to the startup default and the
    file still says `project`. The next Archive open applies `project` once the
    off-thread load lands. Drive that load the way the other lazy-load tests in the file
    do.
  - Keep the "selection, deck choices, and query reset" assertions.
- **Jump legend copy.**
  `tests/ace/tui/widgets/test_agent_jump_legend.py::test_dismissed_marker` asserts
  `"revive"`, but under `jump_opens_archive = yes` the render is `⊘ 1 gone ▶ archive`.
  Assert the Archive copy. Its sibling `test_neighbors_map_shows_dismissed_revive_cells`
  was already fixed in `1230bb32c1`; use that fix as the model.

## 2. Container and marks restore

The parent plan says: on a container row the action reads _Restore clan research.41 (6
members)_ and restores the members that can be restored, and a marked container counts
its matching members. Today restore uses only _loaded_ members.

The relevant code is `actions/agents/_archive_restore_entry.py` (`entry`),
`_archive_restore_keys.py` (`keys`), `_archive_actions.py:archive_marked_run_keys`, and
`_archive_column.py:archive_container_members`.

Current bugs:

- **Collapsed container on `⏎`.** The chooser shows the corpus `member_count`. But
  `collect_archive_restore_keys` finds no loaded members and skips with
  `no loaded members — expand the container (l) first`.
- **Only a collapsed container marked.** `_marked_restore_targets` drops container keys,
  so the chooser offers `Restore to inbox` for the cursor row. But
  `collect_archive_restore_keys` sees non-empty marks and returns only non-container
  marked keys (`[]`). The user sees `Restored 0 · skipped 0 · failed 0`, and the cursor
  row is never restored.

Fix:

- Add one off-thread resolver that returns a container's _matching_ member light rows
  from the warm corpus under the committed query and grouping. Use the same corpus
  rows/window path the column uses when a container is expanded. Never hydrate bundles
  just to count or list members.
  - Use loaded members when the container is expanded.
  - Otherwise query the corpus in a thread worker, through the existing
    `_archive_run_off_thread` / `_archive_spawn` seams.
- Build one resolved target list per chooser open, after the existing off-thread
  hydration and before the chooser shows. This covers both the cursor container and
  marked containers expanded to their members, de-duplicated. Re-capture the selection
  and generation after the await.
- The chooser label count and the executed restore keys must come from that same list:
  - `Restore N marked` counts every resolved run;
  - `Restore clan X (N members)` uses the resolved member count;
  - with marks present, never fall back to the cursor row's `Restore to inbox`.
- The submit path must restore exactly the resolved targets. Keep the off-loop
  capability re-check, and keep per-member skip and failure reasons in the single toast.
- A container whose resolution finds zero members still yields one clear skip reason.
- **Tests.** Add tests through the real `_do_revive_agents` under a temp `SASE_HOME`,
  following `tests/ace/tui/test_agent_archive_restore_proof.py`:
  - collapsed-container restore;
  - expanded-container restore;
  - only a collapsed container marked;
  - a container and a run both marked.

  Also add a chooser-composition test showing that the label count equals the executed
  key count.

## 3. Shelf, Node Finder, and copy correctness

- **Shelf keeps its last snapshot during a recount.**
  - Today `request_archive_shelf_refresh` (`actions/agents/_archive_shelf.py`) replaces
    the snapshot with `status="loading"` and drops `matching`. The widget
    (`widgets/agent_archive_shelf.py`) then paints `◷ Archive · updating…`.
  - Keep painting the last snapshot's text until the new count lands. That means the
    last count wording, or the plain `N runs · M today` line. The plan says "keep the
    last snapshot visible until the new count lands".
  - Keep the no-reflow behavior.
  - `activate_archive_shelf` must still carry the _current_ committed filter on `⏎`
    during a recount, when that filter is answerable. Decide answerability without I/O,
    for example with the same profile parse `_compute_shelf_snapshot` uses. Otherwise
    open the plain Archive.
  - Update `tests/ace/tui/widgets/test_agent_archive_shelf.py`, which currently locks in
    `updating…`.
- **No count per keystroke.** Add a test that types into the Agents filter bar and
  asserts that the shelf recount (`corpus.count`) runs only on commit, never per
  keystroke.
- **Node Finder upgrades when the Archive warms.**
  - `action_jump_to_node` (`actions/navigation/_node_jump.py`) passes
    `archive_name_loader=None` on a cold Archive, so the modal shows `Archive warming…`
    for its whole life.
  - Let the modal re-resolve the loader once the controller turns warm. One option is
    passing a provider callable such as `lambda: archive_name_loader_for_owner(app)` and
    retrying it on each refilter and from a short bounded timer while it is cold.
  - Once it resolves, start the existing off-thread name load.
  - Extend `tests/ace/tui/test_node_finder_e2e.py`'s cold-corpus test so that warming
    the controller while the modal is open fills the Archive section.
- **`%j` captures at keypress.** The Archive bundle-JSON copy reads the selected key
  inside the worker thread (`actions/agents/_archive_reader.py`, about lines 418-425).
  Capture the key and light row on the event loop at keypress, and pass them to the
  worker.
- **Tests that are still missing.**
  - `y` (`agents_copy_reference`) with the Archive active copies the archived run's
    `@agent:` reference.
  - A mounted `,a` from the Beads tab opens Agents ▸ Archive. The Artifacts-tab variant
    exists in `tests/ace/tui/test_agent_archive_remaining_proof.py`; mirror it.

## 4. Chooser goldens

- `tests/ace/tui/visual/test_ace_png_snapshots_agents_archive.py` builds the chooser
  choices by hand (about lines 601-686). As a result the marked golden shows
  `shows it again`, while production says `shows them again`. Build both chooser goldens
  from the real `archive_chooser_model` output.
- The Fork glyph `⑂` (`entry`, about line 268) renders as a missing-glyph box under the
  pinned font stack (Fira Code, DejaVu, Noto Emoji). Replace it with a glyph that stack
  renders, and confirm it in the golden.
- The restored-row golden (`agents_archive_restored_toast`) keeps the restored row under
  the cursor, so the dimming never shows. Move the cursor off that row so the dim row
  and its `⌂ restored` chip are visible.
- Recapture only these goldens with targeted `just fix-tui-screenshots -- <selectors>`.
  Run it inline when it fits; otherwise use `/sase_monitor`. Inspect every created or
  updated image in the report.

## 5. Cleanups and docs

- `_app_action_availability_agents.py:185`: drop "(beta on, parked)" from the docstring.
- `artifact_tabs.py` `normalize_artifacts_subtab`: delete the redundant
  `canonical == "agents" and artifacts_agent_pane_retired()` branch (both arms return
  `DEFAULT_ARTIFACTS_SUBTAB`). Keep its comment on the default fallback. The conformance
  tests for both flag states must still pass.
- `actions/agents/_agent_tabs_switch_strip.py` (about line 401): the
  `if empty_state is None: return None` branch is unreachable when the visible count
  is 0. Replace it with a type-narrowing restructure or an assert, so no dead branch
  remains.
- `tests/ace/tui/test_link_follow.py:243`: drop the `_when_enabled` suffix from
  `test_archived_agent_link_uses_archive_arrival_when_enabled`.
- `Justfile` (about lines 443-446): reword the `sase-1jm.9.4` /
  `reveal_filtered_loaded_agent` comment in the past tense. The row is gone, and
  `sase-1jm.9.5` consumes the helper. Never put a comment line inside the continued
  Symvision command.
- **`docs/ace.md`.**
  - The Artifacts headings at about lines 327, 401, 437, and 451 still list Agent as a
    live pane. So does the body at about 330 and 887 ("five fixed Artifacts views:
    Agent, …").
  - Say the Agent pane is retired by default (`artifacts_agent_pane_retired`) and shows
    only with the opt-out.
  - If a heading changes, fix its anchors at about `docs/ace.md:949`,
    `docs/ace.md:3672`, `docs/configuration.md:905`, and `docs/query_language.md:69`.
  - Lines 516-526, 2802, and 5622 still cite "the revive modal", which was deleted.
    Describe the Archive Record Lifecycle card (Viewable / Restorable / Restartable /
    Missing) and Archive restore instead.
- `docs/artifacts_pane_contract.md` (about lines 204 and 226) still treats Agents as a
  live fixed pane. Mark it retired by default.

## 6. Verify

- Run the focused suites for every touched area:
  - Node Finder modal and e2e;
  - Archive parking;
  - restore proof, chooser, and keys;
  - shelf widget and shelf actions;
  - link follow;
  - jump legend;
  - path-label audit;
  - copy-reference.
- Run the targeted goldens from step 4.
- Run `sase tool run check`. Every remaining failure must be KNOWN with the owners
  listed in Background (`sase-1ic`, `sase-1k4`, `sase-1kf`). Treat anything else as
  yours.
- `just symvision` must list nothing from this epic's files.

## 7. Close out `sase-1jm.9`, then `sase-1jm`

Do this in the same turn as the code. It must not wait for this work's own commit, push,
or CI.

1. Run `sase bead epic-symbols sase-1jm.9`.
   - Resolve every listed `--epic-symbol` entry: wire it up, privatize it, add a
     non-test pragma, or delete it, per `symvision.md`.
   - Re-key an entry only to a still-open later bead that still needs it.
   - At planning time the list was empty.
2. Close the epic: `sase bead close sase-1jm.9 --note "<verification>"`.
   - The note summarizes the landing verification: all 8 phases checked against source
     and commits, the gaps closed by this tale, the integration review, and the
     follow-up triage already noted on the bead.
   - Never use `--force` merely to make the close succeed.
3. Run `just symvision` and confirm that the whitelist is clean.
4. Set `status: done` in the frontmatter of `plan:202610/agents_archive_completion.md`.
   Use the EPIC PLAN path that `sase bead read sase-1jm.9 -r "<why>"` prints.
5. **Parent `sase-1jm`.** It is a plan bead (epic tier) with no parent of its own.
   - Re-review it: `sase bead read sase-1jm -r "<why>"`. Check the previous landing note
     #10 and DISCOVERED ISSUE notes #11-#13; this tale resolves #11-#13.
   - Confirm that every descendant is closed: phases `.1`-`.8`, nested epics `.2.1`,
     `.3.1`, `.3.1.8`, `.4.2`, and `.9`.
   - Re-walk the "Acceptance gates (whole epic)" section of its linked plan
     `plan:202610/agents_archive_view.md` against the tree. `sase-1jm.9.8` note #3 holds
     the last evidence.
   - Check post-child drift: commits on master since this tale's base that touch Archive
     code.
   - If it is still complete:
     - retire any `sase bead epic-symbols sase-1jm` entries;
     - close it with `sase bead close sase-1jm --note "<what you rechecked>"`;
     - confirm with `just symvision`;
     - set `status: done` in `plan:202610/agents_archive_view.md`.
   - If it is incomplete or ambiguous, do not close it. Record the blocker with
     `sase bead note sase-1jm "..."`, and report it in the final response.
   - Never use `--force` to advance a successful nested landing.
