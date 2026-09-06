---
tier: tale
title: Durable fleet mutation journal and launch admission recovery
goal: Make fleet mutations durably idempotent and make launch admission recover the
  one reserved run across crashes and competing controllers.
size: medium
bead_id: sase-xe.6
proposed_by: bbugyi200.athena.sase-xe.6
bead: sase-xe.6
status: done
---

- **PARENT:** [202609/remote_dispatch_fleet.md](remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.6.md)

# Plan: Durable fleet mutation journal and launch admission recovery

## Context

The existing `sase_core::fleet_contract` defines canonical payload fingerprints,
controller-scoped operation keys, exact agent instance locators, resource revisions,
receipts, tombstones, and pure replay decisions. The gateway currently enforces only
fleet enrollment and credential lifecycle; it does not persist operation decisions or
reserve a run before invoking execution. The mobile gateway's `request_id` remains
correlation-only and must not be reused as the fleet reliability mechanism.

This plan implements phase `sase-xe.6` of the approved remote-dispatch epic. It keeps
transport-free validation and state-transition rules in `sase_core`, puts durable
gateway enforcement in `sase_gateway`, and preserves the existing mobile API. Later
dispatch and remote-action phases will call this admission surface rather than inventing
their own retry logic.

## Implementation

1. Extend the transport-free fleet contract with explicit mutation and launch-admission
   records and transitions. Model the documented retry window, receipt states,
   tombstones, reserved run identity, durable worker ownership/lease fencing, spawn and
   run-record observations, settlement, and recovery actions. Validate every record
   against its controller-scoped key, canonical payload fingerprint, exact target
   instance, logical resource revision, timestamps, and monotonically increasing fence
   token. Expose the new pure operations through `sase_core_py` using the existing
   dict-in/dict-out and `allow_threads` conventions.

2. Add a private, lock-protected fleet operation store to `sase_gateway`, rooted under
   the established `<sase_home>/fleet_gateway` directory. Under one exclusive file lock,
   load and validate the bounded journal, classify a request with the core rules,
   reserve a launch's authoritative instance before spawn, claim/reclaim work with a new
   fence, persist state changes atomically with file and parent-directory durability,
   and retain receipts or tombstones through the supported retry window. Corrupt,
   oversized, expired, mismatched-key, mismatched-payload, stale-revision, stale-target,
   and stale- fence inputs must fail closed; an expired operation key must never become
   a new launch.

3. Add the authenticated fleet-v1 admission/recovery wire surface and gateway service
   integration. Derive controller identity exclusively from the authenticated fleet
   credential, require the appropriate mutation/launch scope, canonicalize and
   fingerprint the accepted payload server-side, and map replay/conflict/expiry/
   precondition outcomes to stable API results. A repeated submission returns the
   original receipt; recovery reports or advances the existing reserved run and never
   allocates a second identity. Update capabilities, error codes, public exports, and
   the committed fleet API contract snapshot together.

4. Exercise the launch recovery state machine with an injectable executor/observation
   boundary so tests can stop after reservation, after spawn, after run-record
   persistence, and before reply. Prove that restart recovery finds the existing
   reservation/process/run, that two store or gateway instances cannot both own the
   launch, and that an old worker, reused human name, or reused PID cannot mutate the
   replacement because the exact locator and fence no longer match.

5. Add focused contract, store, route, and Python-binding tests for canonical payload
   deduplication, same-key/different-payload conflict, resource-precondition failure,
   receipt replay and expiry, tombstone retention, concurrent controllers, every crash
   boundary, and name/PID reuse fencing. Regenerate the fleet-v1 contract snapshot, run
   the full `sase-core` verification gate, then update the main repository's pinned core
   revision and strict binding inventory/tests before running the required main-
   repository verification lane.

## Boundaries

- Do not retrofit the mobile-v1 name-based launch/kill endpoints or treat `request_id`
  as a durable operation key; fleet-v1 is the new safety boundary.
- Do not implement `%dispatch`, provider configuration, federation-worker IPC, Follow
  persistence, or ACE remote actions in this phase.
- Do not expose local PIDs, paths, process groups, credentials, or mutable human names
  as fleet identity. A process observation may be stored internally, but all authority
  remains tied to the reserved exact instance and current fence token.
- Do not claim arbitrary exactly-once agent execution. The guarantee is one admitted run
  per launch intent through the documented recovery window.

## Verification

- Run targeted Rust tests for `sase_core` fleet transitions and `sase_gateway` journal,
  route, concurrency, and crash recovery behavior while iterating.
- Run `just check` at the `sase-core` repository root.
- After updating the core revision pin and strict binding surface in the main
  repository, follow the project's mandatory verification memory and run its required
  fast check.
- Immediately before closure, run `sase bead epic-symbols sase-xe.6`, resolve or re-key
  any remaining entries, and close only `sase-xe.6` with a note naming the verified
  fault boundaries and repository checks.
