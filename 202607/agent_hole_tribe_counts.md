---
tier: tale
title: Use agent-hole totals on clan rows and tribe panels
goal: Tribe-panel totals match agent-hole semantics while clan-row hole counts and
  concrete status metrics remain correct.
create_time: 2026-07-22 11:23:46
status: done
---

- **PROMPT:** [202607/prompts/agent_hole_tribe_counts.md](prompts/agent_hole_tribe_counts.md)

# Plan: Use agent-hole totals on clan rows and tribe panels

## Goal

Make every primary Agents-tab total on a synthetic clan row or tribe panel describe agent holes, matching the corrected
global headline. One standalone agent or one sequential family is one hole; a clan container contributes no hole of its
own and each direct clan member contributes one; loaded legacy parallel-family members continue to contribute their
individual holes. Structural descendants, fold state, grouping mode, and tribe-panel layout must not change the total.

Keep the adjacent status buckets intentionally concrete. Their sum may therefore exceed the hole total when a sequential
family has multiple concrete members.

## Current behavior and scope

- Synthetic clan rows are already correct. Clan containers retain only their direct clan members in `runtime_children`,
  `clan_member_counts()` counts those identities once, and the structural fold annotation counts the same direct rows.
  Existing visual coverage shows a clan with three direct holes—including one sequential family—as `×3 [R1 W1 D1]`,
  while the nested family member remains available only below the family row. Preserve this behavior and add a focused
  regression that makes the hole-versus-concrete distinction explicit.
- Tribe-panel border titles are still wrong. `agent_panel_counts()` obtains its total from
  `_panel_concrete_agent_total()`, which expands sequential families and other aggregate rows into concrete agents.
- The selected whole-tribe detail document has the same mismatch. `AgentTribeSummarySnapshot.agent_count` is populated
  from `len(agent_status_projections(...))` and rendered as `N agents`, even though its roster has one top-level unit
  per hole. For example, the current tribe visual shows a two-hole roster (one family plus one standalone) as `3` in
  both the border title and composition line.
- Concrete status chips, concrete nested-member detail, navigation denominators, cleanup scopes, runner-slot capacity,
  and non-Agents-tab uses of agent counts are out of scope and must retain their current semantics.

This is presentation-only Python/TUI work; it does not add or alter shared backend behavior and therefore requires no
Rust-core API change.

## Implementation

1. **Make the scoped panel total explicitly hole-based.**
   - In `src/sase/ace/tui/actions/agents/_display_panel_titles.py`, calculate the panel total with the existing pure
     `agent_hole_count()` projection over the panel's top-level roots.
   - Remove the obsolete concrete-total projection and its tree/aggregate imports.
   - Rename ambiguous presentation fields and arguments such as `AgentPanelCounts.total` and the border-title
     `agent_count` parameter to hole-specific names, then update both full-panel construction and row-patch callers in
     `_display_panel_collection.py` and `_display_panel_patches.py`.
   - Continue deriving stopped/running/waiting/failed/unread/done metrics through `agent_summary_status_counts()` so the
     title can legitimately show, for example, `2 [R1 W1 F1]`.
   - Preserve split/merged layouts, selection and collapse chrome, jump hints, identity colors, stable identity
     deduplication, and fold/group independence. Do not add disk I/O or any new work to the render path.

2. **Make selected-tribe composition use the same projection.**
   - In `src/sase/ace/tui/models/agent_tribe_summary.py`, replace the ambiguous snapshot `agent_count` with a
     `hole_count` computed by `agent_hole_count(roots)`.
   - In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py`, render the composition value as `N hole` or
     `N holes`, including `0 holes` for an empty panel.
   - Preserve `clan_count`, `family_count`, `nested_count`, roster units, attention/error aggregation, and the concrete
     status counts. The change must not remove the useful distinction between holes and nested concrete members.

3. **Pin both the changed and already-correct surfaces with tests.**
   - Extend `tests/ace/tui/test_agent_panel_titles.py` with scoped-count fixtures containing a standalone, a sequential
     family, and a clan whose direct members include a sequential family. Assert the hole total independently from the
     larger concrete status-bucket sum, including merged and folded/collapsed presentation where appropriate.
   - Update `tests/ace/tui/models/test_agent_tribe_summary.py` to assert `hole_count` for mixed, family-only, workflow,
     and larger snapshots while retaining assertions for concrete status and nested counts.
   - Update `tests/ace/tui/widgets/test_agent_display_tribe.py` and any summary lifecycle tests to expect explicit
     `hole(s)` composition wording.
   - Add or strengthen focused clan-row rendering coverage so a clan containing one sequential family plus one
     standalone displays two direct-hole statuses/fold children rather than three concrete family members. This is a
     regression assertion for existing behavior, not a reason to change clan-row production code unless the test exposes
     an unhandled real tree shape.
   - Exercise both `_display_panel_collection.py` and `_display_panel_patches.py` paths so incremental refreshes cannot
     restore the old concrete total after the initial render.

4. **Document and visually verify the cardinalities.**
   - Update the Tribe Side Panels section of `docs/ace.md`: panel-title totals and selected-tribe composition are
     agent-hole counts, while adjacent status chips remain concrete and may sum higher. Clarify that clan-row fold/count
     chrome counts direct clan holes once.
   - Run the relevant Agents-tab PNG comparisons before accepting snapshots. Inspect generated diffs and update only
     goldens whose tribe title or selected-tribe composition changes; clan-row pixels should remain unchanged unless a
     focused regression reveals a real defect.

## Validation

1. Run `just install` before repository commands, as required for an ephemeral workspace.
2. Run focused model, title, panel refresh, clan rendering, and tribe document tests, including:
   - `tests/ace/tui/models/test_agent_summary_status_counts.py`
   - `tests/ace/tui/models/test_agent_tribe_summary.py`
   - `tests/ace/tui/test_agent_panel_titles.py`
   - `tests/ace/tui/test_agent_panel_title_refresh.py`
   - `tests/ace/tui/test_agent_panels_display.py`
   - `tests/ace/tui/widgets/test_agent_display_clan.py`
   - `tests/ace/tui/widgets/test_agent_display_tribe.py`
3. Run the affected family/clan/tribe-panel visual snapshot tests in comparison-only mode, inspect every mismatch,
   update only intentional title/composition goldens, and rerun those tests comparison-only.
4. Run the complete dedicated visual suite with `just test-visual` to catch other family-bearing tribe panels.
5. Run the required repository-wide `just check` and report any demonstrably unrelated baseline failure separately from
   the result of the focused and visual coverage.

## Acceptance criteria

- A tribe containing one sequential family with two concrete members plus one standalone displays a total of `2` in its
  panel border title and `2 holes` in its selected-tribe composition.
- The same tribe's status chips still account for all three concrete agents.
- A clan containing that family and standalone continues to show two direct holes on its clan row, with the family
  descendant available only in nested family detail.
- Hole totals remain stable across clan/family folds, grouping changes, selected/collapsed panels, merged-panel layout,
  and incremental row patches.
- Legacy parallel-family behavior and identity deduplication remain identical to the global `agent_hole_count()`
  contract.
- Focused tests, intentional visual snapshots, and the repository-wide checks pass, aside from any independently
  reproduced pre-existing failure documented in the handoff.
