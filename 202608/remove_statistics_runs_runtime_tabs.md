---
tier: tale
title: Remove Runs and Runtime from Admin Center Statistics
goal: The Admin Center exposes a coherent seven-view Statistics experience without the low-value Runs and Runtime views.
proposed_by: bbugyi200.athena.s4
create_time: 2026-08-02 11:47:35
status: done
---

- **PROMPT:**
  [prompts/202608/remove_statistics_runs_runtime_tabs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/remove_statistics_runs_runtime_tabs.md)
- **AGENTS:**
  - [bbugyi200.athena.s4](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.s4.md)
- **COMMITS:**
  - [bcefbb8](https://github.com/sase-org/sase/commit/bcefbb8e40d48578b3aa221b6bd343669b13a2e9) — feat(ace)\!: remove
    Statistics runs and runtime tabs

# Remove the Runs and Runtime Statistics views

## Goal

Simplify the SASE Admin Center's Statistics pane by removing the low-value **Runs** and **Runtime** numbered views while
preserving the useful run-derived summaries and drill-downs that remain elsewhere. The finished pane should expose seven
views in this order: **Overview**, **Runners**, **Projects**, **Providers**, **Activity**, **XPrompts**, and **Plans &
Questions**.

This is a presentation-layer change. Keep the shared statistics query, Rust wire contract, and Python view-model
builders for run and runtime data intact: the remaining views still consume the composite run payload, and deleting
shared backend statistics is outside the requested scope.

## Implementation

1. Make the seven-view catalog the single source of truth for navigation and presentation.
   - In `src/sase/ace/tui/modals/statistics_pane_data.py`, remove `runs` and `runtime` from `StatisticsView`,
     `VIEW_ORDER`, and every view-keyed label/description map. Remove the Runtime-only grouping order and report
     grouping support only for Projects and XPrompts.
   - Recalculate the numbered strip's responsive boundaries in `statistics_pane.py` from the actual seven-view full and
     compact line widths (111 cells for full labels and 75 for compact labels), so the shorter strip does not switch to
     abbreviated tiers prematurely. Keep micro labels for genuinely narrower layouts.
   - Make prefix-number selection and its hint/help copy use the number of entries in `VIEW_ORDER` rather than
     hard-coded `1-9` text or digit sets. Valid selections become `1` through `7`; an out-of-range digit must not select
     a hidden or removed view. Preserve bare-number Admin Center section navigation when the Statistics prefix is not
     armed.

2. Remove the two views' pane-only behavior without widening the change into the statistics backend.
   - Delete the `runs` and `runtime` rendering branches and their now-dead Rich render helpers/constants from
     `statistics_pane_views.py`, plus their metric legends from `statistics_pane_legends.py`.
   - Remove mutable Runtime grouping state from `StatisticsPane`, `StatisticsViewData`,
     `StatisticsPanePresentationBase`, and `StatisticsHelpModal`; remove Runtime-specific group-chip text, help-control
     text, stale-result checks, and the reload-on-group-change path. Projects and XPrompts must retain their existing
     in-memory group cycling behavior.
   - Keep `load_statistics_view()` on the existing composite query/build path and continue using a fixed runtime-group
     value internally if the current Rust query contract requires one. Do not delete or change `sase.stats` run/runtime
     models, query bindings, or Rust APIs merely because their dedicated Admin Center renderers are gone.

3. Preserve meaningful Overview behavior after Runs disappears.
   - Keep all five summary tiles and their current values.
   - Retarget **Agents Run**, **Success Rate**, and **Commits** to **Projects**, whose rows expose the corresponding
     run, success, and commit breakdowns. Keep **Plans Proposed** and **Questions** targeting **Plans & Questions**.
   - Update the tile-navigation contract and tooltips through `OVERVIEW_TILE_TARGETS`; do not leave clickable controls
     pointing at removed view IDs or silently inert.

4. Bring behavior tests, help coverage, and fixtures in line with the seven-view contract.
   - Update the shared Statistics test helpers and loader mocks to omit Runtime grouping as pane state while still
     producing a valid composite payload for the unchanged statistics builders.
   - Revise navigation tests to assert the exact seven-view order and numbering across keyboard cycling, mouse clicks,
     and prefix selection. Cover the new `2 = Runners` and `7 = Plans & Questions` mappings, dynamic `1-7` hints/help,
     an out-of-range prefixed digit, and unchanged bare-digit top-level navigation.
   - Replace Runs/Runtime grouping, rendering, legend, refresh, and scope assertions with coverage that Projects and
     XPrompts are the only grouping views and that their group changes still reuse the loaded composite result. Retain
     range/project/xprompt stale-result and off-thread refresh coverage.
   - Update Overview tile tests to assert the new Projects drill-down targets and verify navigation reuses the loaded
     result rather than triggering another query.

5. Update the documented and visual user contract.
   - In `docs/configuration.md`, `docs/telemetry.md`, and `docs/ace.md`, describe seven numbered views, their new
     numbers, `0` then `1-7` direct selection, grouping only in Projects/XPrompts, and the revised Overview tile
     destinations. Remove the dedicated Runs/Runtime contents and grouping descriptions without implying that
     run-derived metrics were removed from Overview, Projects, Providers, or XPrompts.
   - Remove the Runs and Runtime PNG snapshot tests and their obsolete goldens
     (`config_center_statistics_runs_120x40.png` and `config_center_statistics_runtime_120x40.png`). Regenerate and
     review every remaining Statistics golden affected by the shorter/renumbered strip, including full-width, narrow,
     loading/empty, help, Projects, Runners, and XPrompts states.

## Acceptance criteria

- The Statistics strip shows exactly seven views in the specified order; cycling, clicking, and `0`-prefixed selection
  cannot reach Runs or Runtime.
- Number labels, help, hints, and documentation consistently map `1` through `7`, and responsive strip tests prove the
  full, compact, and micro tiers fit at their intended boundaries.
- `g` is visible and effective only for Projects and XPrompts. No Runtime group state remains in the pane or its help,
  loading, and freshness contracts.
- All five Overview tiles remain visible; the first three open Projects and the last two open Plans & Questions without
  causing a second statistics load.
- No Runs/Runtime-specific pane renderer, legend, visual test, or golden remains, while shared `sase.stats` run/runtime
  query and model APIs remain unchanged.
- User documentation and reviewed PNG goldens match the seven-view UI.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral SASE workspace.
2. Run the focused Statistics unit/interaction suite (all `tests/ace/tui/test_statistics_*.py` files) and fix any stale
   view IDs, numbering assumptions, loader signatures, or grouping expectations.
3. Update the targeted Statistics PNG corpus with
   `just test-visual -- --sase-update-visual-snapshots tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py`,
   inspect the changed images/diffs, then rerun the same visual file without update mode.
4. Run `just check` and resolve all formatting, lint, type, validation, unit, and visual failures before handing off.
