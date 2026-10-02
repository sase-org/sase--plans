---
tier: tale
title: 'Finish and land pager version clarity (sase-1ef): epic-caused fixes and closeout'
goal: 'Every pager version-clarity surface tells one consistent, correct story: the
  diff body, pill, band, footer, trail, and picker agree, survive syntax highlighting,
  theme changes, navigation, and narrow widths, and the PNG goldens are regenerated
  and stable. Then epic sase-1ef is closed and its plan file marked done.'
size: medium
proposed_by: bbugyi200.athena.sase-1ef.land
bead: sase-1ef
status: done
---

- **PARENT:**
  [202610/pager_version_clarity.md](https://github.com/sase-org/sase--plans/blob/main/202610/pager_version_clarity.md)
- **BEAD:**
  [sase-1ef](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ef/README.md)

# Plan: Finish and land the pager version-clarity epic (sase-1ef)

## Context

Epic `sase-1ef` ("Pager version clarity", plan `plan:202610/pager_version_clarity.md`)
has all five phases closed. Its commits are:

| Commit       | Phase    | What it added                                                                   |
| ------------ | -------- | ------------------------------------------------------------------------------- |
| `45ece4f1d0` | identity | `VersionMoment`, absolute numbering, D3, D9 attachment                          |
| `e0bdb334e9` | badge    | pill, rail, footer verbs, trail suffix, help legend, `HistoryStyles`            |
| `cf0b8e02d9` | band     | playhead scrubber band, diff endpoints, tombstone chrome                        |
| `a1fc3fc8e8` | picker   | aligned picker (the commit title says "privatize …", but it is the picker work) |
| `b5b43de667` | polish   | docs                                                                            |

The land agent verified that work against the plan and against the concurrently landed
split-pane epic `sase-1eg`. Most of it is correct. These parts check out:

- `moment.py` step rules, D3, the boundary notices, and caching
- stepping via `step_target`
- D9 attachment and the provider fallback
- per-pane history state and moments in split view
- the footer following focus
- the `_screen_history_*` and `pager_provider_*` refactors, which preserved the epic's
  logic

The land agent also found the epic-caused defects listed below, plus 17 PNG goldens that
drift on clean master. This tale fixes all of them and then closes the epic.

Read the `tui`, `tui_perf`, and `lint_and_test` memories (via `/sase_memory_read`)
before changing pager code. Their rules apply:

- no IO on render or keystroke paths
- workers via `spawn_pump_free_task` with generation guards
- fail open to today's rendering when history is absent

### Coordination: do not touch the sase-1eg bugs

`sase-1eg.land` is fixing three split-pane bugs as `sase-1eg` work. Do NOT change:

- the history-generation bump in `_resolve_and_dispatch`
  (`src/sase/pager/_screen_actions.py`)
- `_handle_label_key` / `_activate_time_band_label` / `_activate_time_band_commit` in
  `src/sase/pager/_screen_time_band.py`
- `PagerView._build_label_layer` in `src/sase/pager/view.py`

`sase-1eg.5` is also re-landing split-pane docs into `docs/pager.md` (Keys rows plus a
"Split panes" section), and may be adding split tests in `tests/pager/test_app_split.py`
and `tests/pager/test_app_other_pane.py`. Rebase onto current master first, keep their
changes intact, and put any new history-in-split tests in a new file (see §8).

### Already triaged — do not re-file

Every `PROPOSED FOLLOW-UP:` on the phase beads, and the split-pane bugs, were already
triaged in a `sase-1ef` note:

- tasks `sase-1em`, `sase-1en`, and `sase-1eo` were filed
- the split-pane bugs were recorded on `sase-1eg`
- the remaining proposals were declined or folded into this tale

Do not create beads for them again.

## 1. One source for diff endpoints (D1/D7)

**The bug.** The diff body and the label compute endpoints differently.

- `PagerDiffMixin._diff_endpoints_for` (`src/sase/pager/_screen_diff.py`, around
  line 65) and `_history_diff_base_for` (`src/sase/pager/_screen_chrome.py`, around
  line 120) call the old `sase.pager.history.diff.diff_endpoints`. It uses
  `(ordinal - 1, ordinal)` and `max(visible_ordinals)`.
- The pill, band, and trail use `VersionMoment.diff` (`_diff_endpoints` in
  `src/sase/pager/history/moment.py`). That skips hidden versions and honours D3.
- So with v4 hidden and v5 open in the diff view, the label says `Δ v3 → v5` but the
  body diffs v4→v5.
- With a hidden newest version on a clean now, the label says v4→v5 but the body diffs
  v3→v4.

**The fix.**

- Make `moment.diff` the only source: the body diff load, the change marks, the `yy`
  unified-diff copy, and the chrome fallback all read it. Keep the moment's
  hidden-skipping semantics: the base is the newest steppable version below the target,
  and explicit picker bases are normalized older→newer.
- Use the old function only on the fail-open path where no moment can be built. If
  nothing else uses it, delete `diff_endpoints` and its tests.
- Add tests: a hidden middle version and a hidden newest version, asserting that the
  body's loaded endpoints equal `moment.diff`.

## 2. D3 arrival canonicalization swaps in the live section

**The bug.** In `src/sase/pager/_screen_history_discovery.py` (around lines 168-191), an
arrival pin to vN on a now ≡ vN subject only rewrites `state.current_pin`. The document
section stays the derived vN section, whose `version_pin` still names vN. As a result:

- `yy` copies `sha:path` (`_screen_actions.py` around 159/176)
- links resolve against vN's commit
- the trail records a vN pin

**The fix.**

- Perform the same refresh-style live swap that `_load_and_swap_version`
  (`src/sase/pager/_screen_history_swap.py`) uses for a canonicalized pin.
- Push no trail entry. Keep `view`, `compare_base`, and `explicit_base`.
- Test that an `-A vN`-style arrival on a clean D3 subject leaves a live section: no vN
  `version_pin`, a live-path `yy`, and live trail pins.

## 3. Moment edge case: staged-only dirt

`build_moment` derives `now_matches_newest` purely from the three OIDs.

**The bug.** With staged-only edits whose worktree equals HEAD, the status is dirty, so
the pill reads `◌ NOW`, yet `(` skips vN as if now ≡ vN.

**The fix.** Require a clean status for `now_matches_newest`. A dirty now must step
`older` onto the newest visible version (HEAD), per plan §4. Add a moment test.

## 4. Chrome: trail, rail, theme, blend, shedding, footer

1. **Trail crumbs use their own version (§5.5).**
   - **The bug.** `build_pager_trail_snapshot` (`src/sase/pager/_trail_chrome.py` around
     46-66) looks up back/forward suffixes from `_trail_version_info`
     (`src/sase/pager/_screen_trail.py` around 212-246). Those are the current
     document's live pins. `PagerTrailEntry.version_pins` is never read. So after a
     picker jump v24→v21 the back crumb shows `@v21`, and back crumbs for other
     documents show no suffix.
   - **The fix.** Derive each history entry's suffix from
     `dict(entry.version_pins).get(entry.section_identity)`.
   - Make `_current_view_state` record the live state's `current_pin` (falling back to
     `section.version_pin`), so diff-view and canonicalized pins round-trip.
   - Drop the then-unused pin plumbing.
   - Test with a unit test plus an app pilot.
2. **Syntax highlighting erases the rail.**
   - **The bug.** `_publish_syntax_update` (`src/sase/pager/_screen_syntax.py` around
     335-358) calls `compose_body` without `rail_styles` or `history_styles`. Every
     markdown note loses its past rail when highlighting lands, and change marks lose
     their theme colours.
   - **The fix.** Reuse `_compose_body_at_width` (`src/sase/pager/_screen_body.py`).
   - Add a regression test: pin a markdown note to the past, let syntax publish, and
     assert the rail style remains.
3. **Theme change repaints everything history-coloured.**
   - **The bug.** `_on_app_theme_changed` (`_screen_syntax.py` around 129-142) refreshes
     only the trail and band.
   - **The fix.** Also invalidate `HistoryStyles` if that is not already done,
     `_update_subject()`, `_update_footer()`, and recompose the body, so the pill, rail,
     and change marks switch theme.
   - Test with a theme switch.
4. **Blend direction is inverted** (`src/sase/pager/history/styles.py`).
   - **The bug.** `_blend_hex(first, second, f)` moves `first` toward `second` (Textual
     `Color.blend`).
     - `now_pill_bg = _blend_hex(foreground, background, 0.18)` is therefore 82%
       foreground (`#B7B7B7` on textual-dark): loud, not the quiet neutral pill of D4.
     - `band_past_tint` starts at 86% past accent (about `#614C85` on dark, and nearly
       invisible on solarized-light), not the faint tint of D5.
   - **The fix.**
     - Use `_blend_hex(background, foreground, 0.18)` for the now pill.
     - Start the tint at `_blend_hex(background, past, 0.14)` and fade it further toward
       the surface only while body-text contrast is below 4.5:1.
   - Extend `tests/pager/test_history_styles.py` to pin the proportions on every
     built-in theme: the tint is closer to the background than to the accent, and the
     now pill bg is closer to the background than to the foreground. Keep the existing
     contrast and hue rules.
5. **Subject-line shedding must follow §5.2 and be monotonic.**
   - **The bug.** `subject_line` (`src/sase/pager/_chrome.py` around 270-330) lets the
     pill take a shorter form at every stage. The pill shortens while the syntax hint,
     `⌘` count, and context are still shown, and it flips between forms as width
     shrinks:
     - 71 cols: `⟲ PAST · v24/25` with `· md`
     - 61 and 60 cols: `⟲ v24` with `· md`
     - 72, 69, and 57 cols: the full pill again
     - the 60×30 dirty golden: `◌ NOW` while `on top of v3` is still shown
   - **The fix.** Keep the pill at its full form while shedding, in order: the hint, the
     count, then the context (the `Δ` segment kept longest). Only then step through the
     pill's shorter forms, and finally middle-truncate the title. The pill is never
     cropped.
   - Add a test sweeping widths 200→30 for each kind and view. As width decreases, no
     element may reappear and the pill form may only get shorter.
6. **Footer verbs** (`time_verbs_for_moment` and `footer_legend` in `_chrome.py`):
   - **Tombstone verbs are inverted.**
     - On the deletion itself, `}` goes nowhere, yet `} deleted` is shown.
     - In the past on a deleted subject, `} now` is shown although `}` goes to the
       deletion.
     - Fix: show `} deleted` when `to_now` is the tombstone ordinal (`to_now > 0`) and
       differs from `newer` and the current position. Show no `}` on the deletion
       itself.
   - **Order.** Follow D6/§5.4: `( vK · ) vK|now · } … · = diff|read · @ timeline`, with
     `=` before `@`.
   - **Remove the unreachable legacy branch.** Delete the `history_available` /
     `history_diff_view` branch (`( ) version`, `E edits now`) and those parameters from
     `footer_legend` and its call site in `_screen_chrome.py`. Keep a single `E` verb.
     Update the tests that pass those parameters.
7. **Fail-open chip numbering.** When no moment exists, `_history_chrome_state`
   (`_screen_chrome.py` around 76) uses `total = len(state.visible_ordinals)`, which
   brings back the `v24/21` bug. Use the newest committed ordinal, counting hidden
   versions.

## 5. Time band

1. **Dirty worktree in the past (§6.3).**
   - **The bug.** `_moment_readings` (`src/sase/pager/_time_band_model.py` around 298)
     sets `dirty = kind == "now_dirty"`. So past and diff views over a dirty worktree
     never show `+ uncommitted` after `N newer`, nor the amber `◌ now` endpoint.
   - **The fix.** Add a pure worktree-dirty flag to `VersionMoment`, set in
     `build_moment` from the status and rows, and have the band read it.
2. **Diff text must be reachable.**
   - **The bug.** In the diff view, a now (dirty or clean) builds the band in now mode
     (the one-row strip). So `comparing v25 <date> → uncommitted edits`
     (`_time_band_render.py` around 557-580) is never rendered.
   - **The fix.** In the diff view, always render the timeline row with the comparing
     text and the range-coloured scrubber, whatever the kind.
3. **Now-strip shedding (§6.3).**
   - **The bug.** At 60 cols the dirty now strip drops the `v1` / `◌ now` endpoint
     labels but keeps `last changed …` and `◌ edits not durable …`. That leaves
     unlabelled full-height bars at the left edge (proposed follow-up sase-1ef.2#2).
   - **The fix.** Endpoint labels shed last: drop the edits-not-durable notice, then the
     last-changed detail, before the labels.
4. **Tombstone row** (`_time_band_render.py` around 745): it must name the version whose
   content the body shows, i.e. the last version before the deletion. `✖ DELETED · v12`
   → `showing last content (v11)`. Update its test.
5. **Tint only in the past.** `_screen_time_band.py` (around 323) applies
   `band_past_tint` to deleted subjects too. Per §6.5, only `past` is tinted. Deleted
   uses `$surface` and keeps the muted deleted rail.

## 6. Timeline picker

1. **The `≡ now` row.**
   - **The bug.** With now open and the cursor on the vN row tagged `≡ now`, the footer
     reads `⏎ open v25 · = compare v25 → now`. Yet ⏎ is a no-op and the compare is
     byte-identical.
   - **The fix.** Canonicalize an `is_now_alias` row as the open now row in
     `picker_footer_preview` and `_normalize_picker_compare`
     (`src/sase/memory/history/timeline_picker.py`). The footer shows the open-row form,
     and `=` does nothing.
2. **The now row's glyph.** The now pseudo-row uses glyph `●`, so an open now row
   renders `● now  ●  ≡ v25…`. Give the now row a blank glyph, or `◌` when dirty, so `●`
   means only "open".
3. **Remove the `display` string.** Remove the dead pre-joined `display` field from
   `build_picker_rows`; only tests read it. Move those tests to the structured cells.
4. **Hard-coded colour.** `_CURRENT_STYLE = "#9d7cd8"` in
   `src/sase/pager/_timeline_picker.py` must come from `HistoryStyles`. A literal is
   allowed only as the no-styles fallback.
5. **Golden fixture.** The `timeline_picker_*` golden fixture has no history state, so
   its header shows a row count (`4 versions` for a timeline whose newest is v9) and no
   pill. Make the fixture use the moment path so the goldens show real behaviour: honest
   `9 versions · 1 hidden`, the open version's pill, and the theme open-label colour.

## 7. Attachment, colours, and docs

1. **History attaches after navigation.** This is the epic goal: "History attaches to
   every memory section no matter how it was opened". The sase-1eg land agent agreed it
   stays here.
   - **The bug.** `_navigate_to_document` (`src/sase/pager/_screen_actions.py` around
     603-644) and `_restore_view_state` (`_screen_trail.py` around 65-100) never call
     `_start_history_discovery_after_paint()`. Following a link to another memory note,
     in place or via ctrl+w into an already-split other pane, leaves the band at
     `indexing…` with no pill until a time key is pressed.
   - **The fix.** Start discovery after navigating and after restoring. Avoid reloading
     sections already present in `_history_states`.
   - Add pilot tests for an in-place follow and a back-restore.
2. **Provider fallback.** In `_discover_entry_point_factories`
   (`src/sase/pager/history/provider.py` around 95-120), set `builtin_seen` only after
   the built-in entry point loads. A failed load must still reach the string fallback.
   Add a test.
3. **Remaining hard-coded colours (D8).**
   - In `src/sase/pager/history/diff.py`: `REMOVAL_STYLE = "red"` and
     `FRONTMATTER_PROMOTE_STYLE = "magenta"`.
   - In `src/sase/pager/_time_band_render.py`: the direct `UNCOMMITTED_STYLE` uses for
     UNTRACKED, IGNORED, `◌ edits not durable`, and the diverged chip.
   - Route all of them through `HistoryStyles` roles. Keep the `_time_band_vocab.py`
     constants only as the fail-open fallback. If they become unused, remove them and
     their `_time_band.py` re-exports.
4. **Docs** (`docs/memory_history.md`; edit `docs/pager.md` only in history lines,
   preserving sase-1eg's split-pane rows):
   - Fix the "Time band anatomy" mockup:
     - the timeline row comes first and the meaning row sits below it
     - the scrubber has `v1` … `● now` ends
     - no `· v24 ·` text
     - add the past tint note
   - Make both footer examples follow D6:
     `( v21 · ) now · = diff · @ timeline · E edit now`. `} now` appears only when it
     differs from `)`.
   - Mention the picker's `● open` / `▸ cursor` markers and the `≡ now` row briefly in
     "Which version am I reading?".

## 8. Tests, goldens, and verification

- **Tests.** Add or extend unit and pilot tests for every item above.
  - Put a split-view history pilot in a NEW file,
    `tests/pager/test_app_history_split.py`. Two panes on different versions must each
    show their own pill, rail, and band tint, and the footer time verbs must follow a
    focus switch (ctrl+f).
  - Run `tests/pager`, `tests/memory/test_history_pager_provider.py`, and
    `tests/memory/test_timeline_picker.py` directly while iterating.
- **Goldens.** Clean master already drifts on 17 pager goldens:
  - `history_dirty_{dark,light}_60x30`
  - `timeband_{past,cause,diff-range,many-versions}_*`

  For example, `timeband_past_dark_120x40` now renders a chrome rule under the band and
  starts the body one row lower, and `history_dirty_*_60x30` truncates
  `edits not durable …` differently.
  1. Before accepting anything, root-cause whether each drift is a deterministic result
     of landed code (regenerate it) or nondeterminism (fix the fixture, for example by
     pinning the clock).
  2. Then run `just fix-tui-screenshots -- tests/pager/visual`.
  3. Inspect every created or updated PNG in the report, in both themes and at both
     sizes. Look especially at past-frame (now a faint tint), dirty, tombstone,
     diff-range, and the picker.
  4. Confirm that no split-pane or other unrelated golden changed.
  5. Re-run `just test-visual -- tests/pager/visual` twice. Both runs must be clean.

- **Lint and check.**
  - Run `just fix` (or at least `just fmt`).
  - Then run `sase tool run check` and get it passing. Treat UNKNOWN items as yours.
  - Do NOT run `just check-full`. A `just check` pass with a `check-full` failure is a
    test-infrastructure bug: file it with `/sase_new_task`.

## 9. Closeout of epic sase-1ef (final step, same turn)

Close the epic in the same turn as the final code. Do not wait for this tale's own
commit, push, or CI.

1. Run `sase bead epic-symbols sase-1ef`. For every listed `--epic-symbol` entry (there
   were none at planning time), do one of the following:
   - resolve the symbol: wire it up, privatize it, add a non-test pragma, or delete it
     per the `symvision` memory
   - only if a still-open later bead needs it, re-key the Justfile line to that bead
2. Close the epic:

   ```bash
   sase bead close sase-1ef --note "<verification summary>"
   ```

   Summarize three things in the note:
   - the five phases' work verified against the plan
   - each fix from this tale with its tests
   - the goldens regenerated and inspected, and the `sase tool run check` verdict

   Never use `--force` to make the close succeed. If the close is rejected for leftover
   epic symbols, finish that cleanup and close again.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   printed by `sase bead read sase-1ef -r "Need the plan path for closeout"` (plan
   `plan:202610/pager_version_clarity.md`, currently `status: wip`).
5. `sase-1ef` has no `parent_bead`, so nothing further is needed after the close.
