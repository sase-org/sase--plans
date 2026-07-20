---
tier: tale
title: Restore phase PLAN and BEAD context lanes
goal: 'ACE shows a phase agent''s own submitted or approved plan in the PLAN lane
  while independently retaining the authoritative parent-phase metadata in the BEAD
  lane, including for historical runs, without adding work to the TUI event loop.

  '
create_time: 2026-07-20 10:40:27
status: wip
prompt: 202607/prompts/phase_plan_and_bead_context.md
---

# Plan: Restore phase PLAN and BEAD context lanes

## Context and confirmed failure

The selected `sase-83.1` family in the supplied screenshot has two legitimate and distinct plan relationships:

- It is phase 1 of epic bead `sase-83`, whose durable plan is `sase/repos/plans/202607/agent_cli_update_awareness.md`.
  That relationship owns the BEAD lane's phase description, epic plan path, and epic title.
- Its planner member authored and received approval for the tale `sase/repos/plans/202607/provider_update_snapshot.md`.
  That relationship owns a PLAN lane for the selected family and its planner/coder members.

The current metadata and enrichment path collapses those relationships into one `sdd_plan_path`. Plan approval replaces
the parent epic path with the phase agent's newly authored tale path while `epic_bead_id` and `phase_bead_id` remain on
the family. `resolve_agent_plan_enrichment()` sees the phase identity first, interprets the tale as though it were the
parent epic, cannot find phase frontmatter in it, and produces the screenshot's unavailable BEAD fields. The same
phase-role branch always returns `associated_plan=None`, so the otherwise-capable PLAN renderer never receives the
authored tale.

This is an association-model defect, not a lane-rendering defect. A local, cached bead lookup already resolves
`sase-83.1` to the correct parent epic plan and phase description. The fix should make both relationships explicit,
allow them to coexist in one enrichment result, and retain that lookup as a compatibility fallback for records created
before the metadata correction.

## Product and lifecycle contract

Treat BEAD and PLAN as independent SASE CONTEXT lanes with the existing narrative order: BEAD first, then PLAN.

- A phase agent that has not authored a plan continues to show only the selected phase in BEAD. The complete parent epic
  roadmap must not leak into PLAN.
- Once that phase agent submits its own plan, PLAN appears for the family root/planner while BEAD remains anchored to
  the parent epic. After approval, the coder and the selected family continue to show both lanes.
- Pending, approved-without-commit, committed tale, and committed epic handoffs choose the correct archive or durable
  SDD path for the authored PLAN without allowing the parent epic's commit state to masquerade as the authored plan's
  state.
- A missing, unreadable, or stale parent epic degrades only BEAD to honest unavailable fields; it must not suppress a
  valid authored PLAN. Conversely, an unavailable authored plan must not erase valid phase metadata.
- Existing artifacts that lack any new metadata remain readable. For those rows, resolve the parent association from the
  phase bead locally and use the existing bounded caches; never materialize/sync a store or perform network work from
  passive ACE enrichment.

## Separate durable parent and authored-plan identity

Introduce an optional parent-epic plan reference in phase launch metadata instead of relying on `sdd_plan_path` to serve
both meanings. Capture it from the existing `SASE_EPIC_PLAN_REF` input, preserve it across runner re-execs, plan-family
promotion, feedback/question continuations, coder follow-ups, synthetic family rows, and copy/backfill helpers, and load
it into the ACE `Agent` model through both filesystem and snapshot paths.

Because agent snapshots are a shared backend contract, extend the linked `sase-core` `AgentMetaWire` projection and
scanner for the optional field, with serde defaults and wire/parity tests so older `agent_meta.json` files remain valid.
Do not change release versions. Keep existing `sdd_plan_path` compatibility behavior where other consumers still need
it, but stop using that overloaded value as sole proof of the parent epic inside ACE.

Keep the authored plan's existing archive, committed SDD, action, and commit metadata as the PLAN relationship. At
selection time, compare it against the separately resolved parent epic identity so a pending phase plan uses its archive
while an accepted handoff can use its committed SDD copy. This distinction must survive the common coder case where
`plan_action` is absent but the family role and inherited plan artifacts establish the approved handoff.

## Dual plan enrichment and cache behavior

Refactor associated-plan enrichment to resolve two independent outputs for a phase-capable row:

1. Resolve `PhaseBeadSummary` from the explicit parent-epic reference when present. Preserve the current fast path for
   ordinary modern phase rows. For historical rows whose authored plan replaced the old direct epic reference, use the
   existing local-only `BeadIssueLookupSession` and association cache to resolve the phase bead, its parent, and the
   parent's design path. Validate/cache that epic plan by mtime and expose only the selected phase description, epic
   title, and path.
2. When the row has evidence that this phase agent authored a separate plan, build the normal `AssociatedPlanSummary`
   from that plan using the existing archive/SDD selection, frontmatter validation, effective-tier, and file cache
   machinery. Do not infer a PLAN merely because the row belongs to an epic.

Allow one `AgentPlanEnrichment` to carry both summaries. Replace the single claimed/resolved plan path assumption with
all plan paths consumed by BEAD and PLAN, so generic ARTIFACTS filtering cannot duplicate either file. Include every new
in-memory association input in detail-header cache keys and retain short negative TTLs/mtime invalidation so a
temporarily absent sidecar record or an edited plan refreshes without per-keypress I/O.

All filesystem parsing and bead reads must remain in the existing deferred detail-header worker. Rendering, fold
changes, hint navigation, and row selection stay memory-only; no new worker, timer, refresh path, subprocess, or
message-pump await is needed.

## Rendering and family integration

Reuse the existing BEAD and PLAN renderers and `CONTEXT_LANE_ORDER`. Once enrichment can return both summaries, verify
that the standard selected-agent/family header, responsive wrapping, fold levels, file hints, and clan/family context
aggregation preserve both lanes rather than choosing one by role. Each lane must navigate to its own file: BEAD to the
parent epic and PLAN to the phase-authored plan.

Update comments and type contracts that currently state phase workers always carry `associated_plan=None`. Keep the
intentional privacy boundary: BEAD may summarize only the selected parent phase, while PLAN may show the phase agent's
separately authored tale or epic because that is its own work product.

## Regression and compatibility coverage

Add focused tests that reproduce the metadata shape in the screenshot rather than testing only isolated renderers:

- A modern phase with only its parent epic still performs no bead-store lookup, renders a populated BEAD lane, and
  suppresses the parent roadmap from PLAN.
- A phase family with an authored pending plan renders BEAD from the parent epic and PLAN from the archive with correct
  tier/state semantics.
- The approved `sase-83.1`-shaped root and coder records render a populated BEAD for the parent epic plus a tale PLAN
  for the committed phase plan, including the coder case with no `plan_action`.
- A historical record without the new parent reference recovers the parent via the local bead association cache and
  still renders both lanes; missing bead/plan stores degrade each lane independently.
- Both plan files receive distinct numeric hints, neither is duplicated in ARTIFACTS, and detail-header cache
  invalidation responds to relationship/path changes and plan mtimes.
- Filesystem and Rust-snapshot metadata enrichment stay in parity, and runner/follow-up tests prove the explicit parent
  reference survives plan submission, approval, refresh/re-exec, and family projection.

Extend an Agents-tab visual fixture to show a phase plan family with BEAD followed by PLAN, and inspect the PNG diff to
confirm lane ordering, wrapping, colors, counts, and scroll behavior at the representative panel width. Preserve the
existing BEAD-only visual contract for phases that did not author a nested plan.

## Validation and completion

In `sase-core`, run formatting checks and targeted scanner/wire tests, then the relevant crate test suite. In the main
repository, run `just install` before validation, execute the focused runner metadata, loader parity, associated-plan,
header/context, family aggregation, and visual tests, inspect any intentional snapshot update, and run
`just test-visual`. Finish with the repository-required `just check`.

The work is complete when the supplied `sase-83.1` lifecycle shape yields two accurate lanes, legacy phase rows recover
without migration, phase-only rows still hide the parent roadmap, both filesystem and snapshot loading agree, and all
targeted, visual, Rust, and full repository checks pass without introducing event-loop or serial-pump work.
