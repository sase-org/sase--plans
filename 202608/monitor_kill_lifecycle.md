---
tier: tale
title: Stop active monitors with their owning agents
goal:
  Make every explicit agent-kill and cleanup path stop owned monitor proc shells through
  the canonical monitor lifecycle, while preserving intentional monitor handoff and
  repairing the currently orphaned release waiter.
size: medium
proposed_by: bbugyi200.athena.0ak
create_time: 2026-08-22 12:03:30
status: wip
---

# Plan: Stop active monitors with their owning agents

## Outcome

An explicit user kill of a SASE agent will stop every active monitor proc shell owned by
that agent before the agent is dismissed. This will hold for focused family kills,
marked/group/panel/clan/all cleanup, `sase agent kill`, and callers such as the mobile
bridge. Monitor shutdown will use the canonical monitor/proc stop path so it records
stop intent, terminates the command tree, settles the monitor as `stopped`, suppresses
its follow-up, and releases its workspace claim exactly once.

The normal `sase monitor start` handoff remains deliberately different: starting a
monitor inside an agent still writes the pending handoff marker, ends that starter
shell, and leaves the monitor running. Only a later explicit user kill/cleanup of the
owning SASE agent stops the monitor.

## Diagnosis and reproduced failure

- The screenshot's orange gear count reflected real runtime state, not a stale TUI
  counter. `sase monitor list -j` reported two live supervisors: a current
  `just check-full` monitor and monitor `0fmbm91hgytw`, the long-running v0.17 release
  waiter owned by `sase-ru.6`.
- The release waiter was still producing a line every 30 minutes and its detached
  supervisor and command process were alive under PID 1, so dead-supervisor
  reconciliation correctly left it running.
- Durable cleanup proc `dky25sfk32rn`, created at 2026-08-22T11:18:54Z, proves the
  `sase-ru` bulk cleanup selected `sase-ru.6--mon-1` as one of five kill targets. The
  cleanup plan classified that row as ordinary `kind: running`, dismissed the owning
  family, and released workspace 15.
- The TUI then passed the monitor supervisor PID (1665545 in this incident) to the
  generic agent helper. `request_user_kill()` assumes an agent runner PID is also its
  process-group ID and called `killpg(1665545)`. A proc-backed monitor has distinct
  supervisor and command groups, so that group did not exist. `ProcessLookupError` was
  translated to the successful `already_stopped` result even though the supervisor and
  command remained alive. The cleanup therefore reported “Killed 5 agents” and hid the
  monitor's family while leaving the monitor running without its workspace claim.
- The earlier `clan_dismiss_cascade` plan fixed selection and dismissal of terminal
  monitor rows. Its regression fixture used `DONE`, pidless monitors and explicitly left
  a clan containing only a live monitor visible. It did not define how a live monitor
  must be stopped, so the new requirement was not implemented by that work.

## 1. Repair the live incident without disturbing valid monitors

- Re-read `sase monitor list -j` immediately before remediation because the live state
  may have changed while this plan awaited approval.
- If `0fmbm91hgytw` is still `running`, stop that exact monitor with
  `sase monitor stop`; do not stop the unrelated active verification monitor or any
  replacement monitor launched after this diagnosis.
- Verify the monitor and its proc row settle as stopped/killed, its supervisor and
  command group are gone, no `--next` follow-up launched, and the live monitor count
  drops by exactly one. If it already reached a terminal state, record that fact and do
  not mutate it.

## 2. Give cleanup plans first-class monitor-stop semantics in `sase-core`

Open the linked core with `/sase_repo` before editing it. Extend the versioned
agent-cleanup wire and pure Rust planner rather than teaching frontends to reinterpret
an ordinary process kill:

- Bump `AGENT_CLEANUP_WIRE_SCHEMA_VERSION` so an older published binding cannot silently
  return the unsafe `running` classification.
- Add the minimum monitor identity/lifecycle fields needed on `AgentCleanupTargetWire`
  (including the canonical monitor ID and whether it is a live monitor), a `monitor`
  kill kind, and a typed monitor-stop side-effect intent. Carry the monitor ID in the
  returned plan; never use the supervisor PID as a process-group ID.
- Classify a live monitor before the generic `run` case. A directly selected live
  monitor becomes a monitor stop, while terminal monitor rows remain ordinary
  dismissals.
- Build the ownership graph from the existing `parent_timestamp` chain. When a selected
  agent owns active monitor descendants, add those monitors to the kill plan even if the
  UI selection named only the owning family/root. De-duplicate monitors already selected
  by a clan, group, or all-agents scope, and never include an unrelated sibling lane.
- Emit monitor-stop intents before other cleanup intents and omit the generic
  workspace-release intent for monitor kills; canonical monitor settlement owns claim
  release and follow-up suppression.
- Mirror the Rust cases in the Python reference planner so missing/stale-extension
  fallback has identical behavior. Update the PyO3 round-trip fixture and facade
  parity/schema guards for the new wire version and side-effect field.

Regression coverage in the core and Python planners must include direct live-monitor
selection, nested owner-to-monitor cascade, deduplication under clan/custom scopes, an
unrelated monitor that remains untouched, terminal monitor dismissal, and the absence of
a generic workspace-release request for a monitor stop.

## 3. Execute monitor stops through the durable cleanup transaction

Keep slow and stateful shutdown work off the Textual event loop and preserve the
existing optimistic/error-recovery contract:

- Teach the cleanup payload serializers and `KillKind` types about monitor IDs and the
  `monitor` kind.
- Refactor the bulk kill flow to obtain the cleanup plan before signalling targets.
  Signal ordinary agent process groups as today, but never pass a monitor supervisor PID
  to `request_user_kill`. Include planner-cascaded monitor targets in the durable
  cleanup request even when only their owning agent was selected. Apply the same
  per-kind dispatch to the focused-agent path.
- In `sase agent persist-cleanup`, execute every monitor-stop intent first by resolving
  the exact current monitor record and calling the canonical `stop_monitor`/proc-shell
  service. Treat an already-terminal monitor as idempotent success. If a requested
  monitor cannot settle out of `running`, fail the cleanup transaction before writing
  dismissed identities, releasing workspaces, deleting artifacts, or dismissing
  notifications; the existing tracked-proc error refresh must restore optimistic UI
  rows.
- After all monitor stops succeed, apply the remaining cleanup side effects and
  dismissal snapshot once. Do not run the full side-effect plan a second time while
  iterating monitor kill items.
- Preserve the selected-monitor `x` flow, which already submits `sase monitor stop` as a
  durable proc. It should continue to show a settled monitor until the user chooses to
  dismiss it; bulk agent cleanup may stop and dismiss it in one confirmed transaction.
- Make `kill_named_agent` recognize a resolved live monitor/family endpoint and use the
  same canonical stop operation before its existing dismissal bookkeeping. A stop
  failure must return failure and leave the agent/monitor visible instead of reporting a
  successful kill. Mobile/editor callers that delegate to this function inherit the fix
  without runtime-specific branches.

## 4. Prove lifecycle, UI, and headless behavior

Add focused tests at each boundary:

- Planner/binding parity tests assert the monitor kind, monitor-stop intent, ownership
  cascade, deduplication, terminal dismissal, schema rejection, and no duplicate claim
  release.
- Persistence tests assert stop-before-cleanup ordering, idempotent terminal handling,
  and fail-closed behavior when a monitor stays running. Assert a stopped monitor never
  launches its recorded follow-up.
- TUI tests extend the existing clan-dismiss fixture with a live proc-backed monitor.
  Cover focused family kill and bulk clan/marked/group cleanup, assert the generic agent
  signal helper is never called for the monitor, and assert the durable payload contains
  the exact monitor stop. Retain the direct monitor-row `x` regression.
- Headless tests cover both the monitor member name and its family-facing name through
  `sase agent kill`, including the mobile delegation surface.
- Add an integration test with a disposable SASE home: start a long-running monitor with
  a harmless child command and a sentinel `--next`, clean up its owner, and prove the
  supervisor and child terminate, the monitor/proc settle stopped/killed, the follow-up
  is absent, and the workspace claim is released. Also retain the inverse handoff test
  proving that monitor start alone ends the starter but intentionally leaves the command
  alive.

## 5. Ship and verify the binding-backed behavior

- Run `just check` in `sase-core`; do not rely on `cargo test -p sase_core` alone
  because the PyO3 schema tests are required.
- In `sase`, run `just install` before verification so the editable environment builds
  the linked core, then run the focused monitor/cleanup/CLI tests and `just check`.
- Because this changes the Rust cleanup wire and behavior, land/publish the core change
  before finalizing the Python package dependency. Raise the `sase-core-rs` floor in
  `pyproject.toml`, refresh `uv.lock`, reinstall from the published floor, and rerun the
  binding parity tests so production installs cannot use the old classifier.
- Treat this as a broadening change and run `just check-full` only through
  `/sase_monitor`, with a `--next` action that inspects failures and finishes the SASE
  final declaration.
- Recheck `sase monitor list -j` and the TUI indicator after verification. The count
  must equal the genuinely live monitor records, with the repaired v0.17 waiter absent
  unless a new agent intentionally started a replacement.

## Non-goals and invariants

- Do not change the detached-supervisor architecture or make monitors die merely because
  their starter shell completes the normal monitor handoff.
- Do not repair this by resolving `getpgid(supervisor_pid)` inside the generic agent
  killer. That bypasses proc stop intent, PID-identity checks, command-tree teardown,
  terminal monitor markers, follow-up suppression, and canonical claim release.
- Do not stop every monitor globally when one agent is killed; only explicitly selected
  monitors and active monitor descendants owned by the selected SASE agent are in scope.
- Do not add a TUI-only workaround. The Rust planner, Python fallback, durable executor,
  CLI, and connected callers must agree on the lifecycle contract.
