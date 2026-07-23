---
tier: tale
title: Complete the agent hole-to-lane rename
goal: All meaningful agent-hole terminology is replaced by agent-lane terminology
  without changing counting or TUI behavior.
create_time: 2026-07-23 13:16:19
status: wip
---

- **PROMPT:** [202607/prompts/agent_lane_rename.md](prompts/agent_lane_rename.md)

# Complete the agent “hole” to “lane” rename

## Goal

Finish the terminology change started by commit `e803ce63f305` so the SASE codebase consistently calls the unit
represented by either one standalone agent or one sequential agent family an **agent lane**. Preserve the current
counting, status normalization, folding, and rendering behavior; this is a terminology and API-name migration, not a
semantic change.

## Context and current scope

Commit `e803ce63f305` renamed the glossary definition from “Agent Hole” to “Agent Lane,” and the following
memory-initialization commit propagated that definition into generated instruction files. A tracked, case-insensitive
audit of the current tree still finds 118 meaningful text references in 21 files:

- User documentation: `docs/ace.md`.
- Runtime projection and presentation: `src/sase/ace/tui/models/_agent_clan.py`,
  `src/sase/ace/tui/models/agent_tribe_summary.py`, `src/sase/ace/tui/actions/agents/_display_detail_info.py`,
  `src/sase/ace/tui/actions/agents/_display_panel_collection.py`,
  `src/sase/ace/tui/actions/agents/_display_panel_patches.py`,
  `src/sase/ace/tui/actions/agents/_display_panel_titles.py`, `src/sase/ace/tui/widgets/agent_info_panel.py`, and
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py`.
- Unit, integration, and visual tests: `tests/ace/tui/models/test_agent_summary_status_counts.py`,
  `tests/ace/tui/models/test_agent_tribe_summary.py`, `tests/ace/tui/test_agent_display_diff.py`,
  `tests/ace/tui/test_agent_panel_index_integration.py`, `tests/ace/tui/test_agent_panel_title_refresh.py`,
  `tests/ace/tui/test_agent_panel_titles.py`, `tests/ace/tui/test_startup_loading_indicators.py`,
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py`,
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py`,
  `tests/ace/tui/widgets/test_agent_display_clan.py`, `tests/ace/tui/widgets/test_agent_display_tribe.py`, and
  `tests/ace/tui/widgets/test_agent_info_panel.py`.

The raw binary search also reports `demos/out/sase_ace_multi_model_fanout.mp4`, but its only match is an incidental
`HOLe` byte sequence inside compressed binary data. The surrounding extracted strings are gibberish, and the video
predates the commits that introduced the agent-hole UI, so it is not in scope for regeneration. No tracked filename
contains the old term.

## Implementation

1. Rename the projection vocabulary at its source while preserving behavior. In `_agent_clan.py`, rename
   `agent_hole_status_counts` to `agent_lane_status_counts`, `_hole_summary_projections` to `_lane_summary_projections`,
   and update recursion, docstrings, and `__all__`. Update every import and caller. Do not leave a deprecated alias:
   these are private TUI implementation modules, and retaining one would defeat the repository-wide terminology
   migration.

2. Carry the new field and local-variable names through both summary pipelines. Rename `hole_count`, `hole_counts`, and
   `_hole_status_counts` to their `lane_*` equivalents in tribe snapshots and panel-title counts. Rename `hole_total`,
   `agent_hole_count`, and `_agent_hole_count` to their lane equivalents in detail-info and info-panel state, including
   keyword parameters and the collection/patch call sites. Keep the exact existing rules for standalone agents,
   sequential families, clan direct members, legacy parallel families, hidden `STARTING` roots, status buckets, unread
   deduplication, and nested-member exclusion.

3. Replace all user-facing and explanatory old terminology. Render `lane`/`lanes` with the same singular/plural behavior
   in tribe composition, and update ACE documentation, comments, docstrings, test names, fixture/local variable names,
   keyword dictionaries, and assertion descriptions from “agent hole”/“hole(s)” to “agent lane”/“lane(s)”. Leave
   unrelated words such as “whole,” “wholesale,” and “loophole” unchanged.

4. Update the affected tests in lockstep with the renamed APIs and copy. Preserve their existing behavioral coverage
   rather than weakening or deleting assertions. In particular, retain coverage for:
   - standalone, sequential-family, clan, legacy-parallel-family, terminal, unread, and starting-lane projections;
   - tribe snapshot and panel-title `lane_count` propagation through full refreshes and row patches;
   - the info-panel lane headline and its loading/capacity/status behavior;
   - clan/tribe composition copy for zero, singular, and plural lanes.

5. Run the visual suite after the text/API migration. Inspect failures under `.pytest_cache/sase-visual/` and use
   `just test-visual -- --sase-update-visual-snapshots` only to accept goldens whose pixels changed because visible
   `hole(s)` copy became `lane(s)`. Re-run `just test-visual` without the update flag and inspect the resulting diff so
   unrelated PNG changes are not accepted.

## Validation

1. Run `just install` first, as required for an ephemeral SASE workspace.
2. Run the focused model, action, widget, integration, and visual tests covering the files above; ensure renamed tests
   are collected and the zero/singular/ plural rendered lane wording is asserted.
3. Run `just test-visual` cleanly after any intentional golden updates.
4. Run `just check` and require the full formatter, linter, type checker, and test suite to pass.
5. Re-run a tracked, text-only, case-insensitive audit that catches prose, hyphenated names, and identifier fragments
   (including `agent_hole`, `hole_count`, and `_hole_*`). Require zero meaningful old-term matches while confirming
   unrelated `whole`/`loophole` text is untouched. Also verify no tracked filename contains `hole`.
6. Review the final diff to confirm it contains only the terminology/API migration and intentional visual golden
   changes, with no counting or status behavior changes.
