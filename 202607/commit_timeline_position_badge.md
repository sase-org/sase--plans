---
tier: tale
title: Commit timeline position badge
goal: Make the selected commit position and truthful matched-entry total immediately
  visible beside the repository legend while removing the duplicate count from the
  filter bar.
create_time: 2026-07-21 15:16:18
status: wip
---

- **PROMPT:** [202607/prompts/commit_timeline_position_badge.md](prompts/commit_timeline_position_badge.md)

# Plan: Commit timeline position badge

## Context and outcome

The Artifacts Commits pane currently reports a phrase such as `105 matches` at the far right of the persistent filter
bar, while the repository-specific counts begin on the next line with content such as `sase (86)`. This separates the
most important list metric from both the list and its repository breakdown, and it gives no indication of where the
current selection sits within the matched commits.

Replace that presentation with one compact position badge at the start of the repository legend row:

```text
Commits  Scope Current project
[15/105]  ·  sase (86)  ...
```

The badge uses the familiar `selected/total` grammar without a redundant label because it already sits under the Commits
heading. Render the one-based selected position in bold white, the matched total in bold Commits gold, and the brackets,
slash, and separator in a dim neutral style. This gives the total the strongest emphasis while keeping the current
position easy to parse. The repository names and their per-repository counts retain their existing colors and meanings.

The persistent filter bar must stop rendering `N match` / `N matches` entirely. It should continue to show only the
coverage state (`exact`, `preview`, `capped`, or the initial lazy-loading label) in its existing semantic color, and
parse errors must continue to replace that state with the full red diagnostic. Let the compact status area shrink to the
coverage label so the query receives the recovered horizontal space; preserve the wider error treatment. This behavior
is specific to the Commits filter bar, and must not alter the Plans filter bar or the shared filter-bar default.

## Truthful badge states

Drive both values from the currently displayed `VcsLogResult.commits`, using the timeline's commit index rather than its
raw `OptionList` row so disabled day banners never affect the position.

- Exact, non-empty results render `[P/N]`, where `P` is the selected commit's one-based index and `N` is the number of
  displayed matched commits. In the referenced scenario, the selected 12:12 commit is row 15, so the result is
  `[15/105]` immediately before `sase (86)`.
- Results marked `potentially_truncated` render a `+` on the denominator, such as `[1/40+]`, preserving the existing
  guarantee that the visible count is only a lower bound. The filter bar simultaneously says `capped`; do not invent an
  exact total.
- A completed empty result renders `[0/0]` and leaves the existing empty-timeline message and presence legend intact.
- Before any result exists, omit the badge instead of briefly claiming `[0/0]`; retain the current lazy-loading copy. If
  a non-empty result ever has no valid selection during a defensive transient, render a neutral dash for the numerator
  rather than guessing a row.
- Live filter previews use the displayed preview result for both values and retain `preview` in the filter bar. When the
  authoritative result arrives, update the denominator and preserve the selected commit by stable repository/SHA
  identity when it still exists; otherwise follow the timeline's existing first-row fallback.

Compose the badge and repository legend with exactly one dim separator. Keep the current three-row information area and
align any remote-summary continuation beneath the repository legend rather than beneath the badge. At narrow widths, the
badge remains visible and the repository legend takes the ellipsis, making the total dependable at a glance.

## State flow and rendering boundary

Keep this as presentation-only Python/Textual work; no Rust core or VCS-log backend changes are needed. Split the
position badge from the comparatively static scope/repository legend in the commits information area, or use an
equivalent selective-rendering boundary, so a selection move updates only the small badge. Repository counting and
legend construction should happen when the displayed result, scope, coverage, warning, or refresh state changes—not on
each `j`/`k` keypress.

Refresh the badge synchronously whenever selection can change:

- ordinary next/previous and page/first/last timeline navigation;
- programmatic stable-target selection used by artifact jump navigation;
- result replacement after collection, filtering, refresh, scope changes, or sidecar toggles;
- selection preservation or fallback when a filtered result removes the prior commit.

Perform this lightweight update before scheduling the already-debounced detail render. It must use only in-memory state,
trigger no collection or diff work, and keep the existing guard against programmatic `OptionHighlighted` echoes. Result
and scope transitions may rebuild the full information area, but steady-state navigation must not scan all commits or
rebuild the repository legend.

## Product copy and guidance

Update Commits help and user documentation so the new location and lower-bound notation are discoverable. Document
`[P/N]` as selected position over matched entries and `[P/N+]` as selected position over a lower-bound total. Replace
examples such as `40+ matches · capped` with the new paired presentation, for example `[1/40+]` in the legend and
`capped` in the filter bar. Keep the existing explanations of explicit `limit:N`, unlimited queries, and
provider/aggregate truncation unchanged in substance.

## Verification

Add focused rendering and interaction coverage for the complete contract:

- Assert the exact badge text, ordering before the first repository, Rich styles, one-based position, and exclusion of
  day banners.
- Cover exact, capped-by-query, provider/aggregate-capped, empty, pre-load, and defensive no-selection states.
- Exercise keyboard navigation and stable-target programmatic selection to prove `[1/N]` changes immediately to `[2/N]`
  without waiting for the detail debounce.
- Exercise live preview, authoritative reconciliation, Escape restoration, and a filter that drops the selected commit;
  verify the badge total always matches the displayed tuple and the stable-selection fallback remains correct.
- Update Commits filter-bar tests to require coverage-only status and unchanged parse errors, while retaining an
  explicit regression test that the shared/Plans filter bar can still render match counts.
- Update the existing Commits PNG snapshots, especially the normal 120-column timeline, active filter, empty result, and
  capped 80-column view. Inspect actual/expected/diff artifacts before accepting intentional goldens, confirming that
  the badge is visually dominant, the query gains space, repository legend alignment remains clean, and no extra
  vertical space is introduced.

Run the targeted Commits rendering, filtering, interaction, filter-bar, help, and PNG snapshot tests. Then run
`pytest -s -m slow tests/ace/tui/bench_artifacts_jk.py` and retain the existing sub-16 ms p95 budget for `commits.next`,
`commits.prev`, and the fast-navigation actions, demonstrating that the selected-position update stays on the
lightweight navigation path. Finally run `just install` followed by `just check`, as required for repository changes.

## Acceptance criteria

- The referenced 105-entry view shows `[15/105]` immediately to the left of `sase (86)`, and moving the selection
  updates the numerator in place.
- The filter bar contains no `match` or `matches` count in normal, preview, or capped states; coverage labels and parse
  errors remain truthful and readable.
- Empty and truncated results never imply a false exact count, and filtering/refreshing cannot leave the badge out of
  sync with the visible timeline.
- The badge remains visible at the existing narrow-terminal snapshot size, the repository legend and remote-summary
  alignment remain polished, and the pane consumes no additional vertical space.
- Navigation performs no I/O or full-result/legend rebuild and continues to meet the established Artifacts key-to-paint
  budget.

## Scope boundary

Do not change commit collection, filter semantics, row ordering, repository counts, stable-selection identity, detail
debouncing, keybindings, or configuration defaults. No new user configuration is needed for this fixed piece of
Commits-pane chrome.
