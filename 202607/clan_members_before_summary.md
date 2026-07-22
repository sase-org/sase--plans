---
tier: tale
title: Show clan members before the clan summary
goal: Selected clan metadata prioritizes the member roster above the optional authored
  summary without changing folds, navigation, or summary rendering.
create_time: 2026-07-22 11:08:29
status: wip
---

- **PROMPT:** [202607/prompts/clan_members_before_summary.md](prompts/clan_members_before_summary.md)

# Show clan members before the clan summary

## Goal

Prioritize the actionable clan roster in the Agents metadata panel. When a synthetic clan row is selected, render the
fold-aware `CLAN MEMBERS` roster above the clan's authored Rich summary so members remain visible near the top even when
an epic or other generated summary is long.

## Current behavior

`build_clan_detail_text()` builds the selected-clan document entirely from the supplied in-memory snapshot. It currently
emits the identity/status header, optional `clan_summary`, global fold line, numbered member roster, and then the
aggregate error/variable/reply/context/tool/prompt sections. Long summaries can therefore push every member below the
viewport. The roster also publishes the number-to-member jump map and marks its heading and member rows as metadata
navigation anchors; those behaviors must remain intact.

## Implementation

1. In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`, reorder only the clan-detail document assembly:
   keep the compact clan header and global fold indicator first, build and publish the existing numbered member roster,
   then append the optional clan summary before the remaining aggregate sections. Preserve the existing Rich markup
   parsing, malformed-markup raw-text fallback, spacing, fold-level handling, snapshot-only rendering, section IDs, and
   jump-map publication. Do not change clan summary persistence/resolution, roster sorting/contents, disk enrichment, or
   tribe and ordinary-agent documents.

2. Update `tests/ace/tui/widgets/test_agent_display_clan.py` to make the intended order an explicit regression contract
   at every clan fold level: the fold line and `CLAN MEMBERS` roster/member entry appear before the summary, while valid
   summary markup retains its style. Adjust the malformed-markup case to assert that its literal fallback is also placed
   after the roster. Retain coverage that clans without summaries and non-clan agents render as before.

3. Update `tests/ace/tui/visual/test_ace_png_snapshots_agents_clan_panel.py` only where viewport assertions need to
   reflect the new priority. Ensure both the swarm and epic scenarios visibly exercise a member label before their
   summaries, while retaining representative assertions that the summaries still render. Regenerate and inspect the six
   intentional clan-panel goldens for collapsed, expanded, and fully expanded levels:
   `agents_clan_panel_{swarm,epic}{,_level_2,_level_3}_120x40.png`. Confirm the roster is above the summary in each
   image and that no unrelated visual goldens change.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral SASE workspace.
2. Run the focused renderer and metadata-navigation tests:
   `just test tests/ace/tui/widgets/test_agent_display_clan.py tests/ace/tui/widgets/test_prompt_panel_section_navigation.py`.
   Confirm the member section and member anchors remain in their existing navigation order and the jump-map behavior is
   unchanged.
3. Regenerate only the affected visual snapshots with
   `just test-visual -- --sase-update-visual-snapshots tests/ace/tui/visual/test_ace_png_snapshots_agents_clan_panel.py`,
   inspect every changed PNG, then rerun the same targeted visual test without the update flag for exact comparison.
4. Run the required full repository gate with `just check` and verify `git status --short` contains only the renderer,
   intentional tests, and the six reviewed clan-panel PNG goldens.

## Acceptance criteria

- Selecting a clan shows its numbered member roster above its optional authored summary at every supported clan fold
  level, including long epic summaries.
- The summary still renders valid Rich markup and safely falls back to literal text for malformed markup.
- Clan member numbering, ordering, fold overrides, section navigation anchors, and number-key jumps are unchanged.
- Clan documents without a summary, tribe documents, and ordinary-agent metadata documents retain their current layout.
- Focused tests, exact PNG snapshot verification, and `just check` pass.
