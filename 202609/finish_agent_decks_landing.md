---
tier: epic
title: Finish landing agent data decks - Files spread, live spread checks and the
  j/k bench
goal: The Files deck really spreads in the live app, the deck visual tests that are
  broken at master pass again, live spread screenshots of Main, a spread/paged split
  and Files have been inspected and any separator or title-pill defects fixed, and
  before/after j/k bench numbers for the SINGLE and LEFT_RIGHT layouts are recorded
  with every budget miss explained, so the land agent of epic sase-17d can close it.
parent_bead: sase-17d
phases:
- id: files-spread-probe
  title: Make the Files deck spread live and repair the broken deck visual tests
  depends_on: []
  size: medium
  description: 'files-spread-probe: fix DeckPanelFilesMixin.on_worker_state_changed
    so the Files spread probe result is applied (task sase-18m), drop the hand-fed
    probe from the spread-Files golden test, retarget the session fold-levels PNG
    test off the removed agent-xprompt anchor (task sase-18y), then regenerate and
    inspect every Agents-tab golden these changes move.'
- id: spread-live-inspection
  title: Inspect live spread rendering and fix separator and title-pill defects
  depends_on:
  - files-spread-probe
  size: medium
  description: 'spread-live-inspection: capture live sase screenshot PNGs of a single
    spread Main deck, a LEFT_RIGHT split with one spread and one paged panel, and
    a spread Files deck. Inspect separators, the scroll-driven title pill and tiny-width
    degrade, explain the missing separator in the Reply golden, fix any defect, add
    the zoomed-tribe pilot test, and finish with a clean full visual check.'
- id: deck-jk-bench
  title: Record before/after j/k bench numbers for SINGLE and LEFT_RIGHT
  depends_on:
  - files-spread-probe
  - spread-live-inspection
  size: medium
  description: 'deck-jk-bench: run the j/k key-to-paint bench with decks on in the
    SINGLE and LEFT_RIGHT layouts, on the pre-spread baseline a054efc585 and on the
    final tree, interleaved to control for host load. Explain every p95 budget miss
    as host noise or a deck regression, fix any regression, and record the table in
    a bead note.'
proposed_by: bbugyi200.athena.sase-17d.land
create_time: 2026-09-25 08:46:09
status: wip
bead_id: sase-17d.12
---

- **PROMPT:** [prompts/202609/finish_agent_decks_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_agent_decks_landing.md)
- **PARENT:** [202609/agents_tab_decks_and_cards.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_tab_decks_and_cards.md)
- **BEAD:** [sase-17d.12](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17d/sase-17d.12.md)

# Plan: finish landing agent data decks (child of epic sase-17d)

## 1. Context

Epic `sase-17d` ("Agents tab agent data decks and cards", plan
`plan:202609/agents_tab_decks_and_cards.md`) replaced the Agents-tab detail panels with
deck panels (Main, Files and Tools decks made of cards). All eleven phases and the
nested cutover epics (`sase-17d.10.1`, `sase-17d.10.1.4`) are closed. The epic's land
agent checked the tree at master `8fd6a054fd` and found:

- **Already fine.** Every `just check` lint gate is green, including symvision, and
  `sase bead epic-symbols sase-17d` is empty. `widgets/decks/panel.py` is 735 lines, so
  the toobig blocker from the epic's notes is gone. The prompt-panel `CardPart` test
  failures from epic notes #3 and #5 were fixed by `bf3aa6c8a3`. The remaining
  scoped-test failures (family→session copy, a DB-lock, a timing budget, the grok probe)
  and the toobig hit on `command_line/screen_completion.py` belong to other work and are
  not in scope here.
- **Still open, and owned by this epic.**
  1. **Task `sase-18m`, caused by phase `sase-17d.8`.** The Files deck never spreads
     live. `DeckPanelFilesMixin.on_worker_state_changed`
     (`src/sase/ace/tui/widgets/decks/panel_files.py`) returns early unless
     `getattr(event, "is_done", False)`, but Textual's `Worker.StateChanged` only
     carries `worker` and `state`. The probe result is dropped, `_files_pending_probe`
     stays set, and `_render_mode[DeckId.FILES]` stays `PAGED`. The spread-Files golden
     test works around this by feeding the probe by hand (`_apply_files_spread_probe` in
     `tests/ace/tui/visual/test_ace_png_snapshots_agents_decks.py`).
  2. **Task `sase-18y`: an epic-migrated test that is broken at master.**
     `tests/ace/tui/visual/test_ace_png_snapshots_agents_agent_session_panel.py::test_agent_session_panel_fold_levels_and_member_override_png_snapshots`
     scrolls to an `agent-xprompt` Main section. `4af219ebae` (sase-18g.2) moved
     detached xprompts into the identity header, so that section no longer exists and
     the test times out after 15 s. `4af219ebae` landed before the epic's migration
     commit `a674dc91f1`, so the migrated test was already wrong when it landed. The
     land agent reproduced the failure at `8fd6a054fd`.
  3. **User land instruction (epic note #2), items 3 and 4.** Neither item was finished.
     (3) Live screenshots of single-panel spread Main, a split with one spread panel and
     one paged panel, and spread Files. Inspect them for separator styling (blank line
     plus titled rule, card anchor meta on one row, tiny-width degrade) and for the
     title pill following the scroll-derived active card. Fix any defects. (4) The j/k
     bench with decks on, in SINGLE and LEFT_RIGHT, on the parent of `329d4049b` (the
     spread commit) and on the current tree. p95 must stay under 16 ms. Explain each
     budget miss as an environment problem or a regression, and fix any regression. The
     before and after p95 go into the epic's close note. `sase-17d.10.1.4.3` recorded
     only current-tree numbers, compared them with old `sase-17d.3` numbers, and never
     measured LEFT_RIGHT.
  4. **Phase `sase-17d.7`, PROPOSED FOLLOW-UP #1.** Add a pilot test for tribe
     whole-panel focus plus `Z` plus j/k stepping through tribe summaries while zoomed.
- **Suspicious golden.** In
  `tests/ace/tui/visual/snapshots/png/agents_decks_single_main_reply_120x40.png`, the
  spread Main deck on the Reply card begins with a long thin `─` rule, then a short thin
  rule, then blank rows, then `AGENT CHAT`. The spec's heavy `━━ ◆ Reply ━━━` separator
  is nowhere on screen. It may be scrolled just above the viewport by body-start
  anchoring, or it may be a defect. Phase `spread-live-inspection` decides which.

Read these before starting: the epic plan's §3.7 (spread algorithm), §3.8 (visual
language: separators, pill, subtitle `spread` tag), §4.8 (perf rules) and §16
(verification); `plan:202609/deck_spread_mode.md` §5 (separators), §6–§9; and the notes
on `sase-17d`, `sase-17d.8`, `sase-17d.10.1.4.3`, `sase-18m` and `sase-18y`. Read the
`lint_and_test.md`, `tui.md`, `tui_screenshot.md` and `tui_perf.md` memory notes through
`/sase_memory_read`.

## 2. Rules for every phase

- Scope is only what is listed here. Do not touch the command-line, top-bar usage,
  family→session rename or 90-column layout failures. They are tracked as `sase-18o`,
  `sase-18n`, `sase-18s`/`sase-17m.5.1.6` and `sase-18p`.
- Do not recreate anything the cutover deleted (the `p` picker, the zoom modal, the
  panel enums, the `agent_decks` flag).
- Verification: run `just fix`, then `sase tool run check`. Hand it to `/sase_monitor`
  when it may outrun the turn; with no diff it escalates to the full lane, which took
  about 40 minutes on this host. Do not run `just check-full`. Run
  `just fix-tui-screenshots` only in targeted form (selectors after `--`) unless a phase
  says otherwise, and use `/sase_monitor` with `TESTING`/`TESTED` for long runs. Inspect
  the report and every created, removed or updated golden. Generating a golden does not
  approve it.
- Record out-of-scope findings as `PROPOSED FOLLOW-UP:` notes on your own phase bead.
  Close your phase bead with `sase bead close <phase-id> --note "<what you verified>"`.
  Never close `sase-17d` or this child epic.

## 3. Phase `files-spread-probe`

1. **Fix the completion gate** in `DeckPanelFilesMixin.on_worker_state_changed`:
   - Gate on `event.state`, using `textual.worker.WorkerState`.
   - On `SUCCESS`, read `event.worker.result`. In Textual 8, `Worker.result` is an
     attribute, so the `worker.result()` fallback is wrong and must go.
   - On `ERROR` or `CANCELLED`, clear `_files_pending_probe` and leave the deck `PAGED`.
   - Ignore non-terminal states.
   - Keep the stale-subject check in `_on_files_probe_result`, and remove its duplicated
     `if … : pass` block.
   - Confirm that the handler actually runs on `DeckPanel` for the probe worker's
     `StateChanged` (the worker is started by `DeckPanel.run_worker`, the message
     targets that node, and `file_panel/_fetch.py` has its own handler for the file
     view's workers). Also confirm it neither swallows nor steals other workers' events.
2. **Tests.** Add a pilot test that shows the Files deck for an agent with two small
   pages and waits on `panel.is_spread(DeckId.FILES)` with no hand-fed probe. The pages
   must actually spread, and the `spread` subtitle tag must appear. Add a unit test for
   the terminal-state gate, including the cancelled and error paths. Delete
   `_apply_files_spread_probe` and let
   `test_agents_decks_single_files_spread_png_snapshot` wait on the live probe.
3. **Task `sase-18y`.** Retarget the scroll and assert steps in
   `test_agent_session_panel_fold_levels_and_member_override_png_snapshots` to the first
   conversation section that still exists for a detached-xprompt session (for example
   `agent-prompt`). Alternatively, render the fixture non-detached, but only if the
   "conversation at xprompt" capture still has a purpose. Re-baseline
   `agents_session_conversation_level_1_120x40`,
   `agents_session_conversation_level_2_120x40` and
   `agents_session_panel_level_2_120x40`.
4. **Goldens.** Once the fix is in, every Files deck small enough to spread will spread.
   Find every Agents-tab visual module that shows a Files deck without
   `pin_decks_paged`. Candidates include the decks, agent-session panel, agent-session
   gate and agent-session monitor modules; the linked and external repo modules pin
   paged. Run a targeted `just fix-tui-screenshots -- <those modules>` and inspect every
   update. If a golden meant to show paged Files now spreads, decide whether spread is
   the correct new behavior (accept it) or whether the test should pin paged (edit the
   test). Record each decision in your closing note.
5. **Close the tasks.** Run `sase bead close sase-18m --note "<fix commit + evidence>"`
   and `sase bead close sase-18y --note "<fix + regenerated goldens>"`.

**Done when:** a live Files deck with two small pages spreads without help, the three
session goldens pass, targeted `--check` passes on every touched visual module, and
`sase tool run check` passes apart from the known unrelated failures.

## 4. Phase `spread-live-inspection`

1. **Live captures.** Follow `tui_screenshot.md` and use `sase screenshot`, with a
   `sase_home` fixture or a real archive that contains a small Context/Reply agent and a
   multi-file agent. Capture:
   - (a) the single-panel spread Main deck, at the top, then after `Ctrl+D` scrolls past
     the separator;
   - (b) a LEFT_RIGHT split (`|`) with one spread panel and one paged panel, for example
     spread Main beside a long paged Main, or Main beside Files;
   - (c) a spread Files deck;
   - one narrow capture, such as a LEFT_RIGHT split at about 100 columns, to see the
     tiny-width separator degrade.
2. **Inspect every PNG.** Check that:
   - the separator is one blank line followed by a heavy `━` rule in the deck accent,
     with the glyph and card title at the left;
   - there is no separator before the first card;
   - Files separators carry the page label (and check whether repeating the file name in
     the page's own heading reads well);
   - titles truncate cleanly at narrow widths;
   - the border-title pill follows the scroll-derived active card as you scroll, while
     the preferred card stays put;
   - the `spread` subtitle tag is shown only on the spread panel.
3. **Explain the Reply golden** from §1. Dump the rendered lines of `MainDeckView` in
   spread mode for `_reply_agent`, and compare the scroll target with
   `spread_body_start("reply")`. If `Ctrl+J` hides the Reply separator or lands in the
   wrong place, fix it: the spec anchors the reading position so the active card's
   header stays visible. Then regenerate `agents_decks_single_main_reply_120x40` and any
   sibling golden.
4. **Fix every defect you find** in `widgets/decks/separators.py`, `main_view.py`,
   `panel_spread.py`, `files_spread.py` or `titles.py`. Add a unit test or pilot test
   for each fix.
5. **Zoomed-tribe pilot test** (sase-17d.7 follow-up). Select a tribe node, focus the
   Main deck panel, press `Z`, step tribe summaries with j/k, and assert that the zoom
   holds, the deck content follows the selection, and a second `Z` restores the layout.
6. **Full visual check.** Run the full `just fix-tui-screenshots --check` through
   `/sase_monitor`. It must be clean except for failures that reproduce on a clean
   master worktree (known: the `sase-18o` command-line nodes and the `sase-18n` top-bar
   usage nodes). If it finds other Agents-tab or deck failures, fix them here.
7. Put the inspected screenshot paths and findings in your closing note.

**Done when:** all live captures are inspected and their defects fixed, the Reply golden
is explained (and fixed if it was wrong), the zoomed-tribe test passes, and the full
visual check is clean apart from the proven-unrelated nodes.

## 5. Phase `deck-jk-bench`

1. **Trees.**
   - Baseline: `a054efc585`, the parent of `329d4049b`. It already has the deck panel
     core, splits, retargeting, collapse and zoom, and persistence, all behind the
     `agent_decks` flag. Check it out in a scratch git worktree outside the repo and
     give it its own venv. Run it with `SASE_FEATURE_FLAGS='{"agent_decks": true}'`.
   - After: the final tree, where decks are unconditional.
2. **Layouts.** The bench (`tests/ace/tui/bench_tui_jk*.py`) has no layout control. Seed
   LEFT_RIGHT the same way on both trees. Either write a v1 `ace_agents_deck_state.json`
   (`layout` `left_right`, Main in both panels, or Main and Files) under the bench's
   SASE home, or patch
   `sase.ace.tui.models.agent_deck_persistence._agents_deck_state_path` from a scratch
   conftest. Both trees have the same persistence schema. If you add a permanent opt-in
   layout parameter to the bench helpers, keep it generic (for example a
   `SASE_BENCH_DECK_LAYOUT` env var read by the Agents bench setup), and use the same
   scratch mechanism on the baseline so both trees are measured identically.
3. **Runs.** Run `SASE_TUI_PERF=1 pytest -s -m slow tests/ace/tui/bench_tui_jk.py` via
   the guarded runner (see `lint_and_test.md`) for each tree × {SINGLE, LEFT_RIGHT}.
   Interleave the trees (A, B, A, B), take at least two rounds each, and record the load
   average for every run. Use `/sase_monitor` when a set may outrun the turn.
4. **Analysis.**
   - Build a table of p50/p95/max per scenario for each tree × layout.
   - For each scenario over budget, classify it:
     - **host noise:** it misses on the baseline too, by a similar amount under similar
       load, or the miss follows the load average;
     - **deck regression:** the final tree is consistently worse than the baseline in
       interleaved runs.
   - Profile any regression with `tui_perf.md`'s tools and fix it, keeping the fix
     inside the epic's §4.8 rules. Then re-measure.
   - `test_bench_axe_jk` failing on both trees is expected context, not a regression.
     Say so if it holds.
5. **Record** the full table, run ids and classification as a `BENCH RESULTS:` note on
   this phase bead, which the sase-17d land agent copies into the epic's close note.
   Also put the headline before/after j/k p95 for SINGLE and LEFT_RIGHT in the phase's
   closing note.

**Done when:** both trees are measured in both layouts, every budget miss is classified
with evidence, any deck regression is fixed and re-measured, and the `BENCH RESULTS:`
note exists.

## 6. Hand-off

When all three phases are closed, the child epic's land agent verifies them, closes this
child epic, and then resumes the interrupted `sase-17d` landing. Read the `LAND AGENT`
note on `sase-17d` for the verification and follow-up dispositions already recorded.
Then retire any `--epic-symbol` entries, close `sase-17d` with the bench numbers in the
note, run `just symvision`, and mark `plan:202609/agents_tab_decks_and_cards.md` done.
Those steps belong to the land agent, not to any phase above.
