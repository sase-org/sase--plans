---
tier: tale
size: medium
title: Finish and land the Agents node rail epic (sase-1bn)
goal: The last epic-caused node-rail and zoom gaps are fixed and tested, and epic
  sase-1bn is closed with a clean symvision whitelist and its plan marked done.
proposed_by: bbugyi200.apollo.sase-1bn.land
bead: sase-1bn
status: done
---

- **PARENT:**
  [202609/agents_node_rail_and_zoom.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_node_rail_and_zoom.md)
- **BEAD:**
  [sase-1bn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1bn/README.md)

# Plan: Finish and land the Agents node rail epic (sase-1bn)

## Context

Epic `sase-1bn` ("Agents node rail and unmistakable deck zoom", plan
`plan:202609/agents_node_rail_and_zoom.md`) has all seven phases closed. `Ctrl+S` now
turns the Agents-tab node column into a 9-cell row-for-row rail, and `Z` zoom hides the
column and adds structural chrome. The epic commits are 55e4bf73b, 42bb50a80, 74d1ab8e1,
f4cfc51d7, 0a24cf802, 2f03e5059, and 52f7351ae.

The land agent checked every plan step against the code on master 52f7351ae. It found a
short list of remaining **epic-caused** gaps, listed below. This tale fixes them and
then closes the epic. Everything is presentation-only ACE TUI work. Nothing crosses the
`sase-core` boundary and no keymap defaults change.

The land agent has already triaged the follow-ups. That triage is recorded on `sase-1bn`
(note "LANDING TRIAGE"). Do **not** re-triage them or create more beads for them:

- flake task `sase-1bz` (the intermittent first `Ctrl+S`);
- memory task `sase-1bx` (the glossary "Node Panel" strand; do **not** edit memory
  files);
- ci task `sase-1by` (the marker-path audit);
- +1s on `sase-1b8` and `sase-1bq`, and a DISCOVERED ISSUE note on `sase-1bc`.

The failures those beads cover are pre-existing and not caused by this epic. They fail
identically on the pre-epic base b9cfa7386, so they are expected to stay red in
`sase tool run check`:

- `test_agent_completion.py`
- `test_directive_completion_candidates.py`
- `test_axe_run_agent_exec_repeat_env.py`
- `test_agent_artifact_marker_path_passing_audit.py`
- `test_expanded_overflowing_header_claims_half_page_scroll`
- `test_open_action_opens_overlay_on_stash_with_trash_count`
- `tests/completion/test_snapshot.py` and `test_kind_coverage.py`

## Remaining work

### 1. Broken test caused by the rail mixin

`tests/ace/tui/test_agent_list_watch_highlighted.py::test_watch_highlighted_delegates_when_user_navigation`
fails with
`AttributeError: type object 'AgentListRailMixin' has no attribute 'watch_highlighted'`.
It patches `widget.__class__.__mro__[1]`, and `AgentListRailMixin` now sits there
(`class AgentList(AgentListRailMixin, AgentListBase)`).

Fix: patch the next class that `super()` resolves to. Pick the first class in
`__mro__[1:]` whose `__dict__` contains `watch_highlighted` (that is Textual's
`OptionList`). Add a one-line comment explaining why.

### 2. Expanded banner width with 2-character hints (banner-glyphs phase)

In `src/sase/ace/tui/widgets/_agent_list_render_banner.py` (~L151), the line
`hint_cells = 4 if hint_char is not None else 0` is wrong for 2-character hints. The
chip `[ab] ` is 5 cells, so the banner overruns its width by one cell.

Fix: compute the chip text once, append it, and use `cell_len(chip_text)` for
`hint_cells`. Do the same for the mark chip.

Add the tests the phase promised to
`tests/ace/tui/widgets/test_agent_list_grouping_buckets.py`:

- for every bucket in `RAIL_BUCKET_GLYPHS` (all seven), plus a wide-character label,
  render the expanded BY_STATUS L0 banner and the BY_MACHINE L1 banner;
- render each with no hint, a 1-character hint, and a 2-character hint;
- assert `cell_len(prompt) == width` exactly (both the chip and no-chip branches).

Also replace the tautological `test_banner_glyph_matches_rail_glyph` in
`tests/ace/tui/widgets/test_agent_list_render_rail.py` (~L443). It currently only checks
`cell_len == 1`. The new test must render a real expanded banner for each of the seven
buckets and assert that the rendered glyph and its style equal
`RAIL_BUCKET_GLYPHS[bucket]`.

### 3. The rail "Stopped" banner paints a solid amber bar (rail-vocabulary phase)

`rail_banner_cells` in `src/sase/ace/tui/widgets/_agent_list_render_rail.py` (~L390-453)
paints the rule, the fold mark, and the count in `lead_style`. For the BY_STATUS
`Stopped` banner, `lead_style` is the reverse chip `RAIL_NEEDS_YOU_STYLE`
(`bold #1a1a1a on #FFAF00`), so the whole 6-cell banner renders as a solid amber block.
You can see it in `agents_node_rail_by_status_120x40.png`. The design says the `?` chip
is the rail's only reverse-video chip.

Fix:

- Keep `lead_style` only on the lead cell.
- Paint the rule, the fold mark, and the count in a foreground-only rule style derived
  from the lead style. When the style has a background, use that background color as the
  foreground. Otherwise use its foreground color. Drop the background and reverse.
- Keep the hint chip as is.

Add unit tests:

- the expanded and folded `Stopped` banners have exactly one cell with a background (the
  `?` lead);
- the rule cells carry `#FFAF00` as their foreground.

Regenerate the goldens this changes (see Verification).

### 4. Expanded-row badge order regression (rail-vocabulary phase)

Commit 55e4bf73b made `append_agent_row_prefix` in
`src/sase/ace/tui/widgets/_agent_list_render_agent_prefix.py` emit `row_kind_glyph(...)`
early for every row. That moved the **top-level type badges** (`≡`, `❑`, … from
`_TYPE_GLYPHS`) ahead of the hidden icon, the `↻N` retry badge, and the machine chip.
The plan never asked for that. For example, a hidden retry workflow row changed from
`  ↳ ◌ ↻1 ≡ demo` to `  ↳ ≡ ◌ ↻1 demo`.

Fix: restore the pre-epic order while keeping `row_kind_glyph` as the only glyph source.

- Emit the kind glyph early only for tree rows (depth > 0) and for top-level monitor,
  gate, and named-proc rows. Those are the branches that were early before 55e4bf73b.
- Emit the top-level type badge where the old `_TYPE_GLYPHS` badge was, after the
  machine chip.
- Keep the `[X]` fallback for unknown display types.

Add a regression test that renders a hidden retry workflow row (top-level, `hidden`,
`retry_attempt=1`, workflow display type). Assert the badge order `◌ ↻1 ≡` in
`prompt.plain`. Also add a machine-chip case if a fixture makes that cheap. Keep the
existing `row_kind_glyph` parity tests passing.

### 5. Rail title builder must be the only builder (rail-wiring step 4)

`_try_patch_agent_row` in `src/sase/ace/tui/actions/agents/_display_panel_patches.py`
(~L476-494) still has an `else:` fallback that calls `agent_panel_border_title(...)`
directly. That path would paint an expanded title in RAIL mode.

Fix: make `_agent_panel_title` / `_set_agent_panel_title` the only path by deleting the
fallback. Both are always present on the real app via `PanelCollectionMixin`. If a test
harness lacked them, give the harness the real mixin or stubs rather than keeping the
fallback. Drop any import that becomes unused.

### 6. Spacer rows take the rail fallback path (rail-projection phase)

Spacer rows (`Option(Text(""), id="spacer:N", disabled=True)` with a
`(BANNER_ROW, None)` entry and no `_group_at_row` entry) currently fall into the
inconsistent-maps fallback in `AgentListRailMixin._get_visual`
(`src/sase/ace/tui/widgets/_agent_list_rail_mode.py`). Every paint emits a
`widget.agent_list.rail_fallback` trace and builds an uncached blank visual.

Fix: in `_rail_cells_for_option`, recognise spacer options (an id starting with
`spacer:`) and return a blank `Text(" " * RAIL_CONTENT_CELLS)`. The normal path then
caches it and emits no trace. Add a test in
`tests/ace/tui/widgets/test_agent_list_rail_mode.py` that a grouped list with spacers
renders in rail mode without any `rail_fallback` trace event.

### 7. Labels, docstrings, and docs (mode-affordances phase)

- `src/sase/ace/tui/bindings.py:288`: change the `toggle_node_panel` binding description
  from `"Collapse/Expand Node Panel"` to `"Toggle Node Rail"`. Fix any test that pins
  the old text.
- Replace the stale "collapse the node panel" wording in these docstrings with
  rail/hidden wording:
  - `src/sase/ace/tui/widgets/_agent_detail_deck_layout.py` (~L257 "node panel is
    collapsed", ~L285 "Collapse or expand the node panel…");
  - `src/sase/ace/tui/widgets/decks/layout.py` (~L147, ~L162 "hiding the node panel");
  - `src/sase/ace/tui/actions/agents/_deck_layout_actions.py` (~L112,
    `action_toggle_node_panel`).
- Rename `test_zoom_from_single_keeps_panel_and_collapses` in
  `tests/ace/tui/widgets/decks/test_deck_collapse_zoom.py` (~L68). Zoom no longer
  collapses anything.
- `docs/ace.md`, section "Agents Zoom and Node Rail" (~L5089):
  - L5112-5113: drop "collapse" from the snapshot list and "no spine" from the zoom
    description;
  - L5592: say "node-rail preference" instead of "node-panel collapse";
  - add the missing rail anatomy, kept compact:
    - the pip cell (`▪` marked wins over gold `•` unread);
    - container member counts (capped at 99, tinted clan/session);
    - the `│`/`└` tree guides (depth clamped at 2);
    - the folded-banner layout `▸`, lead, rule, hidden count, and urgency roll-up (`?` >
      red `✗` > gold `•`).
  - Keep the Markdown formatting that `just fmt` expects.

### 8. Missing tests promised by sidebar-modes and rail-wiring

- In `test_sidebar_mode_table` (`tests/ace/tui/widgets/decks/test_deck_collapse_zoom.py`
  ~L108), add two rows:
  - RAIL + split key → RAIL, split applied;
  - HIDDEN zoomed from RAIL + split key → zoom ends without a snapshot restore and the
    sidebar returns to RAIL (the preference).
- Add a pilot test in `tests/ace/tui/test_agents_node_rail_wiring.py`, which mounts the
  real app layout. From EXPANDED:
  1. `Z` zooms, so `#agents-content` has `-nodes-hidden` and `#agent-list-container` is
     not displayed.
  2. A split key (`backslash`) ends the zoom.
  3. Assert that `-nodes-hidden` is gone and the container is displayed again with a
     non-zero width.
- Strengthen `test_rail_preserves_row_positions_and_scroll` (~L126). It currently checks
  only option ids and a scroll offset of 0. Use a terminal size small enough that the
  sase panel overflows, scroll it to a non-zero `scroll_offset.y`, toggle `Ctrl+S`, and
  assert:
  - the same scroll offset;
  - the same highlighted index;
  - the same y (`region.y` / line index) for the highlighted row before and after.

## Verification

1. Run `just fmt`, then `sase tool run check`, the wrapped `just check`. Do **not** run
   `just check-full`. The only expected red items are the pre-existing, already-beaded
   failures listed in Context. Anything else is yours.
2. Regenerate the affected PNG goldens with a targeted update. It can outrun a single
   turn, so run it through `/sase_monitor` when needed:

   ```bash
   just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_node_rail.py tests/ace/tui/visual/test_ace_png_snapshots_agents_decks.py tests/ace/tui/visual/test_ace_png_snapshots_agents_retry.py tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py
   ```

   Also include any BY*STATUS or BY_MACHINE banner goldens the run reports, such as the
   `\*_by_status*\*` goldens. Inspect every updated golden. Expected changes:
   - `agents_node_rail_by_status_120x40` loses the solid amber bar: only the `?` cell
     stays reverse and the rule is amber foreground;
   - retry goldens change only if a top-level type badge row also carries a hidden,
     retry, or machine badge.

   Anything else changing is a bug to investigate, not to approve. Treat a `partial`
   status as not current.

3. Run `just symvision` and confirm it is clean. Its `_setup` step may first rebuild the
   Rust extension (the sase-core pin moved recently), which can take more than 10
   minutes, so run `just install` beforehand or route it through `/sase_monitor`.

## Final step: close out epic sase-1bn

1. Run `sase bead epic-symbols sase-1bn`. It was empty when this tale was written. If
   entries now exist, resolve each one per the Symvision epic-whitelist policy: wire it
   up, privatize it, add a non-test pragma, or delete it. Re-key a Justfile line only to
   a still-open later bead that genuinely needs the exemption.
2. Close the epic:

   ```bash
   sase bead close sase-1bn --note "<verification>"
   ```

   The note must summarise:
   - the land agent's verification (all 7 phases checked against code on master
     52f7351ae, and the triage outcomes recorded in the LANDING TRIAGE note);
   - what this tale fixed (items 1-8 above);
   - the golden updates you inspected;
   - the `sase tool run check` result with its tool-run id.

   Never use `--force` merely to make the close succeed. If the close is rejected over
   unfinished phases or epic symbols, fix the cause.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` (currently `status: wip`) in the frontmatter of the epic's plan
   file `202609/agents_node_rail_and_zoom.md` in the plans sidecar. Use the PLAN path
   printed by `sase bead read sase-1bn -r "Need the plan path for closeout"`.
5. Epic `sase-1bn` has no `parent_bead`, so nothing further needs closing. Re-check with
   `sase bead read`. If a parent has appeared:
   - a phase parent: close it only after verifying that its work is done;
   - a plan parent: follow the standard land-agent parent procedure.
