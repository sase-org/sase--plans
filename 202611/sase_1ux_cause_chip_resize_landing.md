---
tier: tale
size: small
title: Recompute the failed-clan cause-chip drop on resize, then close epic sase-1ux
goal:
  The Failed-clan cause chip drops and returns as soon as the Agents list's visible
  width changes, without waiting for a grouping rebuild. Healthy rows keep their cached
  renders across column-width changes. Epic sase-1ux is closed, its plan file is marked
  done, and Symvision is clean.
proposed_by: bbugyi200.athena.sase-1ux.land
proposed_with: claude/opus
bead: sase-1ux
status: done
---

- **PARENT:**
  [202611/failed_clan_visibility.md](https://github.com/sase-org/sase--plans/blob/main/202611/failed_clan_visibility.md)
- **BEAD:**
  [sase-1ux](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ux/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1ux.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1ux.land.md)
- **COMMITS:**
  - [fc397c4](https://github.com/sase-org/sase/commit/fc397c469c9806db05fc396a5ab640bd8d69e16a)
    — feat(tui): recompute failed-clan cause-chip drop on resize

# Plan: recompute the failed-clan cause-chip drop on resize, then close epic sase-1ux

## Context

Epic `sase-1ux` ("Make failed agent sessions and clans unmistakable",
`plan:202611/failed_clan_visibility.md`) has all five phases closed. Its land agent
verified every phase against source and commits: sase-core `1fb9701f`, which is
contained in the `sase-core-revision.txt` pin, plus sase commits `1c4c89f6b4`,
`3dc72ddd4c`, `623c826812`, and `1d4f456f41`. It reviewed the 15 non-epic commits that
landed since the epic started and found no conflicts. It also fixed the one other
epic-caused regression: the `notification-gates.md` wording that tripped the terminology
guard (`sase-1uy`, now closed). Follow-up triage is recorded in the `LAND TRIAGE` note
on `sase-1ux` and is already done; do not refile those follow-ups.

One gap caused by the epic remains: phase `sase-1ux.4` follow-up note #3, "Failed-clan
cause chip drop waits for the next grouping rebuild; first paint and terminal resize do
not recompute the drop." No other agent resumes this landing. This tale must fix the gap
**and** close the epic in the same turn as the code (see the final step).

Read the `tui.md` and `tui_perf.md` memories (`/sase_memory_read`) before changing TUI
code. Rules 2, 6, and 8 apply directly: keep pump callbacks thin and synchronous, patch
rows rather than rebuild, and keep file reads out of render and resize paths.

## How it works today

- `format_agent_option` (`src/sase/ace/tui/widgets/_agent_list_render_agent.py`) drops
  the cause-chip span that `_append_clan_cause_chip`
  (`src/sase/ace/tui/widgets/_agent_list_render_agent_status.py`) returns. It drops the
  span when `left + gap + suffix > width`, using `_drop_plain_span`.
- `width` comes from `widget_viewport_width(widget)`
  (`src/sase/ace/tui/widgets/_agent_list_build_rows.py`): the list's width, capped by
  each ancestor's width. It is sampled at only two points: in `build_row_inputs` during
  a grouping rebuild, stored as each row ctx's `viewport_width`, and in `patch_row`
  (`src/sase/ace/tui/widgets/_agent_list_build_patch_single.py`). Two cases miss it:
  - A build made while the list is unmounted or zero-sized gets width 0, so the chip
    never drops.
  - A terminal resize or split collapse repaints nothing, so the chip stays until the
    next grouping rebuild.
- `agent_render_key` (`src/sase/ace/tui/widgets/_agent_list_render_cache.py`) now keys
  **every** row on `width`, because `_agent_option_key_from_derived` passes it through.
  Only Failed clan rows with a cause chip depend on width. The `#agent-list-container`
  width follows the panels' content-derived `_requested_width`
  (`src/sase/ace/tui/actions/_event_widgets.py`). So whenever the widest row changes
  length, the next build misses the cache for every row.

## Steps

1. **One chip-capability predicate.** Add `row_has_droppable_cause_chip(agent)` next to
   `_append_clan_cause_chip`. It is true for a clan container whose `failure_focus` has
   a `member_label`, which is exactly the condition `_append_clan_cause_chip` checks
   today. Make `_append_clan_cause_chip` use the predicate so the two cannot drift.
2. **Width only keys chip-capable rows.** In `cached_format_agent_option`, compute
   `effective_width = width if row_has_droppable_cause_chip(agent) else 0` once. Pass it
   to both `_agent_option_key_from_derived` and `format_agent_option`. Check every other
   caller that builds the render key with `width`, including the `patch_row` render
   kwargs path, and confirm each goes through this normalization. Rendered output for
   other rows does not change, because they never return a cause span.
3. **Recompute on a width change.** On `AgentList` resize, compare
   `widget_viewport_width(self)` with the width the painted chip-capable rows were last
   rendered at. `AgentListRailMixin.on_resize` in
   `src/sase/ace/tui/widgets/_agent_list_rail_mode.py` already exists; extend it or add
   a sibling hook that keeps the rail behavior. Record the applied width on the row ctx
   (`viewport_width`) during build and patch.
   - If any painted row in `_row_render_ctx` is chip-capable and its recorded width
     differs, schedule one coalesced `call_after_refresh`. It re-patches only those rows
     through `patch_row` and updates each row's ctx `viewport_width`.
   - Coalesce with a scheduled or pending guard, and release the guard if scheduling
     fails.
   - Do nothing when no chip-capable row is painted, so healthy lists do no extra work.
   - The first real sizing after mount is a resize, so it also corrects rows built while
     the width was 0.
   - Follow `patch_row`'s existing width contract (`_max_left` / `_target_width`). If a
     patched row can no longer fit the established column, use whatever that path
     already does; do not add a new rebuild path.
4. **Tests** (widget tests next to
   `tests/ace/tui/widgets/test_agent_list_clan_cause_chip.py`, reusing its
   incident-shaped fixtures):
   - The render key for a non-chip row is identical across widths; a chip-capable Failed
     clan row's key differs.
   - In a mounted `AgentList` (Textual `run_test` plus `pilot.resize_terminal`), the
     chip shows at a wide size, drops after a resize to a narrow size, and returns after
     widening, with no grouping rebuild between. Assert that no full rebuild ran.
   - A list built while unmounted, with width 0, drops the chip after its first sizing
     when the clip is narrow.
   - Resizing a list with no chip-capable rows schedules no patch.
5. **Goldens stay stable.** The existing incident goldens must not change:
   `failed_clan_incident_by_status_collapsed_120x40`,
   `failed_clan_incident_by_status_collapsed_80x40`,
   `failed_session_incident_by_status_expanded_120x40`, and
   `failed_session_incident_by_status_expanded_80x40`, plus the
   `agents_clan_panel_needs_attention_120x40` and
   `agents_session_panel_needs_attention_160x50` panel goldens. Run a targeted check,
   for example `just test-visual -- <selectors>` for
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py` and
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_agent_sessions.py`. Run it once
   more with `TZ=UTC`, per the `tui_screenshot.md` memory. If anything changes, inspect
   it and fix the code rather than regenerating goldens.
6. **Verify.** Read the `lint_and_test.md` memory, run `just fmt`, then run
   `sase tool run check`. At planning time,
   `tests/test_project_display_presentation_audit.py::test_project_metadata_loaders_stay_at_reviewed_tui_worker_boundaries`
   was KNOWN-red on master (owner `sase-1uz`) and is not yours. Do not run
   `just check-full`.
7. **Close out epic `sase-1ux`** (the final step; it does not wait for this work's
   commit SHA, push, or CI):
   1. Run `sase bead epic-symbols sase-1ux`. At planning time it printed
      `No --epic-symbol entries for sase-1ux.` If entries exist now, resolve each one
      per the `symvision.md` memory (wire it up, privatize it, add a non-test pragma, or
      delete it). Re-key an entry only to a still-open later bead that needs it.
   2. Run `sase bead close sase-1ux --note "<verification>"`. The note says that the
      land agent verified all five phases against source and commits (sase-core
      `1fb9701f`, contained in the pin, plus `1c4c89f6b4`, `3dc72ddd4c`, `623c826812`,
      and `1d4f456f41`). It reviewed 15 concurrent commits and found no conflicts. It
      fixed the epic-caused terminology regression (`sase-1uy`) and recorded follow-up
      triage in the `LAND TRIAGE` note. Add what this tale changed and what `just check`
      showed. Never use `--force` merely to make the close succeed. If the close is
      refused, fix the named cause and close again.
   3. Run `just symvision` and confirm it is clean.
   4. Set `status: done` in the frontmatter of the epic's plan file, the PLAN path that
      `sase bead read sase-1ux -r "<why>"` prints
      (`plan:202611/failed_clan_visibility.md`). It currently reads `status: wip`.
   5. `sase-1ux` has no `parent_bead`, so there is nothing further to close.
