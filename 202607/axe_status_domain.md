---
tier: tale
title: Portable AXE runtime status contract
goal: Every frontend can classify the same AXE runtime observations into one stable,
  tested version-1 status snapshot.
bead: sase-8t.1
parent: sase/repos/plans/202607/axe_status.md
create_time: 2026-07-23 07:43:43
status: wip
---

- **PROMPT:** [202607/prompts/axe_status_domain.md](prompts/axe_status_domain.md)

# Portable AXE runtime status contract

## Goal

Complete bead `sase-8t.1` by adding a pure, schema-versioned AXE runtime status contract to the linked `sase-core`
repository. Given the same host-collected observations, every frontend must receive the same normalized orchestrator
coherence, lumberjack states, overall lifecycle state/health, ordered actionable issues, summary, and exit code. The
Rust classifier must perform no filesystem or process I/O. Expose the operation as a JSON-shaped PyO3 binding, test both
the domain and Python conversion surface exhaustively, and leave release-owned Cargo versions untouched.

The implementation is confined to the linked `sase-core` repository. Open it through `sase repo open sase-core` before
reading or editing it.

## Existing conventions to preserve

- Add a focused `crates/sase_core/src/axe_status/` module, following the existing `axe_chop` split between serde wire
  records, pure classification, and colocated domain tests.
- Keep `sase_core` PyO3-free. Re-export the module and its public types, constant, error, and classifier from
  `crates/sase_core/src/lib.rs`.
- Follow the binding's established dict-to-`serde_json::Value`-to-wire and wire-to-plain-Python-dict conversion pattern.
  Domain/request failures become Python `ValueError`; no PyO3 classes cross the boundary.
- Use `#[serde(deny_unknown_fields)]`, snake-case enum values, owned types, and explicit nullable/list fields so the
  version-1 JSON shape is stable.
- Do not edit workspace/crate versions or local path-dependency version pins.

## Contract

Define `AXE_STATUS_SCHEMA_VERSION = 1`,
`classify_axe_status(&AxeStatusRequestWire) -> Result<AxeStatusSnapshotWire, AxeStatusError>`, and the serde wire
records needed for these inputs and outputs:

- Request metadata: schema version and nonblank `generated_at`.
- Nullable desired state with `running`/`stopped`, source, and timestamp.
- Raw orchestrator evidence: whether the lifecycle lock is held plus separate nullable PID/live observations for the
  lock-holder marker, orchestrator PID file, and legacy PID file.
- Nullable active maintenance details: reason, owner PID, started time, and nonnegative age in seconds.
- Hook/agent runner occupancy with current and configured maximum counts.
- Lumberjack observations: name, configured flag, optional positive interval, configured chops, nullable recorded PID
  and reported `running`/`stopped`/`error` state, process liveness, optional start/heartbeat timestamps and ages,
  cycle/error/uptime counters.
- Nullable latest lifecycle event with event kind, timestamp, source, outcome, success, reason, orchestrator PID, and
  optional age.
- Nullable typed collection error (`code` and `message`) so required host collection failures produce a normal `error`
  snapshot rather than being confused with malformed classifier input.
- Derived orchestrator state/coherence and normalized live PID identities; lumberjack state (`running`, `not_reporting`,
  `stale_process`, `stale_heartbeat`, `error`, or `orphaned`) and stale threshold; top-level state (`running`,
  `maintenance`, `stopped`, `not_started`, `down`, `degraded`, or `error`); health (`healthy`, `unhealthy`, or `error`);
  severity-tagged issue records; summary; and exit code.

The complete snapshot must serialize fields in this public order: `schema_version`, `generated_at`, `state`, `health`,
`summary`, `exit_code`, `desired_state`, `orchestrator`, `maintenance`, runner occupancy, `lumberjacks`, latest
lifecycle event, `issues`, and `collection_error`. Preserve raw observations inside the corresponding derived records so
later human and JSON frontends consume one object without reconstructing evidence.

## Validation and deterministic normalization

Return a path-specific `AxeStatusError { code, path, message }` for structural request failures. At minimum reject a
schema mismatch, blank required strings, duplicate lumberjack names, live process observations without a PID, a
configured lumberjack without a positive interval, and inconsistent optional PID/reporting fields. Serde must reject
unknown fields and invalid enum values. A collection-error request is still a valid classifier request and produces
state/health `error` with exit code 2.

Normalize output independently of input ordering:

- Sort/deduplicate live orchestrator PIDs and configured chop names.
- Sort lumberjacks by name. Ignore non-live unconfigured state-directory rows; retain live unconfigured rows as
  `orphaned`.
- Sort issues with an explicit semantic rank and stable subject/name tie-breakers, not incidental insertion or hash-map
  order.
- Keep cumulative historical error counts in output but do not create an issue when a configured process is live,
  reporting `running`, and has a fresh heartbeat.

## Classification rules

Derive orchestrator coherence first:

- Lock held plus exactly one unique live PID identity is coherent/running.
- Lock held with no resolvable live PID is incoherent.
- Any live published/lock-holder PID without the lock is incoherent.
- Multiple distinct live identities are incoherent, whether or not the lock is held.
- No lock and no live identity is stopped.

Emit stable issue codes and exact evidence summaries for lock-without-PID, live-PID-without-lock, and
conflicting-live-identity cases; suggest `sase axe stop --force` for incoherent/partial runtime evidence.

Classify each lumberjack with this precedence:

1. A live unconfigured row is `orphaned`.
2. A configured row with no valid reported status is `not_reporting`.
3. A reported PID that is not live is `stale_process`.
4. A live report whose state is `error` or `stopped` is `error`.
5. A live running report is `stale_heartbeat` when heartbeat age is strictly greater than `max(60, 3 * interval)`, or
   when no heartbeat exists and start age is strictly greater than that threshold.
6. Otherwise it is `running`; equality at the threshold remains fresh.

Reconcile lifecycle and health without treating intentional absence as an outage:

- A typed collection failure is `error`/`error`, exit 2.
- Any incoherent orchestrator evidence, live lumberjack without a coherent orchestrator, unhealthy configured lumberjack
  while the orchestrator is live, or live orphan is `degraded`/`unhealthy`, exit 1.
- Coherent orchestrator plus explicit desired `stopped` is degraded and tells the operator to run
  `sase axe stop --force`.
- No coherent/live runtime plus explicit desired `running` is `down`/`unhealthy`, exit 1, with `sase axe ensure`.
- Coherent orchestrator plus an active maintenance marker and otherwise healthy workers is `maintenance`/`healthy`,
  exit 0.
- Coherent orchestrator with healthy workers is `running`/`healthy`, exit 0.
- No live runtime plus explicit desired `stopped` is `stopped`/`healthy`, exit 0.
- No live runtime and no desired marker is `not_started`/`healthy`, exit 0.

Do not let historical stopped/stale status files make an intentional `stopped` or fresh `not_started` snapshot
unhealthy. When the orchestrator is live, emit name-specific stable issues for `not_reporting`, `stale_process`,
`stale_heartbeat`, and `error`; always emit an orphan issue for a live unconfigured lumberjack. Deduplicate next-step
commands naturally through the stable issue records.

## PyO3 API and documentation

In `crates/sase_core_py/src/lib.rs`:

- Add `axe_status_wire_schema_version() -> int`.
- Add `classify_axe_status(request: dict) -> dict`.
- Deserialize to the Rust request, invoke the pure classifier outside the GIL where appropriate, serialize the complete
  snapshot back to plain Python objects, register both functions in the module, and list them in the binding's
  module-level API documentation.
- Include conversion tests that assert the helper version, exact nested Python dictionary/list/null shape and
  ordering-relevant contents for a representative healthy request, and `ValueError` behavior for schema, structural, and
  unknown-field errors.

Update `crates/sase_core_py/PYPI_README.md` (and the root binding overview if needed for consistency) to document the
two new functions and make clear that Python supplies already-collected observations while Rust performs pure
classification.

## Domain tests

Add table-driven and focused Rust tests covering:

- healthy running, intentional stopped, fresh/not-started, explicit-running outage, and active maintenance;
- lock held without a live PID, live PID without a lock, and distinct live PID conflicts;
- configured lumberjack missing report, dead PID, exactly-at-threshold heartbeat, one-second-past stale heartbeat,
  missing heartbeat inside and outside startup grace, reported error, and reported stopped;
- a live unconfigured orphan and filtering of a dead unconfigured row;
- historical error counters on an otherwise healthy worker;
- desired stopped with a live orchestrator and partial lumberjack runtime without a coherent orchestrator;
- collection-error exit 2;
- malformed/schema-invalid requests;
- reversed/duplicated input ordering yielding sorted workers, chops, live identities, and identical ordered issues;
- stable serialization field names, enum values, nulls, lists, schema version, health, summary, and exit code.

Use helpers/builders to keep the state matrix readable, while making each assertion explicit enough to pin the public
contract.

## Verification and completion

From the linked `sase-core` checkout, run and fix until all succeed:

1. Targeted `sase_core` AXE-status domain tests.
2. Targeted `sase_core_py` binding tests.
3. `cargo fmt --all -- --check`.
4. `cargo clippy --workspace --all-targets -- -D warnings`.
5. `cargo test --workspace`.

Review `git diff --check`, confirm no release-owned versions changed, and inspect both the shell and linked-core
worktrees so unrelated changes are not altered. Then update only bead `sase-8t.1` to `closed`; verify parent epic
`sase-8t` remains open/in progress and do not create any beads.
