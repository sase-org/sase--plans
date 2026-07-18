---
tier: tale
title: Agents status and launch-recency ordering
goal: 'The Agents tab''s by-status view prioritizes running, completed, and waiting
  work in that order and lists the most recently launched agents and agent families
  first without breaking hierarchy, navigation, or refresh behavior.

  '
create_time: 2026-07-18 14:09:24
status: done
prompt: 202607/prompts/agents_status_recency_sort.md
---

# Plan: Agents status and launch-recency ordering

## Context and scope

The Agents tab already receives its flat agent list with top-level rows ordered by descending `start_time`, but
`BY_STATUS` rebuilds that list with a fixed status-bucket order and structural name keys ahead of any time key. This is
why the rendered tree can put Waiting before Done and can order running singleton agents and agent-family groups by
shape/name rather than launch recency.

This tale changes only the Agents tab's `BY_STATUS` presentation. Keep the shared status classification intact:
successful terminal states such as `DONE`, `PLAN DONE`, `TALE DONE`, and `EPIC CREATED` continue to map to the existing
`Done` bucket; `STANDARD`, `BY_DATE`, integration/API status ordering, filter semantics, and the intentionally hidden
`Starting` rows remain unchanged.

## Product ordering rules

- Render visible status buckets in the priority order `Running`, `Done`, `Waiting`, `Stopped`, `Failed`; retain
  `Starting` as the final internal bucket so its existing hidden-row behavior does not change. Unknown future buckets
  continue to sort after known buckets.
- Within each status bucket, order top-level display units by launch time, newest first. Use the existing agent
  `start_time` definition of launch time, place missing timestamps after timestamped rows, and retain deterministic
  structural/name/input-order tie-breakers for equal or missing timestamps.
- Treat an agent family, clan, or workflow subtree as one display unit. Anchor its status and recency to the outer
  presentation/root agent, keep all descendants adjacent in their established preorder, and do not let a newer child or
  follow-up split or unexpectedly re-anchor the family.
- Preserve name-root and name-prefix banners and their fold keys. Recency decides where sibling singleton/family units
  appear; it must not flatten the hierarchy or alter order inside a family.

## Tree ordering implementation

Update the status-bucket priority in `src/sase/ace/tui/models/agent_groups/_buckets.py`, including its explanatory
comments and mode documentation.

Extend the in-memory ordering metadata in `src/sase/ace/tui/models/agent_groups/_keys.py` and the shared grouped walk in
`_tree.py` so `BY_STATUS` computes a descending launch-recency key from the already-loaded outer presentation anchor.
Precompute one effective recency per atomic display unit/name group before sorting, then place that key ahead of the
structural label tie-breakers. This keeps all members of a visible family group contiguous while allowing singleton
agents and families to interleave strictly by launch recency. Keep the computation CPU-only and bounded by the loaded
panel; do not add filesystem access, timestamp parsing, or a new refresh path to the render loop.

Use the same grouped walk for tree rendering and group-key enumeration so banner order, jump hints, fold navigation, and
selection re-anchoring all agree with the visible order. Preserve the existing root-cluster expansion that keeps
workflow and clan descendants adjacent.

Because `start_time` becomes part of a row's position under `BY_STATUS`, extend the status render/patch safety signature
used by `src/sase/ace/tui/actions/agents/_display_panel_patches.py`. Badge-only changes with unchanged bucket,
hierarchy, and launch anchor should continue to use the fast row-patch path; a change to an ordering input must refuse
the in-place patch and rebuild the affected panel through the existing selective refresh path.

## Documentation and regression coverage

Update the Agents grouping-mode section in `docs/ace.md` to describe the new bucket priority, launch-recency rule,
missing-time behavior, and root-anchored family semantics.

Add focused model and widget regressions that cover:

- scrambled inputs producing `Running` before `Done` before `Waiting`, with `Stopped`/`Failed` following and `Starting`
  still absent from rendered rows;
- multiple running agents sorting by descending `start_time`, including equal timestamps and a missing timestamp;
- a newer singleton sorting around an older agent family while the family's root, follow-ups, workflow children, and
  dotted-name banners remain contiguous and in their established internal order;
- terminal and waiting buckets using the same deterministic launch-recency ordering without changing their status
  classification;
- group-key enumeration matching rendered banner order;
- `BY_STATUS` row patching still accepting badge-only updates and rejecting a launch-anchor change that could move the
  row.

Keep existing `STANDARD` and `BY_DATE` ordering tests as regression guards. After implementation, run `just install`,
the focused agent-group/tree, AgentList bucket, and display-patch test modules, then run the repository-wide mandatory
`just check`. Inspect the existing Agents `BY_STATUS` visual snapshot when running the suite, but only update or add a
PNG golden if the exercised fixture intentionally contains multiple affected buckets or recency-ordered siblings.

## Risks and safeguards

The main risk is splitting a family when members carry different timestamps or making incremental refresh leave a row at
its old position. Avoid both by sharing the outer/root launch key across each atomic unit and by treating ordering-key
changes as structural for patch safety. Keep the sort based only on cached `Agent` fields so the change does not add
event-loop I/O or a data-scaled startup scan, and verify that collapsed groups, jump order, and selection continue to
consume the same ordered tree.
