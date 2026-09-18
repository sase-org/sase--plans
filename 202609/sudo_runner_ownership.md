---
tier: tale
title: Preserve detached sudo runner ownership and stream live output
goal:
  Every detached worker is durably owned or proved stopped, command progress is visible
  while running, and detach is advertised only where its identity backend is supported.
size: medium
proposed_by: bbugyi200.athena.sase-12w.6.1
bead: sase-12w.6.1
create_time: 2026-09-18 14:04:41
status: wip
---

- **PARENT:**
  [202609/sudo_detached_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/sudo_detached_landing_repairs.md)
- **BEAD:**
  [sase-12w.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-12w/sase-12w.6.1.md)

# Preserve detached sudo runner ownership and stream live output

Implement phase `sase-12w.6.1` in the opened `sase-core` repository. Keep the manifest,
ledger, and started-handshake schema-version-1 wire contracts intact; this phase owns
the Rust runner behavior and its tests, while Python attempt settlement and remote SSH
recovery remain in later phases.

## Implementation

1. Make detached capability advertisement reflect the process-identity backend. Treat
   detached execution as supported only on targets where the runner can create the
   detached session and derive the durable boot/start identity (Linux for the current
   `/proc` implementation). Return an empty capability list and reject detach mode
   before authentication or spawning on unsupported targets. Structure the predicate so
   tests can exercise both supported and unsupported results on the Linux test host; do
   not change synchronous runner support.

2. Establish a durable startup-ownership barrier between the privileged launcher and
   root worker. Pass the expected `started.json` path to the worker, keep the worker
   from opening the log or executing a reviewed command until it has read and validated
   a handshake matching its PID, process identity, manifest digest, and handoff paths,
   and atomically publish that witness from the launcher. If identity derivation,
   handshake validation, or witness publication fails after spawn, terminate and reap
   the still-barred worker before returning a ledger-shaped definitely-not-started
   runner error. Once the witness is published, preserve it on every outer-runner
   failure path; in particular, a final `sudo -k` failure must still return the
   validated started handshake (with a credential-free warning) rather than claim that
   execution never started. Preserve sealed-manifest revalidation, symlink-safe output
   creation, normal authentication-failure ledgers, stop files, process groups, and
   timeouts.

3. Replace detached command output collection with concurrent incremental pipe draining.
   Write stdout and stderr chunks to `output.log` as they arrive and flush them so
   progress is observable before command exit. At the same time, retain only the
   policy-sized bounded byte tail needed for the ledger (zero for `none`, the existing
   tail bound for `tail`, and the existing maximum for `full`), then apply the existing
   UTF-8-safe ledger truncation. Ensure timeout or cancellation cannot hang while
   joining drainers when a descendant retains a pipe, and preserve process-group
   termination. Leave the synchronous runner's existing externally visible behavior
   unchanged.

4. Extend `sase_gateway` runner tests with deterministic nonprivileged/fake-sudo
   coverage for: capability present/absent by identity backend; auth and true pre-spawn
   errors remaining ledger-shaped; a worker unable to execute before a valid witness;
   post-spawn identity/sentinel failures terminating and reaping the worker; final
   timestamp cleanup failure preserving and returning the durable handshake; output
   visible in `output.log` before command exit; large stdout/stderr retaining bounded
   ledger tails without unbounded collection; and cancellation/timeout completing even
   when descendants retain pipe file descriptors. Continue validating all produced
   handshakes and ledgers against the core version-1 contracts.

## Verification

- Run focused `sase_gateway` sudo-runner tests while iterating.
- Run repository formatting/lint checks as needed.
- Run the `sase-core` repository's required full `just check`, which includes the PyO3
  binding tests; do not edit release-managed versions.
- Confirm the primary SASE repository remains unmodified by this Rust-only phase.
- Before closing `sase-12w.6.1`, run `sase bead epic-symbols sase-12w.6.1` and resolve
  or re-key every remaining symbol, then close only this phase with a note naming the
  focused and full verification performed.
