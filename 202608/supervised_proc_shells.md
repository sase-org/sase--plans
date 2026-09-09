---
tier: epic
title: Supervisor-owned procs and the sase shell model
goal: "Every durable proc is detached and supervisor-owned, monitors are a proc-shell
  facade, ACE owns no proc execution, and SASE presents one coherent agent-and-shell
  taxonomy.

  "
phases:
  - id: shell-taxonomy
    title: Sase agent and shell taxonomy
    depends_on: []
    size: xlarge
    description:
      "shell-taxonomy: plan and land the behavior-preserving terminology migration,
      glossary model, compatibility aliases, CLI language, and generated memory
      surfaces."
  - id: proc-shell-platform
    title: Unified proc-shell platform
    depends_on:
      - shell-taxonomy
    size: xlarge
    description:
      "proc-shell-platform: plan and land the Rust-backed proc schema, atomic lifecycle,
      hardened detached supervisor, named proc shells, family attachment, and monitor
      service facade."
  - id: ace-proc-ownership
    title: Supervisor ownership for every ACE proc
    depends_on:
      - proc-shell-platform
    size: xlarge
    description:
      "ace-proc-ownership: plan and land durable command/result migrations for ACE proc
      producers, global concurrency enforcement, read-only observation, and
      detached-option retirement."
proposed_by: bbugyi200.athena.01x
status: done
bead_id: sase-m9
create_time: 2026-09-09 19:51:41
---

- **PROMPT:**
  [prompts/202608/supervised_proc_shells.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/supervised_proc_shells.md)
- **BEAD:**
  [sase-m9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/README.md)

# Plan: Supervisor-owned procs and the sase shell model

## Outcome and design principles

Deliver three implementation epics in order. Each phase is intentionally `xlarge`: its
worker must inspect the then-current tree, author a focused epic plan, obtain approval,
and execute that child epic rather than attempting this cross-cutting migration as one
change. The child plans must preserve these decisions unless new repository evidence
requires an explicitly documented revision:

1. A **sase agent is a sequence of sase shells**. A shell is the executing member; an
   agent-family container is not a shell.
2. A proc is a durable command execution record. Every newly created proc has immutable,
   non-empty argv and runs under a reparented, identity-checked supervisor; ACE and the
   initiating agent never own its lifetime.
3. A proc shell is a named proc. `shell_name` provides identity and addressability;
   `concurrency_keys` provides atomic set-overlap exclusion. A shell name implies one
   namespaced concurrency key, but the fields remain distinct.
4. A monitor is a family-attached proc shell and remains available through
   `sase monitor`, implemented as a direct service-level facade over proc services. It
   is neither a subprocess invocation of `sase proc` nor an extra wrapper process.
5. Historical evidence is immutable compatibility data. Continue reading legacy proc
   kinds and legacy monitor records even after new writers stop emitting them.
6. Terminal proc state is published only after the command is gone and all required
   output, workspace-claim, family, follow-up, and result settlement is durable.

Do not introduce a top-level `sase shell` command: “sase shell” names the union concept,
while `sase agent` and `sase proc` already provide the useful projections. Reserve
“interpreter” for the command interpreter so `--shell` cannot acquire two meanings.

## Phase 1 — Sase agent and shell taxonomy

Create a child epic for a behavior-preserving terminology migration before the process
architecture changes. Limit “lane” renames to the existing **agent lane** sense; do not
rename AXE scheduling lanes, scoped-test lanes, ACE display lanes, or launch-routing
lanes.

### Canonical glossary contract

Use these concise definitions as the semantic source of truth, polishing only for house
style:

- **Sase Agent** (alias: agent): A sase agent is an agent family or a single agent that
  belongs to no family. It owns an ordered sequence of sase shells. Its name never ends
  in `--<suffix>`; when it has one shell, the agent and shell may share a name, and when
  it becomes a family the bare name identifies the container.
- **Sase Shell** (alias: shell): A sase shell is one executing member of a sase agent,
  either an agent shell or a proc shell. A sase agent is a sequence of sase shells.
- **Agent Shell**: An agent shell is one concrete LLM/provider run. It holds a workspace
  claim while active and is the agent-execution row shown by `sase agent list`.
- **Proc Shell**: A proc shell is a named, supervised proc that belongs to a sase agent.
  It has durable output and lifecycle state; a family-attached proc shell is a sase
  monitor and may carry timeout, workspace-claim, and follow-up policy.

Replace the Agent Lane entry; update Agent Family to describe a sequential chain of sase
shells; update Proc so it no longer claims that `command`, `tui`, and `detached` are
enduring semantic kinds. The user has explicitly authorized these memory changes: edit
the canonical glossary memory and run `sase memory init` to regenerate all derived
instruction files and the memory README.

Introduce the `SaseAgentRef` / sase-agent projection vocabulary with temporary aliases
for serialized fields and internal callers where a hard cutover would be risky. Clarify
that `SASE_AGENT_NAME` identifies the concrete agent shell, while family projection and
the `SASE_AGENT=` commit footer identify the sase agent. Keep `sase agent list` as the
shell listing and make `sase agent kill` help explicit that it kills a live agent shell,
not a family container.

Rename monitor `--lane` language to `--agent`: use `-a/--agent` on start; on list retain
`-a/--all`, use `-l/--agent` during compatibility, and document the deliberate short
alias difference. Update docs, completions, errors, statuses, the monitor skill source,
and tests together. Validate generated files and prove the phase is behavior-preserving.

## Phase 2 — Unified proc-shell platform

Create a child epic that builds one process service and incrementally moves monitors
onto it. Inspect the linked Rust core using `/sase_repo` before changing it, and honor
the shared-backend boundary.

### Store and lifecycle foundation

Add an additive proc wire schema version with nullable/defaulted fields for
`shell_name`, immutable `artifacts_dir`, `reason`, boot-aware `supervisor_identity`,
integer `timeout_ms` and `idle_timeout_ms`, `request_fingerprint`, `concurrency_keys`,
`result_path`, `termination_reason`, and `stop_requested_at`. Integer milliseconds
preserve `ProcWire` equality. Keep supported schema versions an explicit set.

Implement Rust-locked atomic operations equivalent to:

- reserve: validate immutable non-empty argv, reject overlapping active concurrency keys
  or a conflicting active `(project, fully-qualified shell name)`, and append one
  pending row; identical active name plus fingerprint is an idempotent replay;
- request stop: persist intent without declaring the command terminal;
- finish: allow exactly one active-state owner to publish the terminal outcome.

Split legacy read compatibility from new-write validation. Legacy `tui`, `command`, and
`detached` rows—including commandless TUI rows—must remain visible indefinitely. Never
adopt a running legacy monitor into a new proc row; let its original supervisor and
compatibility reconciler finish it.

### One detached supervisor

Promote the hardened monitor supervisor/bootstrap into the proc kernel and retire the
weaker proc supervisor only after compatibility coverage exists. Require double-fork or
equivalent reparenting, bounded startup acknowledgement, a launch barrier, environment
scrubbing, boot-aware pid identity, total and idle timeouts, process-group termination,
reboot/loss reconciliation, and resumable exactly-once settlement. Killing the starter
agent or quitting ACE must not kill the proc.

Use one typed service request for CLI, monitor, bead/epic launch, and ACE callers. The
supervisor executes persisted argv directly. Store execution/control state in the proc
row and family lineage/presentation policy in the artifacts member, with immutable
cross-links. Add `shell_kind="proc"` rather than overloading family role. Move through a
`settling` phase, completing logs, claims, follow-up, artifacts, and result envelopes
before terminalizing the proc. Readers follow `Proc.log_path`; retention deletes only
store-owned paths beneath the proc log root and never artifacts-owned logs.

### Named shells and monitor facade

Add `-N/--shell NAME` to `sase proc run` and `sase proc list`; add exact-name resolution
to show and kill before exact-id and unique-prefix fallback. A value containing `--` is
fully qualified; a bare value attaches beneath the calling sase agent. Reject `/`, proc
id-shaped ambiguity, and invalid agent-name components. Scope active uniqueness to
`(project, fully-qualified name)`, allow reuse after settlement, and retain name
history. Word help as “named proc shell,” not command interpreter.

Make `sase monitor` call the same Python/Rust service layer directly and project the
same proc id. Preserve its family allocation, statuses, timeout, claim, output, and
`--next` behavior while removing duplicate monitor execution state. Compile its command
string to explicit `[/bin/sh, -c, ...]` argv before submission. Correct required-option
CLI-rule violations in the child design rather than perpetuating them; prefer a command
remainder positional and sensible optional policy defaults. Collapse bead task/epic
launch forks onto the service. Standalone proc shells may ship if inexpensive, but are
not a prerequisite for ACE because unnamed procs can carry concurrency keys.

Exercise crash boundaries between every settlement step, idempotent replay, name and key
conflicts from concurrent processes, pid reuse, reboot loss, quiet and invalid-UTF8
commands, partial output, background grandchildren, TERM-resistant groups, claim
transfer, follow-up suppression, log retention, and legacy records.

## Phase 3 — Supervisor ownership for every ACE proc

Create a child epic that inventories both ACE producer APIs and classifies every call
site. Recount `_submit_tracked_proc`, duck-typed callers, and `_submit_proc` against the
then-current tree rather than trusting the research snapshot. The final static invariant
is that no proc API accepts or executes a Python callable and no active proc is owned by
the ACE pid.

For durable, independently inspectable operations, move implementation out of TUI
actions/handlers into existing domain services and commands (`patch`, `agent`, `bead`,
`notify`, `plugin`, `workspace`, `run`, etc.), placing shared backend behavior in Rust
core. Do not create a `sase ace` namespace or a generic serialized-callable dispatcher.
Reduce input to durable identifiers; place large or sensitive payloads in versioned
mode-0600 request sidecars; write typed, versioned mode-0600 result envelopes to
`result_path`. Combined stdout/stderr is presentation, never a structured protocol.

Translate ACE’s in-memory dedup key, patch identity, exclusive-scope overlap, workspace,
AXE-slot, and agent-metadata locks into namespaced `concurrency_keys` reserved
atomically in the shared store. Preserve optimistic UI behavior while making completion
callbacks ephemeral conveniences: ACE restart must reconstruct authoritative state and
results from disk. Convert `ProcMirror` into a read-only, off-event-loop observer that
watches proc ids and active counts and marshals updates through the UI thread.
Declassify short UI-only work such as prompt stashing into ordinary threaded Textual
workers with no proc row; keep the event loop and serial message pump unblocked.

After every new proc path is supervisor-owned, remove `-d|--detached` from
`sase proc run`, `sase proc list`, and the legacy task alias. All procs are detached;
retain `--session` solely for attribution and make unattributed/global procs visible
regardless of an explicit session filter. For one release, accept the obsolete token
only through a suppressed compatibility parser that fails with: “all procs are detached;
remove --detached (use --session none for no attribution).” Keep a hidden legacy kind
filter as needed for history, but stop emitting semantic kinds for new rows.

## Cross-epic acceptance and landing

- One proc id and lifecycle are visible consistently through `sase proc`, the monitor
  facade, ACE Procs, and any family member in Agents.
- Concurrent identical shell starts execute once; differing fingerprints conflict with
  an actionable reference. Overlapping concurrency keys across two ACE instances also
  execute once. Names are reusable only after full settlement.
- Starter death, ACE quit/crash, bootstrap failure, reboot, stale/reused pids, stop, and
  timeout cannot orphan a live command, falsely terminalize it, duplicate a follow-up,
  or release a claim twice.
- Terminal status implies the command is gone and all durable settlement is complete.
  `%wait` and follow views advance only then.
- Historical proc kinds and legacy monitors remain readable and controllable; repeated
  retention cycles never delete artifacts-owned output.
- `sase proc run --help` has no detached option, `--shell` is unambiguous and validated,
  and all new/changed CLI help obeys the canonical CLI rules.
- ACE startup and interaction remain responsive: no new event-loop or message-pump
  blocking, no data-scaled startup work, and selective/coalesced refresh behavior stays
  intact.

Each child epic must run its proportionate focused tests throughout. Before its landing,
run `just install`, then `just check-full` through `/sase_monitor`; run
`just test-visual` for phases that alter ACE rendering and inspect/update PNG goldens
only for intentional changes. The final land agent must repeat whole-repository and
visual verification across the combined tree and document any compatibility window that
remains.
