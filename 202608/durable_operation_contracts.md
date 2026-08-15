---
tier: tale
title: Durable operation and result contracts
goal:
  Give ACE a versioned, secure, argv-only durable operation foundation and a complete
  machine-checked producer inventory for the dependent migration phases.
size: medium
proposed_by: bbugyi200.athena.sase-m9.3.1.1
bead: sase-m9.3.1.1
create_time: 2026-08-15 15:24:36
status: done
---

- **PARENT:** [202608/ace_proc_ownership.md](ace_proc_ownership.md)
- **BEAD:**
  [sase-m9.3.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.3.1.1.md)

# Plan: Durable operation and result contracts

## Outcome

Give the two ACE producer-migration phases a complete, testable durable-operation
foundation. Every durable submission will be representable as explicit argv plus stable
identity and concurrency metadata; private inputs and typed completion will use
versioned mode-0600 sidecars; and the current callable-based surface will have a
machine-checked inventory and an additive argv-only adapter ready for incremental
migration. This phase does not migrate producer call sites or remove the compatibility
callable adapter, because those changes belong to the two dependent phases.

## Current-tree evidence and boundaries

- The production ACE tree currently contains 54 submission calls: 53 producers plus
  `_submit_proc` forwarding into `_submit_tracked_proc`. Of those, 30 are direct calls
  (including the forwarding call) and 24 call a duck-typed `submit` alias obtained from
  `_submit_tracked_proc`. The inventory must be generated or checked structurally so a
  later call-site addition cannot silently escape classification.
- Existing `ProcSubmitRequest` already persists argv, concurrency keys, a request
  fingerprint, and optional request/result paths, while supervisor settlement already
  writes the result before terminal state. The missing pieces are a versioned operation
  contract, strict sidecar validation and permissions, operation/result identity
  matching, typed payload decoding, and an ACE submission path that never accepts a
  callable.
- Preserve legacy callable submissions unchanged for the dependent migration phases. Do
  not serialize a callable, import path, closure, or generic dispatcher into a request.
  Do not parse combined stdout/stderr as structured completion.
- Keep shared proc lifecycle and atomic concurrency reservation in the existing proc
  service/Rust-backed store. Keep domain work in existing patch, agent, bead,
  notification/gate, plugin, workspace/monitor, and run services and commands; do not
  add an `ace` command namespace.

## Implementation

1. Add a checked ACE proc-producer inventory that records, for every current submission
   site, its source identity, owning action, durable-versus-UI-only classification,
   owning domain command, minimal durable identifiers, typed result kind, stable
   fingerprint inputs, namespaced concurrency keys, optimistic UI effects, and restart
   recovery behavior. Include `_submit_proc`, `_submit_tracked_proc`, `ProcQueue`,
   `ProcMirror`, duck-typed protocols, representative test doubles, and worker
   completion paths. Add an AST/source conformance test that fails on an unlisted,
   duplicate, stale, or structurally changed production producer while allowing the one
   documented adapter-forwarding edge.

2. Introduce shared durable operation request/result models with explicit schema
   versions and strict validation. Requests identify the operation and contain only
   JSON-shaped domain data; results carry operation, proc id, success, message/error,
   and an operation-specific typed payload. Provide atomic read/write helpers that
   enforce regular-file ownership, mode 0600, schema support, expected operation/proc
   identity, and clear errors for missing, malformed, mismatched, or partial files. Make
   result publication atomic and preserve settlement's result-before-terminal invariant.

3. Extend `ProcSubmitRequest` and supervisor settlement around those contracts. Create
   private request and result paths before launch, write the versioned request before
   releasing the launch barrier, carry the expected operation in persisted settlement
   state, and validate/publish the typed command result before terminalizing. A command
   that exits successfully without a required valid result must settle as an explicit
   durable error; command failure/kill/timeout must still produce a valid error envelope
   without inferring data from logs. Keep legacy submissions with no operation contract
   backward compatible.

4. Add focused noninteractive command plumbing in the existing patch, agent, bead,
   notify/gate, plugin, workspace/monitor, and run domains. Each entry point accepts
   required stable identifiers positionally and optional policy through sorted CLI-rule
   compliant flags, invokes an existing domain service, and emits its declared typed
   result through the shared result writer. Payload-bearing operations consume their
   private request sidecar; no entry point accepts arbitrary Python callables, import
   paths, or a generic serialized dispatch target. Cover parser help and direct
   service/command execution sufficiently for later producer phases to wire argv without
   inventing a second protocol.

5. Add an additive ACE durable-submission adapter that accepts non-empty argv, an
   operation contract/request, result path, stable fingerprint, namespaced concurrency
   keys, attribution/display metadata, and an ephemeral UI completion callback. It must
   call the detached proc service off the Textual event loop, track only the returned
   durable proc id for presentation, and decode completion only from the typed result
   envelope. Keep callbacks optional and non-authoritative so a restarted ACE can
   recover from disk. Reject callable argv/request values at the API boundary and leave
   the legacy callable methods isolated for the dependent migrations.

## Verification

- Exercise request/result round trips, exact 0600 permissions, unsupported schemas,
  invalid JSON and non-object data, operation/proc mismatches, partial writes, missing
  results, result-before-terminal ordering, command failures, and a crash/reconcile
  boundary after result publication.
- Exercise stable fingerprint replay and overlapping namespaced concurrency keys through
  the existing two-process reservation tests, plus focused tests proving the new adapter
  submits argv off-thread and never executes or serializes a callable.
- Exercise every new/changed domain command's parser/help contract and at least one
  typed success/failure result per command family. Run the inventory conformance test
  against the live source tree.
- Run `just install`, focused proc/CLI/ACE tests while iterating, and finally
  `just check`. No rendered ACE output is intended to change, so the PNG visual snapshot
  suite is not required for this contract-only phase.
