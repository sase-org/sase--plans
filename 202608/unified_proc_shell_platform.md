---
tier: epic
title: Unified proc-shell platform
goal:
  Replace the split proc and monitor execution stacks with one Rust-backed, atomically
  reserved, supervisor-owned proc service that supports named proc shells and preserves
  legacy history.
phases:
  - id: atomic-proc-lifecycle
    title: Atomic Rust proc lifecycle
    depends_on: []
    size: large
    description:
      "atomic-proc-lifecycle: add the additive proc wire schema and Rust-locked reserve,
      stop-request, and finish operations with strict new-write validation, immutable
      argv and ownership fields, idempotent fingerprints, active name/key conflict
      detection, and permanent legacy-row read compatibility; expose the operations
      through the Python binding and migrate the Python proc model/store adapter with
      parity, concurrency, and migration tests."
  - id: detached-proc-kernel
    title: Hardened detached proc kernel
    depends_on:
      - atomic-proc-lifecycle
    size: large
    description:
      "detached-proc-kernel: replace the weak proc launcher with a double-forked,
      acknowledged, environment-scrubbed supervisor that executes persisted argv behind
      a launch barrier, records boot-aware identity, enforces total and idle timeouts,
      terminates process groups, reconciles loss/reboot safely, and settles output and
      result state exactly once before terminal publication; retain compatibility
      control for already-running legacy proc and monitor supervisors."
  - id: named-proc-shell-service
    title: Named proc-shell service and CLI
    depends_on:
      - detached-proc-kernel
    size: large
    description:
      "named-proc-shell-service: introduce one typed Python proc service over the Rust
      lifecycle and detached kernel, including immutable artifact/result cross-links and
      concurrency keys; add validated -N/--shell naming to run/list and exact shell-name
      resolution to show/kill, compile commands to non-empty argv, keep command input
      positional, update completions/help/rendering, and cover name qualification,
      reuse, ambiguity, retention, idempotency, and concurrent overlap behavior."
  - id: monitor-proc-convergence
    title: Monitor facade and family convergence
    depends_on:
      - named-proc-shell-service
    size: large
    description:
      "monitor-proc-convergence: reimplement monitor start/list/show/kill/follow as a
      direct service facade over the same proc id and lifecycle while preserving family
      allocation, shell_kind=proc lineage, claims, statuses, timeouts, logs, follow-up
      policy, and --next behavior; collapse bead and epic monitor launch forks onto the
      service, retain immutable legacy monitor readability/control without adoption,
      remove duplicate execution state only after compatibility tests, and verify
      crash-boundary settlement and starter-death independence end to end."
proposed_by: bbugyi200.athena.sase-m9.2
parent_bead: sase-m9.2
create_time: 2026-09-09 20:00:34
status: wip
---

- **PROMPT:**
  [prompts/202608/unified_proc_shell_platform.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/unified_proc_shell_platform.md)
- **PARENT:**
  [202608/supervised_proc_shells.md](https://github.com/sase-org/sase--plans/blob/main/202608/supervised_proc_shells.md)

# Plan: Unified proc-shell platform

## Outcome

Build one durable process platform whose authoritative lifecycle is reserved and settled
atomically in the Rust core, whose commands always outlive their initiating agent under
one hardened detached supervisor, and whose optional names turn procs into addressable
proc shells. `sase monitor` becomes a family-attached facade over that platform rather
than a second execution system.

The work spans the linked Rust core and the Python/CLI repository. Shared schema,
locking, conflict, transition, and lifecycle rules belong in Rust; Python owns thin
service adapters, process bootstrap glue, CLI presentation, and family/artifact policy.

## Invariants preserved by every phase

- New proc submissions persist immutable, non-empty argv before execution. Historical
  `tui`, `command`, and `detached` rows, including commandless TUI rows, remain readable
  indefinitely even though new writers use the stricter contract.
- New durable commands are reparented and supervisor-owned. Neither ACE, the CLI, nor
  the initiating agent owns the command lifetime. Bootstrap failure cannot release a
  command past the launch barrier.
- A terminal proc means its process group is gone and all required output, result,
  claim, artifact, family, and follow-up settlement is durable. Stop persists intent; it
  never pretends that signaling is completion.
- `shell_name` is identity and `concurrency_keys` are exclusion. A fully qualified shell
  name contributes a namespaced concurrency key, while arbitrary additional keys use
  atomic set-overlap exclusion. Active identical name plus fingerprint replays the same
  proc; a different fingerprint conflicts with an actionable proc reference.
- Proc logs are read through the stored `log_path`. Retention removes only store-owned
  paths below the proc log root and never artifacts-owned output.
- Legacy running monitors keep their original rows and supervisors. Compatibility
  reconciliation completes them in place; no migration adopts them into duplicate proc
  executions.

## Phase details

### 1. Atomic Rust proc lifecycle

Advance the proc schema as an explicit supported-version set. Add nullable/defaulted
fields for `shell_name`, `artifacts_dir`, `reason`, `supervisor_identity`, integer
`timeout_ms` and `idle_timeout_ms`, `request_fingerprint`, `concurrency_keys`,
`result_path`, `termination_reason`, and `stop_requested_at`. Preserve integer-based
wire equality and serde defaults for older rows.

Separate permissive legacy decoding from strict validation for new writes. Under the
existing proc-store lock, implement typed operations with these semantics:

1. Reserve validates immutable argv and identifiers, rejects overlap with every active
   concurrency key or active `(project, fully-qualified shell name)`, returns the
   existing active row for an identical fingerprint replay, and otherwise appends one
   pending row atomically with retention.
2. Request-stop stamps intent exactly once on an active row without changing it to a
   terminal status.
3. Finish allows one active lifecycle owner to move through settlement and publish one
   terminal outcome; stale/repeated finish calls are deterministic no-ops or explicit
   ownership conflicts, never overwrites.

Expose typed PyO3 calls and update Python conversions without reimplementing locking or
conflict logic. Test old schema fixtures, commandless legacy rows, malformed new
submissions, immutable fields, simultaneous reserve calls from independent processes,
fingerprint replay, overlapping key/name conflicts, stop races, and exactly-once finish.
Run the Rust repository's complete `just check` before phase completion.

### 2. Hardened detached proc kernel

Promote the proven monitor bootstrap mechanics into the proc kernel. The starter
persists the reservation, launches a short bootstrap, receives bounded acknowledgement
for the double-forked supervisor and its boot-aware identity, transfers any required
claim, then releases the launch barrier. Scrub initiating-agent and runner-only
environment variables before the supervisor starts. On acknowledgement or transfer
failure, keep the command behind the barrier, stop the supervisor, and settle the proc
as an observable launch error.

The supervisor executes the persisted argv directly without reconstructing a shell
string. It owns the child process group, combined binary-safe output, total/idle timeout
tracking, TERM-to-KILL escalation, stop intent, and recovery checkpoints. Add an
explicit `settling` state: after the command is gone, settlement durably completes log
flush, claim release, result envelope, artifacts/family updates, and optional follow-up
before the Rust finish operation publishes terminal state. Each step is idempotent so a
restarted reconciler can resume it without duplicate release or follow-up.

Cover starter death, bootstrap death, PID reuse, previous-boot identity, quiet and
invalid-UTF8 commands, partial output, background grandchildren, TERM-resistant groups,
timeout/stop races, and injected crashes between every settlement checkpoint. Preserve
control and reconciliation of existing rows driven by either legacy supervisor until
they naturally terminate.

### 3. Named proc-shell service and CLI

Create one typed submission request used by CLI, monitor, bead/epic, and later ACE
callers. It carries explicit argv, cwd/project/session attribution, optional fully
qualified shell identity, concurrency keys, artifacts/result paths, timeout policy, and
settlement policy. Keep execution state in the proc row and presentation/lineage policy
in the artifacts member with immutable cross-links.

Add `-N/--shell NAME` to `sase proc run` and `sase proc list`. A bare name qualifies
beneath the calling sase agent; a value containing `--` is already fully qualified.
Reject `/`, proc-id-shaped ambiguity, empty/invalid agent components, and names that
cannot map to the canonical namespaced key. Resolve show/kill targets by exact shell
name before exact proc id and unique id prefix. Active names are unique per project;
settled names may be reused while history stays visible.

Keep required command data positional and all policy options optional in accordance with
CLI rules. Update alphabetical help, examples, errors, completions, friendly rendering,
JSON output, and filter behavior together. Exercise qualification with and without agent
context, exact resolution precedence, concurrent collisions, settled reuse,
session/global visibility, and artifacts-owned log retention through public CLI tests.

### 4. Monitor facade and family convergence

Compile a monitor command string once to `[/bin/sh, -c, command]` and submit the common
typed request directly; do not subprocess through `sase proc` and do not add a wrapper
process. Allocate the family member and attach `shell_kind="proc"` as lineage policy,
then cross-link it to the same proc id. Preserve monitor-facing status, timeout, claim,
output, follow/follow-up, and `--next` behavior as projections over the proc lifecycle.

Move monitor list/show/kill/follow and bead/epic launch helpers onto the common service
incrementally. Maintain a compatibility reader/controller for historical monitor
artifacts and their running supervisors, but stop emitting duplicate monitor execution
state once all new starts use proc reservations. Ensure family/member rollback and proc
settlement are idempotent at every boundary.

Test identical monitor replay, differing fingerprint conflict, family attachment, claim
transfer/release, follow-up suppression and exactly-once continuation, starter and ACE
death independence, reboot/lost-supervisor reconciliation, legacy live and terminal
monitor records, and consistent visibility through both `sase proc` and `sase monitor`.

## Verification and landing

Each phase runs focused Rust and Python tests while developing. Any phase that changes
the Python repository first runs `just install`, then `just check`; use the monitored
workflow if the check becomes long. Before landing the combined child epic, run the Rust
core's complete `just check`, run `just install` in the Python repository, and run
`just check-full` through the SASE monitor with an explicit next action. No ACE
rendering is planned; if implementation changes ACE visuals, also run `just test-visual`
and inspect intentional snapshot differences.

Acceptance requires one proc id and terminal boundary across proc and monitor views;
atomic execution-once behavior for matching names/keys; no orphan or false terminal
state across starter death, reboot, PID reuse, stop, timeout, or crash recovery; and
continued readability and control of all historical proc and monitor records.
