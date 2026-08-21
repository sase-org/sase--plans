---
tier: tale
title: Restore live output for session-local ACE procs
goal:
  Session-local ACE proc rows stream bounded progress and command output in the Procs
  pane while preserving the read-only durable observer boundary.
size: medium
proposed_by: bbugyi200.athena.09l
create_time: 2026-08-21 11:23:11
status: wip
---

# Plan: Restore live output for session-local ACE procs

## Outcome

Make every ACE proc row that represents a running session-local worker capable of
publishing bounded, thread-safe progress and command output to the Procs pane while the
work is still running. In particular, a comprehensive update must show its current leg
and child-command output instead of remaining on the empty-body `Working...` fallback.
Keep durable supervisor procs on the existing read-only `ProcObserver` log-tail path;
this repair must not reintroduce callable durable procs, ACE proc-store writes, or
process-global stdout/stderr redirection.

## Diagnosis

- The selected `comprehensive update` in the reported screenshot is an ephemeral row
  created by `_submit_session_worker`, not a supervisor-backed durable proc.
- Durable selected rows already have a working off-thread path: `ProcObserver` reads the
  selected `Proc.log_path`, includes `row.output` in its snapshot signature, and the
  pane repaints changed output.
- Session-local rows carry an `ObservedProcLog`, and the pane already watches its
  version every 250 ms, but production never appends to that log. The submission wrapper
  invokes a zero-argument body and hard-codes `SessionWorkerResult.output` to `""`.
- The August read-only-observer cutover removed the former update reporter and streaming
  runner adapters. Current comprehensive, SASE/dev, and agent-CLI update bodies
  therefore use captured subprocess output and publish neither phases nor lines before
  completion.
- Existing live-refresh coverage mutates a compatibility queue buffer directly; it does
  not submit a real session worker and therefore cannot catch this regression.

## Implementation

1. Add a session-local reporting handle around the existing `ObservedProc` presentation
   state. It should expose explicit phase, section, log, command, and result reporting,
   append through the bounded/thread-safe `ObservedProcLog`, and provide the narrow
   subprocess/run-function adapters needed by the existing agent-CLI, uv, and dev-update
   execution seams. Stream combined child output line by line from the worker thread and
   retain the completed output for typed callbacks. Keep this API presentation-only: it
   must not import proc-store mutation APIs, manufacture durable rows, or revive the
   retired `ProcReporter`, `ProcQueue`, or callable durable submission interfaces.

2. Change `_submit_session_worker` to construct and pass that reporting handle to its
   body, append an explicit success/error terminal record, and return the row's retained
   output instead of an empty string. Update all session-worker producers and their test
   doubles to the explicit reporter-aware callable contract; short UI-only operations
   may intentionally ignore the handle, while multi-step work should publish meaningful
   phases/results. Do not capture `sys.stdout`/`sys.stderr` globally, because concurrent
   Textual worker threads would steal one another's output.

3. Restore structured live reporting for the long update paths that expose the defect:
   SASE managed and editable updates, agent-CLI updates, plugin update batches/mode
   changes, agent-cache integration, and the scoped comprehensive update. Thread the
   reporter's streaming run adapters through the existing injectable `run_fn`/`run`
   seams; emit a phase before each selected comprehensive-update leg and concise result
   sections after each leg. Preserve the typed `TrackedProcResult` payloads, ordering,
   failure aggregation, concurrency scopes, completion callbacks, and restart behavior.

4. Leave Procs-pane and durable-observer polling architecture intact. Verify that the
   existing in-memory log-version fast path repaints a selected session row without disk
   I/O or a full list rebuild, follows new output only when the user has not scrolled
   away, and continues to render durable selected-log tails exactly as before. Adjust
   rendering only if needed to avoid duplicate command/header lines from the new
   reporter contract.

## Verification

- Add unit tests for session reporting: bounded line retention, stream styles, phase and
  command metadata, incremental subprocess lines, terminal success/error records, and
  `TrackedProcCompletion.output` containing the retained presentation log.
- Add a Procs-pane integration test that submits a real session worker, blocks it
  between two reported lines, and proves the selected output changes before completion
  without manually mutating a compatibility buffer. Cover selection changes, body-cache
  invalidation, and user-scroll follow behavior.
- Restore focused update tests with a fake reporter/runner to prove comprehensive legs
  report in order, child commands use the streaming adapter, failures remain aggregated,
  and the reported screenshot's update path produces live text.
- Keep observer tests proving durable log growth is read off-thread and unchanged polls
  coalesce; keep the producer-inventory static checks rejecting legacy callable durable
  APIs and ACE proc-store writes.
- Run `just install`, focused session-worker/update/Procs-pane/observer suites, and
  `just check`. Because the visible Procs output changes, run `just test-visual`,
  inspect the Procs-tab actual/expected/diff artifacts, and accept snapshot changes only
  when they represent the intended live-output state.
