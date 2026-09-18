---
tier: tale
title: Authorize headless sudo completion and protect every answer path
goal:
  One durable sudo attempt owns approval through exactly-once headless settlement
  without weakening terminal-only approval elsewhere.
size: medium
proposed_by: bbugyi200.athena.sase-12w.6.2
bead: sase-12w.6.2
create_time: 2026-09-18 15:00:37
status: wip
---

- **PARENT:**
  [202609/sudo_detached_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/sudo_detached_landing_repairs.md)
- **BEAD:**
  [sase-12w.6.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-12w/sase-12w.6.2.md)

# Authorize headless sudo completion and protect every answer path

## Objective

Complete phase `sase-12w.6.2` by making one durable sudo execution attempt own the
approve decision from before authentication until its validated ledger is settled.
Permit completion without a controlling TTY only for the internal finalizer whose
operation request, persisted attempt, selected command IDs, manifest digest, started
handshake, and terminal ledger all match. Preserve the TTY restriction for every
ordinary CLI, generic gate, TUI, mobile, Telegram, fleet, and arbitrary detached caller.

This tale changes both the `sase-core` linked repository and the primary `sase`
repository. Shared attempt-state validation, liveness classification, and headless
settlement authorization belong in Rust core; Python remains responsible for locks,
durable files, local process/proc inspection, operation-request loading, and runner
adapters. Do not change manifest or ledger version-1 wire contracts and do not edit
release versions or memory files.

## Implementation

1. Add a Rust-owned sudo attempt policy and Python binding.
   - Extend `crates/sase_core/src/sudo.rs` (or a focused sudo submodule) with an
     additive version-1 attempt wire that binds the gate/request identity, acceptance
     identity, selected command IDs, manifest digest, controller/auth-start owner, local
     versus remote target metadata, handoff locations, startup state, validated started
     handshake, and finalizer proc where known.
   - Parse existing in-flight version-1 Python records explicitly: missing additive
     ownership/target/startup fields must become conservative legacy/unknown state,
     never proof that an executor is dead. Reject malformed or contradictory new
     records.
   - Given host-collected facts, classify executor ownership as `live`, `dead`, or
     `unknown`. Remote ownership and active/uncertain startup must never be checked
     against local `/proc` or classified dead merely because a local finalize proc is
     absent.
   - Add a settlement-authorization policy which succeeds only for the same persisted
     attempt and accepted approve decision after matching the durable internal operation
     request, selected IDs/digest, validated handshake, and validated terminal ledger.
     Make replay of the same completed authorization idempotent and reject stale,
     forged, mismatched, cancelled, denied, or conflicting facts.
   - Export the validators/policy through `sase_core_py` and thin wrappers in
     `src/sase/sudo/core.py`; cover Rust wire, legacy compatibility, state-transition,
     liveness, mismatch, and replay cases.

2. Reserve approval ownership before every execution mode.
   - Refactor `src/sase/sudo/cli.py`, `src/sase/sudo/detach.py`, and
     `src/sase/sudo/execution.py` so approval first performs one lock-ordered preflight
     under the gate acceptance/decision discipline, before the auth lease, capability
     selection, local/remote runner, or foreground fallback.
   - Reject existing `response.json`, cancellation, a conflicting accepted decision, and
     live or unknown attempts before any authentication or command execution. Claim the
     approve decision and persist an attempt/controller record first, then transition it
     durably through authentication, startup uncertainty, handshake, finalizer
     submission, settlement, and retirement.
   - Use the same reservation for detached, `--no-detach`, unsupported-runner fallback,
     and old-target fallback so no branch can repeat commands while another attempt is
     live or unresolved. Keep ordinary auth failures ledger-shaped and safely retryable
     only when core policy proves execution never started.
   - Serialize the generic deny and cancellation routes with that ownership. A live or
     unknown approve attempt cannot be answered as deny/cancel; its supported
     interruption is the recorded stop channel. Preserve the generic behavior for
     non-sudo gates.

3. Make recovery conservative and ownership-complete.
   - Persist target kind/host, local and remote handoff metadata, controller identity,
     and startup state before their corresponding side effects. Reconcile the durable
     runner `started.json` witness whenever stdout/handshake validation or finalize-proc
     submission fails, and retain the record/handoff whenever startup or executor
     liveness remains unknown.
   - Keep local process and proc probes in Python, but pass only facts to Rust for the
     decision. Do not perform SSH probes from `project_execution` or any TUI render
     path; cached durable `unknown` projects as executing/pending. Leave bounded remote
     probing and transport mechanics for phase `sase-12w.6.3` while preserving enough
     host/path state for that phase to recover the executor.
   - Distinguish the finalizer proc's failure from executor death. On settlement error,
     journal the failure and retain recoverable receipt/attempt/handoff evidence unless
     the policy proves safe retirement. On successful or idempotent settlement, retire
     local state exactly once; clean remote state only through the target-aware adapter.

4. Add the narrow headless completion seam.
   - Split generic gate execution only as much as necessary to complete an already
     accepted sudo approve decision. The internal path must consume a positive Rust
     authorization derived from durable facts, revalidate the current response/
     cancellation/decision state under the normal locks, run the existing sudo approve
     command/result and response persistence machinery, and preserve gate-shell
     settlement and failure journaling.
   - Do not key authorization on `source="sudo_cli"`, an environment variable, or any
     other caller-controlled string. Keep `_reject_unavailable_option_transport` and the
     controlling-TTY requirement unchanged for ordinary `execute_gate_selection` calls
     and all external surfaces.
   - Make a repeated finalizer after response publication return the existing answer
     without rerunning commands or continuations, then safely reconcile any remaining
     attempt state.

5. Add regression and acceptance coverage.
   - Extend `tests/test_sudo_execution.py`, `tests/test_sudo_detach.py`,
     `tests/test_sudo_gate.py`, and `tests/test_sudo_acceptance.py` for a genuinely
     headless finalizer (`has_controlling_tty` false), forged/stale/mismatched operation
     requests and attempts, exact replay, and a proof that direct/generic/non-sudo
     callers still receive `tty_required` or `unsupported_sudo_approval`.
   - Cover live and unknown attempts across detached, foreground, capability fallback,
     and remote metadata; lost startup output/handshake, proc-submission failure,
     already answered/cancelled gates, deny/cancel races, executor-versus-finalizer
     failure, preserved recovery evidence, and exactly one response/continuation.
   - Keep remote liveness tests adapter-based: assert no local PID probe and no blocking
     TUI projection for remote/unknown state. Do not absorb the SSH quoting, target-side
     probing, output-fetch, or integrated remote acceptance work assigned to phase
     `sase-12w.6.3`.

## Verification and completion

1. In `sase-core`, run focused sudo/core/binding tests, then its required `just check`.
2. In `sase`, run the focused sudo parser/core/runner/execution/detach/gate/acceptance
   and affected TUI projection tests. Run `just fix`, then the repository-required
   `just check`; use the SASE monitor workflow if verification becomes long-running.
3. Inspect both working trees for accidental release, generated, memory, or unrelated
   changes. Record any genuinely out-of-scope discovery only as a `PROPOSED FOLLOW-UP:`
   note on `sase-12w.6.2`.
4. Run `sase bead epic-symbols sase-12w.6.2` and resolve every remaining entry or re-key
   its Justfile line to the parent epic or a later open phase. Close only `sase-12w.6.2`
   with a note naming the headless, duplicate-prevention, recovery, Rust, and Python
   verification performed; do not close `sase-12w.6`, `sase-12w`, or any ancestor plan
   bead.
