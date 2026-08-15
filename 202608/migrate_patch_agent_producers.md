---
tier: tale
size: medium
title: Migrate ACE Patch and agent producers to supervisor-owned durable operations
goal:
  Every cataloged durable ACE Patch and agent action submits explicit argv through the
  shared proc supervisor, uses stable namespaced concurrency keys and typed result
  envelopes, and preserves correct optimistic UI completion and rollback behavior.
proposed_by: bbugyi200.athena.sase-m9.3.1.2
bead: sase-m9.3.1.2
create_time: 2026-08-15 16:48:08
status: wip
---

- **PROMPT:**
  [prompts/202608/migrate_patch_agent_producers.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/migrate_patch_agent_producers.md)
- **PARENT:** [202608/ace_proc_ownership.md](ace_proc_ownership.md)
- **BEAD:**
  [sase-m9.3.1.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.3.1.2.md)

# Plan: Migrate ACE Patch and agent proc producers

## Context

Phase `sase-m9.3.1.2` follows the durable-operation contract work already present in the
tree. That dependency added:

- `src/sase/ace/tui/durable_submit.py` and `ProcActionsMixin._submit_durable_proc()` as
  the argv-only, off-event-loop ACE submission boundary;
- versioned mode-0600 request/result envelopes under `sase.ops`;
- focused `sase patch`, `sase agent`, and `sase run` entry points; and
- a machine-checked inventory in `src/sase/ace/tui/proc_producer_sites.py`.

Patch and agent action modules still call `_submit_proc()` or `_submit_tracked_proc()`
with Python callables. Those workers are owned by ACE, their deduplication is only
process-local, and their callback results cannot be reconstructed after ACE exits. This
tale migrates only the Patch and agent families assigned to this phase. Bead,
notification, plugin, monitor, AXE background-command, and other utility producers
remain for sibling phase `sase-m9.3.1.3`; replacement of `ProcQueue` and `ProcMirror`
remains for `sase-m9.3.1.4`.

## Required invariants

- Durable work crosses the ACE boundary only as non-empty argv plus JSON-shaped request
  data. Large or mutable inputs live in the private request sidecar, not argv,
  fingerprints, labels, or logs.
- Completion behavior reads the typed result envelope. Combined output is presentation
  only and is never parsed as a result protocol.
- Shared-store concurrency keys are stable, namespaced, and sufficient to make
  collisions authoritative across simultaneous ACE instances. Patch mutations share a
  key derived from project identity plus Patch name; agent metadata mutations key the
  owning artifacts/identity; launch, cleanup, revert, workspace, and AXE-slot claims
  retain their existing exclusion relationships.
- Submission and envelope waits stay in thread-backed workers. Completion callbacks
  revalidate the selected Patch/agent before touching UI state, retain existing
  optimistic changes and rollback behavior, and tolerate stale or absent callbacks
  because durable state remains authoritative.
- Workspace claims, runner/AXE slots, and cleanup ownership settle exactly once through
  the proc request/settlement path, including submission failure and replay.
- Revert preview calculations that the inventory classifies as session-local UI work
  become ordinary thread-backed Textual workers with no durable proc row; revert
  execution remains durable.

## Implementation

### 1. Complete the reusable durable-producer boundary

Refine `src/sase/ace/tui/actions/proc_actions.py`, `src/sase/ace/tui/durable_submit.py`,
and small adjacent helpers so migrated callers can construct canonical `sase` argv,
deterministic request fingerprints, and namespaced concurrency keys without duplicating
encoding rules. Preserve the existing local `ProcQueue` presentation handle for this
phase, but make the supervisor's proc id and typed result authoritative. Where current
launch or cleanup commands do not yet emit the operation-specific result required by the
dependency contract, complete the focused existing domain command/handler integration
rather than adding an ACE-specific dispatcher.

The boundary must distinguish an atomic shared-store collision from an ordinary command
failure, surface the current duplicate warning, and avoid applying failure rollback to a
newer optimistic action. Request fingerprints must be stable for idempotent replay but
exclude volatile result paths and sensitive payload values.

### 2. Migrate every Patch producer in the inventory

Update the Patch actions in `status.py`, `sync.py`, `base.py`, `proposal_rebase.py`, and
`hints/_rewind.py` to submit their matching `sase patch` argv and operation request
payload through `_submit_durable_proc()`. Include status, submit, archive, restore,
revert, sync, reword, tag, mail, accept, rebase, and rewind. Use project-qualified Patch
concurrency keys so all mutations of one Patch exclude one another across ACE processes
without colliding with the same display name in another project.

Move workspace identifiers and workflow-only values such as descriptions, accepted
entries, rewind options, and resolved workspaces into request payloads. Preserve start
notifications, hook resets, optional mail-after-accept behavior, Patch refreshes, and
selection checks by mapping `TrackedProcCompletion.payload` from the typed envelope back
to the current callback logic. Do not release a workspace claim locally when the durable
proc settlement policy owns that release.

### 3. Migrate every agent producer in the inventory

Update launch, cleanup/dismiss/kill, approve, wait/unwait, rename, tribe assignment, and
revert execution modules to use focused `sase run` / `sase agent` argv and private
request payloads. Use stable keys for launch identity, target agent/family, artifacts
metadata, tribe-store mutation, workspace claims, and any AXE runner slot involved.
Preserve optimistic rename/approve/wait/tribe changes and their guarded rollback, agent
list refreshes, launch-result deltas, and cleanup notifications.

Convert single and bulk revert previews from `_submit_tracked_proc()` to normal
thread-backed Textual workers because they are session-local computations; keep the
execute paths supervisor-owned and carry previewed SHAs/workspace identifiers through
the durable request. Completion must suppress stale previews and stale mutation
callbacks when selection or optimistic state changed while work was running.

Update `proc_producer_sites.py` as each site changes so the AST inventory remains an
accurate migration ledger: removed callable submissions must disappear or be
reclassified to their durable/UI-worker form, while unrelated sibling-phase producers
remain cataloged.

### 4. Prove durability, concurrency, and UI compatibility

Add focused tests beside the existing durable adapter, proc inventory, Patch action, and
agent action tests. Cover:

- exact argv, private request payload, stable fingerprint, and namespaced keys for each
  operation family;
- two independent submitters colliding on the same Patch/agent key while different
  projects or identities remain independent;
- typed success and failure payloads, malformed/missing results, idempotent replay, and
  restart-style completion without relying on captured stdout or a live callback;
- optimistic success, guarded rollback, current-selection revalidation, and stale
  callback suppression;
- workspace claim and AXE-slot release exactly once on success, failure, collision, and
  replay; and
- no remaining callable proc submission in the Patch/agent inventory, while UI-only
  revert previews produce no proc row and run off the event loop/message pump.

Keep tests at the narrowest stable seams: operation command tests for domain behavior,
adapter/process tests for shared-store exclusion and envelopes, and action tests for UI
effects. No rendered layout is intended to change, so the PNG visual suite is not
required unless implementation unexpectedly changes visible widgets or text.

## Verification

1. Run focused operation, durable-submission, producer-inventory, Patch-action, and
   agent-action tests while iterating.
2. Run `just install` before repository verification, as required for an ephemeral SASE
   workspace.
3. Run `just check`. If its selector escalates or reports unusual selection, use the
   project-prescribed `just check-full` monitor workflow rather than running the full
   suite inline.
4. Re-run the AST inventory/conformance tests and grep the Patch/agent action modules to
   confirm no scoped producer still passes a callable to `_submit_proc()` or
   `_submit_tracked_proc()`.
5. Record why `just test-visual` was skipped if no rendered output changed; otherwise
   run it and inspect any generated PNG diff artifacts.

## Non-goals and phase handoff

- Do not remove the callable submission APIs globally; sibling phase `sase-m9.3.1.3`
  still owns unrelated producers until it migrates them.
- Do not replace ACE proc observation, `ProcQueue`, or `ProcMirror`; that belongs to
  `sase-m9.3.1.4` after both producer-migration phases finish.
- Do not change detached-option compatibility or legacy proc-history semantics; that
  belongs to `sase-m9.3.1.5`.
- Do not close the parent epic or any ancestor bead. Close only `sase-m9.3.1.2` after
  the implementation and verification above pass.
