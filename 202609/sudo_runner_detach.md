---
tier: tale
title: Add detached sudo runner execution and handshake validation
goal: Implement the sase-12w.1 auth-then-spawn runner contract in sase-core without
  changing synchronous runner behavior.
size: medium
proposed_by: bbugyi200.athena.sase-12w.1
bead: sase-12w.1
status: done
---

- **PARENT:**
  [202609/sudo_proc_execution.md](https://github.com/sase-org/sase--plans/blob/main/202609/sudo_proc_execution.md)
- **BEAD:**
  [sase-12w.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-12w/sase-12w.1.md)

# Plan

Implement the first phase of the approved detached sudo execution epic in the linked
`sase-core` repository. Keep the existing manifest and ledger schemas at version 1 and
preserve the current synchronous invocation byte-for-byte when `--detach-dir` is absent.

## Core handshake contract

- Extend `crates/sase_core/src/sudo.rs` with the schema-version-1 `sudo_exec_started`
  handshake wire, strict JSON parsing/validation, bounded field and path validation,
  manifest-digest validation, PID and timestamp checks, and tests for valid and rejected
  shapes.
- Re-export the handshake API and constants from `crates/sase_core/src/lib.rs`.
- Add `sudo_validate_handshake` to `crates/sase_core_py/src/lib.rs`, register it in the
  module, document it in the exported-function inventory, and extend binding tests to
  prove normalization and validation errors reach Python.

## Runner modes

- Refactor `crates/sase_gateway/src/sudo_runner.rs` argument parsing so `--capabilities`
  emits the pinned schema-version-1 capabilities document without a TTY or manifest,
  normal manifest execution remains unchanged, and `--detach-dir DIR` selects
  auth-then-spawn. Add complete help for all public flags; keep the root executor mode
  internal and reject inconsistent argument combinations.
- In detach mode, validate the handoff directory and sealed manifest, perform the
  existing `sudo -k` and interactive `sudo -v`, then invoke this binary once through
  `sudo -n -u root --` with internal executor arguments. Require a bounded spawn-success
  sentinel before invalidating the sudo timestamp and returning one validated handshake
  JSON.
- In the internal root executor, re-read and re-hash the manifest, detach with a new
  session and redirected output, resolve the requested account and drop supplementary
  groups/GID/UID for each direct command, preserve per-command timeouts and ledger
  output-tail rules, and poll the handoff `stop` file while waiting. Terminate a running
  command group on timeout or cancellation and retain stop-on-failure semantics.
- Write the unchanged ledger schema atomically inside the handoff directory with
  no-follow protections, refuse symlink targets, and leave the ledger/log readable by
  the invoking user. Generate a Linux process-identity token from boot ID and process
  start ticks for the handshake.

## Verification

- Extend the runner's fake-sudo fixture and non-detaching executor test path to cover
  capabilities, detached authentication failure, root-spawn failure, handshake fields,
  executor manifest re-verification, stop-file cancellation, `run_as` resolution
  failure, ledger/log behavior, and symlink refusal, while retaining every existing
  synchronous test.
- Run focused Rust tests while iterating, then run the linked repository's required
  `just check` gate. Inspect the final diff and repository status, run
  `sase bead epic-symbols sase-12w.1`, resolve or re-key any phase-owned symbol entries,
  and close only `sase-12w.1` with a note naming the checks that passed.
