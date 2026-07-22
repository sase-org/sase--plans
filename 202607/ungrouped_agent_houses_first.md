---
tier: tale
title: Keep standalone agent houses above panel subgroups
goal: 'Every agent house that is not a member of a visible name subgroup renders before
  all subgroup banners in its containing Agents-panel group, while preserving status-bucket
  priority, recency ordering, and subtree continuity.

  '
create_time: 2026-07-22 07:20:40
status: wip
---

- **PROMPT:** [202607/prompts/ungrouped_agent_houses_first.md](prompts/ungrouped_agent_houses_first.md)

# Plan: Keep standalone agent houses above panel subgroups

## Context and root cause

The Agents tab already intends dotless and singleton-name-root houses to render directly below their containing project,
ChangeSpec, or status bucket before any name-root subgroup banner. `walk_order()` encodes that distinction through the
name-root sort key, but its `BY_STATUS` path compares each display unit's shared launch-recency value before comparing
whether the unit is standalone or belongs to a visible subgroup. A subgroup whose root launch is newer than a standalone
house therefore sorts ahead of that house. This reproduces the reported shape: `hh` can render first, followed by the
`hk` subgroup (`hk` and `hk.f0`), with standalone `hi` incorrectly left below the subgroup.

The regression came from adding newest-first status ordering across all display units without retaining subgroup
membership as the primary within-bucket partition. The structural grouping data and banner builder are correct; the
fault is confined to the ordering tuple used by the existing pure in-memory tree projection.

## Implementation

Refine the `BY_STATUS` ordering in the agent grouping key module so each status bucket first partitions top-level
display units into standalone houses and visible name-root subgroups. Sort standalone houses first, newest launch first
within that standalone partition, then sort subgroup units by their established root/shared launch recency. Retain
deterministic tie-breakers and the current atomic-cluster expansion so families, clans, workflow children, and dotted
prefix groups cannot be split or re-anchored.

Keep `STANDARD` and `BY_DATE` behavior unchanged. Preserve the analogous nested rule for houses directly under a
name-root versus visible dotted-prefix subgroups, and keep exact prefix parent markers inside their subgroup. Update the
nearby ordering comments/module documentation and the Agents grouping-mode documentation to state that recency applies
within the standalone and subgroup partitions rather than allowing those partitions to interleave.

## Regression coverage

Add focused model tests using unequal timestamps that mirror the reported `hh`, `hi`, `hk`, and `hk.f0` layout. Assert
that both standalone houses render before the `hk` banner even when `hi` is older than the subgroup, and that the
subgroup remains contiguous with its members. Adjust the existing rooted-family recency expectation to the clarified
invariant, and retain or extend coverage showing that standalone-only units remain newest-first, multiple subgroup units
remain newest-first, missing timestamps stay last within their partition, and family/workflow preorder is unchanged.

Run the focused agent-group model tests first, followed by the repository's required validation sequence
(`just install`, then `just check`). If the change affects an intentional Agents PNG snapshot, inspect the
actual/expected/diff artifacts and update a golden only when it reflects this ordering correction; rerun the dedicated
visual suite after any intentional snapshot update.

## Risks and boundaries

The main risk is fixing the screenshot case by disabling launch-recency sorting or by breaking a grouped subtree into
independent rows. Avoid both by changing only the relative priority of the existing structural-membership and shared
recency components. No Rust core, persistence, keymap, or refresh-path change is needed: this is presentation-only
ordering over already-loaded agents, and it must not introduce I/O or additional data-scaled work into the render path.
