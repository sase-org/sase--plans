---
tier: epic
title: 'Deck views landing fixes: badge-first prebuilt paint fidelity, FINAL-flag
  test drift, and golden rebaseline'
goal: The sase-1b1.8 deck-views landing remainder can close. The badge-first prebuilt
  Main body paints pixel-identically to the synchronous render. The epic's tests and
  docs match the always-on FINAL deck. The six agents_deck_view PNG goldens pass `--check`
  on master.
phases:
- id: fidelity
  title: Make prebuilt deferred bodies match the synchronous render and re-apply the
    landing edits
  depends_on: []
  size: medium
  description: 'fidelity: render prebuilt Main bodies through the same console settings
    and widget post_render base style that Textual''s RichVisual uses, with a strip-equality
    regression test. Re-apply the land agent''s FINAL-flag test and Deck Views docs
    edits.'
- id: goldens
  title: Rebaseline and inspect the six agents_deck_view goldens
  depends_on:
  - fidelity
  size: small
  description: 'goldens: regenerate the six agents_deck_view PNG goldens with the
    targeted update form, inspect every change, and confirm `--check` and `sase tool
    run check`.'
proposed_by: bbugyi200.athena.sase-1b1.8.land
parent_bead: sase-1b1.8
create_time: 2026-09-27 17:36:33
status: done
bead_id: sase-1b1.8.4
---

- **PROMPT:** [prompts/202609/deck_views_prebuilt_paint_fidelity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/deck_views_prebuilt_paint_fidelity.md)
- **PARENT:** [202609/deck_views_landing_remainder.md](https://github.com/sase-org/sase--plans/blob/main/202609/deck_views_landing_remainder.md)
- **BEAD:** [sase-1b1.8.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1b1/sase-1b1.8.4.md)

# Deck views landing fixes

## Background

The land agent for epic `sase-1b1.8` (plan
`plan:202609/deck_views_landing_remainder.md`, parent epic `sase-1b1` deck views)
verified all three phases. It found epic-caused work that keeps the epic from closing.

1. **Badge-first prebuilt bodies paint differently from the synchronous path.** Phase
   `sase-1b1.8.2` (commit `80fbe7020`) added
   `src/sase/ace/tui/widgets/decks/panel_view_deferred.py`. On a user `P` press,
   `build_prebuilt_offthread` renders the destination Main body off the loop on a
   **private** `rich.console.Console(width=..., highlight=False)`. It stores
   `(strips, height, anchors)`, and `SectionTrackingVisual.render_strips` / `get_height`
   (`src/sase/ace/tui/widgets/prompt_panel/_section_navigation.py`) serve those strips
   instead of `self._visual.render_strips(...)`. The module docstring claims "Pixels ...
   match the synchronous path". They do not. In
   `tests/ace/tui/visual/snapshots/png/agents_deck_view_fixed_page_cards_120x40.png` and
   `agents_deck_view_split_narrow_120x40.png`, the `P`-transitioned panel's
   `── AGENT (code) ── 10:06:00` block-header rows (and the rule rows under them) now
   carry a grey background. The automatic panel beside it in the same split capture,
   which went through the synchronous path, does not. The live-verify phase bisected
   this drift to `sase-1b1.8.2`.

   Likely root cause: Textual's `RichVisual.render_strips` (`textual/visual.py`) does
   three things the prebuilt path skips. It renders with `app.console`, which is built
   in `textual/app.py` with
   `markup=True, emoji=False, safe_box=False, force_terminal=True, soft_wrap=False` and
   `color_system=constants.COLOR_SYSTEM`. It derives options from
   `app.console_options.update(highlight=False, width=..., height=...)`. And it first
   wraps the renderable with
   `self._widget.post_render(self._renderable, style.rich_style)`, which applies the
   widget's base style. Confirm this with a strip comparison before fixing. Do not
   assume it.

2. **The FINAL flag removal invalidated two epic tests.** Commit `2d8f2f056`
   (sase-1b2.19) deleted the `ace_final_deck` flag, so FINAL is always on.
   `override_flags` silently accepts unknown names.
   - `tests/ace/tui/models/test_agent_deck_persistence.py::test_final_panel_decoded_to_main_keeps_views`
     (the R4 test from sase-1b1.8.1) now fails. A FINAL panel stays FINAL.
   - `tests/ace/tui/widgets/decks/test_deck_view_main_pilot.py::test_final_panel_shows_no_badge_and_no_cycle`
     still passes, but it wraps its body in the dead
     `override_flags(ace_final_deck=True)`.

   The land agent made these edits but could not commit them, because a plan handoff
   skips the host finalizer. The `fidelity` phase re-applies them.

3. **All six agents_deck_view goldens drift.** Besides item 1, two intentional
   cross-epic changes reached these goldens. The footer hint `[/] blocks` became
   `(/) blocks` (sase-1bc.1, bracket-to-paren move). The deck rail count row gained
   `final 0` / `final 1` now that FINAL is always on (sase-1b2.19). Those are correct
   and only need rebaselining.

Everything here is presentation-only Textual work, so nothing belongs in `sase_core`.

## Shared constraints

- Read `tui.md`, `tui_perf.md`, and `tui_screenshot.md` with `/sase_memory_read` before
  changing TUI code or goldens, and `lint_and_test.md` before finishing.
- Keep the badge-first behavior. Heavy Rich rendering stays off the event loop, the
  badge flips on the first frame, and the generation guard still makes stale work a
  no-op. The fix is fidelity, not reverting to a synchronous render.
- Keep Tools and FINAL permanently automatic (no badge, no `P`). Keep dispatch explicit
  (`deck is DeckId.MAIN`, `deck in (MAIN, FILES)`).
- Keep every touched file under the `toobig` 1,000-line limit.
- `just symvision` is currently red on four unrelated
  `src/sase/integrations/usage_windows.py` sase-telegram pragma errors, tracked as task
  `sase-1bj`. Treat them as pre-existing and do not fix them here. Record anything else
  out of scope as `PROPOSED FOLLOW-UP:` notes on your own phase bead.
- `test_deck_block_spread_pilot.py::test_scroll_derived_cursor_and_streaming_stays`
  flakes under host load, tracked as task `sase-1bl`. It is not phase work unless your
  change makes it fail deterministically.

## Phase: fidelity

1. **Prove the divergence.** Write a focused test (for example
   `tests/ace/tui/widgets/decks/test_deck_view_prebuilt_fidelity.py`). Mount a Main deck
   panel in a pilot app with a multi-block session Reply card like the golden fixture in
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_views.py`. Build its
   destination renderable with `build_destination_renderable(...)` for page cards,
   spread, and page blocks. Compare `build_prebuilt_offthread(...)` strips (segments
   text and style) against what the panel's `SectionTrackingVisual` produces through the
   synchronous `RichVisual.render_strips` for the same width. This test must fail before
   the fix, at least on the block-header rows.
2. **Fix the prebuilt render.** On the UI thread, where `panel_view_transition.py`
   schedules the off-thread build, capture three things. First, the console
   configuration (mirror `app.console`'s settings on a fresh private `Console`; do not
   share `app.console` across threads). Second, `app.console_options`. Third, the
   widget's `post_render`-wrapped renderable with the same base `rich_style` that
   `RichVisual.render_strips` would use. Then render the post-rendered renderable off
   the loop with those options. Keep anchors derived from the same strips. Keep the
   `(digest, width)` cache key, and add the base style to the key if it can differ
   between panels (focused vs unfocused, accent). Update the module docstring so it
   states the guarantee truthfully.
3. **Re-apply the land agent's integration edits.**
   - In `tests/ace/tui/models/test_agent_deck_persistence.py`, rename
     `test_final_panel_decoded_to_main_keeps_views` to `test_final_panel_keeps_views`
     with the docstring "A FINAL panel round-trips as FINAL and keeps its Main/Files
     views." Drop the `override_flags` import and context manager. Assert
     `loaded.panels[0].deck is DeckId.FINAL` and keep the `preferred_cards` and `views`
     assertions.
   - In
     `tests/ace/tui/widgets/decks/test_deck_view_main_pilot.py::test_final_panel_shows_no_badge_and_no_cycle`,
     drop the `override_flags` import and the
     `with override_flags(ace_final_deck=True):` block and dedent its body. Change the
     docstring's first line to "A FINAL panel has no badge and P is unavailable."
   - In `docs/ace.md` "Deck Views", make four changes. "The Tools deck has no views: it
     always pages automatically and shows no badge." becomes "The Tools and FINAL decks
     have no views: they always lay out automatically and show no badge." The no-badge
     sentence names "the Tools or FINAL deck". The `P`-unavailable list names "the Tools
     or FINAL deck". After "routine presses stay silent because the badge changes in
     place.", add: "The badge and panel chrome repaint on the first frame after `P`; the
     Main body for the new layout is built off the event loop and swapped in when ready,
     so on a very long Reply the badge can lead the body by a moment."
   - Grep `src/`, `tests/`, and `docs/` for any other `ace_final_deck` reference and
     remove it.
4. **Verify.** Run the new fidelity test, `tests/ace/tui/widgets/decks/` (including
   `test_deck_view_main_pilot.py`, `test_deck_view_files_pilot.py`, and
   `test_deck_view_keys.py`), the prompt-panel section-navigation cache tests,
   `tests/ace/tui/models/test_agent_deck_persistence.py`, and the four keymap tests that
   use `B`. Then run `sase tool run check`. Rerun the deck-view bench
   (`tests/ace/tui/bench_tui_deck_view.py`, `slow` marker, run by path, hand long runs
   to `/sase_monitor`) once on the standard 5k fixture to confirm the badge p50/p95
   budgets still hold, and record the numbers and host load in your bead notes. Do not
   regenerate goldens in this phase.

## Phase: goldens

Runs after `fidelity` lands.

1. Run
   `just fix-tui-screenshots --check -- tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_views.py`.
   The block-header background drift in `agents_deck_view_fixed_page_cards_120x40` and
   `agents_deck_view_split_narrow_120x40` must be gone. If it remains, fix it in code
   (reopen scope with a note) rather than blessing it.
2. Regenerate with the targeted update form,
   `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_views.py`,
   through `/sase_monitor`, since the update form needs a monitor.
3. Inspect every updated golden in the retained report. Accept only these differences:
   the footer `(/) blocks` hint, the rail count row gaining `final N`, and any pixel
   change the `fidelity` phase intentionally made. Deck badges, rails, block rails, and
   bodies must otherwise be unchanged. Check `fixed_page_cards` and `split_narrow` at
   the AGENT header rows: the fixed panel and the automatic panel must match.
4. Rerun the `--check` form until it is clean, then run `sase tool run check`. Record
   what changed per golden in your phase bead notes.
