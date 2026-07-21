---
tier: tale
title: Persistent configurable filter for Artifacts commits
goal: 'Make the Artifacts Commits sub-tab continuously explain and control its commit
  selection through a visible, configurable default query while keeping collection
  lazy, navigation responsive, and the summary limited to matched repositories.

  '
create_time: 2026-07-21 08:35:14
status: done
prompt: 202607/prompts/commits_persistent_default_filter.md
---

# Plan: Persistent configurable filter for Artifacts commits

## Context and desired contract

The Commits pane already has a capable live query editor, but it is normally hidden and its empty-query model currently
excludes sidecars by itself. That makes the effective scope difficult to discover and prevents the requested
`sidecar:false since:24h` text from being the actual reason sidecars are absent. The existing information header also
repeats filter state and renders every resolved repository, including repositories with a zero count after the active
query is applied.

Keep this work scoped to the ACE Artifacts Commits experience. The `sase vcs log` CLI should retain its existing sidecar
opt-in behavior, collection should remain lazy until the Commits pane is first activated, and all VCS collection and
diff work must remain on the established background-worker paths.

## Configuration and query semantics

- Add `ace.artifacts.commits.default_query` to the bundled SASE configuration, public JSON schema, configuration
  inventory/documentation, and regression coverage. Its bundled value is the literal query `sidecar:false since:24h`.
  Resolve it once from the already-loaded merged ACE config and inject the parsed value into the pane before its first
  collection; do not read or stat config files from widget render, focus, or key-event paths.
- Validate configured text with the same commit-query parser used by the live editor. A malformed runtime value should
  produce a clear diagnostic and fall back to the bundled query instead of leaving the pane in a partially parsed state
  or crashing ACE. Document when a changed initial query takes effect.
- Change the commit-filter value/query contract so an unconstrained query includes sidecar repositories. Canonical query
  rendering should explicitly show the current `sidecar:true` or `sidecar:false` token, making collection's existing
  `include_sidecars` argument solely a projection of parsed query state. Initializing the pane with the bundled query
  will therefore be the only reason sidecars are hidden by default in this tab. Preserve directional cache/snapshot
  coverage: a sidecar-inclusive result may preview an excluding query, but an excluding result cannot authoritatively
  preview an inclusive one.
- Keep the compatibility sidecar toggle, but make it rewrite the same visible canonical query and use the normal filter
  reconciliation path. Clearing or editing the query, toggling sidecars, changing project scope, refreshing, and
  fetching must all continue to agree on one `CommitLogFilterValues` state.

## Persistent filter interaction

- Convert the Commits filter bar from a transient row into a permanently visible row at the top of the sub-tab. Its
  inactive state shows the committed canonical query without stealing focus; `/` and the local `f` shortcut focus that
  same row for live editing rather than mounting or revealing another control.
- Retain the current debounced, in-memory live preview and background reconciliation behavior. `Enter` commits the query
  and returns focus to the timeline while leaving the row visible. `Escape` restores the last committed query and
  result, also leaving the row visible. Programmatic changes such as the sidecar toggle must update the displayed text
  without emitting a false user edit, starting duplicate collections, or disturbing timeline selection.
- Keep match/error status useful in both editing and inactive states, and preserve completion behavior and focus
  routing. Adjust the shared filter-bar API only where a reusable persistent/non-focusing update primitive is useful;
  Plans should retain its existing transient behavior.
- Update the Commits layout styles for the permanently allocated filter row and completion overlay, checking constrained
  terminal sizes as well as the normal split-pane layout. The added row must not introduce synchronous work on
  navigation or rendering paths.

## Concise scope and repository summary

- Remove the standalone `Sidecars hidden` / `Sidecars included` label from the Commits information header and rely on
  the persistent canonical query for that state. Avoid repeating the query's filter tokens in the header; retain the
  project scope, refresh/warning state, presence legend, and useful remote freshness information.
- In the Commits-pane legend, calculate counts from the fully filtered and row-limited result and render only
  repositories represented by at least one displayed commit. Remote-state annotations should follow that same visible
  repository set. Keep the shared CLI renderer's output contract unchanged, either through a TUI-specific projection or
  an explicit legend option, so this concision does not silently alter `sase vcs log` output.
- Preserve stable repository colors, commit selection by identity, empty-state messaging, and warning visibility when
  the query matches no commits.

## Documentation and verification

- Update the ACE guide, contextual help, and configuration reference to show the always-visible row, the bundled/default
  configuration field, explicit sidecar tokens, and the `Enter`/`Escape` behavior. Keep the VCS CLI docs clear that
  CLI-side sidecar defaults are independent of the Commits-pane query.
- Extend pure query tests for empty-query inclusion, explicit canonical sidecar serialization, configured-default
  parsing, and round trips. Extend config-schema/config-resolution tests for the new nested field, overrides, wrong
  types, and malformed query fallback.
- Extend Commits-pane behavior tests to prove the bar is visible before and after edit sessions, the first collection
  receives a 24-hour bound plus `include_sidecars=False`, custom configuration seeds the first query, the sidecar toggle
  rewrites the row, and the header no longer carries the old prose. Add a mixed result where one resolved repository has
  no commits after filtering and assert that only nonzero repository counts render.
- Refresh the Commits PNG goldens for the persistent row and concise legend, including default, empty, edit/completion,
  sidecar-toggle, narrowed-query, and parse-error states. Make fixed-time fixtures independent of the new rolling
  24-hour default, and retain focused small-terminal/layout coverage.
- Before implementation verification, run `just install`. Then run the focused query, schema, Commits widget/pane, help,
  and visual tests; capture the existing Artifacts navigation benchmark before and after the layout change and confirm
  key-to-paint p95 remains below the documented 16 ms target. Finish with `just test-visual` as needed for intentional
  golden updates and the mandatory `just check`.

## Acceptance criteria

- Opening Artifacts → Commits always shows the effective query at the top, with the bundled configuration displaying
  `sidecar:false since:24h`.
- Sidecars are excluded because that query parses to exclusion; an empty or `sidecar:true` query includes them, and the
  visible row always reflects the toggle.
- The old sidecar prose is absent, and the summary lists only repositories with at least one commit in the currently
  displayed filtered result.
- A configured valid default query controls the first collection, malformed configuration degrades to the bundled query
  with a diagnostic, and no new config or VCS I/O occurs on render/navigation paths.
- Existing lazy activation, live preview, cache directionality, selection, completion, fetch/refresh, CLI output, visual
  snapshots, and TUI performance contracts remain covered and passing.
