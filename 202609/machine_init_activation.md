---
tier: tale
title: Canonical machine initialization with verified activation
goal: All machine initialization entry points share an offline-plan, explicit-discovery
  workflow that reports enrollment honestly and verifies applied configuration before
  success.
size: medium
proposed_by: bbugyi200.athena.sase-xe.16.6
bead: sase-xe.16.6
status: done
---

- **PARENT:** [202609/remote_dispatch_completion.md](remote_dispatch_completion.md)
- **BEAD:**
  [sase-xe.16.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.6.md)

# Canonical machine initialization with verified activation

## Context

Phase `sase-xe.16.6` must replace the split ownership between `sase init machine`, the
init registry, and the existing `sase machine add` handler with one machine-owned
initialization workflow. Planning and check/preview surfaces must remain pure and
offline. Explicit apply must always be able to rescan beside enrolled machines, obtain
enrollment bundles without echoing secrets, preserve installation pins, report the full
enrollment outcome, deploy chezmoi-backed writes, reload the applied config, and
authenticate before claiming success.

The implementation must preserve the existing Rust-backed `MachineService` enrollment
and credential contracts and the remote-dispatch trust rules: discovery is not trust,
already-enrolled identities are skipped, a changed identity is never silently repinned,
and recovery after a consumed bootstrap is actionable.

## Implementation

1. Add a dispatch-owned machine-initialization planner/apply service and make
   `plan_init_machine` plus `run_init_machine` thin adapters. The planner will inspect
   only local merged configuration, emit no perpetual drift for zero-machine or
   all-enrolled registries, and never invoke providers. The apply workflow will run
   discovery only when explicitly invoked, list enrolled records alongside candidates,
   reconcile candidate identities against existing pins, skip exact enrolled matches,
   and route identity changes to `sase machine repair` without overwriting a pin.
2. Add sorted canonical `sase machine init` parser/handler wiring, retaining
   `sase init machine` and the init-registry spec as delegates to that same service.
   Support schema-versioned JSON where applicable, bounded gateway timeouts, and
   enrollment bundles from `-B/--bootstrap-file`, stdin, or a `getpass` hidden prompt.
   Preserve the `_init_input_func` and injected-stream seams so the interactive
   candidate/alias loop is behaviorally testable and no bundle is accepted through a
   bare echoed `input()` prompt.
3. Reuse the existing `machine add` enrollment-result serialization and success rules
   instead of printing unconditional enrollment. Quarantine and failed enrollment must
   exit nonzero with the quarantine/recovery reason, while JSON rows retain all
   enrollment fields. Only emit an enrolled success after activation verification.
4. Add an activation step after the machine record and credential are written. Resolve
   the actual applied machine-overlay target; on a chezmoi controller run the existing
   scoped `apply_chezmoi` operation from a durable tracked proc and honor its nonraising
   return code. Then clear/reload merged configuration, verify the applied record still
   has the expected endpoint/provider/pin and its credential resolves, and perform a
   bounded authenticated hello. A failed apply or partial activation will retain the
   recovery state and explain whether to retry apply or issue a new bootstrap and run
   `sase machine repair`.
5. Update `docs/init.md`, parser/help snapshots, and focused behavioral tests for the
   new canonical command, five-spec init order, offline check purity, hidden/file/stdin
   bundle inputs, rescan with one existing and one new identity, quarantine honesty,
   chezmoi source-versus-applied activation, authenticated verification, and partial
   recovery messaging.

## Verification

Run focused machine parser, init, dispatch-service, and chezmoi activation tests first.
Regenerate or update the committed CLI completion snapshot through the repository's
normal test/update surface if parser changes require it. Then follow the repository
verification memory: run `just check` after installing the workspace prerequisites if
needed. Before closing the phase, inspect `sase bead epic-symbols sase-xe.16.6`, resolve
or re-key every remaining symbol, and close only `sase-xe.16.6` with a note naming the
verified behaviors.
