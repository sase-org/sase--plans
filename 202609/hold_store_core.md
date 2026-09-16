---
tier: tale
title: Rust hold-record store and bindings
goal:
  Later hold phases can arm, list, release, and evaluate durable TTL-bounded holds
  through one fail-open Rust domain API and its Python bindings.
size: medium
proposed_by: bbugyi200.athena.sase-11l.2
bead: sase-11l.2
create_time: 2026-09-15 22:53:46
status: wip
---

- **PARENT:**
  [202609/hold_directive.md](https://github.com/sase-org/sase--plans/blob/main/202609/hold_directive.md)
- **BEAD:**
  [sase-11l.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11l/sase-11l.2.md)

# Rust hold-record store and bindings

## Goal

Implement phase `sase-11l.2` as the reusable Rust backend for durable `%hold` records.
The result must be a TTL-bounded, fail-open, flock-guarded store under the SASE home, a
pure candidate-blocking predicate, and PyO3 bindings that later epic phases can consume
without reimplementing domain behavior in Python.

## Scope and constraints

- Work in the linked `sase-core` repository first; the primary `sase` repository changes
  only for the pinned core revision after the Rust change is published.
- Persist `agent_holds.json` under the supplied SASE home, guarded by `agent_holds.lock`
  through the shared `store_lock` helper and written by atomic tempfile replacement.
- Store one record per armer key. Re-arming the same key replaces its previous record.
- Every record has a current schema version, a typed armer (`agent`, `proc`, or `cli`)
  with key/display/project and liveness coordinates, mandatory finite
  `created_at`/`expires_at`, `project` or `host` scope, and a match block containing
  frozen artifact directories plus live names, hoods, tribes, and `future`.
- Reads prune expired, malformed, stale-schema, and dead-armer records. Missing,
  malformed, or unreadable state is treated as an empty active set so scheduling is
  fail-open. Mutating operations still report validation, lock, serialization, and write
  failures.
- Armer liveness is supplied as data by the caller (PID-alive/done-marker facts for
  agents and CLI armers; terminal-status facts for proc armers); the core performs no
  process, artifact-directory, or proc-store discovery beyond its own hold store.
- The pure predicate applies project/host scope, frozen artifact-dir membership,
  name/family/clan/workflow matching, hood component-boundary matching through
  `agent_name_in_hood`, tribe matching, future timestamps, and armer-kin exclusion. It
  returns a typed block carrying enough armer and expiry information for later
  admission/UI phases.
- Do not implement runner-slot enforcement, release hooks, CLI commands, prompt parsing,
  config defaults, or UI in this phase.

## Implementation

1. Add `crates/sase_core/src/agent_hold.rs` with strict serde wire types for armers,
   liveness facts, selectors, candidates, stored records, snapshots, and block results.
   Validate non-empty identities and projects, finite positive timestamps, mandatory
   `expires_at > created_at`, scope/kind values, and selector collections. Normalize
   set-like inputs deterministically for stable JSON and test results.
2. Implement relative- and exact-expiry arm APIs, idempotent per-key release, and list
   with prune-on-read. Use a `BTreeMap` keyed by armer key, the shared bounded store
   lock, atomic replacement, and removal of an empty/corrupt state file. Keep reads
   fail-open while preserving bounded lock-timeout reporting.
3. Implement `hold_blocks_candidate(record, candidate)` as an I/O-free function.
   Short-circuit inactive scope and armer kin, then union selector matches and return a
   deterministic `HoldBlock` only when at least one selector applies. Cover historical
   `--` shell suffixes and hood boundaries using the existing identity helpers.
4. Export the module, public functions, wire types, constants, and errors from
   `sase_core::lib`.
5. Add PyO3 bindings for schema version, relative/until arm, release, list, and the
   predicate. Accept typed Python dictionaries for structured inputs, serialize wire
   outputs to Python dictionaries/lists, map validation to `ValueError`, shared lock
   timeout to `TimeoutError`, and I/O/JSON failures to `RuntimeError`. Add every binding
   to the module registration and module-level binding index.
6. Add focused Rust and binding tests for round trips, expiry boundary, re-arm
   replacement, idempotent release, malformed/stale/unreadable cleanup, dead armer and
   terminal proc pruning, bounded lock waits, each selector, project/host scope, future
   before/after arm time, self/family/clan/dotted-descendant exclusion, hood component
   boundaries, shell-suffix ancestry, and Python error mapping.
7. Format and run the linked core repository's targeted tests during iteration, then its
   required `just check`. Publish the core change through the normal host-owned
   finalization path, ratchet `sase-core-revision.txt` in the primary repository with
   `tools/ratchet_core_revision`, and run the primary repository's binding validation
   plus required `just check`.

## Verification

- `sase-core`: targeted `agent_hold` and PyO3 binding tests pass, followed by
  `just check`.
- `sase`: `tools/check_sase_core_rs_bindings` and `tools/validate_sase_core_rs` see the
  new module surface, the revision pin targets the published core revision, and
  `just check` passes.
- `sase bead epic-symbols sase-11l.2` reports no unresolved symbols before closing the
  phase, and only `sase-11l.2` is closed with a note naming the verified checks.
