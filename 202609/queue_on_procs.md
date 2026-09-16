---
tier: tale
title: Allow queue-capacity admission on proc launch units
goal: Stand-alone proc units honor authored queue capacity, priority, and weight before
  dispatch while preserving existing agent admission behavior.
size: medium
proposed_by: bbugyi200.athena.sase-11l.1
bead: sase-11l.1
status: done
---

- **PARENT:**
  [202609/hold_directive.md](https://github.com/sase-org/sase--plans/blob/main/202609/hold_directive.md)
- **BEAD:**
  [sase-11l.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11l/sase-11l.1.md)

# Allow queue-capacity admission on proc launch units

## Goal

Complete phase bead `sase-11l.1` by allowing `%queue` / `%q` fields on a stand-alone
`%proc` launch unit. A proc with authored queue intent must wait at the typed-launch
admission boundary until the shared runner-capacity engine admits it. Proc candidates
default to queue weight `0`, so `%proc ... %q:1` drains existing occupied load without
consuming a runner slot itself; an authored weight participates normally. Existing agent
`%queue` behavior and the shared queue parser's validation remain unchanged.

## Implementation

1. In `sase-core`, extend `ProcUnitWire` with optional queue capacity and priority plus
   queue weight/provenance fields. Parse `%queue` for proc units through the same shared
   queue collector used by agent units instead of reporting `agent-directive-on-proc`.
   Populate a proc's implicit weight as `0` while preserving whether a non-default
   weight was authored. Include the proc queue fields in serialization, launch content
   digests, and the approval preview. Keep invalid and zero capacities governed by the
   existing shared queue contract, and update Rust tests for accepted spellings,
   authored weight, validation, preview, and digest sensitivity.

2. In the Python launch-wire adapters, add the matching proc queue fields to dataclass
   hydration and serialization, with backward-compatible defaults for plans that predate
   the fields. Add contract tests that prove queue intent round-trips from Rust through
   Python without changing agent payload behavior.

3. Add a proc capacity-admission helper that projects a stable synthetic candidate into
   the existing Rust-backed runner-capacity snapshot using the same host scan, liveness
   probe, effective limit, priority/FIFO ordering, and blocker interpretation as agent
   admission. Use proc weight `0` by default and the authored weight when supplied. Keep
   this layer as Python plumbing; the authoritative capacity decision remains in
   `sase-core`.

4. In `AdmissionEngine`, evaluate that helper before journaling a proc as `dispatching`.
   If the decision is blocked, leave the unit `eligible`, expose the blocker details in
   admission progress, and retry on the normal poll; never reserve or dispatch a proc
   while blocked. Treat malformed/invalid capacity decisions as launch errors rather
   than admitting work without an authoritative result. Units without authored queue
   fields retain the existing direct-dispatch path.

5. Add focused tests showing that `%queue` on `%proc` no longer raises
   `agent-directive-on-proc`, approval previews and digests reflect queue fields, legacy
   proc plans still hydrate, and dispatch behavior is correct: a weight-0 `%q:1` proc is
   blocked while a weight-1 agent occupies the host and dispatches after the host
   becomes idle; an authored proc weight affects fit; blocker information is surfaced;
   and ordinary proc and agent admission remains unchanged. Keep the directive-contract
   parity suite green.

## Verification and completion

Run focused Rust and Python tests while iterating, then run `just check` in both
`sase-core` and the primary repository. In the primary repository, run the core binding
validation/ratchet workflow required for a linked core change. Before closure, run
`sase bead epic-symbols sase-11l.1`, resolve or re-key every remaining symbol, and close
only `sase-11l.1` with a note naming the verified parser, wire, admission, and
regression coverage. Do not close the parent epic or create follow-up beads; record any
discovered out-of-scope work as a `PROPOSED FOLLOW-UP:` note on this phase.
