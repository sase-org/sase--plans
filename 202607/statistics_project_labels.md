---
tier: tale
title: Repair every Statistics project label
goal: Make every human-facing Statistics project and ChangeSpec label use configured
  display names while canonical keys continue to drive queries, joins, filters, selection,
  and stable colors.
create_time: 2026-07-20 13:12:16
status: wip
prompt: 202607/prompts/statistics_project_labels.md
---

# Plan: Repair every Statistics project label

## Context and constraints

Statistics currently passes canonical project keys from the Rust query payload into ambiguous Python view fields and
renders those fields directly. The shared project-display contract already provides immutable `ProjectDisplaySnapshot`
values, deterministic canonical-key fallback, visible-label sorting, duplicate-label support, and pure helpers for
project and project-prefixed ChangeSpec labels. This phase will make Statistics the reference consumer of that contract
without changing Rust aggregation, persisted identities, or query wire formats.

All lifecycle metadata reads must remain in the existing threaded Statistics load/refresh path. View-model construction
and Rich/Textual rendering must remain pure: no renderer may load, stat, or glob project records. A missing or deleted
project record will render its canonical key. Canonical keys will continue to own joins, outgoing project filters,
selected filter state, and categorical colors.

## Implementation

1. Extend the Statistics load result and pure view-building boundary to accept the fresh display snapshot loaded by the
   worker. Keep query arguments canonical, load the snapshot once per refresh, and carry enough immutable projection
   metadata back to the UI for an active filter heading to refresh its label without render-time I/O.
2. Replace ambiguous project-bearing view-model fields with explicit identity and presentation values across workspace
   activity, ChangeSpec work, aggregate project work, and runtime groups. Build project rows by joining ChangeSpecs on
   canonical project keys, derive labels through the supplied snapshot, humanize project-prefixed ChangeSpec/runtime
   labels, and retain canonical group values for interaction or future lookup. Preserve deterministic fallback for
   unknown keys.
3. Update Statistics renderers to show only projected labels in Overview Top projects; Projects-by-project,
   Projects-by-ChangeSpec, and drilldown rows; Activity workspaces; project/ChangeSpec runtime grouping; project filter
   choices; and the active-filter heading. Use canonical project keys for categorical colors and filter cycling, and
   order user-facing project choices/row tie-breaks by visible label followed by canonical key so duplicate labels stay
   distinct and deterministic.
4. Keep refresh and selection behavior coherent: an unfiltered result refreshes the ranked canonical filter-option
   identities, every result refreshes the immutable display snapshot used by headings, and stale worker results continue
   to be rejected by canonical filter identity. Selecting a visible label must call both run and activity queries with
   its canonical project key, including after range changes and refreshes.

## Verification

- Add view-model tests with a deliberate mismatch (`gh_acme__widgets` → `widgets`) covering explicit key/label fields,
  missing-record fallback, canonical ChangeSpec-to-project joins, project and ChangeSpec runtime groups, visible-label
  ordering, and duplicate display names with canonical-key tie-breaking.
- Add loader tests proving one display snapshot is resolved in the worker-side load path and is refreshed/reassociated
  without changing canonical query filters.
- Add renderer and Textual interaction tests proving all listed Statistics surfaces show labels, stable colors use
  canonical keys, the active-filter heading is human-facing, and choosing `widgets` submits `gh_acme__widgets` across
  selection, range change, and refresh.
- Update deterministic Statistics visual fixtures to use mismatched canonical keys and configured labels, then run the
  focused view and pane tests plus the dedicated Statistics PNG snapshot cases. Inspect any actual/expected/diff
  artifacts before accepting intentional snapshot changes.
- Run `just install`, targeted tests during implementation, `just test-visual` for the final visual verification, and
  finish with `just check`.

## Risks and safeguards

- Prevent display names from entering query or join paths by keeping canonical keys explicit on every
  interactive/project-bearing row and asserting outgoing calls.
- Avoid identity collisions when labels duplicate by never selecting by label and by using canonical keys as
  deterministic tie-breakers.
- Avoid stale labels by loading a fresh snapshot during every existing Statistics refresh rather than relying on
  directory mtimes or the convenience cache.
- Avoid TUI responsiveness regressions by performing lifecycle I/O only inside the existing thread worker and passing
  immutable data into pure renderers.
