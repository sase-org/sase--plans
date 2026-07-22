---
tier: tale
title: Count agent holes in Agents-tab status summaries
goal: Make global, tribe-panel, and selected-tribe primary status metrics count the
  same agent holes as their adjacent totals.
create_time: 2026-07-22 12:11:26
status: wip
---

- **PROMPT:** [202607/prompts/agent_hole_status_counts.md](prompts/agent_hole_status_counts.md)

# Plan: Count agent holes in Agents-tab status summaries

## Goal

Make the primary status summaries on the `sase ace` Agents tab use the same agent-hole cardinality as their adjacent
totals. The global header and every split, collapsed, selected, or merged tribe panel must count one status per agent
hole rather than one status per concrete sequential-family member. For the motivating `31 holes / 56 concrete agents`
case, an all-done roster must therefore render `31` with `31 done`, not `56 done`.

An agent hole is one standalone agent or one sequential family. A rootless clan is not a hole; each direct clan member
is a hole, and a direct sequential family still contributes only one. Loaded legacy parallel-family members continue to
contribute one hole each, while an unloaded compatibility aggregate contributes one fallback hole. Workflow aggregates
remain one hole. Fold state, grouping mode, tribe ownership, and panel layout must not change either totals or status
counts.

## Status semantics

- Classify a standalone, workflow, or sequential-family hole from its normalized top-level owner status. Sequential
  family roots already mirror the active or latest logical member during status normalization, so historical family
  members must not add settled `done`, `running`, or other primary-summary counts.
- Recurse through clan containers to their loaded direct members. For a direct sequential family, classify the family
  root once; for a direct standalone or workflow, classify that direct row once.
- Preserve the legacy parallel-family rule: when loaded parallel members replace their aggregate root as holes, classify
  each loaded member independently. Ignore unrelated serial descendants.
- Deduplicate projections by stable agent identity so the same hole cannot be counted twice when it appears through
  multiple containers or panel inputs.
- Preserve existing status-bucket meanings and styling: stopped/asking, running, waiting, failed, unread, and done. An
  unread terminal hole contributes to `unread` instead of `done`; the existing manual-unread behavior for nonterminal
  rows remains unchanged.
- Preserve transient `STARTING` behavior. A hidden top-level `STARTING` row is one hole in the global total and one
  `starting` count, but is absent from tribe-panel slices. A starting member reached through a visible clan or legacy
  aggregate continues to roll up as running where the compact panel chip has no starting metric.

## Current behavior and scope

- `agent_hole_count()` in `src/sase/ace/tui/models/_agent_clan.py` already supplies the correct total projection.
  `agent_summary_status_counts()` separately expands sequential families through `concrete_agent_statuses()`, which is
  why its status sum can exceed the hole total.
- `AgentInfoDisplayMixin._agent_info_metrics()` uses the concrete projection for the global header while using
  `agent_hole_count()` for its leading number.
- `agent_panel_counts()` uses the same concrete projection for split/collapsed/merged panel title chips while using the
  hole projection for the panel total.
- `build_agent_tribe_summary_snapshot()` also uses concrete counts for the selected whole-panel `TRIBE` header. Its
  primary `Status:` chip should agree with the corresponding panel title and therefore moves to hole semantics too.
- Nested detail remains intentionally concrete: expanded member rows, nested-count composition, and per-family/per-clan
  member-rollup chips inside the detailed tribe roster continue to describe the concrete members being shown. Clan-row
  chips already count direct member holes correctly and should not change. Group-banner summaries, runner-capacity
  counts, navigation/selectable counts, cleanup/action counts, and saved-group metadata are outside this change.

## Implementation

1. Add a pure, in-memory hole-status projection beside the existing concrete projection in
   `src/sase/ace/tui/models/_agent_clan.py`.
   - Return the existing summary fields plus a `total` that is the deduplicated hole count, allowing callers to derive a
     total and all adjacent metrics from one traversal.
   - Carry the existing projected-from-container context needed to map nested `STARTING` members into the compact
     running bucket, while leaving hidden top-level `STARTING` accounting to the global header.
   - Resolve unread from the projected hole owner's identity and merge duplicate projections without duplicating unread
     state.
   - Share bucketing code with the concrete projection where practical, but keep `agent_summary_status_counts()` (or an
     explicitly renamed concrete equivalent) available for nested member rollups and concrete-cardinality tests.
   - Make `agent_hole_count()` delegate to, or share its deduplicated unit projection with, the new helper so total and
     status semantics cannot drift.

2. Switch primary summary callers to the hole-status projection.
   - In `src/sase/ace/tui/actions/agents/_display_detail_info.py`, use the hole projection for unread/stopped/running/
     waiting/failed/done and its total for visible roots, then add the already-indexed hidden top-level `STARTING` holes
     to both the headline total and starting metric exactly once. Preserve the existing in-memory metrics cache and
     selectable-position denominator.
   - In `src/sase/ace/tui/actions/agents/_display_panel_titles.py`, derive `AgentPanelCounts.hole_count` and every
     status metric from the same hole projection. This must cover split panels, collapsed panels, selected panels, and
     the merged `All agents` layout without adding render-path I/O.
   - In `src/sase/ace/tui/models/agent_tribe_summary.py`, use hole counts for the top-level snapshot `counts` and derive
     `hole_count` from the same projection. Keep the existing concrete helper for nested family/clan unit
     `status_counts`, member glyphs, and `nested_count`.
   - Preserve the existing selective title/row patch paths so incremental status or unread refreshes recompute the
     affected panel title and global header without a full list rebuild.

3. Update `docs/ace.md` to state that the global header strip, tribe title strips, and selected `TRIBE` header count
   holes and use the normalized owner status. Remove the now-obsolete statements that these primary metrics are concrete
   or may exceed their adjacent total, while documenting that nested roster/member summaries retain concrete semantics.

## Tests

1. Extend `tests/ace/tui/models/test_agent_summary_status_counts.py` with focused projection contracts:
   - an active planner/coder sequential family is one running hole rather than running plus done;
   - a finished multi-member family is one done hole;
   - a clan containing a family and a standalone contributes two status-classified holes;
   - duplicate identities and unread terminal owners count once;
   - loaded and unloaded legacy parallel-family behavior is preserved;
   - the `31 holes / 56 concrete agents` fixture reports `total == 31` and `done == 31` from the hole projection while
     the retained concrete projection remains `total == 56`.

2. Update `tests/ace/tui/test_agent_panel_index_integration.py` so the global header regression proves its status strip
   and headline use the same mixed family/clan hole projection. Retain coverage for the hidden standalone `STARTING`
   hole and legacy parallel-member counts.

3. Update `tests/ace/tui/test_agent_panel_titles.py` and `tests/ace/tui/test_agent_panel_title_refresh.py` so
   sequential-family panels render one owner-status metric per hole in split, collapsed, and merged layouts. Keep legacy
   parallel members independently classified, and add or adapt an incremental patch assertion proving a status/unread
   change refreshes the hole-based title without rebuilding the panel collection.

4. Update `tests/ace/tui/models/test_agent_tribe_summary.py` and the tribe display tests so the selected whole-panel
   header matches its title's hole statuses while nested family/clan unit rollups remain concrete. In particular, the
   reference `6 holes / 10 concrete statuses` tribe should become `2 running / 4 done` in its primary summary while its
   nested/member data remains unchanged.

5. Run the affected Agents-tab visual tests in comparison-only mode first, including family, clan, panel-title, and
   tribe-panel scenarios. Inspect every actual/expected/diff artifact and update only snapshots whose global or panel
   status digits intentionally change. Rerun those tests comparison-only, then run the complete dedicated visual suite
   to find and audit any additional family-bearing screens.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run formatting and the focused model, info-panel, panel-title/refresh, tribe-summary, and tribe-rendering tests named
   above.
3. Run the targeted visual comparisons, inspect diffs, update only intentional goldens, and rerun them without snapshot
   updates.
4. Run `just test-visual` comparison-only across the complete PNG suite.
5. Run the required repository-wide `just check` and `git diff --check`.
6. Audit the final diff to confirm that primary status chips use holes, nested concrete detail is preserved, no
   render-path I/O or full-rebuild path was introduced, and only intentional PNG goldens changed.
