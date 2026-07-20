---
tier: epic
title: Eliminate canonical project-key leaks from human-facing surfaces
goal: Make every human-facing project and ChangeSpec label use the configured project
  display name while preserving canonical directory keys for identity, persistence,
  paths, joins, filters, and machine-readable contracts.
phases:
- id: display_contract
  title: Establish the identity and display projection contract
  depends_on: []
  size: medium
  description: 'Implement Phase 1: Establish the identity and display projection contract,
    including a batchable display-name snapshot and explicit key-versus-label presentation
    models.'
- id: statistics_views
  title: Repair every Statistics project label
  depends_on:
  - display_contract
  size: medium
  description: 'Implement Phase 2: Repair every Statistics project label while retaining
    canonical project keys for query grouping, joins, filters, and selection state.'
- id: remaining_surfaces
  title: Repair and audit the remaining human-facing surfaces
  depends_on:
  - display_contract
  size: medium
  description: 'Implement Phase 3: Repair and audit the remaining human-facing surfaces
    across the TUI and CLI without changing stored or machine-readable identity.'
- id: regression_audit
  title: Add cross-surface regression coverage and validate the codebase
  depends_on:
  - statistics_views
  - remaining_surfaces
  size: medium
  description: 'Implement Phase 4: Add cross-surface regression coverage and validate
    the codebase with targeted, visual, performance-sensitive, and full checks.'
create_time: 2026-07-20 12:45:58
status: wip
---

# Problem and evidence

The project directory key and the user-facing project name are different concepts:

- Canonical key: `gh_sase-org__sase`
- Configured display name: `sase`

The screenshot at `@.sase/home/tmp/screenshots/20260720_123015.png` shows the canonical key in the Admin Center
Statistics overview under **Top projects**. The same failure reproduces for other GitHub-backed projects such as
`gh_bobs-org__bob-cli`, which should display as `bob-cli`.

This is not isolated to one cell. A read-only audit found the same identity/display conflation in:

- Statistics overview, project tables, ChangeSpec drilldowns, workspace activity, runtime grouping, project filter
  options, and the active-filter heading.
- The human `sase workspace list` single-project heading.
- Human `sase doctor` project headings and project-current summaries.
- `sase changespec search --format markdown` project metadata, ChangeSpec headings/anchors, and running-claim labels.
- Prompt-stash rows and previews.
- Completed launch-approval task labels.
- Project/ChangeSpec selection and replay paths where one string currently serves inconsistently as both selectable
  identity and visible label.
- Additional operation/progress renderers that interpolate raw ChangeSpec names and therefore can expose a canonical
  project-key prefix.

The Statistics feature landed after the earlier display-name consistency work, which explains the regression: its view
rows expose only ambiguous fields such as `project` or `group`, and renderers display those fields directly. The
statistics backend is returning the correct canonical identity; this is a presentation-boundary bug, not an aggregation
or storage bug.

# Design contract

| Concept             | Example                     | Required use                                                                                                                        |
| ------------------- | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Project key         | `gh_sase-org__sase`         | Project lookup, directory layout, persisted records, joins, query filters, color/selection identity, structured API and JSON fields |
| Project label       | `sase`                      | Human-facing TUI cells, headings, choices, notifications, task labels, and human CLI/Rich/Markdown output                           |
| ChangeSpec identity | `gh_sase-org__sase_feature` | Stored ChangeSpec references, exact lookup, workflow operations, anchors or paths that are machine contracts                        |
| ChangeSpec label    | `feature`                   | Human-facing ChangeSpec names after project-prefix humanization                                                                     |

The implementation must follow these invariants:

1. Presentation code must not infer a project label from a directory basename.
2. Any model crossing into a renderer must make identity and presentation explicit (`project_key` plus `project_label`,
   or an equivalent typed projection); an ambiguous `project` string must not silently serve both roles.
3. Missing/deleted project metadata falls back deterministically to the canonical key instead of dropping or
   misattributing data.
4. Sorting is by the visible label with a canonical-key tiebreaker; selection, joins, filtering, and stable colors
   remain keyed by canonical identity.
5. TUI render paths perform no project-record disk I/O, `stat`, or globbing. Display metadata is loaded or refreshed in
   the existing worker/off-thread data path and passed into pure renderers.
6. Canonical identifiers remain visible where the product is explicitly managing or diagnosing storage identity (for
   example, a directory-key detail field), but the primary human label is still the configured project name.
7. Rust aggregation and domain behavior remain unchanged unless investigation proves a wire contract cannot carry the
   required identity. The expected fix is a Python presentation projection over canonical Rust results.

# Phase 1: Establish the identity and display projection contract

Create one reusable projection boundary backed by project lifecycle records and `effective_project_name`, rather than
adding one-off calls in individual cell renderers.

- Add a batch-oriented, immutable project-display snapshot or equivalent mapping that resolves canonical project keys to
  configured display names once per load/refresh.
- Keep the existing single-key convenience behavior for non-rendering call sites, but define explicit fallback,
  refresh/invalidation, and duplicate-label behavior. Cover the case where a nested ProjectSpec display name changes
  without relying accidentally on the projects-root directory mtime.
- Provide presentation helpers for project labels and project-prefixed ChangeSpec/VCS labels that consume the snapshot
  when used in a TUI path.
- Introduce or update presentation row types so identity-bearing consumers receive `project_key` separately from
  `project_label`. Rename ambiguous fields where doing so prevents future misuse.
- Document the boundary in the types and helper APIs: canonical keys enter from storage/query layers, while renderers
  consume projected labels and retain keys only for interactions.
- Add focused unit tests using a deliberately mismatched fixture (`gh_acme__widgets` -> `widgets`), a missing-record
  fallback, two keys sharing a label, and a display-name refresh/rename.

This phase should produce the shared mechanism used by both downstream phases. It must not migrate persisted files or
rewrite canonical query payloads.

# Phase 2: Repair every Statistics project label

Make Statistics the reference implementation of the identity/display contract.

- Resolve the display snapshot in the existing threaded Statistics load/refresh path and pass it into the pure view
  builder; do not resolve names from cell render functions.
- Extend project-bearing Statistics rows to carry canonical keys and visible labels explicitly, including workspace
  activity, ChangeSpec work, aggregate project work, and runtime groups.
- Preserve canonical keys for the ChangeSpec-to-project join, project-scoped query filters, selected filter state, and
  stable project color assignment.
- Render configured project labels in:
  - Overview **Top projects**.
  - Projects-by-runs and Projects-by-ChangeSpecs tables.
  - Project drilldown headings and rows.
  - Activity workspace rows.
  - Runtime grouping when grouped by project.
  - Project filter choices and the active-filter heading.
- Humanize project-prefixed ChangeSpec labels in project drilldowns and runtime ChangeSpec grouping while retaining
  their canonical values for lookup/grouping.
- Sort project choices and rows by visible label with deterministic canonical-key tiebreaking, including
  duplicate-display-name cases.
- Add view-model, renderer, interaction, refresh, and visual snapshot tests. Tests must prove that selecting `widgets`
  still submits the canonical `gh_acme__widgets` filter and that refreshes do not lose the label/key association.

# Phase 3: Repair and audit the remaining human-facing surfaces

Apply the shared projection to every confirmed non-Statistics leak, then complete a systematic audit of user-facing
project and ChangeSpec renderers.

## Confirmed CLI and report repairs

- Use the inventory/project record's display label in the single-project `sase workspace list` heading while keeping its
  paths and structured output canonical.
- Use display labels in the human `sase doctor` report and project-current summary; keep diagnostic identity/data fields
  canonical where they are consumed programmatically or name an exact storage target.
- Humanize `sase changespec search --format markdown` project metadata, ChangeSpec headings, matching links/anchors,
  parents, and running-claim labels. Ensure generated links still target the displayed headings and that JSON/structured
  search results are unchanged.
- Audit human command output that directly interpolates raw `ChangeSpec.name`, `parent`, `claim.cl_name`,
  `project_basename`, `record.project_name`, or equivalent fields (including commit/archive/mail/revert/progress paths).
  Project labels and project-derived ChangeSpec prefixes must be humanized unless the field intentionally communicates
  an exact identifier, path, or command token.

## Confirmed TUI repairs

- Project prompt-stash rows and previews through display metadata loaded alongside stash data off-thread; preserve
  canonical project keys in the stash schema and restore behavior.
- Resolve the completed launch-approval task's project label in its existing worker-side metadata load before updating
  the task queue.
- Separate project selection/replay identity from its label. Project options and ChangeSpec options show
  configured/humanized labels, while saved selections, prompt contexts, lookup, and replay use a canonical project key.
- Audit adjacent history, notification, task-queue, modal, preview, banner, and progress renderers for raw project keys
  or project-prefixed ChangeSpec names. Reuse the already-loaded snapshot rather than introducing render-time filesystem
  access.

## Audit method and exclusions

- Trace all project-bearing presentation paths from canonical sources (`project_name`, `project_basename`, `project`,
  project-group values, ChangeSpec names, and claim names) to `Text`, widgets, option labels, notifications, task
  labels, `print`, Rich tables/panels, and Markdown output.
- Classify each occurrence as **identity-only**, **human-facing label**, or **intentional dual display**. Fix every
  human-label occurrence and record explicit rationale in tests/comments for intentional canonical output.
- Do not rewrite canonical values in project directories, ProjectSpecs, workspace registries, stash/history storage,
  ChangeSpec files, task metadata, artifact paths, exact commands, query payloads, JSON, or stable machine contracts.
- Do not treat storage/debug logs that are never rendered to a user as display bugs. Human log dashboards that already
  derive `sase` from checkout metadata remain unchanged unless a mismatched-key fixture proves otherwise.

Add focused regression tests for each repaired surface, always using a project whose canonical key differs from its
display name. Where a surface supports both human and JSON output, assert the human label and the unchanged canonical
machine value side by side.

# Phase 4: Add cross-surface regression coverage and validate the codebase

Close the regression path rather than relying only on isolated snapshots.

- Add a shared mismatch-key fixture/factory that downstream tests can reuse without accidentally testing the easy
  `key == label` case.
- Add a presentation-boundary regression suite or narrowly scoped architecture check covering the audited human
  renderers. It should fail when a known canonical-only project/ChangeSpec field is newly rendered without projection,
  while allowing explicitly documented identity/path/machine-output sites.
- Re-run a source audit for raw project/key interpolation after the fixes and review every remaining hit against the
  classification from Phase 3. Avoid a blind ban on canonical strings because they are required internally.
- Exercise representative end-to-end human output for:
  - Statistics overview, project filter, project drilldown, activity, and runtime grouping.
  - Single-project workspace listing.
  - Doctor's human report.
  - ChangeSpec Markdown search.
  - Project/ChangeSpec selection, prompt stash, and launch-approval task updates.
- Run focused unit and TUI interaction suites after each area changes, then the dedicated PNG visual suite for
  intentional Statistics/TUI rendering changes.
- Run `just install` before repository-wide validation, accept visual snapshots only after inspecting
  actual/expected/diff artifacts, and finish with `just check`.

# Acceptance criteria

- No human-facing Statistics label contains `gh_sase-org__sase` when the configured name is `sase`; the same is true for
  any mismatched canonical/display project fixture.
- Every confirmed CLI/TUI leak above displays the configured project name or humanized ChangeSpec name.
- A user selection made through a display label still operates on the correct canonical project when labels are
  duplicated or changed.
- Persisted identities, paths, query filters, joins, structured JSON/data contracts, and backend aggregations remain
  canonical and backward compatible.
- TUI rendering performs no new synchronous project metadata I/O and remains responsive during refresh.
- Targeted tests, inspected visual snapshots, the presentation-boundary regression audit, and `just check` all pass.

# Risks and mitigations

- **Accidentally querying by display name:** Keep a separate canonical key on every interactive/view row and assert
  outgoing filters in tests.
- **Duplicate display names:** Use canonical keys for identity and deterministic tiebreaking; labels are never assumed
  unique.
- **Stale labels after rename:** Define snapshot refresh/invalidation semantics and test a ProjectSpec name change.
- **TUI regressions from synchronous lookup:** Resolve snapshots only in worker/off-thread loaders and keep renderers
  pure.
- **Over-humanizing exact identifiers:** Maintain the audit classification and preserve canonical values in paths, exact
  commands, persistence, and machine-readable output.
- **Visual churn:** Limit snapshot updates to intentional label changes and inspect renderer diffs before acceptance.
