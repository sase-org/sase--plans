---
tier: epic
title: 'Deck views landing remainder: integration fixes, P-transition budgets, and
  the live check'
goal: Finish the work the sase-1b1 (deck views) land agent found before that epic
  can close. Re-apply its uncommitted integration fixes so master is green for deck
  views. Bring `P` view transitions within the D10 budgets, or put the documented,
  measured mitigation in place. Perform the never-run live wide/narrow drive of the
  deck-view badge and cycle.
phases:
- id: integrate
  title: Re-apply the sase-1b1 landing integration fixes
  depends_on: []
  size: small
  description: 'integrate: move four keymap tests that remap actions onto the now-owned
    P key to the free key B. Privatize three view_policy symbols flagged by symvision.
    Make DeckViewPolicies.with_deck and distinct layouts dispatch explicitly on Main/Files.
    Delete the shadowed card_documents view_policy duplicate and add the R4 FINAL-panel
    persistence test.'
- id: perf
  title: Bring P view transitions within the D10 budgets or a measured guard
  depends_on: []
  size: large
  description: 'perf: profile P key-to-paint on the 5,000-line and 14,000-line Reply
    fixtures. Apply the D10 mitigations in order: remove redundant render and measurement
    work, then badge-first pump-safe painting, then (last resort) an explicit measured
    guard the UI explains. Re-measure with the deck-view bench and record the numbers.'
- id: live-verify
  title: Live wide/narrow drive of deck views and the acceptance checklist
  depends_on:
  - integrate
  - perf
  size: small
  description: 'live-verify: drive the real TUI with sase screenshot at wide and narrow
    widths (P, Ctrl+J, split, zoom). Confirm the D5 title-legibility test and walk
    the parent plan''s acceptance checklist. Fix small gaps and regenerate and inspect
    any golden that changes.'
proposed_by: bbugyi200.athena.sase-1b1.land
parent_bead: sase-1b1
create_time: 2026-09-27 14:52:58
status: done
bead_id: sase-1b1.8
---

- **PROMPT:** [prompts/202609/deck_views_landing_remainder.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/deck_views_landing_remainder.md)
- **PARENT:** [202609/deck_views.md](https://github.com/sase-org/sase--plans/blob/main/202609/deck_views.md)
- **BEAD:** [sase-1b1.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1b1/sase-1b1.8.md)

# Deck views landing remainder

## Background

Epic `sase-1b1` (plan `plan:202609/deck_views.md`) shipped deck views: the per-panel
`spread` / `page cards` / `page blocks` view, the top-border badge, `P`, palette
commands, persistence, and docs. All seven phases closed. The land agent verified the
work on master and found three things that keep the epic from closing:

1. **Integration fixes it could not commit.** A plan handoff skips the host finalizer,
   so the landing edits listed in phase `integrate` were not committed. Four of them
   repair master-red tests caused by the epic.
2. **D10 performance budgets are missed.** The parent plan requires, for every `P`
   transition, p50 at most 150 ms and p95 at most 300 ms key-to-paint on the standard
   5,000-line Reply (10 turns × 500 lines). It also requires max under 1,000 ms with no
   stall-watchdog row on a 14,000-line pathological Reply. The acceptance checklist
   allows "Benchmarks meet the D10 budgets (or the documented, measured mitigation is in
   place)". Neither holds. Phase `sase-1b1.6` measured (ms, p50/p95/max) on a loaded
   host:
   - standard: page blocks→page cards 664/757/776, page blocks→spread 615/1027/1399,
     page cards→page blocks 44/636/737, page cards→spread 112/727/770, spread→page
     blocks 24/35/39, spread→page cards 689/753/850.
   - pathological: every transition into an inline layout took 1.4–3.0 s max, and
     watchdog rows fired.
   - forced Files spread already passes: the keypress stays under 1 ms, and the paint
     lands about 120 ms after the probe.

   Samples swing with host load. Entering an inline layout (page cards or spread) on a
   huge Reply recomposes the whole Reply. The parent plan's D10 predicted exactly that
   risk and prescribed ordered mitigations. No mitigation or guard was applied.

3. **The live drive was never done.** Parent verify step 2 (`sase screenshot`, driving
   `P`, Ctrl+J, `|`, `Z` at wide and narrow widths) was replaced by golden inspection.

Read the parent plan (`sase plan` / the `plan:202609/deck_views.md` artifact) for the
D1–D10 design. Everything here is presentation-only Textual work, so nothing belongs in
`sase_core`.

## Shared constraints

- Scope stays Main and Files. Tools and FINAL (flag `ace_final_deck`) are permanently
  automatic: no badge, and `P` and the "Deck view: ..." commands are unavailable. View
  code dispatches explicitly (`deck is DeckId.MAIN`, `deck in (MAIN, FILES)`), never
  "not Tools" or an else fall-through. Keep
  `tests/ace/tui/widgets/decks/test_deck_view_main_pilot.py::test_final_panel_shows_no_badge_and_no_cycle`
  green.
- Automatic (`AUTO`) behavior stays byte-for-byte equivalent unless a phase's measured
  mitigation explicitly targets it.
- Read `tui.md` and `tui_perf.md` with `/sase_memory_read` before changing TUI code, and
  `lint_and_test.md` before finishing. Verify with `sase tool run check`.
- Keep every touched file under the `toobig` 1,000-line limit. `panel_view.py` is
  already about 820 lines, so put new transition or perf logic in a new module rather
  than growing it past the limit.
- When rendered output changes, regenerate goldens with targeted
  `just fix-tui-screenshots -- <selectors>` and inspect every created or updated golden.
- Unrelated master-red failures are already triaged (see the land note on `sase-1b1`).
  Do not treat them as phase work. Record anything out of scope as `PROPOSED FOLLOW-UP:`
  notes on your own phase bead.

## Phase: integrate

Re-apply these exact edits. They were verified in the landing workspace: 175 deck-view
tests and 70 keymap tests passed, and `just symvision` no longer listed any
`view_policy.py` symbol. If master already contains any of them, skip that item.

1. **Keymap tests collide with the new `P` default.** `cycle_deck_view: "P"`
   (`src/sase/default_config.yml`, `ace.keymaps.app`) is a real app binding. The
   registry (`src/sase/ace/tui/keymaps/registry.py`, duplicate check near
   `_CONTEXTUAL_APP_DUPLICATES`) therefore reverts any user override that remaps another
   app action onto `P`. Four tests used `P` as a "free" key and have failed on master
   since `a5e2a34dab`. Keep `P` as the deck-view default: the plan chose it, and the
   owner's deployed config has no `P` overrides. Change only the tests, to `B`, the only
   uppercase letter no default app action uses:
   - `tests/test_keymaps_registry_loading.py::test_partial_app_override`:
     `next_patch: "B"`, assert `reg.app.next_patch == "B"`.
   - `tests/test_keymaps_registry_loading_legacy.py::test_legacy_commits_action_override_migrates_to_stitches`:
     `commits_next: "B"`, assert `reg.app.stitches_next == "B"`.
   - `tests/test_keymaps_display_help.py::test_agents_help_uses_configured_direct_visible_fold_selector_key`:
     `expand_all_folds: "B"`, and expect `("B", "Toggle tribe fold by hint key")`.
   - `tests/test_keymaps_e2e.py::test_remapped_navigation_key`: `next_patch: "B"`, press
     `"B"`, and update the docstring and comment.
2. **Symvision: epic-owned unused publics** in
   `src/sase/ace/tui/widgets/decks/view_policy.py`: rename `BlockState` → `_BlockState`,
   `layout_signature` → `_layout_signature`, `distinct_layouts` → `_distinct_layouts`
   (all references, word-boundary), and drop the three names from `__all__`. Update the
   imports and uses in `tests/ace/tui/widgets/decks/test_deck_view_policy.py` and
   `tests/ace/tui/widgets/decks/test_final_deck_shell.py`; test files may import private
   names.
3. **Explicit dispatch (rule R2).**
   - In `_distinct_layouts`, replace the "Tools/FINAL → (), Files → two, else three"
     chain with: `MAIN` → `_LAYOUT_DEPTH`; `FILES` → `(SPREAD, PAGE_CARDS)`; every other
     deck → `()`. Word the docstring as "every other deck (Tools, FINAL) offers
     nothing".
   - In `src/sase/ace/tui/widgets/decks/model.py` `DeckViewPolicies.with_deck`: `MAIN` →
     replace `main`. `FILES` → raise `ValueError("Files deck cannot use page_blocks")`
     for `PAGE_BLOCKS`, else replace `files`. Any other deck → raise
     `ValueError(f"{deck.value} deck has no view policy: {view!r}")`. Update the
     docstring to "Rejects every deck other than Main/Files (Tools, FINAL) and Files
     `PAGE_BLOCKS`".
   - Update the comment above `policies.with_deck(...)` in
     `panel_view.py::set_view_policy` to match.
4. **Shadowed duplicate.** `6702105da8` (sase-1b2.11) added a `view_policy()` to
   `DeckPanelCardDocumentsMixin` in `src/sase/ace/tui/widgets/decks/card_documents.py`.
   It walks parents to read area state. `DeckPanelViewMixin.view_policy()`
   (`panel_view.py`) precedes it in the `DeckPanel` MRO and fully shadows it. Delete the
   method and the now-unused `DeckView` import. Also delete its stub-only test
   `test_view_policy_defaults_to_auto` in
   `tests/ace/tui/widgets/decks/test_card_document_view.py` (keep the `DeckView` import
   there; other tests use it).
5. **Stale docstring.** In `panel_chrome.py::_chrome_policy`, replace "AUTO until the
   engine lands ... main-engine phase owns view_policy()" with "Return the panel's view
   policy for `deck` (AUTO on failure). `DeckPanelViewMixin.view_policy()` owns the
   stored policies; chrome reads it defensively so a chrome refresh never raises."
6. **R4 test.** In `tests/ace/tui/models/test_agent_deck_persistence.py`, add
   `test_final_panel_decoded_to_main_keeps_views`. Write a v1 file whose panel is
   `{"deck": "final", "preferred_cards": {"main": "reply"}, "views": {"main": "page_cards", "files": "spread"}}`
   and load it under `sase.feature_flags.override_flags(ace_final_deck=False)`. Assert
   that the deck is `MAIN`, that `preferred_cards == {MAIN: "reply"}`, and that the
   views equal `DeckViewPolicies(main=PAGE_CARDS, files=SPREAD)`.

Verify: the four keymap tests, `tests/ace/tui/widgets/decks/`,
`tests/ace/tui/models/test_agent_deck_persistence.py`, and `just symvision` (no
`view_policy.py` entry), then `sase tool run check`. Known master-red failures you did
not cause include:

- FINAL deck tests (`test_card_document_decks_is_main_only`,
  `test_picker_catalog_covers_every_deck`, `test_final_live`), owned by sase-1b2.
- The import budget and the finalizer symvision reds, owned by sase-1b2.
- The memory README drift and `test_agent_completion`, owned by sase-1bc.
- The header-panel scroll test (sase-1b8) and the Files Ctrl+J flake (sase-1a7).

## Phase: perf

Work the D10 procedure, measure-first. This phase is sized `large` because the root
cause is not yet profiled: plan before implementing.

1. **Measure honestly.** Run `tests/ace/tui/bench_tui_deck_view.py` (marker `slow`; run
   explicitly by path; hand long runs to `/sase_monitor`) at least twice and note the
   host load. Profile the worst transitions (page blocks→page cards, page blocks→spread,
   spread→page cards) with `SASE_TUI_PERF` and/or pyinstrument per `tui_perf.md`. Record
   which part of key-to-paint is Rich/Markdown rendering, Textual layout/compose,
   measurement (`measure_card_rows`), the anchor-restore retries, and chrome/footer
   refreshes. Instrument the path from
   `AgentDetailDeckViewMixin.cycle_focused_deck_view` → `DeckArea.set_panel_view` →
   `DeckPanelViewMixin.set_view_policy` → `_apply_main_view_change` (in
   `panel_view.py`), plus the `CardDocumentView` show path (`document_view.py`) and
   `document_transitions.py`.
2. **Mitigation 1: remove redundant work.** Look for a double render (for example a
   paged recompose followed by a spread recompose), repeated measurement despite the
   fixed policy (fixed views must skip measurement), repeated anchor-restore work, or
   repeated `refresh_chrome` / footer refreshes per press. Remove only what the profile
   shows.
3. **Mitigation 2: pump-safe, badge-first painting.** If the whole-Reply recomposition
   itself is the cost, follow the established pump-safe patterns in `tui_perf.md`. The
   badge and panel chrome must paint on the first frame after the keypress, before the
   heavy body recomposition. The body then renders without starving the event loop
   (chunked or deferred, per the existing deck patterns), still restoring the reading
   anchor per D7. Keep rapid-press correctness: the generation counter and pending
   anchor must still make stale work a no-op.
4. **Mitigation 3 (last resort): explicit measured guard.** Only if a budget is still
   missed after 1–2, add an explicit, measured size guard that the UI explains. For
   example, a fixed inline layout above a measured line threshold could show a concise
   badge status or toast. Nothing may silently refuse the press or silently show a
   different layout than the badge claims (D5). Document the guard in `docs/ace.md`
   "Deck Views" and base the threshold on the recorded numbers.
5. **Re-measure** with the bench after each mitigation, and record p50/p95/max per
   transition for both fixtures, plus the Files forced-spread numbers, in your phase
   bead notes. The phase is done when the D10 budgets hold on a reasonably quiet host,
   or when a documented measured mitigation is in place and explained in docs. If host
   load makes numbers unreliable, say so and report the best quiet-host samples.
6. **Correctness guard rails.** These must stay green:
   - `test_deck_view_main_pilot.py` (all six ordered transitions keep block and offset,
     the bottom pin, rapid P-P-P, and partial documents).
   - `test_deck_view_files_pilot.py`, `test_deck_view_keys.py`, and the deck chrome and
     title suites.
   - The six `agents_deck_view_*` goldens
     (`just fix-tui-screenshots --check -- tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_views.py`).
     If a mitigation intentionally changes pixels, regenerate and inspect those goldens.

## Phase: live-verify

Runs after `integrate` and `perf` land.

1. Read `tui_screenshot.md` with `/sase_memory_read`. Use `sase screenshot` (`--keep`)
   on a live `sase tui` session with an agent whose Reply has several blocks and a Files
   deck with several text files, and another with an image. Drive `P` repeatedly,
   Ctrl+J/Ctrl+K, `|` (left-right split), `\`, `Ctrl+F`, and `Z`. Recapture at a wide
   width (about 160–200 columns) and a narrow one (about 90–120 columns, split halves).
2. Confirm the parent D5 visual test on each capture. With the body covered, the title
   alone must tell which panel and deck, spread vs paged, blocks inline vs paged, and
   automatic vs fixed. Check the Files `spreading…` and `spread unavailable` statuses
   and the rail cue (`page N/M` / `all N`).
3. Walk the parent plan's acceptance checklist and fix small gaps. Record
   `PROPOSED FOLLOW-UP:` for anything larger. If output changes, regenerate and inspect
   the affected goldens.
4. Record what was captured (widths, keys, and results per checklist item) in your phase
   bead notes, attaching the screenshots as artifacts if useful.
