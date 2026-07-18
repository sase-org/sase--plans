---
tier: tale
title: Sort clan members by status priority
goal: 'Agents and agent families inside every agent clan render failed members first,
  then stopped, running, waiting, and completed members, in every Agents-tab grouping
  mode, while preserving clan atomicity, family adjacency, launch-recency ties, and
  the incremental refresh fast paths.

  '
create_time: 2026-07-18 16:26:41
status: done
prompt: 202607/prompts/clan_member_status_sort.md
---

# Plan: Sort clan members by status priority

## Context and scope

Clan members currently render inside their clan in global launch order, newest first. `project_clan_tree` in
`src/sase/ace/tui/models/_agent_tree.py` emits each clan container followed by its member rows in the order they arrived
from `sort_and_reorder`, and `walk_order` treats every clan subtree as one atomic cluster whose internal "projected
preorder" is preserved verbatim in all grouping modes. The result is the reported screenshot: a clan in the `Running`
bucket lists nine `WAITING` members above its single `RUNNING` member because the running member launched first.

This is new product behavior, not a remnant of the restored bucket fix: neither the July 18 recency tale nor the
bucket-priority restoration specified any ordering _within_ a clan. The change is confined to the Agents tab's
Python/Textual presentation layer — the same boundary stance as the approved bucket-priority restoration — and needs no
Rust core wire/API change, no new refresh path, and no filesystem access in the render loop.

## Product ordering rules

- Within every agent clan, direct member display units always sort by status priority: `Failed`, `Stopped`, `Running`,
  `Waiting`, `Done` — in that order — regardless of the Agents tab's active grouping mode (`STANDARD`, `BY_DATE`, or
  `BY_STATUS`).
- `Starting` members share the `Running` rank, matching how clan headers and the collapsed-panel roster already count
  starting members as running. Unknown future buckets sort after all known buckets.
- This priority is deliberately different from the L0 `BY_STATUS` bucket order (which puts `Stopped` before `Failed`).
  The L0 order is user-specified attention priority across all agents; the within-clan order is user-specified triage
  priority inside one clan. Do not unify them.
- A sort unit is one direct clan member (`tree_depth == 1`) together with its attached deeper descendants (family
  follow-ups and workflow steps keyed to it). The unit's bucket comes from the depth-1 anchor row's displayed status —
  the family status-override machinery already projects a chain's current state onto its visible anchor row, so sort
  order always agrees with the status the user can read on the row.
- Members that share a status bucket keep their existing relative order (newest launch first with the established
  deterministic tie-breakers), i.e. the new sort is stable over today's projected order.
- Descendant rows inside a unit keep their established internal order and stay adjacent to their anchor.
- A clan's own position among sibling display units does not change: its container's aggregate status and launch anchor
  still decide the clan's bucket and cross-unit position, and the clan subtree remains atomic.

## Implementation

Apply the ordering inside `project_clan_tree`, which is the single choke point: loader normalization, compute-merge,
kill, dismiss, and filter re-projections all rebuild clan trees through it, so every grouping mode and every mutation
path inherits the sorted order with one change. Group each clan's rows into units by their projected parent link, sort
units with a stable status-priority key, and emit the container followed by each unit's anchor and descendants. Rows
whose parent is missing from the clan already project as depth-1 anchors and simply become their own units.

Define the within-clan priority next to the existing clan status helpers in `src/sase/ace/tui/models/_agent_clan.py`,
deriving buckets from the shared `status_bucket_for_values` mapping rather than duplicating status strings. Keep clan
container construction order-independent (status, times, tags, and project fields must not drift because members were
reordered). The sort must stay CPU-only over already-loaded `Agent` fields.

## Patch safety and refresh

Under `BY_STATUS`, the existing row-patch guard already refuses in-place patches when a row's own status bucket changes,
so status-grouped panels rebuild correctly today. `STANDARD` and `BY_DATE` have no such guard: a clan member's status
flip currently patches its row in place, which would leave the member at a stale position once position depends on
status. Extend the single-row patch safety check in `src/sase/ace/tui/actions/agents/_display_panel_patches.py` so a
clan-projected row whose status bucket changed refuses the in-place patch in every mode and falls back to the existing
selective rebuild, recording the fallback reason in the display-patch trace. Badge-only changes must keep the fast path,
and the optimistic row-removal path stays valid because removing rows never reorders the survivors.

## Documentation

Update `docs/ace.md` in two places: the clan/family hierarchy paragraph and the grouping-modes section. State that
direct members of a clan always sort by the within-clan status priority in every grouping mode, that launch recency only
orders members sharing a status, and that this is intentionally distinct from the `BY_STATUS` L0 bucket order.

Out of scope: the clan detail panel's MEMBERS section (`clan_section_member_rows` is a launch-order inventory and keeps
its current ordering), the L0 bucket priority, status filters and counts, and ordering of non-clan hood rows.

## Regression coverage and validation

Add focused model regressions for `project_clan_tree`: scrambled member statuses produce the full
`Failed, Stopped, Running, Waiting, Done` unit order; same-bucket members keep newest-first order; a family subtree
sorts by its anchor's displayed status while its follow-ups stay adjacent and internally ordered; a `Starting` member
ranks with `Running`. Add grouping-mode tree regressions proving the sorted member order renders identically under
`STANDARD` and `BY_STATUS`, including the screenshot scenario of one `RUNNING` member below many `WAITING` members now
rendering first, while the clan subtree remains one atomic, contiguous cluster in its aggregate bucket. Add a
display-patch regression: a `STANDARD`-mode clan member status-bucket change refuses the in-place patch and rebuilds,
while a badge-only change still patches in place.

Run `just install`, then the focused agent-tree, agent-group model, AgentList widget, and display-patch suites, then the
mandatory repository-wide `just check`. Unlike the bucket restoration, this change may legitimately reorder PNG goldens
whose fixtures contain a clan with mixed member statuses: inspect every failing snapshot diff and update only goldens
whose change is exactly the new member order, with no unrelated layout or styling drift.

## Risks and safeguards

The main risk is disturbing ordering outside the clan: keep the sort strictly inside each clan's unit list so cross-unit
recency, name grouping, bucket placement, and the atomic-cluster guarantees stay untouched, and lean on the existing
atomic-subtree and recency regressions as guards. A second risk is stale rows from incremental refresh; the new patch
guard plus its trace-backed regression covers the only in-place path that skips re-projection. Finally, keep the
projection sort stable and free of I/O so clan-heavy panels do not regress TUI responsiveness.
