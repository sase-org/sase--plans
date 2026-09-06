---
tier: tale
title: Durable fleet mutation journal and launch recovery
goal:
  Fleet mutations and launch retries recover one exact admitted run through durable,
  controller-scoped receipts and fencing.
size: medium
proposed_by: bbugyi200.athena.sase-xe.6
bead: sase-xe.6
create_time: 2026-09-06 18:19:00
status: wip
---

- **PARENT:** [202609/remote_dispatch_fleet.md](remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.6.md)

# Durable mutation journal and launch admission recovery

## Goal

Complete phase `sase-xe.6` by turning the existing pure fleet operation contracts into
durable, controller-scoped mutation and launch-admission machinery. The result must
guarantee that retries recover one previously admitted run, conflicting payloads never
reuse an operation key, expired keys never become fresh launches, and mutations never
act on a replacement process or run that merely reused a human name or PID.

This phase changes the linked `sase-core` repository (the transport-free `sase_core`
crate plus the `sase_gateway` enforcement layer) and, where required by the
two-repository boundary, the primary `sase` repository's binding contract tests,
validator expectations, and pinned core revision. Open the linked repository through
`sase repo open sase-core` and use only the path it prints. Do not put an ephemeral
workspace path into source or documentation.

The accepted design context is the epic plan `plan:202609/remote_dispatch_fleet.md` and
the consolidated report
`research:202609/remote_dispatch_and_fleet_focus/remote_dispatch_and_fleet_focus.md`.
Read the report with `sase artifact read` before implementation. Preserve these settled
boundaries:

- operation keys are scoped to the authenticated controller and paired with a canonical
  payload fingerprint;
- a same-key/same-payload retry returns the original receipt, a same-key/different-
  payload retry conflicts, and an expired/tombstoned key is rejected rather than
  silently accepted as new;
- feed cursors are unrelated to mutation preconditions;
- run identity is reserved durably before spawning, and recovery looks up the admitted
  reservation/run before it is ever allowed to spawn;
- human names, PIDs, paths, and provider labels are not mutation identity;
- `sase_core` remains transport-free, while HTTP/authentication and executor wiring
  remain in `sase_gateway`;
- `%dispatch` parsing, source-side intent/follow persistence, portable project
  preparation, Fleet read routes, and concrete remote kill/retry/gate/question routes
  belong to later epic phases and are not part of this tale.

## Implementation

### 1. Complete the transport-free mutation and fencing contracts

Refine `crates/sase_core/src/fleet_contract.rs` only where the existing version-1 types
are insufficient, and place stateful persistence/state-machine code in a focused fleet
operation module exported from `crates/sase_core/src/lib.rs` rather than growing the
already-large contract file further.

Keep `operation_payload_fingerprint()` as the one canonical, key-order-independent
fingerprint implementation. Add the wire records and validation needed to represent:

- an operation kind and controller-scoped key;
- a mutation precondition using an exact `AgentInstanceLocatorWire` plus its
  `ResourceRevisionWire` (or a launch reservation identity before an instance exists);
- operation states from admitted/pending through settled, rejected, and tombstoned;
- a stable receipt that can carry the authoritative exact instance/run locator and a
  bounded, non-secret result/error payload;
- a launch admission/reservation ID, the reserved exact locator, payload fingerprint,
  claim/fencing generation, owner identity, and any bound run/process identity needed
  for recovery;
- an exact-instance fence decision that compares the requested locator, revision, and
  recorded process identity with freshly owner-resolved facts. A mismatch must be a
  typed stale/precondition result before any mutation callback executes.

Prefer additive/defaulted fields where that keeps the just-landed version-1 binding
fixtures compatible. If an incompatible shape is genuinely necessary, bump the relevant
fleet schema deliberately and update every Rust/PyO3/contract snapshot fixture in the
same change; do not allow two meanings under one schema version.

Define and document one explicit retry-window policy in code and in the fleet API
contract. Full receipts must remain queryable for at least that window. After full
receipt compaction, retain a compact durable tombstone for every used scoped key so an
old/expired operation key can never become an unseen launch. The tombstone must retain
enough information to distinguish expired replay from conflicting reuse without
retaining full prompts, credentials, auth headers, or other secret/large payload data.

### 2. Add a crash-safe durable journal and launch-admission store

Implement a `FleetOperationJournal`-style store under the existing permission-
restricted `<sase_home>/fleet_gateway` state root. Use a transactional durable format
(SQLite is already an allowed `sase_core` dependency and is preferred here) with
exclusive uniqueness constraints for `(controller_id, operation_id)`, admission IDs, and
reserved exact run identities. Configure crash durability, create the directory and
database with owner-only permissions, and keep secrets/full prompts out of stored
receipts and audit events.

Expose narrow operations that each validate their wire inputs and execute one atomic
transaction:

1. Admit a general mutation or launch reservation. In one commit, decide replay versus
   conflict/expiry/precondition failure, reserve the exact target/run identity for a new
   launch, write the pending receipt, and append an audit/state transition. Nothing may
   spawn before this succeeds.
2. Claim an admitted launch with a monotonically increasing fencing generation and an
   opaque worker identity. Identical live-owner claims replay; another owner may take
   over only through an explicit stale/dead-owner recovery decision.
3. Bind a discovered or newly spawned run/process to the admission using the current
   fencing generation. Stale workers must be unable to bind, settle, or target the
   reservation after ownership changes.
4. Settle the operation and return/read the original receipt. Repeated settlement by the
   current owner is idempotent; incompatible settlement conflicts.
5. Expire/compact receipts to tombstones without dropping the permanent used-key guard.

Model the reserve/claim/finish protections in `crates/sase_core/src/procs/store.rs`, but
do not overload the proc JSONL store or copy its name-based replay rule. Fleet records
are keyed by controller operation identity and exact locators. Ensure multi-process
access is serialized by the database transaction itself (and use the project's
store-lock conventions only if an external migration/permission lock is still needed).

### 3. Enforce admission and recovery in the gateway layer

Add a focused gateway module that owns the journal service and a small executor/resolver
trait. Wire the journal into `GatewayState` beside `FleetCredentialStore`; production
state uses the configured SASE home, while tests can inject a temporary store and a
deterministic fake executor.

The orchestration contract for a launch must be:

1. authenticate the fleet bearer credential and derive `controller_id` from that
   credential, never from trusted request JSON;
2. canonicalize/fingerprint the mutation payload and reserve the operation plus exact
   run/admission identity transactionally;
3. on both first execution and retry, query the executor for an existing run bound to
   the admission before requesting a spawn;
4. acquire/renew a fenced ownership claim; the launched supervisor/worker must claim the
   reservation before it can start the real agent workload;
5. bind a found/new run and settle the original receipt; a lost HTTP reply therefore
   replays the same receipt;
6. return typed conflict, expired, stale-precondition, unavailable, or pending-recovery
   outcomes without local/other-host fallback and without minting a new operation key.

Provide authenticated fleet API wire/routes only for the reusable operation receipt and
launch-admission surface that later `dispatch-launch` will consume. Keep the payload
portable and opaque enough that this phase does not execute xprompt expansion or invent
the later phase's project/attachment validation. Add dedicated least-privilege fleet
scopes for operation receipt lookup and launch admission, include them in default
enrollment capabilities, enforce them per route, apply the existing body limit and
protocol negotiation, and derive key scope from the authenticated credential.

Update `crates/sase_gateway/src/wire.rs`, route exports/error mapping, and
`fleet_api_v1_contract_snapshot()` plus the committed
`contracts/api_fleet_v1/fleet_api_v1.json` snapshot together. The public contract must
state the retry/tombstone policy, exact-instance requirement, recovery outcomes, and
that a source timeout is not evidence of nonexecution. Avoid exposing PIDs, filesystem
paths, raw process identities, bearer tokens, or full prompts in receipts/errors.

If the current target launcher cannot yet execute a portable launch, keep the production
executor explicitly unavailable after durable admission and return a recoverable
pending/unavailable outcome; do not call the legacy mobile name-based launch path. The
injected executor and state-machine tests must nevertheless prove the admission/recovery
protocol that the later dispatch phase will connect to the real launcher.

### 4. Test every idempotency, crash, concurrency, and fencing boundary

Add table-driven unit/integration tests in `sase_core` and `sase_gateway` covering at
least:

- canonical fingerprints and controller scoping;
- first admission, same-payload replay, different-payload conflict, exact-target or
  revision mismatch, receipt expiry, compaction, and durable tombstone rejection after
  reopening the store;
- concurrent submissions from two handles/process-like store instances proving one
  committed admission and one authoritative receipt;
- permission modes and reopen/recovery of the journal;
- crashes/fault injection after reservation, before executor invocation, after executor
  spawn, after run discovery/binding, after settlement, and before the reply; every
  replay must find or return the same admitted run and the fake executor must observe at
  most one real workload start;
- stale claim takeover and rejection of work from an older fencing generation;
- same human name with another exact locator, and the same PID with another recorded
  process identity, both rejected before the mutation callback runs;
- route authentication, controller-ID spoof resistance, capability/scope denial,
  body/version validation, stable receipt lookup, lost-reply replay, and redacted
  response/error bodies;
- freshness of the committed fleet API snapshot.

Keep or extend the existing PyO3 smoke tests for any changed fleet contract function,
and update `tools/validate_sase_core_rs` plus its tests if new bindings are part of the
public boundary. Do not duplicate journal behavior in Python.

## Verification and completion

Before finishing, read `lint_and_test.md` through `sase memory read`. Run the linked
`sase-core` repository's mandatory `just check` (never only `cargo test -p sase_core`).
In the primary `sase` repository, run `just install` if the isolated editable
environment is stale, run the focused fleet binding/validator tests while iterating,
then run mandatory `just check`. Keep `tools/validate_sase_core_rs` green and update
`sase-core-revision.txt` to the resulting linked-core revision according to the
two-repository workflow; do not change release-plz-owned crate versions manually.

Do not create follow-up beads. Record any genuinely out-of-scope discovery on
`sase-xe.6` as `PROPOSED FOLLOW-UP: ...`. Before closing, run
`sase bead epic-symbols sase-xe.6` and resolve every remaining `--epic-symbol` entry or
re-key its Justfile annotation to the still-open parent/later phase that owns it. Close
only this phase with:

```text
sase bead close sase-xe.6 --note "<journal, recovery/fencing, contract, and checks verified>"
```

Do not close `sase-xe` or any ancestor bead.
