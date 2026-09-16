---
tier: tale
title: Enforce agent holds at runner-slot admission
goal:
  Matching pre-run agents park behind durable holds and release only after terminal
  armer settlement, while running agents and hold-store failures remain unaffected.
size: medium
proposed_by: bbugyi200.athena.sase-11l.3
bead: sase-11l.3
create_time: 2026-09-16 09:04:57
status: wip
---

- **PARENT:**
  [202609/hold_directive.md](https://github.com/sase-org/sase--plans/blob/main/202609/hold_directive.md)
- **BEAD:** sase-11l.3

# Hold-barrier admission for agents

## Outcome

Make active agent-hold records authoritative at the existing runner-slot admission
boundary. A matching WAITING or QUEUED agent receives a typed `hold-barrier` blocker and
parks; a candidate that has already claimed and become RUNNING is never re-evaluated.
Release agent-authored holds only when the armer's family generation has settled in any
terminal outcome, and release proc-authored holds when the proc reaches a terminal
status.

This phase spans the primary `sase` repository and the linked `sase-core` repository.
Keep shared matching/blocker policy in Rust and Python limited to identity/liveness
projection, locking, persistence calls, and settlement hooks.

## Required semantics

- Read the active hold snapshot during each admission attempt while the existing
  `runner_slots.lock` is held, and pass that exact snapshot into the capacity request
  evaluated before the claim callback. Do not introduce another admission lock and do
  not write dependencies into another launch's `waiting.json` or `ready.json`.
- Preserve the arm-versus-claim linearization rule: a hold visible before claim blocks
  and parks the candidate; a hold armed after claim cannot affect that RUNNING agent.
- Treat hold-store read, decode, lock-timeout, stale-armer, and malformed-record
  failures as fail-open for holds. Existing runner-capacity failures retain their
  current fail-closed behavior.
- Evaluate every active hold independently. If multiple holds match, emit one blocker
  per armer so the candidate remains blocked until all matching holds are gone.
- Make `hold-barrier` a resource blocker so held waiters park instead of spinning or
  retaining FIFO eligibility ahead of runnable work.
- Do not release an agent hold at the first shell handoff. Use the same
  family-generation settlement interpretation as
  `wait_dependency_resolution._index_queries.family_candidate_for_root`; success,
  failure, kill, skip, and loss are all terminal releases. Proc release keys only off a
  terminal proc row, never launch/submission.
- Keep the hot admission path bounded: one hold snapshot per poll under the existing
  lock, no unbounded lock wait, and no TUI event-loop disk I/O. Any hold snapshot used
  for ACE presentation must be captured in its existing worker-thread load boundary.

## Implementation

### 1. Extend the Rust capacity wire and blocker policy

In `sase-core:crates/sase_core/src/runner_capacity.rs`:

- Bump `RUNNER_CAPACITY_POLICY_SCHEMA_VERSION` for the wire change.
- Add a serde-defaulted active `holds` payload to `RunnerCapacityRequestWire` using the
  phase-4.2 `AgentHoldRecordWire` type.
- Extend `RunnerCapacityRecordWire` with the optional candidate identity and launch-time
  facts required to construct `AgentHoldCandidateWire`. At minimum this includes the
  requested `agent_name`; also carry the semantic workflow, clan, tribe, family, and
  numeric creation time when available so the existing pure predicate can honor all of
  its selector kinds and `future` without confusing the storage directory name with a
  workflow name. Keep fields optional/defaulted for old persisted scan rows and
  synthetic capacity records.
- Add optional, serde-defaulted `held_by` and `hold_expires_at` fields to
  `RunnerCapacityBlockerWire`.
- In `waiter_blockers`, project the waiting record into `AgentHoldCandidateWire`, call
  `hold_blocks_candidate` for each supplied hold, and emit deterministic `hold-barrier`
  blockers containing the armer identity, expiry, and the message
  `held by <armer> (expires in <t>)`. An invalid individual hold must not block the
  candidate. Keep the ordering stable for snapshot parity.
- Include `hold-barrier` in `has_resource_blocker` so the waiter is marked parked.
  Running records remain immune naturally because only waiter records pass through
  `waiter_blockers`.
- Update all Rust constructors, pyo3 round-trip fixtures, schema assertions, and
  serialization tests for the new defaulted fields. Add focused capacity tests for a
  WAITING candidate, a QUEUED candidate, a RUNNING record, project scoping, a future
  match, multiple simultaneous holds, removal of holds one at a time, blocker metadata
  and message text, and parked/resource classification.

Do not duplicate selector, scope, expiry, or kin-exclusion policy in the capacity
module; use `agent_hold::hold_blocks_candidate` as the authority.

### 2. Add a fail-open Python hold runtime adapter

Add a small Rust-backed facade under `src/sase/core/` for the phase-4.2 hold bindings.
It should:

- Strictly validate the Rust snapshot/record shape at its public boundary while
  returning an empty active snapshot to admission callers on missing bindings,
  malformed/unreadable state, bounded lock timeout, or liveness-collection failure.
- Supply host liveness facts for agent, proc, and CLI armers. For agent records, use the
  recorded root artifact/done-marker identity plus the wait-dependency family index so a
  completed handoff shell is still live until its effective family generation is
  settled. For procs, use `TERMINAL_PROC_STATUSES`; for PID-backed CLI records, use the
  existing process-liveness helper. Keep this host observation in Python and leave the
  Rust store/predicate I/O-pure beyond its own state file.
- Expose idempotent helpers to release a stored armer key, release all records whose
  proc identity matches a settled proc, and reconcile/release agent records whose
  recorded family generation has settled. Settlement and cleanup helpers must swallow
  hold-specific failures after recording/logging them: a courtesy hold must never make
  agent/proc settlement fail.
- Avoid repeated full artifact scans inside a single admission poll. Reuse the scan
  already captured by the runner where possible, cache per-project family indexes only
  within that call, and pass one validated active-hold list onward.

Add unit tests around the facade for valid active snapshots, malformed/unreadable store
fail-open behavior, dead PID pruning, family handoff versus final settlement, terminal
proc classification, and idempotent release.

### 3. Thread candidate identity and holds through every capacity adapter

Update `src/sase/core/runner_slots/_admission_capacity_records.py` so scanned and
synthetic records populate the new wire fields from `AgentMetaWire` (`name`, family,
workflow, clan, tribe) and derive the candidate creation epoch from the launch artifact
timestamp using the configured local timezone. Preserve `None` for identities or
timestamps that cannot be established; never manufacture a future match from the current
poll time. Update `enrich_candidate_from_records` so the runner's synthetic candidate
receives the same identity as its scanned artifact row.

Update `src/sase/core/runner_slots/_admission_snapshot.py` to accept an explicit active
hold snapshot and include it in the Rust request. Keep the snapshot helpers pure: they
must not read the hold store themselves. Update compatibility/request-shaping code and
all independent capacity producers (runtime admission, integration/CLI list projection,
and ACE's worker-thread capacity calculation) so a given source snapshot and hold
snapshot produce the same blockers.

For the ACE path, capture active holds in the existing off-event-loop agent-load compute
boundary and pass them to `refresh_runner_slot_context`; do not add store reads to
rendering, key handling, or Textual pump callbacks. The existing unknown-blocker
fallthrough in `_agent_queue_section.py` should then display the Rust message without a
new presentation branch. Full `held_by` row/JSON presentation remains phase 4.8.

Expand `tests/test_capacity_snapshot_parity.py` and targeted adapter tests to assert
that runtime, `sase agent list -j`, and ACE receive identical `hold-barrier` blocker
payloads and that candidate name/family/workflow/clan/tribe/future facts survive both
scan-backed and synthetic paths.

### 4. Enforce under the existing runner-slot lock

In `src/sase/axe/run_agent_wait_slots.py::_try_claim_runner_slot`, load the validated
active holds after acquiring `runner_slots.lock` and before building/evaluating the
candidate request. Pass them to `runner_capacity_snapshot`; leave the existing claim
callback under that same lock. A hold failure supplies an empty hold list, while errors
in capacity accounting continue to follow the existing admission error path.

Add runner-slot tests covering:

- WAITING and already-QUEUED candidates parking on a matching hold;
- admission after the last matching hold is released;
- multiple matching holds requiring all releases;
- malformed state and dead armer fail-open;
- `future` matching a candidate submitted after arm time but not an older candidate;
- both race orders under `runner_slots.lock` (hold visible before claim blocks; claim
  completed before arm remains RUNNING and is never re-evaluated);
- blocker text/metadata flowing through the current queue detail fallback.

### 5. Release on terminal settlement

After `write_done_marker_and_update_index` has made a terminal marker visible, invoke
the agent-hold reconciliation helper. It must locate the recorded root generation and
release only when `family_candidate_for_root(...).is_resolved` says the effective family
generation has settled. Hooking the shared done-marker writer should make ordinary
success, failure, gate/monitor handoff, and the TUI kill path converge on the same
behavior; add regression tests proving an intermediate handoff does not release and
success, failure, and kill of the final family member do.

After `finish_proc` returns a terminal row in
`src/sase/procs/settlement.py::settle_proc_shell`, idempotently release every hold whose
armer proc identity matches that row. The already-terminal early return should also
reconcile the hold so crash/retry settlement repairs stale records. Test success,
error/failure, kill, and repeated settlement. Do not treat a merely launched proc as
terminal.

### 6. Cross-repository integration and verification

- Update the pyo3 module documentation/index and registration only if the Python facade
  needs an additional binding beyond the phase-4.2 API. Keep all binding names
  statically discoverable by `tools/check_sase_core_rs_bindings`.
- Run the focused Rust suites for `agent_hold`, `runner_capacity`, and the pyo3 capacity
  round trip, then the focused Python facade, admission, settlement, parity, and queue
  rendering suites.
- Land/publish the Rust core change first, then run `tools/ratchet_core_revision` so
  `sase-core-revision.txt` names the core revision containing the new wire. Run
  `tools/check_sase_core_rs_bindings` and `tools/validate_sase_core_rs` against the
  matching build.
- Before finishing changes in the primary repository, read
  `sase/memory/lint_and_test.md` through the audited memory command and follow it. Run
  `just check`; leave `just check-full` to the epic land agent as specified by the
  parent design.
- Recheck both repositories for unintended changes. Run
  `sase bead epic-symbols sase-11l.3`, resolve every symbol or re-key it to the parent
  or a later still-open phase, then close only `sase-11l.3` with a note naming the Rust,
  Python, race/release, parity, and `just check` verification performed. Do not close
  `sase-11l` or any ancestor.
