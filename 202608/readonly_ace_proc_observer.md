---
tier: tale
title: Read-only ACE proc observation
goal:
  Make ACE a read-only observer of supervisor-owned durable procs while preserving
  responsive UI state, restart recovery, and live completion behavior.
size: medium
proposed_by: bbugyi200.athena.sase-m9.3.1.4
bead: sase-m9.3.1.4
create_time: 2026-08-15 19:07:41
status: done
---

- **PROMPT:**
  [prompts/202608/readonly_ace_proc_observer.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/readonly_ace_proc_observer.md)
- **PARENT:** [202608/ace_proc_ownership.md](ace_proc_ownership.md)
- **BEAD:**
  [sase-m9.3.1.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.3.1.4.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-m9.3.1.4](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-m9.3.1.4.md)
- **COMMITS:**
  - [8c48404](https://github.com/sase-org/sase/commit/8c48404581cabc8b49f1534ef4e64f542363141d)
    — feat(ace): observe durable procs read-only

# Plan: Read-only ACE proc observation

## Outcome

Make the durable proc store and detached supervisor the only authorities for
ACE-launched commands. ACE will submit argv, observe immutable proc/result/log snapshots
off the Textual event loop, and apply small coalesced UI updates without writing proc
rows or owning command lifetime. Restarting ACE will reconstruct its proc surfaces from
durable state, while quitting or restarting ACE will leave active commands running.

## Constraints and current seams

- Preserve the existing supervisor/store lifecycle and typed operation envelopes; shared
  durable behavior stays in `sase.procs` and the existing domain commands.
- Store reads, sidecar parsing, session discovery, and log reads must occur on an
  observer thread. Only immutable snapshots and completion records may cross back via
  `call_from_thread`.
- Preserve Procs-pane selection, scope, output following, active counts, explicit `K`
  control, and live-session optimistic completion callbacks. Durable rows and envelopes
  override optimistic state as soon as they are observed.
- The current producer inventory still contains legacy `_submit_tracked_proc` users
  (including bead/issue mutations, notification responses, agents sync, and update
  workflows). Complete their already-designed domain-command cutover, or classify truly
  session-local work as `_submit_session_worker`, before deleting callable proc
  execution. Do not leave an ACE-owned durable body behind the observer facade.
- Historical `tui`, `command`, and `detached` rows remain renderable and controllable
  under existing compatibility behavior, but ACE must not create or terminalize them.

## Implementation

1. Finish the residual producer cutover. Route each inventory entry classified as
   durable through `_submit_durable_proc` with its existing domain argv, typed request,
   fingerprint, and concurrency keys; retain only genuine UI-only thread workers. Remove
   `live_body`, `_submit_proc`, `_submit_tracked_proc`, `ProcReporter` execution, and
   inventory classifications that permit callable durable work. Update focused producer
   tests to assert argv/result-envelope behavior and cross-process collision semantics.
2. Replace `ProcQueue` and the write-owning `ProcMirror` with a read-only
   `ProcObserver`. Its daemon thread will resolve the ACE session/project context, poll
   proc rows with an mtime fast path, reconcile submitted ids plus relevant session,
   origin, project, shell-name, and tag matches, read only the selected log tail, and
   decode terminal typed results (including explicit malformed/missing-result records).
   It will coalesce unchanged polls and deliver immutable snapshots through one
   `call_from_thread` boundary. The UI-side projection will expose read-only lookup,
   active-count, scope-conflict, and ordered-row queries needed by existing surfaces.
3. Split durable submission from durable completion. A short thread worker will validate
   and submit argv, then register the returned durable id and live callback metadata
   with the observer; it will not wait for or execute the command. Observer transitions
   will reconcile optimistic placeholders, invoke live callbacks once from decoded
   envelopes, apply rollback/refresh/toast behavior on the UI thread, and tolerate the
   command settling before registration. Store-level concurrency remains authoritative,
   with a small pending-key guard only for the pre-reservation click window.
4. Move the indicator, Procs pane, runners modal, update-restart deferral, and lifecycle
   helpers onto observer projections. Replace the pane's independent store polling with
   observer detail selection and selective row/body refreshes while preserving identity
   restoration and scope filtering. Stop prompting to kill commands during normal ACE
   quit/restart or AXE stop; only the Procs-pane `K` action may request supervisor stop.
   Teardown must stop the observer thread without waiting for, cancelling, signaling, or
   terminalizing any active command.
5. Remove `proc_mirror.py`, queue mutators, child-process ownership fields, mirror write
   tests, and obsolete quit-kill copy. Add static invariants proving ACE imports no proc
   append/update APIs, submits no callable durable work, and never records the ACE pid
   as a proc owner. Keep the presentation row/log types only where they remain useful,
   under observer-oriented names.

## Verification

- Add focused observer tests for off-main-thread store/log/result I/O, unchanged-poll
  coalescing, selective detail-log growth, session/all-session and unattributed scope,
  external-proc discovery, submitted-id races, restart reconstruction, terminal callback
  exactly-once behavior, and malformed/missing/mismatched envelopes.
- Update Procs-pane, indicator, lifecycle, runner, producer-inventory, and
  durable-submit tests for read-only state and explicit `K` control.
- Add process-level tests that launch a supervisor-owned command, tear down or kill the
  ACE process, and verify the command settles successfully and is reconstructed by a
  fresh observer; separately verify observer teardown never sends a signal or proc-store
  terminal write.
- Run focused ACE proc/producers/lifecycle suites while iterating. Then run
  `just install` and `just check`. Run the TUI performance benches relevant to polling
  and Procs navigation. Run `just test-visual` only if rendered Procs/quit output
  changes; otherwise record that no visual output changed.

## Completion

Re-scan the tree for `ProcQueue`, `ProcMirror`, callable proc submitters, proc-store
write APIs under ACE, and quit-time command cancellation. Record any out-of-scope defect
as a `PROPOSED FOLLOW-UP:` note on `sase-m9.3.1.4`; do not create a bead. Close only
`sase-m9.3.1.4` after the focused tests and required repository checks pass.
