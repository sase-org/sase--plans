---
create_time: 2026-07-11 13:19:52
status: done
prompt: 202607/prompts/sort_custom_revival_by_date.md
tier: tale
---
# Plan: Sort Custom Revival Results Newest First

## Summary

Restore the ordering promised by **Custom revival search...**: the visible dismissed-agent rows should be ordered by
agent recency, newest first, regardless of project, project-agent status, or ChangeSpec/PR name. Keep the existing
global archive query, 250-parent pagination, same-session merge, local filtering, workflow-child support, and
off-event-loop loading unchanged.

This is a focused TUI presentation fix. The dismissed-bundle index and page loader already choose archive membership in
newest-first order, so no `sase-core`, persistence, index-schema, or linked-repository change is needed.

## Root cause

`load_dismissed_bundles_page()` already pages top-level archive summaries with:

`ORDER BY COALESCE(start_time, raw_suffix) DESC, filename ASC`

However, after each archive page is merged with same-session dismissed agents,
`AgentReviveArchiveMixin._dismissed_agent_rows()` re-sorts the full merged list by:

1. project-level agents before ordinary agents;
2. `cl_name` alphabetically; and
3. `start_time` newest first only within those groups.

That presentation sort overrides the archive's recency order, producing the project-grouped behavior. It is inherited
from the former project/PR-scoped picker flow and is no longer appropriate for the unscoped recent-results panel.

The existing paging regression test does not catch this because its “recent”, “older”, and “oldest” fixtures override
only `raw_suffix`; all retain the helper's identical default `start_time`. It also asserts a set of identities instead
of their displayed order.

## Desired behavior and acceptance criteria

- Opening **Custom revival search...** still loads at most the newest 250 visible archive parents initially and opens
  the dismissed-agent modal directly.
- Visible top-level rows are globally newest-first across projects and ChangeSpecs/PRs. Project-level agents receive no
  ordering priority.
- Loading an older page with `Ctrl+K` preserves a single newest-first ordering across the accumulated same-session and
  archive rows; older pages do not form project clusters or appear ahead of newer rows.
- Recency uses `Agent.start_time` when available. Legacy rows without `start_time` use a parseable timestamp from
  `raw_suffix` as a fallback, matching the archive index's intent; rows with no usable date sort after dated rows.
- Equal-recency rows have a stable deterministic order so repeated page application does not cause avoidable cursor or
  mark movement.
- Workflow children remain excluded from visible options but remain in `all_dismissed` for step counts and previews;
  child steps continue to be ordered by `step_index` inside previews.
- Filtering does not regroup or re-sort results: it narrows the already newest-first visible list.
- Page limits, offsets, exhaustion handling, deduplication, marks, highlighted identity, response-corpus preparation,
  and archive/projection background work remain unchanged.

## Implementation approach

### 1. Replace the obsolete project-first presentation sort

In `src/sase/ace/tui/actions/agents/_revive_archive.py`, change `_dismissed_agent_rows()` so it derives visible
top-level rows and orders those rows solely by recency descending. Remove the project-agent and `cl_name` precedence
from the sort key.

Use a small explicit recency-key helper rather than embedding another opaque tuple in the method. The helper should:

- prefer `start_time` for normal/current bundles;
- fall back to the supported timestamp representation in `raw_suffix` for legacy/null-start rows;
- put rows with neither value after dated rows; and
- provide a deterministic identity-based tie-breaker without changing the primary date order.

Keep `all_dismissed` as the complete merged parent-and-child corpus. Its membership, rather than its project ordering,
is what the modal uses for workflow step counts and child lookup, and `_get_child_steps()` already sorts children by
step index.

Apply the sort in the existing page-loader/background boundary. Sorting up to the accumulated bounded pages is cheap and
must not introduce disk reads, JSON parsing, or response-file work on the Textual event loop.

### 2. Add focused ordering regressions

In `tests/test_agent_group_revival_routing.py`, add or strengthen coverage around the custom archive path:

- construct agents from different projects/ChangeSpecs whose alphabetical/project-agent grouping conflicts with their
  actual dates, and assert `_dismissed_agent_rows()` returns visible identities strictly newest first;
- include an older project-level agent to prove it no longer jumps ahead of a newer ordinary agent;
- cover a missing-`start_time` legacy row with a timestamp suffix and an undated row to lock in fallback and
  unknown-last behavior; and
- retain a workflow child to prove it remains available in `all_dismissed` while absent from the visible ordering.

Update `test_custom_search_pages_global_archive_and_revives_without_scope` so every fixture has a distinct,
date-consistent `start_time` and its page assertions compare ordered identity lists rather than sets. After the second
page, assert the accumulated result remains newest-first across both pages and the same-session row. Preserve the
existing assertions for archive offsets, exhaustion, and scope-free single/batch revival.

If a modal-level assertion is useful, keep it narrow: verify filtering preserves source order. Do not duplicate the
archive mixin's sorting logic inside `DismissedAgentSelectModal`, because the modal is a generic ordered-list consumer
and currently preserves the order supplied by its caller.

## Verification

Run the focused routing and revive-modal tests while iterating, including the new date-order and pagination cases. No
copy, style, or layout changes are planned, so PNG golden updates should not be necessary.

Before completion, run `just install` as required for the ephemeral workspace, followed by the repository-required
`just check`. If any visual test changes unexpectedly, inspect the generated artifacts and treat that as a regression
rather than accepting a new snapshot for this ordering-only fix.

## Non-goals

- Do not change the archive page query, page size, or offset model; backend archive membership is already correct.
- Do not redefine “recent” as dismissal time or add a new persistence field; this fix follows the existing
  `start_time`/`raw_suffix` recency contract.
- Do not reintroduce project/PR scoping or add a user-selectable sort mode.
- Do not change saved/recent revival-group ordering, other `ProjectSelectModal` flows, filtering corpus behavior, or
  revival execution/audit semantics.
