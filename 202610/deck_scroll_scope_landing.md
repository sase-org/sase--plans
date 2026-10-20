---
tier: tale
title: Finish deck scroll scope and close sase-1n7
goal:
  Spread titles report viewport truth, zoom keeps paged continuation counts, and
  sase-1n7 closes with verified code and its plan marked done.
size: medium
proposed_by: bbugyi200.athena.sase-1n7.land
proposed_with: codex/gpt-6-astra
bead: sase-1n7
create_time: 2026-10-20 08:01:23
status: wip
---

- **PARENT:**
  [202610/deck_scroll_scope.md](https://github.com/sase-org/sase--plans/blob/main/202610/deck_scroll_scope.md)
- **BEAD:**
  [sase-1n7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1n7/README.md)

# Finish deck scroll scope and close sase-1n7

## Outcome and scope

Finish two presentation defects in epic `sase-1n7`, verify the repaired chrome, and
close the epic in this coding turn. This is one bounded implementation task, not a new
phased epic. Do not wait for this turn's commit SHA, push, or CI result: the host
commits after the turn ends, and closing the epic with the final code still dirty is the
normal landing.

The original approved plan is `plan:202610/deck_scroll_scope.md`. Read it through
`sase artifact read` and read `sase-1n7` and its landing notes before coding. Its
decisions remain final: Shift+B toggles the lock, P retains its widening behavior, and
the authorized Agent Data Card glossary change is already complete. No memory changes
are needed. All repairs are presentation-only Python/Textual behavior.

## Evidence already established

The lander reviewed all four closed phases and every note, the approved plan, the actual
implementation, and commits `9ece19c4af`, `798e786c88`, `de1d960d11`, and `3dbe570462`.
The reviewed tip was `e77280c873`; a fetch showed the same origin/master tip. Review any
later changes before editing and again before closing.

The lander's optional focused `just test` audit was stopped during setup after it waited
on the shared Rust wheel-build lock; pytest never started, so there is no new
focused-suite pass to inherit. Both defects below were reproduced directly with the
installed Python environment. Complete the verification in step 3.

The original tabs/chip grammar, effective-versus-shown distinction, partial-paint
holding, lock action/config/palette/footer/help wiring, cached scroll/layout hooks, and
documentation are present. Two seams remain:

1. **False all-in-view title.** In `widgets/decks/title_scope.py`, both `_cue_body` and
   `spread_title_candidates` use `cue.all_in_view or not (cue.above or cue.below)`.
   Empty direction lists do not prove that the active card is fully visible. A direct
   executable reproduction on the reviewed tip is:

   ```python
   from sase.ace.tui.widgets.decks.model import DeckId
   from sase.ace.tui.widgets.decks.scope_cues import (
       resolve_viewport_spread,
       viewport_spread_cue,
   )
   from sase.ace.tui.widgets.decks.titles import CardTab, deck_title

   ordered = ("context", "reply")
   above, below, all_in_view, ready = resolve_viewport_spread(
       ordered,
       "context",
       (("context", 0), ("reply", 80)),
       scroll_y=70,
       viewport_height=30,
       max_scroll_y=70,
   )
   assert (above, below, all_in_view, ready) == ((), (), False, True)
   cue = viewport_spread_cue(
       ordered,
       {"context": "Context", "reply": "Reply"},
       "context",
       above,
       below,
       all_in_view=all_in_view,
   )
   title = deck_title(
       DeckId.MAIN,
       (CardTab("context", "Context"), CardTab("reply", "Reply")),
       0,
       width=100,
       accent="red",
       focused=True,
       paged=False,
       spread_cue=cue,
   )
   print(title.plain)  # Currently: "◆ MAIN both cards in view"
   ```

   Context occupies rows 0..80 and Reply 80..100. Only rows 70..100 are visible. The
   title must retain the here-card identity without claiming the whole deck is visible.
   Do not invent an off-screen-card arrow for a fully visible later card.

2. **Zoom drops the paged continuation.** In `widgets/decks/titles.py`, `deck_subtitle`
   falls back to a restore-only candidate when complete counts do not fit. Build a cue
   with
   `continuation_texts(DeckId.MAIN, tab_count=2, block_count=4, block_paged=True, card_key="Ctrl+J", block_prev_key="(", block_next_key=")", accent="red")`
   and render with `status=None`, empty availability,
   `ZoomChrome(DeckLayout.SINGLE, 0, 1, "Z")`, and width 25. The current result is just
   `Z restore`; the `+3 blocks · +1 card` counts disappear. The approved design says
   zoom keeps the continuation after the restore hint and counts are the last thing
   truncated.

Later commit `3217324d9a` split the deck-view PNG module. All 11 tests, assertions, and
golden names survived; their changes are moved/renamed helper calls. Put new Main
captures in `tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_views_main.py`,
using the existing shared helpers. Files captures now live in the sibling `_files.py`
module. The other overlapping commit, `25d14fae6c`, only changed monitor-cleanup
comments in `default_config.yml`; it did not affect the keymap. Other intervening
commits were release/CI/monitor changes with no deck integration needed.

## Work

1. **Make viewport truth authoritative in the spread ladder.**

   Read the ACE, TUI performance, and screenshot memory guidance before editing. Fix
   both all-in-view shortcuts in `title_scope.py`. Only the explicit proven
   `all_in_view` state should select that wording. When it is false and neither
   direction has an off-screen card, retain the here-name as width permits, then degrade
   through the existing chip/name ladder without introducing tabs or false arrows.
   Preserve the real no-scroll-range and reserve-excluded all-in-view cases, precise
   Files nouns, focus styles, and ZOOM priority.

   Add regression coverage that joins the viewport resolver to the title builder for the
   reproduction above, tests wide and narrow rungs, and confirms a truly fitting deck
   still gets the long/compact all-in-view phrase. Add a controlled mounted case if
   necessary to prove the same condition through the scroll spy. Preserve state-key
   gating: no I/O, measurements, or chrome refresh on every scroll tick.

2. **Keep continuation counts when zoomed width shrinks.**

   Repair the zoomed subtitle candidate/fallback path in `titles.py`. Preserve the
   restore hint and the continuation after it; drop switcher counts, the switcher, cue
   key segments, and optional zoom position before truncating the remaining counts to
   the available cell budget. A restore-only result is appropriate only when no
   continuation cell can fit. At width 25 in the reproduction, a visible continuation
   prefix must remain after `Z restore`.

   Preserve Rich styles when truncating, including bold accent at bottom and dim
   unfocused cues, and respect cell width rather than Python character count. Cover
   single and split zoom, bound/unbound navigation keys, cards and block counts, and
   widths around the fallback boundary. Leave non-zoomed behavior intact unless a shared
   helper is needed to preserve the same established contract.

3. **Verify the repaired output and current integration.**

   Run focused tests for `test_scope_cues.py`, `test_deck_titles.py`,
   `test_deck_chrome.py`, `test_deck_view_keys.py`, `test_deck_view_policy.py`, and
   `test_deck_view_main_pilot.py` under `tests/ace/tui/widgets/decks/`. Include the
   added regressions. Capture representative PNG cases for the partially visible first
   spread card and narrow zoomed paged continuation using the existing visual harness;
   run targeted `just fix-tui-screenshots -- <selectors>` and inspect every
   creation/update in its report. Use `/sase_monitor` for long commands.

   Reconfirm the original acceptance cases: spread Reply names Context above; paged Main
   signals other pages; a page-cards scrollbar does not imply Context is below;
   automatic panels have no lock while P/B locks; one-card decks have no tabs/chip.
   Reuse existing goldens plus the new cases rather than expanding into a full
   visual-suite cleanup.

   Run `just fix`, inspect its changes, then file-change verification through
   `sase tool run check` (the wrapped `just check`). Do not run `just check-full`. A
   known failure of check-full-only infrastructure is not remaining epic work. Any newly
   uncovered issue caused by this epic must be resolved before closing.

4. **Complete the original epic's closeout in this turn.**

   Re-read `sase-1n7`, verify all four phase beads remain done, review any new notes and
   post-review commits, and confirm the two repairs above. Follow-up triage is already
   finished; carry these exact outcomes into the epic close note:
   - `sase-1n7.1` note 1: declined a new manifest task because both named entries
     (`test_github_actions_install_smoke.py` and
     `test_github_actions_testpypi_drill.py`) are present, added by `110214f51c`. This
     is not a blanket claim that unrelated release tests or the whole manifest have
     passed.
   - `sase-1n7.1` note 2: corroborated the exact loaded bounded-drain failure on
     existing task `sase-xo` with +1; no new task.
   - `sase-1n7.2` note 1: corroborated the pre-existing sticky Reply timeout on
     `sase-1ii` with +1 and recorded a DISCOVERED ISSUE on active epic `sase-1jz`, whose
     phase `.3` explicitly owns that node; no new task.

   Run `sase bead epic-symbols sase-1n7`. Resolve every entry or re-key it only to a
   still-open later bead that actually needs it; the lander's check found none, but
   recheck now. Close normally:

   ```bash
   sase bead close sase-1n7 --note "<source/decision and later-commit review; repaired viewport/zoom behavior; actual passing verification and inspected captures; all three follow-up outcomes>"
   just symvision
   ```

   Never use `--force` merely to make this close succeed, and never use it to advance a
   successful nested landing. If named phases or symbols block closing, finish the
   required cleanup or deliberately handle a genuinely canceled scope.

   Open the plans repository through `/sase_repo`
   (`sase repo open plans -r "Mark the completed deck scroll scope epic plan done"`).
   Use the printed checkout, read its AGENTS.md if present, and set `status: done` in
   the YAML frontmatter of `202610/deck_scroll_scope.md`, the plan behind
   `plan:202610/deck_scroll_scope.md`. Read artifact contents through
   `sase artifact read`; do not bypass audited sidecar reads. Include that opened
   repository in the host finalizer declaration.

   After the close, run `sase bead read sase-1n7 -r "Need the parent link"`. The landing
   audit found no parent bead, so no ancestor close is currently required. If a parent
   link has since appeared, follow the original landing rules: a parent phase closes
   only after its scope is verified, leaving its epic to its lander; a directly parented
   plan requires full descendant/note/linked-plan and drift review, its own symbol
   cleanup, normal close, symvision, and plan status update. Stop at an incomplete or
   ambiguous parent and record its blocker instead of forcing it.

   Submit `/sase_final` only after all commands settle. Do not postpone this epic's
   close to a nonexistent later lander or to the host commit.
