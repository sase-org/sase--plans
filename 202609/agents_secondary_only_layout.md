---
tier: tale
title: Add fullscreen Agents detail layouts
goal:
  The Agents view picker supports reliable metadata-only and File/LLM-Calls-only
  fullscreen layouts without adding them to the split-layout cycle.
size: medium
proposed_by: bbugyi200.apollo.0e.f0.f0
create_time: 2026-09-18 13:00:00
status: wip
---

# Plan: Agents fullscreen detail layouts

## Objective

Extend the Agents-tab `p` view picker with a secondary-only fullscreen layout that shows
the selected File or LLM Calls panel without the agent metadata panel. Move the existing
metadata-only shortcut from `0` to `[`, assign `]` to the new secondary-only layout, and
keep the three split layouts as the only members of the `p`/`P` cycle.

The result should read as one visual continuum inside the picker:

| Key | Layout                        |
| --- | ----------------------------- |
| `[` | Metadata only (100/0)         |
| `1` | Metadata larger (70/30)       |
| `=` | Equal split (50/50)           |
| `2` | File/LLM Calls larger (30/70) |
| `]` | File/LLM Calls only (0/100)   |

`p` and `P` continue to cycle forward and backward only among `1`, `=`, and `2`. When
either fullscreen endpoint is active and a secondary panel is available, either cycle
key exits fullscreen directly to `=`. Bare `[` and `]` outside the `p` modal remain
inert on the Agents tab.

## Design

Keep content selection and geometry orthogonal:

- File versus LLM Calls remains the detail panel mode. Changing between `f` and `t` must
  preserve the active layout, including either fullscreen endpoint.
- Represent metadata-only and secondary-only as explicit detail layout states alongside
  the three split ratios. Remove the current special treatment of metadata-only as an
  independent `INFO` content mode (or provide a narrow internal migration adapter if a
  staged refactor is safer), so entering metadata-only does not forget whether File or
  LLM Calls should reappear.
- Keep the canonical layout cycle tuple limited to metadata-larger, equal, and
  secondary-larger. Make the next/previous helper return equal when its input is either
  fullscreen state; it must never wrap from a split ratio into a fullscreen state.
- Derive the visible layout from one centralized state-application path. It must clear
  stale sizing/visibility classes before showing exactly the active metadata and
  secondary scrolls, including the metadata-search scroll variant, and it must preserve
  the existing metadata fallback when File has no content or the selected row is a
  summary, proc shell, or pinned historical attempt.
- Keep secondary availability separate from current visibility. In metadata-only mode,
  `p`/`P`, the three split choices, and `]` are enabled when the selected File or LLM
  Calls panel can be shown even though it is currently hidden. If no secondary panel is
  available, keep those choices disabled with the existing actionable reason. Forced
  metadata contexts show `[` as the effective current layout and do not overwrite the
  user's session-local layout preference.
- Preserve current fast paths: switching layouts performs only synchronous widget/class
  updates and uses the existing cached File/LLM Calls content. Do not add disk reads,
  refresh workers, or asynchronous work to picker key handlers or render methods.

## Implementation

1. Refine the Agent detail state model in
   `src/sase/ace/tui/widgets/_agent_detail_panels.py` and
   `src/sase/ace/tui/widgets/agent_detail.py`.
   - Add metadata-only and secondary-only layout states while retaining a three-entry
     split-cycle constant.
   - Make direct layout application explicitly show/hide the active metadata scroll and
     selected File/LLM Calls scroll, apply full-height styling for the sole visible
     panel, and remove stale fullscreen/split classes on every transition.
   - Preserve File/LLM Calls selection while fullscreen, and make both cycle directions
     leave either fullscreen state for equal.
   - Reapply the active layout after File/LLM Calls availability events, agent
     navigation, metadata-search swaps, and empty-file fallback so refreshes cannot
     resurrect a hidden panel or leave a blank detail area.
   - Update `panel_mode_label`, `is_info_mode` (or its replacement), scroll routing, and
     related compatibility helpers so metadata-only still reports/targets metadata,
     while secondary-only reports/targets the active File or LLM Calls panel.

2. Update the picker model and presentation in
   `src/sase/ace/tui/actions/agents/_agent_view_picker.py`,
   `src/sase/ace/tui/modals/agent_view_modal.py`, and `src/sase/ace/tui/styles.tcss`.
   - Replace the `0` row with `[` Metadata only and add `]` File/LLM Calls only; use
     Textual's bracket key names for bindings while continuing to match the printable
     characters in the modal's input containment path.
   - Arrange the layout rows in the visual order `[`, `1`, `=`, `2`, `]`, followed by
     `p` and `P`, so the two fullscreen endpoints frame the three split choices.
   - Mark the current content choice and current layout independently, focus the active
     fullscreen row when opening from fullscreen, and keep `0`/`n` contained as inert
     printable keys rather than leaking to app bindings.
   - Compute choice availability from selected secondary capability/content rather than
     from whether that panel is visible. Ensure stale modal submissions are still
     rejected after row/attempt/capability changes.
   - Update cycle subtitles so split layouts describe their true next/previous member
     and both fullscreen layouts advertise `... -> Equal split` for `p` and `P`.

3. Align dependent Agents behavior.
   - Update zoom target selection in `src/sase/ace/tui/actions/agents/_panel_detail.py`:
     metadata-only starts zoom on metadata, secondary-only starts on the selected
     File/LLM Calls panel, and the existing split-ratio target rules remain intact.
   - Update detail scrolling/navigation and incremental file-refresh predicates to use
     the effective visible panel, without changing behavior for forced metadata rows or
     empty File results.
   - Keep the picker and layout session-local; do not add configuration, persistence, or
     Rust-core behavior for this presentation-only feature.

4. Update discovery and documentation.
   - Refresh Agents help rows, command-palette search terms/description, `docs/ace.md`,
     and `docs/configuration.md` to document `p[` / `p]`, the five direct layouts, the
     three-state-only cycle, and the fullscreen-to-equal escape behavior.
   - Update shared visual-test helpers that select metadata-only and remove stale claims
     that `[`/`]` are inert inside the picker. Continue to state that bare brackets are
     inert on the Agents tab.

5. Add focused behavioral and visual coverage.
   - Modal tests cover direct bracket selection, inert `0` and `n`, keyboard navigation,
     disabled secondary-only rows, current badges/focus, and both cycle directions from
     each fullscreen endpoint.
   - Mounted picker tests cover File-only and LLM-Calls-only rendering; preservation of
     fullscreen layout across `f`/`t`; direct recovery to all three split ratios; `p`
     and `P` mapping fullscreen to equal; and unchanged three-state wraparound for
     ordinary split layouts.
   - Widget/state tests cover stale-class cleanup, refresh/availability reapplication,
     no-file fallback, forced summaries/historical attempts, scroll targeting, and
     File/LLM Calls incremental refresh behavior.
   - Zoom tests cover both fullscreen endpoints for File and LLM Calls.
   - Update the picker PNG golden to show the bracketed five-layout continuum and add
     deterministic File-only and/or LLM-Calls-only visual coverage that proves the
     metadata panel is absent and the secondary panel consumes the full detail height.
     Inspect the rendered PNG diffs before accepting the baselines.

## Acceptance criteria

- `p[` shows only agent metadata; `p]` shows only the currently selected File or LLM
  Calls panel.
- `p0` no longer changes layout, and unused printable keys remain contained by the
  modal.
- `pp` and `pP` cycle only through 70/30, 50/50, and 30/70 while on a split layout.
- From either fullscreen layout, both `pp` and `pP` select the 50/50 layout when the
  secondary panel is available.
- Choosing File or LLM Calls does not discard the current fullscreen/split layout, and
  choosing a layout does not discard the selected secondary content mode.
- Secondary-only cannot expose an empty/stale detail area: unavailable File/LLM Calls
  choices are disabled or fall back to metadata using the established rules.
- Navigation, refresh events, scrolling, metadata search, and `Z` zoom all target the
  panel that is effectively visible.
- Help, command search, documentation, tests, and PNG snapshots use the new keys and
  semantics consistently.

## Verification

1. Run focused non-visual tests for the modal, mounted picker, detail layout state,
   scrolling/refresh routing, and zoom behavior.
2. Run the touched Agents visual snapshot tests, review the actual PNG diffs, and use a
   local `sase screenshot` capture of a representative fullscreen File or LLM Calls
   state as an interactive sanity check when suitable agent data is available.
3. Run `just fmt`, `git diff --check`, and the repository-required `just check`; fix
   failures caused by this change and re-run the affected focused lanes plus the final
   gate.
