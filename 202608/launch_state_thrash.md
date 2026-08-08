---
tier: tale
title: Stop launch-time STARTING/RUNNING refresh thrash
goal:
  Agent launch state advances monotonically while marker bursts stay on the bounded
  exact-delta refresh path.
proposed_by: bbugyi200.athena.vt
create_time: 2026-08-08 12:38:56
status: wip
---

# Stop launch-time STARTING/RUNNING refresh thrash

## Goal

Make a newly launched agent progress monotonically from `STARTING` to its durable
runtime state in the ACE Agents view, without turning bursts of loader-visible marker
writes into repeated whole-list reloads. Preserve the existing fast-path behavior:
launch results and artifact watcher/poll events should still make a real transition
visible promptly, and all filesystem work must remain outside Textual's serial message
pump.

This is a `tale`: the runner-side publication race and the TUI-side scheduling race are
two halves of one launch-state consistency bug and can be implemented and verified as
one coherent change. There is no independent phase boundary that would justify an epic.

## Evidence and root cause

The incident artifacts show a launch taking roughly two minutes to become admitted,
while the Agents header reports a provisional `STARTING` row. The corresponding runtime
artifacts establish that this was not a process restart loop:

- the affected run had one launch-spawn event, one PID, and one artifact directory;
- `run_started_at` was eventually recorded and the workflow state became `running`;
- there is only one "Starting agent run" entry and no retry/restart marker.

The TUI load telemetry during that interval does show a refresh loop. In particular,
`starting_poll` requests that should be exact artifact-directory deltas appear as full
Tier-1 loads. Several took roughly 2-6 seconds during the reported launch, with later
examples under host contention taking roughly 9-10 seconds. Other broad refreshes in the
same window were slower still. This explains the visible churn and why it is most
obvious during a slow initial admission.

The code path has two compounding races:

1. A live `RUNNING` claim is intentionally projected as `STARTING` until enrichment
   reads `run_started_at` from `agent_meta.json`. The runner rewrites that file several
   times during directive resolution, workspace setup, admission, and execution-loop
   startup. The primary `write_agent_meta` implementations currently truncate and
   rewrite the destination in place. A watcher-triggered or polled load can therefore
   observe an absent/partial JSON snapshot, miss `run_started_at`, and temporarily
   project the same live run as `STARTING` again.
2. `_schedule_agent_artifact_delta_refresh` has no scheduled/in-flight exact-delta
   queue. If either a broad refresh or another delta already owns `_agents_loading`, it
   discards the known artifact directories, labels ordinary contention as
   `delta_read_failure`, and schedules a broad Tier-1 refresh. There is also a dispatch
   window before an exact-delta task sets `_agents_loading` in which multiple tasks can
   be spawned. The countdown poll treats every `(mtime_ns, size)` change to
   `agent_meta.json` or `waiting.json` as useful, so normal bootstrap writes amplify
   both races.

The root fix is to publish complete metadata snapshots atomically and to retain exact
delta intent across refresh contention. A status-only rendering override is not an
acceptable substitute: it would conceal inconsistent reads while leaving the costly
broad-load loop intact.

## Implementation

### 1. Publish agent metadata atomically

- Introduce or consolidate a small shared axe helper for `agent_meta.json` persistence:
  serialize to a uniquely named temporary sibling in the artifact directory, finish and
  close the file, then use `os.replace` to publish it. Clean up a leftover temporary
  file on failure and propagate the same errors the current required writers propagate.
  The artifact-index mutation notification must happen only after the replacement is
  complete, so index readers and filesystem watchers can only observe the old complete
  document or the new complete document.
- Route the generic run-agent metadata writes in `src/sase/axe/run_agent_markers.py` and
  the duplicate setup writer in `src/sase/axe/run_agent_runner_setup.py` through that
  implementation. Route the specialized axe-runner writer in
  `src/sase/axe/runner_artifacts.py` through the same atomic primitive while retaining
  its current best-effort error handling and metadata shape.
- Audit direct `agent_meta.json` mutation sites used by launch/admission/family
  promotion. Convert any remaining truncate-in-place writer on these paths or document
  why it is already atomic; do not broaden the change to unrelated JSON markers.
- Preserve canonical tribe normalization, newline/format compatibility, and the current
  artifact-index update semantics. This is process-side persistence plumbing, not shared
  domain behavior, so it remains in Python rather than moving into the Rust core.

### 2. Give exact artifact deltas first-class coalescing state

- Extend the Agents loading state declarations and startup initialization with explicit
  exact-delta scheduled/pending state. The pending request must retain a bounded,
  insertion-stable, deduplicated set of artifact directories, the subset known to be
  deleted, completion callbacks, and a normalized source. Reuse the watcher queue's
  existing artifact-delta bound where practical so a pathological burst has one
  consistent limit.
- Refactor `src/sase/ace/tui/actions/agents/_loading_refresh.py` around a single
  lossless drain rule:
  - when idle, mark an exact delta scheduled before spawning its pump-free task, closing
    the current multiple-spawn window;
  - when a broad or exact load is scheduled/in flight, merge new exact directories into
    the pending exact request instead of converting them to a broad load;
  - after the active refresh completes, drain any already-required broad/full-history
    work and then the retained exact delta, so an exact marker event that arrived during
    an older broad snapshot is applied afterward;
  - when a broad request arrives during an exact load, preserve its existing
    full-history precedence and callback behavior, then continue draining any newer
    exact events;
  - coalesce repeated events for the same launch directory into one trailing exact read,
    without losing deletion knowledge or firing callbacks more than once.
- Keep broad fallback only for conditions that truly require it: missing/unmappable
  artifact directories, active search, bounded-queue overflow, or an actual delta-load
  failure. If the exact load itself reports an incomplete scan or raises, schedule one
  broad recovery and retain the callback for that recovery.
- Preserve navigation-gate deferral and pump-free task ownership. Scheduling and queue
  mutation remain event-loop-local and cheap; no disk I/O or await is added to the
  Textual message pump.

### 3. Make refresh telemetry describe the real work

- Remove the inference that every broad `starting_poll` refresh represents
  `delta_read_failure`; contention will no longer be a fallback.
- Record exact-delta scheduling/coalescing/draining with `artifact_delta_load` cost and
  directory counts. Record a broad fallback reason only at a real fallback boundary.
  Keep existing trace taxonomy stable unless a precise new reason is needed for an
  explicit scheduler overflow.
- Ensure `tui_agent_loads.jsonl` can distinguish a genuine delta failure from a queued
  delta that waited behind other work. This makes the acceptance check observable and
  prevents this bug from being misdiagnosed as runner restarts in the future.

## Tests

Add focused deterministic coverage around the two race boundaries:

- Atomic metadata publication: pause a writer before `os.replace`, verify a concurrent
  reader still sees the complete old metadata (including `run_started_at`), release it,
  and verify the complete new metadata and post-publication index update. Exercise the
  generic and specialized writer entry points or their shared primitive, including temp
  cleanup/error behavior.
- Exact scheduling before dispatch: issue multiple watcher/poll delta requests before
  the first task begins and assert that only one task is spawned with deduplicated
  directories.
- Broad-load contention: hold a Tier-1 load in flight, issue repeated
  `starting_poll`/watcher changes for the same directory plus another directory, then
  release the load. Assert there is one trailing exact batch, no contention-induced
  broad fallback, correct deleted-directory propagation, and FIFO one-shot callbacks.
- Delta-load contention: hold an exact load in flight and issue a burst of further
  changes. Assert one bounded trailing exact batch and no broad load. Cover an explicit
  full-history request racing the delta so it is neither downgraded nor allowed to drop
  the later exact work.
- Failure and overflow: verify an actual delta read failure and an over-limit exact
  queue each produce exactly one broad recovery with the correct trace reason.
- Launch-state regression: construct the live-claim sequence used by the loader, publish
  `run_started_at` while refresh work is contended, and assert the applied statuses do
  not regress from `RUNNING` to `STARTING` and that `starting_poll` never records a full
  load merely because another refresh was active.
- Retain and update the existing starting-poll, agent artifact-delta, refresh
  coalescing, callback, event-dirty-flag, and trace tests so the original promptness,
  navigation, and cleanup contracts remain covered.

## Validation

1. Run `just install` because this is an ephemeral workspace and dependencies may be
   stale.
2. Run focused tests for the metadata writer plus
   `tests/ace/tui/test_starting_agent_poll.py`,
   `tests/ace/tui/actions/test_agent_artifact_delta_loader.py`,
   `tests/ace/tui/test_agents_refresh_coalescing.py`,
   `tests/ace/tui/test_loading_callbacks.py`,
   `tests/ace/tui/test_event_handlers_artifact_dirty_flags.py`, and
   `tests/ace/tui/test_agents_refresh_trace.py`, including the new deterministic race
   cases.
3. Run `just check` as the repository-required whole-repo lint plus diff-scoped test
   gate. If test selection escalates, reports unusual coverage, or the implementation
   touches the broadening set, run `just check-full`.
4. Re-run the deterministic contended-launch regression after the full gate and inspect
   its captured refresh records: the expected sequence is an older broad load followed
   by at most one coalesced exact delta; there must be no `starting_poll` full-load
   entry, no `delta_read_failure` for mere contention, and no `RUNNING` to `STARTING`
   regression. No PNG snapshot run is required because this changes scheduling and
   persistence, not rendering.

## Acceptance criteria

- A launch creates one runner process and the ACE row transitions monotonically from
  `STARTING` to `RUNNING`, `WAITING`, `QUESTION`, or a terminal state as durable markers
  arrive; an unreadable in-place `agent_meta.json` window no longer exists.
- Repeated metadata/waiting marker writes during a slow launch are deduplicated into at
  most one trailing exact-delta batch per active refresh, rather than a chain of Tier-1
  broad loads.
- Exact artifact directories, deletion state, full-history requests, and completion
  callbacks survive every scheduling race without being dropped or duplicated.
- Broad recovery remains available and accurately traced for real loader failures,
  active-search limitations, missing paths, and bounded-queue overflow.
- Existing launch latency, watcher fallback, navigation responsiveness, and full-history
  reconciliation tests continue to pass, followed by `just check` (and `just check-full`
  when required by the repository gate).

## Non-goals

- Do not shorten provider startup, workspace acquisition, dependency installation, or
  intentional `%wait` admission. A long but stable `STARTING`/`WAITING` interval is
  valid; the bug is oscillation and redundant reload work.
- Do not render provisional `STARTING` rows differently or add a sticky presentation
  override. Correct the publication and scheduling races at their sources.
- Do not change global refresh intervals, remove the watcher/poll safety net, or move
  frontend scheduling policy into the Rust core.
