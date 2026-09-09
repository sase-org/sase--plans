---
tier: epic
status: done
title: Unified proc-shell platform
goal:
  Replace the separate proc and monitor execution engines with one Rust-backed,
  atomically reserved, detached proc-shell service while preserving historical rows,
  family-attached monitor behavior, durable settlement, and existing observation
  workflows.
phases:
  - id: proc-store-lifecycle
    title: Atomic proc schema and lifecycle
    depends_on: []
    size: medium
    description:
      "proc-store-lifecycle: extend the Rust proc wire and Python parity models with
      additive/defaulted proc-shell, ownership, timeout, result, stop, and settlement
      fields; split permissive legacy reads from strict new-write validation; and
      replace generic append/update ownership with locked reserve, stop-request,
      supervisor-claim, and single-owner finish operations. Enforce immutable non-empty
      argv, active shell-name and concurrency-key conflicts, idempotent fingerprint
      replay, explicit supported schema versions, and settling-before-terminal
      invariants while keeping legacy command/tui/detached and commandless TUI rows
      readable. Add Rust, PyO3/wire-parity, and Python facade tests for concurrent
      reservations, invalid transitions, replay, legacy parsing, and retention ownership
      boundaries."
  - id: unified-proc-supervisor
    title: One detached proc service and supervisor
    depends_on:
      - proc-store-lifecycle
    size: medium
    description:
      "unified-proc-supervisor: introduce one typed Python proc submission request and
      service over the new Rust lifecycle, then promote the monitor
      bootstrap/supervision guarantees into the proc kernel: double-fork/reparenting,
      bounded acknowledgement and launch barrier, scrubbed child environment, boot-aware
      supervisor identity, direct persisted argv execution, merged binary-safe output,
      total and idle timeouts, process-group TERM-to-KILL escalation, stop intent, and
      reboot/pid-reuse/loss reconciliation. Move through settling and make settlement
      resumable and idempotent so the command, output, result envelope, workspace claim,
      artifacts, and follow-up policy are durable before terminal state. Keep
      Proc.log_path authoritative and only prune store-owned logs below the proc log
      root. Migrate ordinary proc CLI/API callers to the service, retaining
      compatibility reconciliation for already-running legacy rows and removing the
      weaker supervisor only after focused crash-boundary tests pass."
  - id: named-proc-shell-cli
    title: Named proc-shell addressing and CLI
    depends_on:
      - unified-proc-supervisor
    size: small
    description:
      "named-proc-shell-cli: add -N/--shell to proc run and list, resolve show/kill
      references by exact fully qualified shell name before exact id and unique id
      prefix, and derive bare names beneath the calling sase agent. Validate names
      against slash, proc-id ambiguity, invalid agent components, and malformed
      qualification; map each shell name to its distinct namespaced concurrency key
      without conflating the fields; scope active uniqueness by project and allow reuse
      only after settlement. Update filtering, JSON and rich renderers, help,
      completions, and tests using “named proc shell” language while preserving
      historical name visibility and avoiding a top-level sase shell command."
  - id: monitor-proc-facade
    title: Family-attached monitor facade and settlement
    depends_on:
      - unified-proc-supervisor
      - named-proc-shell-cli
    size: medium
    description:
      "monitor-proc-facade: reimplement monitor start/list/show/stop as direct calls and
      projections over the shared proc service so one monitor has one proc id and no
      duplicate execution state or wrapper process. Compile the monitor command
      remainder to explicit [/bin/sh, -c, command] argv, attach a shell_kind=proc
      artifacts member to the target sase agent, preserve family allocation, status
      labels, workspace-claim transfer, bounded output, total/idle timeout, stop, and
      exactly-once --next behavior, and store immutable proc/artifacts cross-links.
      Correct required-option CLI-rule violations by using a command positional and
      optional policy defaults, retain hidden compatibility aliases/readers for
      historical monitor records, and never adopt a live legacy monitor. Cover starter
      death, launch failures, quiet and invalid-UTF8 output, background grandchildren,
      resistant process groups, claim transfer/release, and follow-up
      replay/suppression."
  - id: proc-platform-cutover
    title: Service cutover and compatibility verification
    depends_on:
      - monitor-proc-facade
    size: medium
    description:
      "proc-platform-cutover: collapse bead task/epic launch monitoring onto the shared
      service and audit all remaining proc/monitor writers so new records use the
      unified lifecycle while legacy readers and control paths remain intact. Verify
      consistent ids, state, logs, names, and family projections across proc CLI,
      monitor CLI, agent listings, and current ACE observation without migrating
      ACE-owned producer APIs reserved for the following parent phase. Exercise
      concurrent processes, replay/conflict diagnostics, crash recovery at every
      settlement boundary, reboot and pid reuse, retention of artifacts-owned logs, and
      old store fixtures. Update the generated monitor skill source and user
      documentation, run just install and focused suites throughout, then run just
      check-full via sase monitor with a follow-up action; run visual tests only if
      existing ACE rendering changes."
proposed_by: bbugyi200.athena.sase-m9.2
parent_bead: sase-m9.2
bead_id: sase-m9.2.1
create_time: 2026-09-09 19:52:02
---

- **PROMPT:**
  [prompts/202608/unified_proc_shell_platform_1.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/unified_proc_shell_platform_1.md)
- **BEAD:**
  [sase-m9.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.2.1.md)

# Plan: Unified proc-shell platform

## Current architecture and boundary

The current Rust core (`crates/sase_core/src/procs`) stores schema-v2 `ProcWire` rows
and exposes append, partial update, read, and prune operations through `sase_core_rs`.
Those operations serialize access, but reservation and terminal ownership are still
assembled in Python and cannot atomically reject competing names or overlapping
concurrency keys. The Python proc path (`src/sase/procs`) records a row and starts a
single detached supervisor, while `src/sase/monitor` independently owns the stronger
launch transaction, family member, workspace claim, timeouts, output, and follow-up
settlement. This plan makes the shared Rust store the lifecycle authority and makes both
CLIs consumers of one Python service adapter.

Shared state transitions, name/key conflicts, and lifecycle validation belong in
`sase-core`; subprocess bootstrapping and integration with Python-owned artifacts,
claims, follow-up launches, CLI rendering, and compatibility adapters remain in this
repository. Changes to the linked core must land with their bindings and wire-parity
tests before Python callers rely on them.

## Durable lifecycle contract

New writers persist immutable non-empty argv and store-owned log identity at reserve
time. A reserve either creates one pending row, returns the same active row for an
identical `(project, shell_name, request_fingerprint)` replay, or returns a structured
conflict naming the active proc. Any overlap in normalized concurrency-key sets is also
an atomic conflict. A fully qualified shell name contributes one namespaced key, but
callers may add unrelated keys and readers must continue exposing both fields.

The supervisor claims the reserved row with a boot-aware identity before crossing the
launch barrier. Stop records intent and signals the identity-checked supervisor; it does
not publish a terminal outcome. Completion first enters `settling`, records that the
command and process group are gone, drains and closes output, writes any typed result,
settles claim/family/follow-up state idempotently, and only then performs the
single-owner finish operation. Reconciliation resumes settlement where possible and uses
explicit termination reasons for loss, reboot, timeout, stop, and launch failure.

Legacy schema versions and semantic kinds remain an explicit read contract. Their
missing fields receive compatibility defaults and their old supervisors retain
ownership. Strict validation applies only to new reserves, preventing legacy commandless
TUI rows from disappearing while ensuring no new commandless proc can be created.

## Compatibility and scope

`sase monitor` remains the ergonomic family-attached proc-shell interface, including its
handoff behavior and generated skill. It becomes a service-level facade rather than
invoking `sase proc` or maintaining a second store. Historical artifacts remain
resolvable by monitor id and family names; live legacy monitors settle through their
existing path and are never copied into new proc rows.

This child epic prepares but does not perform the following parent phase's ACE producer
migration. Existing ACE proc observation may be adapted to display the additive wire
fields, but callable execution APIs, global ACE concurrency conversion, and retirement
of `--detached` remain phase 3 work. That boundary keeps this plan independently
landable while delivering the process service it requires.

## Acceptance criteria

- Concurrent identical named starts execute once; mismatched fingerprints and
  overlapping concurrency keys return actionable conflicts.
- Killing the starter or closing ACE cannot kill or orphan a newly submitted proc; stop,
  timeout, reboot, pid reuse, and supervisor loss cannot falsely terminalize a live
  command.
- Terminal status means the command/process group is gone and output, result, claim,
  family metadata, and follow-up settlement are durable; replay cannot duplicate a
  follow-up or claim release.
- A new monitor and proc projection share one id, log, lifecycle, and control target;
  family/agent views retain monitor presentation and historical monitor records remain
  readable and controllable.
- Named shell resolution and reuse follow project-scoped active uniqueness, while bare
  names attach to the calling sase agent and invalid/ambiguous forms fail clearly.
- Retention never deletes artifacts-owned output, legacy proc kinds and commandless TUI
  rows remain readable, and new writers emit only the unified schema/lifecycle.
- New and changed CLI help follows the canonical option rules, particularly replacing
  required monitor options with positional input or optional defaults.
- Focused Rust and Python suites pass per phase; final verification runs `just install`
  and `just check-full` through the monitor workflow, plus visual snapshots only if ACE
  rendering changed.
