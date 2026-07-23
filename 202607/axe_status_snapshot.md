---
tier: tale
title: Side-effect-free AXE host status snapshot
goal: Python collects one deterministic read-only AXE host snapshot, classifies it
  through the shared Rust contract, and keeps deep doctor in health parity.
bead: sase-8t.2
parent: sase/repos/plans/202607/axe_status.md
create_time: 2026-07-23 08:09:06
status: done
---

- **PROMPT:** [202607/prompts/axe_status_snapshot.md](prompts/axe_status_snapshot.md)

# Implement the side-effect-free AXE host status snapshot

## Goal

Complete bead `sase-8t.2` by adding the Python host-I/O half of the schema-version-1 AXE status contract delivered by
`sase-8t.1`. The result must expose one typed, Textual/argparse/Rich-free snapshot for the later `sase axe status`
command, classify all observations through `sase_core_rs`, remain read-only even when runtime files are stale or race
with collection, and make the deep AXE doctor check use the same current-health answer while retaining its genuinely
deeper diagnostics.

This tale changes the SASE shell repository only. It must not reimplement Rust classification, change the existing
TUI-oriented `get_axe_status()` compatibility shape, add the public CLI, or edit release-owned versions in `sase-core`.

## Existing contracts and integration points

- The Rust binding exposes `axe_status_wire_schema_version() -> int` and `classify_axe_status(request: dict) -> dict`.
  Schema version 1 contains desired state, raw lock/PID observations, maintenance, runner occupancy, lumberjack
  observations, the latest lifecycle event, a nullable collection error, and the derived snapshot/issue records.
- `sase.axe._process_probe.probe_orchestrator(cleanup=False)` is the required non-mutating lock/PID probe. Its three PID
  sources must each be paired with an explicit liveness observation before calling Rust; do not reduce the evidence to
  only `running_pid`.
- Existing read helpers provide desired state, structurally valid maintenance, recent valid lifecycle journal events,
  configured lumberjack status/PID files, state-directory lumberjack names, and global hook/agent runner counts. Missing
  or malformed optional runtime/history records already degrade to absent observations and must remain best-effort.
- `sase.doctor.checks_deep_axe.check_axe_state()` currently duplicates desired/orchestrator/lumberjack classification.
  It must instead adapt the shared snapshot, while preserving pinned-log, orphan rotation-temp, and cumulative
  historical lumberjack error diagnostics.

## Implementation

### 1. Add typed AXE status wire models and the Rust-backed facade

Create a small AXE status model/facade module under `src/sase/axe/` with frozen dataclasses and constrained literal
types for every request and response record used by schema version 1:

- desired state, process observations, raw and derived orchestrator evidence;
- maintenance and hook/agent runner occupancy;
- raw and derived lumberjack records;
- latest lifecycle event;
- issue and collection-error records; and
- the complete request and classified snapshot.

Provide explicit recursive request serialization and response rehydration so callers receive typed immutable records
rather than an unvalidated nested dictionary. Keep a local `AXE_STATUS_WIRE_SCHEMA_VERSION = 1`, require the binding's
reported schema to match it, call `classify_axe_status` through `require_rust_binding`, and reject malformed or
wrong-version binding responses clearly. Do not add a Python fallback classifier. Export the stable collector/snapshot
surface from `sase.axe` for the CLI phase without changing `get_axe_status()`.

### 2. Collect one deterministic, read-only host observation

Add a collector module under `src/sase/axe/` whose public entry point returns the typed classified snapshot and whose
clock is injectable. The default clock must be timezone-aware; a fixed aware datetime must make generated timestamps and
all ages deterministic. Convert ages to non-negative whole seconds so the Rust rule remains healthy at exactly
`max(60, 3 * interval)` and becomes stale one second later.

Build the request as follows:

1. Load the effective AXE config once. Record configured runner maxima, sorted configured lumberjacks, intervals, and
   enabled chop names.
2. Call `probe_orchestrator(cleanup=False)` exactly as a read-only probe, retain lock-held plus all lock-holder,
   orchestrator-PID-file, and legacy-PID-file identities, and independently probe every present PID for liveness.
3. Read the validated desired-state and maintenance markers without invoking stale-marker cleanup. Include maintenance
   reason, owner PID, start time, and age.
4. Count live hook and agent runners using the existing global counters and pair them with the configured limits.
5. Read the most recent valid lifecycle journal event, preserving its event/outcome/source/success/reason/PID fields and
   calculating its optional age.
6. Inspect the union of configured lumberjack names and current state-directory names. For configured names, represent
   absent status as `not_reporting` inputs and use any readable PID observation; for readable status records, carry the
   reported state, timestamps/ages, counters, uptime, configured interval, and configured chops. For unconfigured names,
   include an observation only when its recorded PID is still live, so Rust can label it `orphaned`; ignore dead or
   vanished unconfigured artifacts.
7. Invoke only the shared Rust classifier and return its typed snapshot.

Treat file disappearance between listing, existence checks, and reads as an absent/best-effort observation. Optional
missing or malformed desired-state, maintenance, journal, and lumberjack records must not crash collection. Convert a
required config or host-input read/count failure into a typed collection-error request that Rust classifies as
`state=error`, `health=error`, and exit code 2. Do not convert binding/schema/programming failures into host collection
errors. The collection path must never call cleanup, ensure, start/stop/restart, systemd/subprocess management,
directory creation, or any state writer.

### 3. Make deep doctor consume the shared snapshot

Refactor `sase.doctor.checks_deep_axe.check_axe_state()` to call the collector once and derive current desired state,
orchestrator coherence/PID, lumberjack rows, current issue details, and present health from that snapshot. Map healthy
snapshots to `OK`, unhealthy snapshots to `WARN`, and collection-error snapshots to `ERROR`. Intentional maintenance,
intentional stop, and never-started states must not become warnings merely because no work is running.

Retain deep-only behavior after snapshot collection:

- scan for pinned AXE/lumberjack logs at the configured cap;
- measure excessive orphan log-rotation temp litter; and
- report nonzero cumulative lumberjack error totals as historical warnings without changing the shared snapshot's
  current health.

Preserve useful compatibility data keys where practical, add the snapshot's state/health/issues and runner occupancy,
and build deterministic, bounded details/next steps from Rust issue summaries/commands plus deep-only findings.
Deduplicate next steps. This refactor must remove the duplicate heartbeat threshold, desired-state, process-liveness,
and lumberjack-state classifier from doctor.

### 4. Add focused integration and regression coverage

Add focused AXE status tests using redirected temporary SASE state/config roots, fake PID liveness, and a fixed
timezone-aware clock. Cover:

- typed request serialization, binding schema enforcement, and exact typed response rehydration;
- healthy running, explicit stopped, fresh/not-started, desired-running/down, active maintenance, and every
  lock/PID-coherence shape visible to the host collector;
- configured lumberjacks that are missing, dead, error/stopped, fresh, exactly at the heartbeat threshold, and one
  second stale;
- cumulative historical errors remaining currently healthy;
- sorted/deduplicated configured chops and lumberjack output;
- live unconfigured lumberjacks included as orphans, while dead, disappearing, or malformed unconfigured state is
  filtered safely;
- missing/malformed optional marker, journal, and status files plus directory/file race simulations;
- required config/input failures becoming an error snapshot;
- exact `cleanup=False` probing and guards proving no cleanup/write/ensure/systemd helpers are invoked; and
- deep-doctor parity with snapshot health, including intentional states, collection errors, historical error totals,
  pinned logs, orphan temp files, deterministic issue details, and next steps.

Update existing deep-doctor tests to patch or build the shared snapshot rather than patching the removed duplicate
classifier inputs.

## Validation

During implementation:

1. Run `just install` before any repository checks so the workspace uses the completed linked-core binding.
2. Run the new AXE status tests and the existing deep-doctor/AXE process-state tests while iterating.
3. Re-run threshold, race, orphan, collection-error, and no-side-effect cases explicitly after the implementation is
   complete.
4. Run `just check` as the required full repository validation.
5. Confirm `git diff --check`, inspect the final diff for accidental CLI/TUI/core/version changes, verify bead
   `sase-8t.2` is the only bead being closed, and leave parent epic `sase-8t` open.

## Acceptance criteria

- Repeated collection against the same host observations and fixed clock returns the same typed schema-version-1
  snapshot.
- Lock/PID incoherence and partial lumberjack evidence are preserved for Rust classification rather than hidden by
  cleanup or Python-side simplification.
- Optional runtime-file races do not crash status collection, while required collection failures produce a stable error
  snapshot.
- No status collection path writes, removes, heals, starts, stops, ensures, or invokes systemd.
- Deep doctor and the shared snapshot agree on current desired/orchestrator/lumberjack health, and doctor still reports
  its deep-only historical/log/temp findings.
- Targeted tests and `just check` pass, `sase-8t.2` is closed, no new beads are created, and parent epic `sase-8t`
  remains open.
