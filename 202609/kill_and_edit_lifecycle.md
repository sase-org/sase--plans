---
tier: tale
title: Fix kill-and-edit targeting and cancelled prompt launch holds
goal: "Make ACE kill the exact launch requested by ,X across completion and refresh
  races, and prevent cancelled kill-and-edit prompts from delaying or replaying through
  a new prompt.

  "
size: medium
proposed_by: bbugyi200.athena.0go
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0go](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0go.md)
- **COMMITS:**
  - [88e6f4e](https://github.com/sase-org/sase/commit/88e6f4ef7ebb555f7ced1e2447b8efa7c1b64304)
    — fix(ace): harden kill-and-edit relaunch lifecycle

# Fix kill-and-edit targeting and cancelled prompt launch holds

## Scope and chosen tier

This is one bounded TUI lifecycle repair that one follow-up coding agent can implement.
Use a medium tale: launch records, asynchronous cleanup, and prompt ownership must be
changed together and exercised through the existing Textual harnesses.

Preserve the current keymap meanings: `,X` targets the most recent accepted launch in
this ACE session; `,x` targets the focused or marked agents. Preserve confirmation,
restartable-prompt preparation, and exact family-member name handling. The reported
problem concerns executing that action and cancelling its replacement prompt, not
changing which action the keymap denotes.

No implementation changes were made during planning. The initial and final working-tree
checks were clean. This scratch plan lives outside the repository.

## Diagnosis and evidence

The user's historical occurrence has no identified agent name or log timestamp, so do
not claim a particular historical PID failure. The current source has reproducible
state-machine defects that explain the reported symptoms:

1. **Cancelled prompts leave a global launch hold.**
   `actions/agent_workflow/_relaunch_barrier.py` checks all app cleanup barriers and
   `has_pending_launch_kill(app)` for every `_submit_resolved_launch` call.
   `PromptBarSubmitMixin.on_prompt_input_bar_cancelled` clears `_prompt_context` and
   unmounts the widget but does not detach this prompt's wait ownership. An unrelated
   prompt opened afterward therefore gets the old waiting toast.
2. **A cancelled submission can replay under a new prompt.**
   `_relaunch_cleanup_launch_waiters` stores bare callbacks. Its drain checks only
   whether `_prompt_context is None`. `_submit_resolved_launch` captures the text but
   reads the current context and bulk selection when replayed. Reopening a prompt makes
   the old callback eligible again, potentially launching the wrong text with the new
   context. The existing cancellation test merely assigns `None` and never reopens the
   prompt, so it misses this case.
3. **Already-returned results escape an in-flight kill.**
   `_begin_inflight_deferred_kill` marks the record `KILL_PENDING` but does not process
   `record.results`. Later callbacks kill only their newly returned results. If p1
   completes, the user presses `,X`, and p2 completes, only p2 is killed and
   `_finish_pending_launch_kill` nevertheless consumes the entire record.
4. **An unloaded newest row is mistaken for a dead launch.** Completion stamps a record
   resolved before the asynchronous artifact-delta refresh populates its rows.
   `_matched_agents_for_record` searches only loaded rows, and
   `_kill_and_edit_last_launch` consumes an unmatched record and immediately targets an
   older one. A partly loaded launch set likewise silently drops missing units.
5. **Completion and successful kill are conflated.** Deferred completion consumes a
   record once its launch procs are terminal, even if kill initiation returned false.
   While a multi-proc record remains pending, duplicate completion callbacks can repeat
   cleanup. `_refresh_launch_record_state` also marks the whole launch `FAILED` on the
   first failed proc, hiding other successful or still-running units from the
   last-launch action.

Read-only harnesses imported this checkout's modules with the installed SASE Python
runtime and stubbed kill/cleanup effects. They produced these observations:

| Event order                                               | Observed result                                             |
| --------------------------------------------------------- | ----------------------------------------------------------- |
| p1 finishes, `,X`, p2 finishes                            | Only `second` killed; record `consumed`                     |
| Hold old submit, cancel context, open new context, submit | New prompt held; drain replays both old and new submissions |
| Newest record resolved but only older row loaded, `,X`    | Older agent targeted; newest record `consumed`              |

These are isolated reproductions, not full test-suite results or real process kills. The
local `uv run --no-sync` environment cannot import its required `sase_core_rs`
extension; the implementing agent must repair that environment with `just install`
before verification. The installed runtime worked for the read-only harnesses.

All source paths below are relative to `src/sase/ace/tui/` unless stated otherwise.

## Required behavior

- Cancelling the replacement editor cancels editing and any not-yet-submitted launch
  owned by that editor. It does not rescind an already requested source kill or cancel
  its cleanup proc. A fresh unrelated prompt can launch while that cleanup continues.
- A replacement associated with a pending source cleanup still waits for the cleanup
  ordering required by the existing relaunch contract. Re-entering `,X` on the same
  pending record attaches the newly restored editor to that same operation.
- A delayed callback cannot launch text from a cancelled editor, borrow a later editor's
  project/context/bulk targets, detach a later widget, or revive an old wait.
- `,X` stays pinned to its exact launch until its targets are resolved. Absence from a
  UI cache is not proof that a target was killed or dismissed. Only confirmed handled or
  explicitly dismissed targets permit the action to advance to an older record.
- Every concrete result of the targeted launch is handled once, regardless of whether it
  arrived before or after `,X`, in which order procs complete, or whether a sibling
  launch failed. Failed kill initiation remains visible and retryable.

## Implementation

### 1. Give relaunch waits an explicit owner

Refactor `_relaunch_barrier.py`, `_types.py`, and relevant app state into small typed
records for a kill/edit operation and a parked submission. Give each prompt editing
session an opaque identity independent of prompt text, display name, project name, and
widget stack rebuild generation. Associate restored prompt sessions with their source
kill/edit operation; the operation owns its pending launch results and cleanup barriers.

Capture a submission's original prompt context and bulk targets, payload, submit shape,
and owner identity before parking it. Replay only a still-valid submission of that
owner, at most once. Do not recover ownership by testing whether some `_prompt_context`
exists or by swapping global app context around a callback.

Gate submissions on their associated operation, rather than all app cleanup barriers.
Multiple cleanups within one bulk operation still jointly hold its replacement. Other
prompt sessions and unrelated kill/edit operations can progress independently. Preserve
existing backend launch/name-admission checks for a fresh prompt that explicitly asks to
reuse an identity; this TUI change must not bypass those checks or add a Python
implementation of name conflict policy.

Cover `_entry_relaunch.py`, `_kill_last_launch.py`, and
`actions/agents/_marking_kill.py`: resolved flows currently open a barrier before
mounting their editor, whereas the in-flight flow mounts before registering a pending
kill. Create and attach the operation explicitly so both orders are safe. Repeated `,X`
may refocus only its own editor; an unrelated mounted prompt is not proof that the
pending operation already has an editor.

### 2. Invalidate cancelled editing sessions consistently

Wire explicit session invalidation through `_prompt_bar_submit.py`,
`_prompt_bar_mount.py`, and the relevant stash/replace paths. Invalidate before detached
widgets or asynchronous callbacks can act. Whole-bar `Ctrl+C`, whole-stack cancellation,
empty submit, editor cancellation, and replacement by a new prompt must retire the old
session's unsubmitted waits. Save cancelled history once using existing helpers.

Keep per-pane cancellation distinct: `keep_bar=True` retains the surviving stack's
session. Selected-pane submission already removes the pane and rebuilds the stack before
posting `Submitted`; do not reject an accepted parked pane merely because its widget no
longer exists or `_generation` advanced. Use the accepted submission record and session
lifecycle for that case. Preserve multiple legitimate pane submissions and ensure
repeating the same pending whole-bar submit cannot queue duplicate launches.

Account for successful-submit unmount separately from cancellation. The first replayed
submission must not invalidate other legitimately accepted submissions from that stack.
Capture and check the owner through any provider/input preflight callback that can
outlive a cancelled prompt before reaching `_submit_resolved_launch`.

### 3. Track handled launch results separately from proc completion

Update `_launch_records.py`, `_kill_last_launch.py`, and `_launch_procs.py` so proc
completion, kill intent, and per-result handling are distinct facts. On entering the
pending-kill state, process already stamped results; on subsequent callbacks, handle
only unhandled results. Use exact result identity and in-progress/handled bookkeeping to
guard reentrant callbacks and duplicate deliveries before cleanup submission.

Do not consume a record solely because `record.results` is nonempty or all procs have
terminal stamps. Distinguish successful initiation/settlement, initiation failure,
confirmed already-handled targets, and unresolved targets. Preserve retry access to
failed units and avoid killing successful units again. A partial launch failure must not
make the whole gesture disappear while successful or outstanding units remain.

Retain and test the existing explicit timeout and incomplete-admission policies: the
180-second in-flight timeout warns and makes the launch retryable, and an incomplete
admission result covers only the concrete returned units. Do not claim that dismissing
WAITING/QUEUED rows cancels their admission coordinator. Keep diagnostics specific about
pending, failed, timed-out, and initiated cleanup. Late callbacks after timeout or
cancellation of an editor must not launch cancelled text or affect another operation.

The existing 30-second cleanup timeout remains a documented fallback, not evidence that
kill succeeded. Avoid expanding this repair into new admission-coordinator cancellation
or a process-signalling redesign.

### 4. Resolve targets independently of visible-row timing

Replace the loaded-row-only inference in `_matched_agents_for_record` and resolved `,X`
dispatch with explicit resolved, confirmed absent/handled, and unresolved outcomes. Use
all exact results belonging to the record, never the focused row or a name-only match.
Prefer the authoritative `AgentLaunchResult.artifacts_dir` where supplied; retain
established canonical/output-path fallbacks for older result payloads in
`_launch_delta.py` and cover them with tests.

Use the existing bounded artifact-delta/loader path to refresh unresolved targets off
the UI thread and revalidate identities before acting. An unresolved newest launch stays
pinned and gets an actionable status; do not loop through older records or use a stale
bare PID as a substitute for a current target. For a partly loaded bulk result, resolve
every outstanding target instead of quietly operating on the visible subset. Explicit
dismissal evidence may permit skipping a previously handled launch.

Reuse existing restartable-prompt loading, kill/dismiss planning, process-identity
verification, and tracked cleanup submission. Preserve revalidation after asynchronous
prompt resolution and confirmation. New presentation/session state belongs here; shared
lifecycle or target-resolution behavior, if an existing API proves insufficient, belongs
in `sase-core` behind `sase_core_rs`. Open that repo through `/sase_repo` before any
reads or edits and update its API/binding/tests rather than introducing a Python backend
fallback. The diagnosed defects should primarily require TUI orchestration.

### 5. Add regressions and update behavior documentation

Extend the existing focused suites, splitting helpers/files if needed for size limits:

- `tests/ace/tui/test_kill_and_edit_launch_barrier.py`: real prompt `Ctrl+C`, open a new
  bar before completion, submit, and deliver late cleanup. Assert new text/context
  launches once, old text never launches, no inherited wait toast, and requested source
  cleanup still completes. Cover cancellation both before and after a held submission.
- `test_kill_and_edit_inflight.py`: one result before `,X`, several afterward; duplicate
  callback while another proc remains pending; partial failure; rejected kill
  submission; timeout plus late completion; repeated `,X` with an unrelated prompt
  mounted.
- `test_kill_and_edit_last_launch_dispatch.py` and `..._join.py`: completion before row
  refresh, partial row visibility, explicit artifact paths, and confirmed dismissal.
  Update the existing test that equates an empty matcher result with an already-dead
  target to require actual dismissal/terminal evidence.
- Existing last-launch single/bulk tests: confirmation cancellation, disappearance
  during async prompt resolution, target identity preserved across selection changes,
  and exact family/prompt restoration remain covered.
- Prompt stack submit/cancel handlers: per-pane cancellation, multiple accepted held
  pane submissions, repeated whole-bar submit, whole-stack cancel, and late callbacks
  after replacing a prompt. Preserve cancelled-history and focus behavior.

Use event-driven fake completions/timers for the races and at least one Textual Pilot
regression pressing actual `Ctrl+C`; assigning `_prompt_context = None` alone is not
sufficient. Assertions should inspect launched payload/context, target identities,
cleanup requests, and preserved state, not just toast strings. No real user agents
should be terminated by tests.

Update the ACE help description or nearby user documentation if the clarified wait and
cancel behavior is documented there. Keep default key bindings and timeout values; if a
configuration value must change, update `src/sase/default_config.yml` too.

## Verification and acceptance

Read the required `lint_and_test.md` and `tui_perf.md` memory through
`/sase_memory_read` in the implementation turn. Run `just install` as needed for this
clone's native extension/dependencies. Run the focused affected tests and `just check`.
If scoped test selection escalates, broadens unusually, or full verification is required
for landing, run `just check-full` only through `/sase_monitor` using TESTING/TESTED.

Do not block Textual's loop or serial message pump while resolving targets or waiting
for cleanup. Keep existing tracked procs, coalesced refreshes, and cancellation at
teardown. No new keypath disk reads, subprocess waits, or blocking timer callbacks.

Acceptance requires all of these together: the newest requested launch is never silently
replaced by an older target; every result covered by its kill intent is accounted for; a
cancelled editor's old launch cannot replay; a fresh unrelated prompt launches without
the cancelled editor's wait; and an active replacement still respects its own source
cleanup ordering. Report any environment or baseline-check failure separately from
actual test results.
