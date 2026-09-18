---
tier: tale
title: Detached sudo answer and finalize proc
goal: Local sudo approvals can release the terminal after authentication while a supervised
  proc safely completes and settles the gate.
size: medium
proposed_by: bbugyi200.athena.sase-12w.2
bead: sase-12w.2
status: done
---

- **PARENT:**
  [202609/sudo_proc_execution.md](https://github.com/sase-org/sase--plans/blob/main/202609/sudo_proc_execution.md)
- **BEAD:**
  [sase-12w.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-12w/sase-12w.2.md)

# Detached sudo answer and finalize proc

## Objective

Complete phase `sase-12w.2` by adding an opt-in local detached execution path to
`sase sudo answer`, backed by the auth-then-spawn runner contract from the closed runner
phase. The foreground CLI must return once authentication and executor startup are
proven, while a durable gate-kind proc streams output, validates the unchanged sudo
ledger, answers the gate, and settles its shell. Existing synchronous behavior remains
the default and must not change.

## Constraints and invariants

- Add mutually exclusive `-D/--detach` and `-N/--no-detach` answer options, with
  synchronous execution still selected when neither is present. Register `sudo finalize`
  as a hidden internal subcommand; do not expose its sidecar payload on argv.
- Detached execution is local-only in this phase. A remote target requested with
  `--detach` must remain pending and explain that the user can retry without detach.
- Probe `sase_sudo_runner --capabilities` before creating a detached attempt. If the
  installed runner does not advertise `detached_execution`, emit a visible fallback
  notice and run the existing synchronous path.
- Keep the reviewed manifest durable for the executor in a user-owned `0700` handoff
  directory under the SASE state tree. Validate the runner's `sudo_exec_started`
  handshake through `sase_core_rs.sudo_validate_handshake` and never duplicate the Rust
  wire validation in Python.
- Serialize in-flight record transitions with the gate bundle's `.response.lock`. A live
  record prevents a second approval and names its finalize proc; a record is stale only
  when both its proc and its identity-matched executor are dead. Clean only validated,
  SASE-owned handoff paths.
- Preserve the gate pending state until the finalizer has a valid terminal ledger.
  Non-terminal runner outcomes and finalizer failures must leave it answerable, while an
  already-written `response.json` makes a retried finalizer settle-only and idempotent.
- Submit the finalizer through `submit_proc_request` with `shell_kind="gate"`, a
  readable label, the summed command timeout, and a durable operation request carrying
  the handoff and handshake facts plus the selected command ids and answer metadata.

## Implementation

1. Extend the Python runner/core seams without disturbing synchronous calls.
   - Add handshake validation to `src/sase/sudo/core.py`, calling the exact
     `sudo_validate_handshake(handshake, manifest)` Rust binding and translating
     failures to a targeted `GateError`.
   - In `src/sase/sudo/runner.py`, add a capability probe and a file-based detached
     invocation that passes `--detach-dir`, parses either the started handshake or the
     existing ledger-shaped authentication failure, and retains current executable
     resolution, timeout, interrupt, and exit-code mapping. Keep
     `run_sudo_runner`/`run_sudo_runner_file` unchanged for non-detached callers.

2. Add a focused detached-execution lifecycle module under `src/sase/sudo/`.
   - Define versioned execution-state serialization for the gate id, selected commands,
     manifest digest, handoff directory, validated handshake/executor identity, and
     finalize proc id. Use atomic, permission-safe JSON and the existing gate lock.
   - Create/write the per-attempt handoff directory and sealed `manifest.json`; expose
     helpers to load, update, clear, and project execution state. Classify record
     liveness using the proc store and `process_identity_matches`, clear dead/dead
     records with a notice, and reject any record whose proc or executor is still live.
   - Centralize bounded output-tail streaming, stop-file creation, ledger polling, and
     ownership-checked cleanup so CLI presentation and finalization do not each invent
     filesystem rules.

3. Wire the detached answer path and durable finalizer into the CLI.
   - Extend `src/sase/main/parser_sudo.py` with the alphabetized answer flags and hidden
     `finalize` parser, including internal operation-I/O support where needed. Update
     dispatch in `src/sase/sudo/cli.py` while preserving feature gating and all current
     answer error envelopes.
   - Refactor `_approve` only enough to share manifest selection and final receipt
     application. For `--detach`, refuse remote targets, probe capabilities, then under
     the auth lease and response-lock lifecycle create/claim the record, invoke the
     auth-then-spawn runner, validate its handshake, and submit
     `sase sudo finalize <gate-id> --json`. Persist the proc id before releasing the
     lease and return stable JSON/human `execution_started` output with `proc_id` and
     `request_id`; on any pre-submission error, retain only the state needed to
     recognize a live executor and safely clean an otherwise dead attempt.
   - Implement the internal finalizer to load and cross-check its required operation
     sidecar against the execution record, incrementally copy `output.log` to stdout,
     and wait for `ledger.json` while verifying executor pid identity. On SIGTERM write
     `stop`, allow a bounded grace period, and exit nonzero as killed. On a ledger,
     reuse `_reject_non_terminal_auth`, `validate_sudo_receipt`,
     `execute_gate_selection`, and `_settle_shell`; carry the original feedback/retry
     metadata so detached answers match synchronous semantics. Record redacted journal
     failure evidence for executor death/timeout, clear the record, and clean the
     handoff after durable completion or failure. If the response already exists, skip
     execution directly to shell settlement before cleanup.
   - Extend `sudo show`/`list` payloads and human output with a live `executing` state
     and finalize proc id, without changing the persisted gate-shell state or treating
     execution as an answered gate.

4. Add regression and lifecycle coverage.
   - Extend parser, runner, and sudo unit tests for flag mutual exclusion/defaults,
     hidden finalizer dispatch, capability parsing/fallback, handshake validation,
     durable handoff permissions, atomic execution-record transitions, live duplicate
     rejection, stale recovery, and executing-state projection.
   - Extend `tests/test_sudo_acceptance.py` with a fake capable runner/executor that
     writes delayed output and a ledger. Verify detached answer returns after the auth
     phase with the gate pending, the finalize proc uses an operation sidecar and gate
     shell kind, output is visible in its log, and successful finalization produces the
     same receipt/settlement shape as the synchronous flow.
   - Cover remote refusal, old-runner synchronous fallback, non-terminal ledgers,
     executor death without a ledger, timeout/SIGTERM stop-file behavior, response
     idempotency, safe cleanup, and unchanged synchronous answer behavior.

## Verification

Run focused sudo parser/runner/gate/acceptance tests first, then the repository's
standard agent verification (`just check`). Before closing `sase-12w.2`, run
`sase bead epic-symbols sase-12w.2`, resolve or re-key every remaining phase symbol, and
close only this phase with a note naming the focused tests and `just check` result.
