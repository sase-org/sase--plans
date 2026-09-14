---
tier: tale
title: Remote fleet rows become family and clan nodes
goal: Remote serialized rows project into stable local-equivalent family and clan
  trees without local-only side effects.
size: medium
proposed_by: bbugyi200.apollo.sase-xe.16.11.7.15.2
bead: sase-xe.16.11.7.15.2
status: done
---

- **PARENT:**
  [202609/remote_agents_display_parity.md](https://github.com/sase-org/sase--plans/blob/main/202609/remote_agents_display_parity.md)
- **BEAD:**
  [sase-xe.16.11.7.15.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.11.7.15.2.md)

# Remote fleet rows become family and clan nodes

## Objective

Complete phase `sase-xe.16.11.7.15.2` by turning each host's serialized fleet summaries
into the same in-memory family/clan tree shape used for local agents. Remote projection
must consume the owner-resolved `row_kind`, `family_role`, `current_instance`, and
`container_projected_concrete_agent` facts, preserve host-qualified identities and
parent lineage, and never invoke local PID, filesystem, cleanup, or hydration behavior.

## Baseline and constraints

1. Confirm the checkout is clean, run `just install`, and record the focused fleet and
   agent-tree test baseline before editing. Treat existing failures as baseline evidence
   rather than weakening assertions.
2. Keep membership, instance, and promotion decisions sourced from the fleet wire and
   existing Rust-backed promotion surface. Python owns only `Agent` construction,
   host-local relationship assembly, ordering, and presentation-tree projection.
3. Keep remote synthesis pure and in-memory. Do not add event-loop I/O, local process
   checks, filesystem hydration, cleanup effects, feature flags, or broad refresh paths.

## Implementation

1. Extend the fleet summary projection so every row records and acts on the structural
   wire facts. Within each host, select the renderable representation of a logical agent
   from `current_instance` and `container_projected_concrete_agent`, removing duplicate
   container/concrete and superseded non-current top-level instances while retaining the
   historical/member shell rows needed to present family history.
2. Resolve every retained `parent_timestamp` against host-qualified row identities, then
   run a remote-only node-normalization pass that reuses the local family relationship
   primitives (`apply_status_overrides` and ordering/runtime-child attachment) without
   running local PID filtering, local deduplication, or imported filesystem behavior.
   Synthesize a stable, origin-qualified family root when a bounded page contains
   member/shell rows but not their root, and link members, monitors, gates, and procs
   beneath the correct container through `followup_agents`/`runtime_children`.
3. Ensure the final mixed source is projected through `project_clan_tree` in one shared
   helper used by both fleet refresh reprojection and query refiltering. Preserve the
   selected row by stable host-qualified identity across a refresh and make repeated
   projection idempotent.
4. Expand the real serialized fleet fixtures only with contract fields already emitted
   by the wire. Add focused regressions for a remote family container and rendered `xN`
   behavior, monitor/gate/proc nesting, container-plus-concrete duplicates, non-current
   and historical shells, a missing root page that synthesizes a stable container,
   refresh selection stability, grouping by machine, clan projection, and equality of
   refresh/refilter tree shapes.

## Verification and completion

1. Run the focused fleet projection, fleet refresh, grouping-mode tree, and agent-tree
   rendering tests, then any adjacent suites implicated by failures.
2. Read the required lint/test reference memory and run `just check`, escalating only as
   its documented rules require. Compare the result with the recorded baseline.
3. Run `sase bead epic-symbols sase-xe.16.11.7.15.2`; resolve every remaining phase
   symbol or re-key it to an open later phase/parent as appropriate.
4. Close only `sase-xe.16.11.7.15.2` with a note naming the focused and full checks that
   passed. Record any genuinely out-of-scope discovery only as a `PROPOSED FOLLOW-UP:`
   note on this phase.
