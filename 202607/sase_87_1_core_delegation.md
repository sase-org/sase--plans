---
tier: tale
title: Core delegated-phase closure and scheduling
goal: Make delegated epic phases close when their child work lands, prevent retries
  from relaunching delegated phases, and expose the bead identifiers needed for bead-gated
  waits.
create_time: 2026-07-20 11:25:09
status: done
prompt: 202607/prompts/sase_87_1_core_delegation.md
---

# Plan: Core delegated-phase closure and scheduling

## Context

Epic phase agents can delegate consequential work to a child epic. The phase agent then finishes while that child epic
continues independently. The core currently cascades plan closure only downward, so landing the child epic leaves its
parent phase open. Retrying the parent epic also schedules every non-closed phase, which can relaunch that delegated
phase and create duplicate child work. Finally, the work-plan wire payload exposes only agent waits for phases scheduled
in the current run; later consumers need stable bead IDs for every in-epic dependency and every phase, including closed
and temporarily unscheduled delegated phases.

The work belongs in the linked `sase-core` repository because closure, scheduling, and payload semantics are shared
backend behavior. The existing wire fields and Python binding behavior must remain compatible; all new payload data is
additive.

## Upward delegated-work closure

Extend `crates/sase_core/src/bead/mutation.rs` so each plan bead closed by `close_issues` can complete its direct parent
when that parent is a non-closed phase and all of the phase's children are closed. Close the phase through the normal
mutation/event path, using a distinct reason such as `delegated work landed`, so event streams, JSONL, SQLite-backed
reads, and search projections observe the same state transition. Apply the check as closure proceeds so nested and
multi-child structures settle correctly, while never auto-closing a plan/epic ancestor.

Keep explicit close behavior and downward plan cascades intact, avoid duplicate close events when requested IDs overlap
with cascaded IDs, and leave `remove_issue` unchanged: removing delegated work is rollback and must keep the phase open
for a future retry.

Add mutation coverage for a child epic closing its parent phase, a second open child preventing that closure, nested
delegation closure behavior, emitted close reason/event projection, and removal not triggering the upward cascade.

## Delegation-aware work planning

Update `crates/sase_core/src/bead/work.rs` to distinguish all non-closed phases from phases schedulable in the current
launch. A phase with a direct non-closed plan child is already delegated and must be excluded from the wave DAG. A
closed or removed child plan must not exclude it. Dependencies among scheduled phases continue to determine agent wait
names and wave order; a scheduled phase that depends on a delegated phase is launchable without an agent-name wait for
that absent agent and remains represented by its blocker bead ID for the downstream bead barrier.

Retain the existing error when an epic truly has no non-closed phase children, but allow a valid plan with no waves when
all remaining phases are delegated so its land segment can be launched and bead-gated by the caller. Keep total phase
counting and existing launch/land names stable.

Add two additive serialized fields: an ordered `blocker_bead_ids` list on each phase assignment containing every in-epic
dependency regardless of closure or delegation status, and an ordered epic-wide `phase_bead_ids` list containing all
direct phase children. Preserve `waits_on` and `land_waits_on` as the agent names launched in the current plan. Extend
Rust planner tests for closed blockers, delegated blockers, delegated-only plans, closed/removed child behavior, wave
ordering, and the new complete ID lists.

## Binding compatibility and verification

Because the PyO3 functions serialize the Rust wire structs directly, retain their public entry points and extend the
binding-level work-plan test in `crates/sase_core_py/src/lib.rs` to assert that the new fields cross the Python boundary
with the expected JSON shape while existing fields remain present.

Run focused Rust tests while iterating, format the Rust workspace, then run the required repository-integrated checks
from the main SASE checkout: `just rust-check` and `just bead-perf-smoke`. Review the final diff and statuses in both
the main checkout and linked core repository, ensuring only the intended core files changed before closing `sase-87.1`
and leaving parent epic `sase-87` open.

## Risks

- Closure ordering can accidentally emit duplicate events or give explicit closures the delegation reason; tests must
  assert outcome IDs, reasons, persisted state, and event counts around overlapping/downward cascades.
- Filtering delegated phases from waves can break topology if delegated blockers remain in the scheduled DAG; agent
  waits and bead blocker IDs must be derived from intentionally different sets.
- Empty waves are valid only when open phases exist but all are delegated; an actually completed or malformed epic must
  retain its current validation behavior.
- Payload ordering is user-visible in rendered prompts and snapshots, so derive both new lists deterministically without
  changing the ordering of existing fields.
