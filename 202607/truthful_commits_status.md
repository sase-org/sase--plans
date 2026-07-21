---
tier: tale
title: Truthful commits-pane status and cache behavior
goal: The Artifacts Commits pane reports capped result sets honestly, exposes the
  active row limit when it affects the display, and preserves stable filter caching
  while relative date windows continue to move with refresh time.
bead: sase-8h.3
parent: sase/repos/plans/202607/commits_filter_correctness.md
create_time: 2026-07-21 11:23:10
status: wip
prompt: '[202607/prompts/truthful_commits_status.md](prompts/truthful_commits_status.md)'
---

# Plan: Truthful commits-pane status and cache behavior

## Context

Phases 1 and 2 of the commits-filter correctness epic have already made date bounds text-stable and re-anchorable and
added truncation metadata to `VcsLogResult`. The remaining phase is presentation and pane-state work: the commits filter
bar still unconditionally reports authoritative results as exact, the default `limit:40` stays hidden when it clips the
timeline, and the new equality and collection-time resolution contracts need end-to-end regression coverage. This work
stays in the Python TUI and documentation; it does not alter the Rust core or create a new refresh path.

## Implementation

### Truthful result coverage

Use the existing aggregate/provider truncation signals as the authoritative source for whether collection may have
omitted rows. When the pane's exact in-memory matcher produces more rows than the visible `limit:`, preserve that fact
on the displayed `VcsLogResult` as well, including the raised candidate cap used by `until:` collections. Centralize the
commits-pane status decision so authoritative and live-preview paths agree: complete results render an exact count,
while potentially truncated results render a lower-bound count such as `40+ matches` with the distinct `capped` coverage
label. Extend the shared filter-bar status API only as needed to express the lower-bound marker, with defaults that
leave Plans and other callers unchanged.

Update `snapshot_covers` to reject potentially truncated snapshots even when the semantic filter values are equal,
rather than inferring coverage from row count and collection limit. Preserve `preview` for genuinely provisional local
filtering and use `capped` when missing rows, rather than reconciliation, is what prevents an exact count.

### Visible active cap and stable pane state

Surface `limit:<N>` in the commits pane only while the displayed result is potentially truncated; do not add a cap
indicator for exact bounded results or `limit:all`. Reuse the existing info/filter rendering and refresh flow so this
adds no event-loop work and no alternate collection path.

Keep cache keys, in-flight comparison, and live reconciliation based on the text-normalized `CommitLogFilterValues`
supplied by phase 1. Opening an unchanged persistent filter must not increment the generation or schedule a collection.
When locally re-filtering a cached relative-window snapshot, resolve a fresh reference time so an aging `since:24h`
preview can narrow immediately; authoritative collection results still use the single resolved clock captured by their
worker. Regular refreshes continue through the existing thread worker and therefore resolve the same semantic
`filter_spec` against a new operation time.

### Help, documentation, and defaults

Update the Commits filter completion hints, Artifacts help-modal copy, `docs/ace.md`, and the `sase vcs log` and Commits
configuration sections in `docs/configuration.md` to explain that day-granular `until:` includes the named day, capped
counts are lower bounds, and `limit:N`/`limit:all` controls the cap. Review the bundled default-query comment/schema
descriptions and change them only where needed to keep the documented sliding-window behavior accurate.

## Verification

Add focused coverage across the existing commits suites for:

- exact versus provider/aggregate/visible-limit truncation status, including `N+ matches · capped` and conditional
  `limit:N` visibility;
- `snapshot_covers` behavior with truncated snapshots and live previews;
- opening/submitting an unchanged relative query without a generation bump or extra collection, stable cache lookup
  after reparsing, and a later refresh resolving a newer relative window;
- fresh-time narrowing of cached relative-window previews without mutating the semantic cache key;
- completion and help text for inclusive day-granular `until:` semantics.

Add or adapt an `artifacts_commits_*` PNG case that visibly pins the capped status and active cap. Run the focused
unit/interaction tests first, then the dedicated visual suite. Inspect actual/expected/diff artifacts before accepting
intentional PNG updates with `--sase-update-visual-snapshots`. Finally run `just install` followed by `just check`, and
re-run any affected focused suite after formatting or snapshot updates.

## Risks

The main edge is distinguishing a bounded but complete result from a result that actually lost rows: cap visibility must
follow truncation metadata or an observed presentation slice, never merely `len(commits) == limit`. Relative window
preview timing must not replace the collection worker's shared clock or make old cached data look authoritative; it may
narrow an older snapshot while the existing reconciliation path obtains the refreshed result. Visual changes should be
limited to the intended status/cap surfaces.
