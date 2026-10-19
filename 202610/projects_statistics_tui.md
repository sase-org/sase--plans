---
tier: tale
title: Migrate Statistics Projects to the shared project report
goal:
  Complete sase-1m3.11 with progressive project statistics, shared rendering, safe
  refreshes, and verified TUI coverage.
size: medium
proposed_by: bbugyi200.apollo.sase-1m3.11
proposed_with: codex/gpt-6-astra
bead: sase-1m3.11
create_time: 2026-10-19 09:05:45
status: wip
---

- **PARENT:**
  [202610/sase_stats_project.md](https://github.com/sase-org/sase--plans/blob/main/202610/sase_stats_project.md)
- **BEAD:**
  [sase-1m3.11](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1m3/sase-1m3.11.md)

# Migrate Statistics → Projects to the shared project report

Implement assigned phase `sase-1m3.11` of `sase-1m3`, using
`plan:202610/sase_stats_project.md`, especially sections 4, 6.5, 8, and 19. The Projects
sub-tab must report default-branch output, committed plans, lifetime code, and this
machine's runs through the existing project-stats loader and shared renderers. Its
current run-recorded Commits table gives way to these explicitly scoped facts.

## Scope and verified starting point

This is a medium tale for one coding agent: the Rust facets, Python loader, and Rich
renderers already exist, and the remaining work is one bounded TUI integration with
tests and documentation. On inspection the assigned phase was in progress with no
previous implementation/remainder notes. The checkout was clean. The renderer phase
`sase-1m3.9` is closed and its shared renderer goldens passed. No implementation changes
were made while authoring this plan.

Relevant existing interfaces:

- `src/sase/stats/project/loader.py`: `load_project_stats(query, *, facets=...)`.
  `ProjectStatsQuery` carries range and project selectors; fetching is a CLI preflight,
  never a loader operation. `ProjectStatsView` contains immutable decoded records, raw
  `data`, `fingerprint`, `facets`, display names, machine, and generated time.
- `src/sase/stats/render/compose.py`: `render_project_stats(view, group_by, ctx)`
  composes the card, selected table, coverage footer, and legend. `RenderContext`
  accepts width, clock, timezone, machine, color, and Unicode explicitly.
- `src/sase/stats/loader.py`: `load_statistics_view` currently executes the composite
  run/activity/previous-window queries for every view. `StatisticsViewData.views` is
  currently mandatory, and the TUI assumes it exists in several paths.
- `src/sase/ace/tui/modals/statistics_pane.py` owns a debounced thread worker and
  30-second refresh timer. The actions, presentation, and view mixins currently reuse
  the composite result when changing most views. Projects has three legacy groupings.
- `src/sase/stats/registry.py` already has `PROJECT_REPORT_GROUPINGS` and
  `PROJECT_REPORT_LEGENDS`, while the live Projects spec still uses legacy definitions.
- `tests/stats/_project_render_fixtures.py` supplies frozen report examples;
  `tests/ace/tui/_statistics_pane_helpers.py` and the visual statistics helpers supply
  the existing pane fixtures.

Honor the final epic decision `stats_decision_record = no`. No memory files may be
edited, and no decision is being re-asked. The required `PROPOSED FOLLOW-UP:` note for
the skipped decision record has already been appended to `sase-1m3.11`. Leave the CLI
implementation and epic-wide acceptance work to their assigned phases. Shared backend
behavior stays in Rust; this integration must not recreate counting, attribution,
filtering, or aggregation in Python.

## Implementation

### 1. Give Projects its own loader result

Make `StatisticsViewData.views` optional and add an optional `project_stats` field.
Early-route `load_statistics_view("projects", ...)` to exactly one `load_project_stats`
call with `facets={"output", "runs"}`. Build its query from the selected range and
canonical project filter, with fetch disabled. Return `views=None`, the project report,
its display snapshot, and compatible result metadata. This branch must not call the old
run/activity/previous-window queries or build the composite models. The other seven
views keep their existing loader behavior, including Overview's run-derived Top projects
and Perf's specialized data.

Audit every `result.views` access in the Statistics modules and tests. Narrow result
types at the rendering/action boundaries rather than manufacturing empty composite
models. In particular, fix project option discovery, the current-project seed, filter
cycling, empty-state detection, macro focus, and navigation. A project with zero local
runs can still have output or lifetime code and must remain visible. When all projects
are loaded, cache project-filter keys from `project_stats.projects`; keep that option
list when a filtered query completes.

Switching into Projects must load its report; switching back must obtain a valid
composite result. Existing transitions between compatible composite views may retain
their cheap reuse. Never paint a result for the wrong view/filter/range while a new
selection is loading, and preserve the one-shot current-project seeding behavior.

### 2. Implement progressive loading and safe refreshes

Keep timer, debounce, selection, and worker completion handlers synchronous and thin.
All source resolution and loading happens in thread workers. The initial worker paints
Output, Runs, and available lifetime history. Only after that paint, start a separate
worker group requesting `facets={"code"}`; the shared card displays `measuring code…` in
the meantime.

Carry an explicit selection identity and load generation through both workers, including
the resolved range, view, project filter, and requested facets. Reject late results and
errors from canceled generations, changed selections, or unmounted panes. Keep worker
groups distinct so the Code worker does not cancel the primary worker or current-project
seed. Coalesce periodic refreshes rather than repeatedly restarting unfinished work.
Cancel both worker lanes on teardown and retain the existing hidden-tab refresh guard.

Merge the Code result as a presentation adapter, matching projects by canonical key and
repos by ID; do not zip rows or recompute totals. Carry Code/snapshot/records data,
facet-specific issues, and coverage without replacing Output, Runs, model attribution,
Patch rows, or the raw wire's existing non-Code fields with the nulls of a Code-only
response. Retain the primary result's project membership. Preserve consistent decoded
and raw representations, source metadata, and facet completion state. Guard against
combining mismatched source revisions where the two responses expose different refs;
revalidate instead of presenting mixed revisions as one measurement. Keep this helper
local to the TUI unless an existing neutral presentation seam already fits.

Every active 30-second tick still revalidates through the shared loader. Compare the
appropriate facet fingerprint and selection identity before replacing state or
rebuilding the report body. Track facet completion separately: equal fingerprints must
not suppress the first Code completion, and a primary refresh must not downgrade an
unchanged full report to a loading card. Use actual range identities; do not freeze
rolling ranges to force equality. Group changes and width changes remain valid
presentation repaint reasons even when fingerprints match.

On refresh failure, retain the last good report for that selection and show
`stale · <time>` with the failure visible. A first-load failure shows an error rather
than measured zeros. A Code failure must keep Output/Runs usable, end the misleading
permanent measuring state, and remain retryable. Clear the stale marker after recovery.
Do not attribute a failed new selection to the old selection's report.

### 3. Use shared rendering, grouping, and terminology

Replace the old tables in `statistics_pane_projects.py` with the shared
`render_project_stats` composition, including its coverage footer. Construct an
injectable `RenderContext` using the report's machine/timezone and the actual body
content width. Avoid render-path config or filesystem reads. Resize must rerender for
the shared wide, compact, and stacked layouts, including after lazy mount settles.

Move still-used `_legend_note`, `_project_cell`, and `_format_timestamp` into
`statistics_pane_views.py`; remove obsolete Projects-only helpers. Use `project_accent`
for Statistics project cells and the project scope chip, keeping macro coloring intact.

Use `ProjectsGroupBy = Literal["project", "repo", "model", "patch"]`, deriving the live
spec's grouping IDs/order from `PROJECT_REPORT_GROUPINGS` while preserving its
`StatisticsGrouping` label records. Update descriptions, labels, and the help modal from
the registry; the live Projects legends become `PROJECT_REPORT_LEGENDS`. Use the
approved description: “Compare what projects produced, which models produced it, their
code, and this machine's runs.”

`g` changes only presentation. Several projects default to project grouping; one project
defaults to model, skips project when cycling, and permits repo/model/patch. Normalize
the effective grouping consistently for the table, scope chip, and help, while
respecting valid explicit user choices. Patch grouping is the shared Project → Patch
tree and replaces drilldown.

Relabel Overview's Commits tile `Run commits (this machine)` and update its legend and
tile-caption metadata. Preserve its numbers and click-through to Projects. Explain that
this Overview metric remains run-recorded, distinct from the report's default-branch
commits and its code-only Patch Run commits.

### 4. Update behavioral tests and documentation

Extend the existing Statistics tests and fixtures rather than testing private structure
alone. Required regression evidence:

- Projects makes one primary project-loader call with Output/Runs and zero legacy
  queries; the other views retain their loader contracts.
- Output/Runs paint while a controlled Code worker is blocked; Code completes without
  erasing other facets or mismatching reordered rows. Cover unavailable and run-only
  projects, local plan stores, and zero-run projects with meaningful output.
- Late successes and errors after range/filter/view changes are ignored; obsolete
  workers cannot overwrite a newer generation. Exercise rapid navigation into and out of
  Projects, filter cycling/seeding, hidden ticks, and teardown.
- Unchanged fingerprints cause no report-body update; first Code completion still
  paints; changed facets update; failed refreshes retain data and recover from stale.
- All four groupings, one-project grouping rules, and `g` without loader calls work.
  Existing macro/perf/navigation/bindings behavior remains covered.
- At identical context and width, the pane's report renderable exported to plain text
  equals `render_project_stats(view, group, ctx)`. Cover compact/stacked widths and
  resize, project-accent seams, help/legends, and the Overview relabel.

Adjust tests under `tests/stats/` and `tests/ace/tui/test_statistics_*.py`, especially
loading, rendering, filters, scope header, help, bindings, navigation, and current
project seeding. Preserve the `statistics_pane.load_statistics_view` test seam and add
an explicit seam for the Code worker so visual fixtures never read real repos.

Update existing-feature Reference prose in `docs/reference/tui/admin-center.md` and
`docs/reference/tui/machines-and-tools.md`. Update the contributor contract in
`docs/contribute/telemetry.md`: data sources, three scopes/clocks, groupings, empty
states, first paint, freshness, and `(no Patch)`. Remove stale drilldown references in
Statistics keymap descriptions/docs. No key change is needed; update
`src/sase/default_config.yml` only if a source description/configuration requires it,
then regenerate generated documentation through `just fix`.

## Verification and completion

Read the required ACE/TUI/performance/screenshot/docs/testing memory through
`sase memory read` before implementation. Follow `docs/contribute/writing-docs.md`. Use
`sase tool run check` for the agent verification gate; do not run `check-full`. Use
`/sase_monitor` for long verification commands and wait for the handoff command itself
to exit. Run `just fix` before a final verification monitor.

Update deterministic visual fixtures to exercise Projects project/model/repo/patch, the
measuring-Code state, and compact layouts. Replace the drilldown snapshot test with
patch and add model/repo coverage. Pin epoch, timezone, machine, fixture inputs,
current-project seed, and waits for the intended primary/final loaded state. Regenerate
the Projects and Overview goldens and affected help/scope surfaces using:

```sh
just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py
```

Run the mutating visual command through `/sase_monitor`, then inspect its retained
report and every created/removed golden and each update group. An exit-zero `partial`
result is not proof that all images are current. Targeted runs cannot authorize stale
golden pruning: run the full visual maintenance command if needed to remove the orphaned
drilldown golden, and inspect unrelated changes before accepting any. Do not use a
prepared completion that skips visual review. Prove host independence with two targeted
`--check` runs, one with `TZ=UTC`, with neither reporting unstable nodes.

Extend the existing refresh soak to Projects with controlled slow workers and observed
completion conditions. Collect responsiveness evidence with `SASE_TUI_PERF=1` across
several refresh ticks, targeting p95 below 16 ms and no new loop/pump stall rows.
Statistics has no row cursor: exercise its actual navigation/scroll controls and j/k
where supported, and state precisely what was measured. Record measurements and check
results on the assigned phase.

Before closing, run `sase bead epic-symbols sase-1m3.11` again. No entries existed
during planning, but consume/remove any that now apply, or re-key remaining legitimate
deferred consumers to a still-open later phase/parent. Do not leave exemptions keyed to
a phase that is about to close.

Record discovered out-of-scope work only with
`sase bead note sase-1m3.11 'PROPOSED FOLLOW-UP: ...'`; create no beads. For a check
failure reproduced identically on the clean base, record the reproduction and any
existing tracking bead, then close this phase as the user instructed. The renderer phase
notes mention pre-existing hint/pin test failures; that is context, not proof that any
new failure is unrelated.

After implementation and verification, close only the assigned phase:

```sh
sase bead close sase-1m3.11 --note "<implemented behavior, tests, visual review, and responsiveness evidence>"
```

Do not set its status by hand or close the parent epic or this tale's ancestor beads.
Leave final commits and publication to the host via `/sase_final` at the end of the
coding turn. If launched attached to an already-closed phase, first confirm whether the
implementation landed and reopen only if genuinely unfinished.
