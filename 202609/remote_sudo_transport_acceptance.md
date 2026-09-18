---
tier: tale
title: Complete remote sudo transport and detached acceptance
goal:
  Remote detached sudo execution is shell-safe, recoverable, streaming, and settled
  exactly once through the shared completion contract.
size: medium
proposed_by: bbugyi200.athena.sase-12w.6.3
bead: sase-12w.6.3
create_time: 2026-09-18 16:33:42
status: wip
---

- **PARENT:**
  [202609/sudo_detached_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/sudo_detached_landing_repairs.md)
- **BEAD:**
  [sase-12w.6.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-12w/sase-12w.6.3.md)

# Complete remote sudo transport and detached acceptance

## Context

Phase `sase-12w.6.3` must finish the remote half of detached sudo execution on top of
the ownership and settlement contracts landed by phases `sase-12w.6.1` and
`sase-12w.6.2`. The current adapter passes scripts and arguments as separate SSH argv
items even though OpenSSH reconstructs one remote shell command, uses `kill -0` for a
root-owned remote worker, never relays the remote `output.log`, and removes remote
handoff state after failures that may have happened after spawn. The durable Python
attempt records now distinguish local, remote, and unknown ownership, but do not retain
the remote handoff paths needed to reconcile or clean an uncertain remote attempt.

Keep manifest, handshake, ledger, and attempt schema versions at version 1. Preserve
old-target and old-runner synchronous fallback, terminal authentication failures,
headless-settlement authorization, duplicate-answer protection, stop-file cancellation,
and nonblocking TUI projection. Do not add credentials to argv, environment, logs, or
durable state, and do not change release versions manually.

## Implementation

1. Extend the shared Rust sudo attempt wire with optional validated remote handoff
   metadata, and expose it unchanged through the existing Python binding. Treat its
   absence as the valid legacy version-1 shape; require a remote target host and safe,
   absolute target paths when it is present. Add Rust and binding regressions for valid
   remote metadata, malformed metadata, and legacy records without it.

2. Extend `SudoExecutionState` and its version-1 reader/writer to round-trip the remote
   path metadata. Allocate the remote path set before staging, persist it with the
   reserved remote attempt before SSH side effects, and use the durable state rather
   than transient finalize-request data as the authority for reconciliation, stop,
   output, and cleanup. Keep old in-flight records readable and conservatively unknown.

3. Refactor `src/sase/sudo/ssh.py` around one remote-command encoder that passes a
   single shell-quoted command string after the SSH target. Apply it to contract probes,
   staging, synchronous and detached exec, JSON/ledger reads, output reads, liveness,
   stop, and cleanup. Make staging fail atomically on directory creation, permission, or
   manifest-write errors before any command launch, while keeping manifest bytes on
   stdin and secrets out of the remote command.

4. Replace signalling-based liveness with target-side `/proc` existence and boot/start
   identity inspection. Return a three-way live/dead/unknown result: permission or
   parsing problems and SSH failures remain unknown, while only a missing process or a
   verified identity mismatch is dead. Give every poll, stop, recovery, and cleanup SSH
   operation a bounded timeout; use bounded exponential backoff against the overall
   finalize deadline.

5. Stream the remote output file incrementally from a byte offset in bounded chunks to
   the finalizer's stdout, including a final drain when the ledger appears. Preserve the
   offset across transient transport failures, avoid duplicate output, and ensure stop
   requests continue to address the recorded host and handoff. A temporary unreachable
   target must leave the attempt pending with all recovery evidence intact.

6. Reconcile uncertain startup before cleanup or another approval. After staging has
   succeeded, SSH/handshake loss must retain the durable remote paths and classify the
   attempt as unresolved; a later recovery probes the recorded target for the started
   witness, ledger, and identity. Proven pre-spawn failures may clean up. Completed,
   cancelled, failed, and timed-out ledgers continue through the same validated local
   settlement path. Successful and idempotent settlement retry remote cleanup and only
   retire the durable attempt once cleanup is complete or safely unnecessary.

7. Replace argv-shape-only SSH tests with an isolated fake endpoint that joins the
   remote argv exactly as OpenSSH does and executes the resulting shell string in a
   temporary target root. Cover quoted/space-containing paths, staging sub-step
   failures, root-owned process inspection without signal permission, identity mismatch,
   network loss before and after possible spawn, incremental output without duplication,
   stop retention, cleanup retry, and detached-capability skew. Extend acceptance tests
   so a finalizer in a new session with no controlling TTY observes early output,
   settles exactly once, and leaves the gate pending plus recovery state for uncertain
   errors. Retain synchronous fallback and existing local acceptance coverage.

## Verification

- Reinstall the editable Python/Rust binding as needed with `just install` so tests use
  the linked core changes rather than a stale or missing extension.
- Run focused Rust sudo wire/binding tests and the linked `sase-core` repository's full
  required `just check`.
- Run focused Python sudo SSH, execution, detach, parser, gate, acceptance, TUI
  projection, and completion tests.
- Run `just fix`, then the primary repository's required `just check`; use the SASE
  monitor workflow for any long `just check` and for combined `just check-full` if the
  selection broadens or the phase's acceptance scope requires the exhaustive lane.
- Before closing `sase-12w.6.3`, run `sase bead epic-symbols sase-12w.6.3`, resolve or
  re-key every leftover, and close only this phase with a note naming the verified Rust
  and Python coverage.
