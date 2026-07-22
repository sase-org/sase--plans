---
tier: tale
title: Show agent-hole count in the Agents-tab headline
goal: The Agents-tab headline reports agent holes, including direct clan members and
  hidden starting agents, while concrete status and navigation counts retain their
  existing semantics.
create_time: 2026-07-22 10:51:04
status: done
---

- **PROMPT:** [202607/prompts/agent_hole_headline_count.md](prompts/agent_hole_headline_count.md)

# Plan: Show agent-hole count in the Agents-tab headline

## Goal

Change the leading numeric headline on the `sase ace` Agents tab from the number of concrete agents represented by the
current view to the number of **agent holes**. In the reported screenshot, the headline must therefore read `31` rather
than `56`.

An agent hole is one standalone agent or one sequential agent family. Family members do not add holes: after a
standalone agent gains a follow-up and becomes a family, the family still occupies the same one hole. A clan is a
rootless container rather than a hole, so each direct clan member contributes one hole; a direct member that is itself a
sequential family still contributes only one. This definition is independent of whether clans/families are folded, which
tribe panel owns them, or which Agents-tab grouping mode is active.

## Current behavior and root cause

`AgentInfoDisplayMixin._agent_info_metrics()` in `src/sase/ace/tui/actions/agents/_display_detail_info.py` passes
`agent_summary_status_counts(...).total + starting_count` to `AgentInfoPanel` as the leading total. The shared summary
projection intentionally expands clan containers, sequential family containers, and workflow aggregates into concrete
agent rows so adjacent status metrics can report every running/waiting/done agent. That concrete projection makes the
reported strip internally read `56 [3 running · 1 waiting · 52 done]`, but `56` is the wrong unit for the headline.

The implementation must separate these two valid cardinalities:

- The **headline** counts holes: one standalone agent or sequential family; a clan contributes its direct member holes.
  A hidden top-level `STARTING` agent contributes one hole even though it is not selectable yet.
- The **status strip** continues to count concrete agents, so its buckets may intentionally sum to a number larger than
  the hole headline. This preserves member-level status visibility.
- Position/navigation totals continue to count rendered selectable roots. Clan containers remain one selectable row, so
  the position denominator is neither the hole count nor the concrete-agent count.
- Panel-title totals, group-banner summaries, tribe/clan/family composition, roster counts, runner capacity, and fold
  annotations retain their existing semantics. In particular, the per-panel `N` stays aligned with its adjacent
  concrete-agent status chip; this request changes only the global leading headline called out in the screenshot.

The count is a presentation projection over the already-loaded, filtered Agents-tab tree. The Rust scanner already
supplies clan/family metadata, and no shared wire or backend API needs to change. Keep the implementation in the Python
models/actions layer, with no `sase-core` edits.

## Implementation

1. Add an explicit, pure agent-hole counter beside the existing clan/family summary projection in
   `src/sase/ace/tui/models/_agent_clan.py` (or a narrowly named sibling model module if that avoids coupling). Give the
   result a first-class `holes`/`hole_total` name rather than overloading the existing concrete `total`.
   - For each top-level non-clan unit, count one hole without expanding a loaded sequential family or workflow step
     tree.
   - For a synthetic clan container, use its loaded direct members (`clan_members()` / `runtime_children`) as the hole
     units. The clan container contributes zero by itself, and descendants of a direct sequential-family member do not
     add holes.
   - Preserve legacy parallel-family compatibility: loaded parallel members replace an aggregate compatibility root,
     while a root with no loaded members falls back to one hole, matching the existing projection boundary.
   - Deduplicate by stable agent identity where compatibility fixtures or merged projections can expose the same direct
     member twice.
   - Operate only on in-memory rows already traversed by the cached summary path. Do not add filesystem access,
     subprocesses, a new refresh path, or render-time stat/glob work.

2. In `_agent_info_metrics()`, continue deriving unread/stopped/running/waiting/failed/done from
   `agent_summary_status_counts()`, but derive the headline from the new hole count plus the existing
   `hidden_starting_indices` count. A hidden top-level `STARTING` row is a not-yet-loaded standalone hole and must be
   included exactly once, preserving the earlier `1 [1 starting]` regression fix.

   Keep this work inside `_agent_info_metrics_cache` and retain its current invalidation inputs. The loaded agent-list
   identity already invalidates structural clan/family changes, and unread-state changes still invalidate the concrete
   status buckets. Continue using `AgentPanelIndex.non_child_total` and `non_child_position()` for selectable position
   semantics.

3. Update the headline data-flow terminology in `src/sase/ace/tui/actions/agents/_display_detail_info.py` and
   `src/sase/ace/tui/widgets/agent_info_panel.py` so comments, docstrings, local variables, and the stable-state field
   identify this value as an agent-hole count. Rename the internal/named `visible_agent_count` state where practical;
   retain the legacy positional `update_agent_counts(..., total)` fallback only as a compatibility adapter and document
   that `total` now means the headline hole count. Do not change styling, spacing, loading text (`Agents: …`), or any of
   the capacity/status badges.

4. Update `docs/ace.md` to describe the leading value as the visible agent-hole total and spell out the family/clan and
   hidden-STARTING rules. Explicitly distinguish it from the following concrete-agent status buckets and the
   rendered-row position total so future work does not re-couple the three cardinalities merely to make them sum.

## Tests and verification

1. Add focused model coverage for the hole projection (alongside
   `tests/ace/tui/models/test_agent_summary_status_counts.py` or in a dedicated model test):
   - standalone agents each contribute one;
   - a multi-member sequential family contributes one while its concrete-agent total still includes every member;
   - a clan container contributes one per direct standalone/family member, never one for the container or its family
     descendants;
   - multiple clans and panels deduplicate stable identities;
   - loaded legacy parallel members replace their compatibility root, and an unloaded legacy root falls back to one;
   - a screenshot-cardinality regression constructs 31 holes representing 56 concrete agents and asserts both values,
     pinning the requested `56` to `31` correction without weakening concrete status counting.

2. Update `tests/ace/tui/test_agent_panel_index_integration.py` to prove the complete info-panel data flow:
   - mixed standalone, sequential-family, and clan roots send the hole total to the headline while concrete status
     buckets retain member counts;
   - hidden top-level `STARTING` rows add one to both the hole headline and starting bucket but remain excluded from the
     selectable position denominator;
   - changing grouping/fold presentation does not change the headline;
   - existing ordinary-agent and legacy parallel-family behavior remains covered.

3. Update `tests/ace/tui/widgets/test_agent_info_panel.py` for the renamed state/argument contract and assert that the
   widget renders the supplied hole count unchanged before the capacity and concrete-status chips. Keep the countdown
   stable-cache tests intact so a countdown-only tick still avoids rebuilding the stable Rich text.

4. Exercise the family/clan visual fixtures in `tests/ace/tui/visual/test_ace_png_snapshots_agents_families.py`,
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py`, and related Agents-tab scenarios. Add a semantic
   assertion for at least one mixed clan/family fixture, regenerate only genuinely affected PNG goldens, inspect the
   actual/expected/diff artifacts under `.pytest_cache/sase-visual/`, and rerun comparison-only to prove the accepted
   change is limited to the headline digits.

5. Run `just install` before repository checks, then run the focused model/integration/widget tests and the dedicated
   visual suite. Finish with the repository-required `just check`.

## Non-goals

- Do not change concrete-agent status aggregation or settled family-member bucketing.
- Do not change panel-title, group-banner, tribe/clan/family composition, runner-capacity, fold, selection, or
  navigation counts.
- Do not add an explicit `holes` label to the compact header; the requested visual remains the same except for the
  corrected number.
- Do not modify SASE memory files or the Rust core/backend wire.
