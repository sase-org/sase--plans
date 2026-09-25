---
tier: tale
title: Finish and close the deck cutover landing-repairs epic (sase-17d.10.1.4)
goal:
  Every ACE visual test that drove the deleted legacy Agents detail UI runs on the deck
  API, deck chrome renders deterministically without clipped titles, the missing
  coverage goldens and live screenshots are done, the full visual check is clean apart
  from known unrelated failures, and sase-17d.10.1.4 plus its three phase beads are
  closed.
size: medium
proposed_by: bbugyi200.athena.0ru
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0ru](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ru.md)
- **COMMITS:**
  - [a674dc9](https://github.com/sase-org/sase/commit/a674dc91f186c653399a681f97d456876edbcb19)
    — test(ace): finish the deck cutover's visual migration and chrome fixes
    (sase-17d.10.1.4)

# Plan: finish and close epic `sase-17d.10.1.4` (deck cutover landing repairs) as one tale

## 1. Goal

Do all the remaining work of epic bead `sase-17d.10.1.4` ("Finish the deck cutover's
visual migration, coverage goldens and live checks", plan
`plan:202609/deck_cutover_landing_repairs.md`) in one agent. Then close its three phase
beads and the epic bead itself. The user asked for a single plan instead of the epic's
three phase agents.

Read these before starting:

- The epic plan:
  `sase artifact read plan:202609/deck_cutover_landing_repairs.md "<why>"`.
- The notes on `sase-17d.10.1.3` and `sase-17d.10.1`:
  `sase bead read sase-17d.10.1.3 sase-17d.10.1 -r "<why>"`.
- The `tui_screenshot.md` and `lint_and_test.md` reference memories, through
  `/sase_memory_read`.

When this tale is done:

- Every ACE visual test that drove the deleted legacy Agents detail UI uses the deck
  API, or has been deleted when its subject no longer exists.
- Two deck-chrome bugs found during planning are fixed:
  - a nondeterministic Main deck accent color
  - clipped border titles and subtitles
- The three missing coverage goldens exist.
- Live `sase screenshot` captures have been inspected.
- A full `just fix-tui-screenshots` update has been run and every change inspected, and
  a full `--check` leaves only the known unrelated failures listed in step 7.
- j/k bench numbers are recorded.
- `sase-17d.10.1.4.1`, `.2`, `.3` and `sase-17d.10.1.4` are closed. Do **not** close the
  parent `sase-17d.10.1`; its land agent owns it.

## 2. State at planning time (master `c83e3bd91`, 2026-09-24)

None of the epic's phase agents ever ran, so no phase commits exist. All three phase
beads are `in_progress` and have no notes.

### 2.1 Land-agent repairs (epic plan section 2)

The land agent's "fixed in this turn" edits were lost; its landing was interrupted. The
current status of each repair:

- **Repair 1 (mypy)** and **repair 2 (symvision)** are already on master in a different
  form. Commit `114fbca89` (`sase-18f.1`) uses class-level `if TYPE_CHECKING:` host
  declarations in `widgets/_agent_detail_display.py` and
  `widgets/_agent_detail_state.py` instead of inheriting `Static`. It also privatized:
  - `_FileSourceLabel`
  - the `vim_search_controller` helpers
  - `_status_text` in both render modules

  Do not redo or restructure either repair.

- **Repair 3 is missing.** Both fallbacks still query the deleted ids:
  - `AgentLLMCallsPanel._get_scroll_container` (`widgets/llm_calls_panel.py`) still
    falls back to `#agent-llm-calls-scroll`.
  - `AgentFilePanel._get_scroll_container` (`widgets/file_panel/_content.py`) still
    falls back to `#agent-file-scroll`.
- **Repair 4 is missing.** These non-visual tests fail today:
  - `tests/ace/tui/test_agent_fold_transitions_llm_calls.py`, two tests. The
    `_LLMCallsDetail` stub has no `focused_tools_view()`.
    `_route_llm_calls_detail_level` now requires a focused Tools view, so the keys fall
    through to the tree, and the `StubFoldApp` has no `_arm_panel_fold_hint_mode`.
  - `tests/ace/tui/test_agents_tab_current_project_seed.py::test_update_state_rebuilds_when_seeded_flag_changes`
    still passes `view_mode` (around line 204).
  - `tests/ace/tui/widgets/test_agent_jump_panel_visibility.py::test_search_overlay_keeps_jump_panel_visible`
    queries `#agent-prompt-scroll` (around line 164).
  - `tests/test_file_panel.py::test_zoom_file_cap_subtitle_points_to_editor` imports the
    deleted `modals/zoom_panel_events`.

### 2.2 Full visual check

`pytest -m visual -n 8 tests/ace/tui/visual tests/pager/visual` gave 42 failed and 989
passed. The failures fall into four groups:

- **Cutover migration (this tale).** Every test listed in epic plan section 3 fails,
  plus these additions:
  - `test_family_panel_shells_monitor_metadata_png_snapshot` and the first capture of
    the family fold-levels test. Their goldens predate decks, so they need regeneration
    only.
  - `test_agents_commit_messages_panel_png_snapshot`. Its `ctrl+f` scroll loop is now a
    no-op, and it asserts Main-deck and Files-deck text on screen at the same time.
- **Main deck accent flake (this tale; source bug, see step 2).** Affected tests
  include:
  - `agents_decks_{single_main_reply,context_reply_no_files,single_main_context,left_right_ratio_70_focus_left,collapsed_*,zoomed,top_bottom_focus_bottom}`
  - `agents_list_120x40`
  - `agents_neighbor_jump_*`
  - `agents_clan_unread_count_*`

  Which tests fail changes from run to run.

- **Stale after `c83e3bd91` (`sase-17x.12`, palette moved to `;`).**
  `agents_decks_single_empty_120x40`: the only difference is the onboarding "Command
  palette" key hint.
- **Unrelated; leave to their owners.**
  - `test_ace_png_snapshots_command_line.py`, 5 tests
  - `test_command_palette_png_snapshot`
  - `test_ace_png_snapshots_config_center_*`, 5 tests (the agent-CLI list data changed)
  - `test_top_bar_usage_attention_narrow_png_snapshot` and
    `test_top_bar_usage_badges_crowded_narrow_png_snapshot`, which time out on both
    trees

### 2.3 Orphan goldens

These goldens have no producer any more:

- `agents_context_zoom_modal_120x40`
- `agents_file_zoom_modal_120x40`
- `agents_file_zoom_search_120x40`
- `agents_metadata_zoom_modal_120x40`
- `agents_multi_file_zoom_modal_120x40`
- `agents_view_picker_five_layouts_120x40`
- `agents_family_panel_member_override_120x40`

`agents_waiting_unknown_zoom_modal_120x40` joins this list once step 3 deletes its test.

## 3. Scope rules

- Presentation and test work only. No `sase-core` change.
- Do not edit `docs/` or `sase/memory/`; `sase-17d.11` owns them.
- Do not fix the unrelated failures from section 2.2, and do not repair the bench
  harness.
- Only the two deck-chrome source fixes in step 2 and the two dead-fallback removals in
  step 1 change `src/`.
- Keep `wait_for_visual_idle(page)` as the final await before every capture.
- Run every full or broad `just fix-tui-screenshots`, the bench, and any `check` that
  may outrun your turn through `/sase_monitor` (`TESTING`/`TESTED`). Scoped
  `just fix-tui-screenshots -- <selectors>` runs are fine inline.
- Generation is not approval. Open and inspect every changed PNG.

## 4. Steps

### Step 0: Environment

Run `just install`, then `just install-visual`. Without the pinned visual stack, every
visual test errors at setup with `RendererEnvironmentError` (mismatched
`textual`/`rich`/`pillow` versions).

### Step 1: Remaining land-agent repairs (phase `.1` scope)

1. Remove the dead legacy-id fallbacks:
   - `AgentLLMCallsPanel._get_scroll_container` and
     `AgentFilePanel._get_scroll_container` return their `VerticalScroll` parent or
     `None`. Remove the `app.query_one("#agent-...-scroll")` fallback.
   - Keep the `try/except` around `self.parent`, because unmounted test panels raise
     there.
2. Migrate `test_agent_fold_transitions_llm_calls.py` to deck semantics:
   - Give `_LLMCallsDetail` a `focused_tools_view()` that returns a sentinel object when
     the stub is "focused" and `None` otherwise.
   - A focused Tools view takes `l`/`h`/`L`/`H`, and the existing assertions hold for
     that case.
   - An unfocused one leaves the keys to the tree. Assert that no detail actions were
     recorded. Stub `_arm_panel_fold_hint_mode` on the test app, or pick a scenario that
     doesn't reach it.
3. Drop the deleted `view_mode` kwarg or key in
   `test_agents_tab_current_project_seed.py`.
4. In
   `test_agent_jump_panel_visibility.py::test_search_overlay_keeps_jump_panel_visible`,
   drive the overlay with `detail.deck_area.focused_panel().show_search_overlay()`.
5. Delete `tests/test_file_panel.py::test_zoom_file_cap_subtitle_points_to_editor`.
6. Run those four test modules until they pass.

### Step 2: Deck chrome source fixes (do these before regenerating any golden)

#### 2a. Deterministic Main deck accent

`DeckPanelChromeMixin._resolve_accent` (`widgets/decks/panel_chrome.py`) reads
`self.app.theme_variables["secondary"]`. That dict is sometimes stale.

Evidence from a monkeypatched probe over `test_ace_png_snapshots_agents_decks.py` on
master:

- In 224 of 250 resolutions it returned `#004578`, the `textual-dark` secondary.
- Throughout, `app.theme == "flexoki"` and `app.current_theme.secondary == "#24837B"`.
- So the Main title, the active-tab pill, the `main N` subtitle entry and the spread
  card separators (`MainDeckView._spread_accent` calls the same resolver) render dark
  blue or teal depending on test order and timing.
- The committed goldens contain a mix of both colors.
- Resolving from `current_theme` made the serial module fail the identical 7 tests on
  two runs, so the fix is deterministic.

Fix:

- Resolve the Main accent from `self.app.current_theme.secondary`, falling back to
  `_FALLBACK_ACCENTS[DeckId.MAIN]`.
- Recommended: have `DeckPanel` subscribe to `app.theme_changed_signal` on mount, and
  call `refresh_chrome()` and re-render the Main spread separators, so that a runtime
  theme switch recolors live.
- `titles.DECK_ACCENTS` (`"$secondary"`) has no users. Leave it unless a linter
  complains.
- Add a unit test under `tests/ace/tui/widgets/decks/`. The Main accent must equal
  `app.current_theme.secondary` even when `app.theme_variables["secondary"]` holds a
  different value.

#### 2b. Border title and subtitle clipping

`_chrome_width()` passes `self.size.width` to `deck_title`/`deck_subtitle`. That is the
content width, outer − 2.

Textual renders a border label into outer − 2 cells, reserves 2 cells per corner, and
truncates with `…` (`textual/_border.py` `render_border_label`, `cells_reserved = 4`).
So the real label budget is `size.width - 4`. A title within 4 cells of the width passes
the fit check and is then clipped, instead of dropping to the compact or micro tier.

Master shows this already: the right panel of `agents_decks_context_reply_no_files`
renders `◆ MAIN ┃ ‹ 2/2 › …`, and its subtitles end in `files 0…`.

Fix:

- Pass the true label budget, `max(1, size.width - 4)` (keeping the 80 fallback before
  layout), to both `deck_title` and `deck_subtitle`.
- Add a unit test. Pick a panel width where the full title length L satisfies
  `size.width - 4 < L <= size.width`. It must pick the compact tier, and the rendered
  title must not contain `…`.
- Also check the search-command help subtitle in a LEFT_RIGHT half panel. The deck
  explorer saw it truncated.

### Step 3: Migrate the legacy visual tests

Add two shared helpers to `tests/ace/tui/visual/_ace_agents_png_snapshot_helpers.py` and
reuse them:

- `main_deck_scroll(page, panel_index=0)`: returns
  `detail.deck_area.panel(i).active_scroll()`.
- `scroll_main_section_to_top(page, identity, panel_index=0)`: implements the pattern
  below.

`scroll_main_section_to_top` is modeled on `_current_agent_metadata_section_id` in
`actions/navigation/_fold.py`:

- Get `panel.main_view`, and call `view.enable_section_layout_reserve()`.
- Wait until `view._section_anchor_generation == view._section_generation` and an anchor
  with `identity == <id>` exists in `view._section_anchors`.
- Scroll with
  `scroll.scroll_to(y=view.virtual_region.y + anchor.row, animate=False, immediate=True)`.
- Call `wait_for_visual_idle`.
- Assert that
  `view.resolve_section_at_row(int(scroll.scroll_y) - view.virtual_region.y, width=view.size.width) == <id>`.

Replace every wait on the hidden `#agent-prompt-panel`'s `active_section_identity` with
deck-visible predicates:

- `panel.active_main_card()`
- `view.resolve_section_at_row`
- the scroll offset

Main cards are only `context`, `reply` (titled "Output" for steps and monitors) and
`summary`. Sections live inside cards. In paged mode, select the card first with
`ctrl+j`/`ctrl+k`. In spread mode, `set_deck_preferred_card` does not scroll.

The table below lists each test's migration. Paths are under `tests/ace/tui/visual/`.
The first eight rows are phase `.1` scope; the rest are phase `.2` scope.

| Test                                                                                                                           | Migration                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `test_ace_png_snapshots_agents_family_panel_monitor.py::test_monitor_state_detail_png_snapshots` (7 states × 90/120)           | `scroll_main_section_to_top(page, "monitor")` (MONITOR is in the Context card). Assert at **both** widths; today the 90-column run silently captures a different scenario. Keep the MONITOR/Result:/Next:/Evidence: checks.                                                                                                                                                                                                                                                                                                                                 |
| `…family_panel_monitor.py::test_family_conversation_monitor_phase_png_snapshot`                                                | AGENT REPLY is the `reply` card. Press `ctrl+j` until `panel.active_main_card() == "reply"`, then assert MONITOR and AGENT REPLY are visible.                                                                                                                                                                                                                                                                                                                                                                                                               |
| `…family_panel_monitor.py::test_family_panel_shells_monitor_metadata_png_snapshot`                                             | No legacy API. Regenerate and inspect only.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `test_ace_png_snapshots_agents_family_panel_gate.py::test_selected_gate_shell_output_png_snapshot`                             | Gate OUTPUT is in the Output (`reply`) card. Select it, then scroll `main_deck_scroll(page)` until "gate output line 01" and the truncation marker are visible. Also migrate the leftover near line 149.                                                                                                                                                                                                                                                                                                                                                    |
| `test_ace_png_snapshots_agents_family_panel.py::test_family_panel_fold_levels_and_member_override_png_snapshots`               | Replace the `ctrl+j` loops with `scroll_main_section_to_top(page, "agent-xprompt")`. Keep the anchor across `z z`, and assert the resolved section again. "Wrap back to top" becomes `scroll_to(y=0)` plus a `None`/top assertion.                                                                                                                                                                                                                                                                                                                          |
| `test_ace_png_snapshots_agents_panels.py::test_agents_collapsed_panel_png_snapshot` (helper `_assert_collapsed_panel_summary`) | Replace the `info._view_mode == "tribe"` and view-chip assertions with `detail.deck_area.panel(0).active_main_card() == "summary"` and `"view:" not in info._build_display_text().plain`. Keep the TRIBE text checks.                                                                                                                                                                                                                                                                                                                                       |
| `test_ace_png_snapshots_agents_tribe_clan_summaries.py`                                                                        | Rewrite `_jump_to_clan_summaries` as `scroll_main_section_to_top(page, "tribe:clan-summaries")` (inside the Summary card).                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `test_ace_png_snapshots_agents_tribe_prompts.py`                                                                               | Rewrite `_jump_to_prompts` as `scroll_main_section_to_top(page, "tribe:prompts")`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `test_ace_png_snapshots_agents_tribe_panel.py::test_tribe_panel_four_level_png_snapshots`                                      | Use `main_deck_scroll(page)` in `_settle_tribe_visual` (near line 50) to hide the scrollbar and force the top.                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `test_ace_png_snapshots_agents_clan_panel.py::test_swarm_clan_panel_png_snapshots`                                             | Replace `#agent-prompt-scroll` (near lines 320 and 332) with `main_deck_scroll(page)`, and drop the unused `VerticalScroll` import if any.                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `test_ace_png_snapshots_agents_slow_tools.py::test_agents_slow_tool_calls_fold_levels_png_snapshots`                           | Replace `_focus_slow_tool_section`, `_slow_tool_section_top_aligned` and `_metadata_viewport_top_section` with `scroll_main_section_to_top(page, "slow-tool-calls")` and deck scroll helpers. Delete the dead `AgentDetail` import, `_LLM_CALLS_FOOTER_RE` and `_rendered_llm_calls_footer`.                                                                                                                                                                                                                                                                |
| `test_ace_png_snapshots_agents_linked_repos.py::test_agents_linked_repo_diff_file_panel_png_snapshot`                          | `#agent-file-scroll` becomes the Files deck's `active_scroll()`. Drop the Main-scroll hiding unless Main is visible.                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `…linked_repos.py::test_agents_commit_messages_panel_png_snapshot`                                                             | Its bare `AssertionError` comes from the no-op `ctrl+f` loop and from asserting Main-deck "Deltas:" together with Files text. Either show a Main\|Files split (`vertical_line`, then `show_deck(1, DeckId.FILES)`) or split the assertions per deck. Scroll with `ctrl+d` on the focused deck.                                                                                                                                                                                                                                                              |
| `test_ace_png_snapshots_agents_external_repos.py::test_agents_external_repo_diff_file_panel_png_snapshot`                      | Fails the convergence guard: the Files spread probe re-renders after idle. Wait for the Files render mode to settle (`panel.is_spread(DeckId.FILES)` or its paged state) before the final `wait_for_visual_idle`.                                                                                                                                                                                                                                                                                                                                           |
| `test_ace_png_snapshots_llm_calls.py`, case `agents_llm_calls_panel_full_120x40`                                               | `#agent-llm-calls-scroll` (near line 438) becomes `#agent-deck-panel-0-tools-scroll`. Re-measure the pinned `virtual_size.height` (79 was measured on the legacy layout). Replace the `modals/zoom_panel_modal.py` path string (near line 244) with an existing path; this changes both the EXPANDED and FULL goldens.                                                                                                                                                                                                                                      |
| `test_ace_png_snapshots_agents_metadata_search.py::test_agents_metadata_search_typing_and_committed_png_snapshots`             | Use the deck overlay: `,` `/` + query (typing), then `enter n`. Wait for `page.app._agent_metadata_search.mode == "committed"` and `"[2/"` in `detail.deck_area.focused_panel().search_command().render().plain`. Regenerate both goldens.                                                                                                                                                                                                                                                                                                                  |
| `test_ace_png_snapshots_agents_waiting.py::test_agents_waiting_unknown_zoom_modal_png_snapshot`                                | The zoom modal is deleted: delete the test and its golden. Its unique coverage is the "ghost" unknown target and the `[beads] run-bead ◐, done-bead ●, open-bead ○` line in the detail, which `test_agents_waiting_missing_target_row_png_snapshot` does not cover. Move it into a deck scenario, `test_agents_waiting_unknown_deck_png_snapshot` with golden `agents_waiting_unknown_deck_120x40`: Main deck on the waiter with the Wait section scrolled into view (reuse the fixture and bead seed). If the deck wraps the bead line, assert its tokens. |

Optional clean-ups in modules you touch anyway:

- Rename `_ace_agents_png_snapshot_zoom_fixtures.py`'s "zoom-modal" docstring.
- The `" · view:"` split in `test_ace_png_snapshots_agents_proc_shells.py`.

### Step 4: Coverage goldens (phase `.3` scope)

Add these tests to `test_ace_png_snapshots_agents_decks.py`. The spread Main case is
already covered by `agents_decks_single_main_reply_120x40`.

1. **Spread Files** (`agents_decks_single_files_spread_120x40`):
   - Use an agent with 2 or more small file pages. `zoom_multi_file_agent` in
     `_ace_agents_png_snapshot_zoom_fixtures.py` is currently unused and fits.
   - Call `show_deck(0, DeckId.FILES)` and wait for `panel.is_spread(DeckId.FILES)`.
   - Assert that the `spread` subtitle tag is shown.
2. **Paged after threshold** (`agents_decks_single_main_paged_120x40`):
   - Give Main content over `spread_max_screens` (default 1.5) × viewport rows, for
     example a very long `response.md`.
   - Wait for `panel.main_view.render_mode is RenderMode.PAGED`, and assert that there
     is no `spread` tag.
   - Do not use `pin_paged`. The golden must prove the threshold path, as the
     `sase-17d.8` notes ask.
3. **LEFT_RIGHT committed-search overlay**
   (`agents_decks_left_right_search_committed_120x40`):
   - Press `vertical_line`, choose the panel with `ctrl+f` **before** starting the
     search, then `,` `/` + query, `enter`, `n`.
   - Wait for committed mode and `[2/` on that panel's `search_command()`, as the
     `sase-17d.6` notes ask.

### Step 5: Scoped regeneration and inspection

Run
`just fix-tui-screenshots -- <every module touched in steps 3–4 plus test_ace_png_snapshots_agents_decks.py>`.
Open every changed or created PNG and check for:

- the deck tab strip and subtitle, with no clipped `…` title where a shorter tier fits
- the Main accent (teal `#24837B`)
- the empty states
- no `view:` chip and no `(p)` hint
- the intended card or section in view

Iterate on test or source fixes until the scoped `--check` passes.

### Step 6: Full update run and inspection (via `/sase_monitor`)

Run the full `just fix-tui-screenshots` through `/sase_monitor`. Its follow-up must read
the retained report (`.pytest_cache/sase-visual/latest-report.json`). Inspect:

- every creation and removal: the orphans in section 2.3 should be removed
- every update group, expanding any group with unexpected differences

Expected update groups:

- Main-accent-only recolors (from step 2a)
- title and subtitle tier changes on narrow or split panels (from step 2b)
- `agents_decks_single_empty_120x40`'s `;` palette hint (accept it; it's stale from
  `c83e3bd91`)

Then restore the unrelated goldens with `git checkout -- <paths>`, so their owners
review them:

- `command_line_*`
- `command_palette_default*`
- `config_center_*`

Any other unexpected diff must be explained or fixed before continuing.

### Step 7: Full check (via `/sase_monitor`)

Run `just fix-tui-screenshots --check` through `/sase_monitor`. It must be clean except
for the unrelated set in section 2.2:

- the command-line tests
- the command palette test
- the config center tests
- the two top-bar usage narrow tests

Any remaining failure in an Agents-tab or deck module must be fixed. Otherwise, prove
that it reproduces with the three cutover commits reverted. To build that baseline:

1. In a scratch worktree, run `git revert --no-commit` of `bda83083b`, `742c1df38` and
   `37c8bb264`.
2. Resolve the trivial conflicts.
3. Run pytest with `PYTHONPATH=<worktree>/src -m visual`.

Run `/sase_new_task` for each unrelated residual failure that has no task yet (task type
`ci`). It checks for duplicates first.

### Step 8: Live screenshots

Capture these PNGs with `sase screenshot` and inspect them. Use `--keep`, then
`tmux send-keys -t <sase_tmux_target>` and `sase screenshot --window <target>`.

- single Main
- a Main|Files split (`|`, then `ctrl+n` to Files on the new panel if needed)
- Context|Reply (Main in both panels, with `ctrl+j` on one)
- a collapsed node panel (`ctrl+s`)
- a zoomed panel (`Z`)

The captured TUI must run this workspace's code. Check that the Main accent is teal and
that the titles aren't clipped. Run the command from the workspace venv if the launched
TUI is a different install. Confirm there is no remaining border-title clipping near the
tab strip; if there is, fix it and regenerate only that scope.

### Step 9: Perf (via `/sase_monitor`)

- Run `pytest -s -m slow tests/ace/tui/bench_tui_jk.py`.
- Record p50/p95 per scenario and the host load average.
- Compare with the `sase-17d.3` flag-on numbers (next rail-absent p50 14.44/p95 19.28;
  prev p50 14.84/p95 21.85; rail-painting next p50 15.53/p95 21.92; prev p50 13.91/p95
  18.30).
- Known pre-existing misses under load: clan and tribe fold level 1, the fleet
  `hung_host`/`reconnect_churn`/`event_burst` scenarios, and the link-rail delta.
  `test_bench_axe_jk` fails on both trees.
- An overrun counts as a cutover regression only if it is worse than the no-cutover
  baseline on the same scenario. Do not repair the harness.

### Step 10: Verification

- Run `just fix`.
- Run `sase tool run check`. Prefer a prepared-completion verify monitor if it may
  outrun the turn; see `/sase_final`. At planning time `check` had no failures from this
  area. Confirm that any failure also reproduces without your diff before setting it
  aside.
- Run `pytest --collect-only -q tests/ace/tui/visual`; it must still collect.
- Rerun the four step-1 modules and `tests/ace/tui/widgets/decks/`.

### Step 11: Bead notes and closing (last, after verification passes)

1. `sase bead note` each phase bead with what was done in its scope:
   - `.1`: step 1 and the first eight rows of the step 3 table
   - `.2`: the remaining step 3 rows
   - `.3`: steps 2 and 4–9, including:
     - bench p50/p95
     - every baseline or unrelated failure and its task bead
     - live-screenshot findings
2. Close them in order with `sase bead close <id> --note "<what you verified>"`:
   `sase-17d.10.1.4.1`, `sase-17d.10.1.4.2`, `sase-17d.10.1.4.3`.
3. Close `sase-17d.10.1.4` with a summary note. The user explicitly asked for this epic
   to be closed by this plan.
4. Add a note to `sase-17d.10.1` saying its child epic is complete, so its land agent
   can resume. Do **not** close `sase-17d.10.1`.
5. Finish with `/sase_final`.

## 5. Files expected to change

- `src/sase/ace/tui/widgets/decks/panel_chrome.py`, and possibly
  `widgets/decks/panel.py`/`main_view.py` for the theme-change refresh
- `src/sase/ace/tui/widgets/llm_calls_panel.py`
- `src/sase/ace/tui/widgets/file_panel/_content.py`
- `tests/ace/tui/widgets/decks/` (new unit tests)
- the four non-visual test modules from step 1
- the visual test modules and helper listed in steps 3–4
- goldens under `tests/ace/tui/visual/snapshots/png/`, as inspected and accepted in
  steps 5–6
