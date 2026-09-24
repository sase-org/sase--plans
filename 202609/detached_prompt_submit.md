---
tier: epic
title: Release the prompt bar at submit via detached pending launches
goal: 'Submitting from the ACE prompt input bar removes the bar on the very next paint
  in every launch flow. Kill/dismiss cleanup waits, provider/hold/dispatch preflights,
  and launch bookkeeping continue as a visible, cancellable pending launch that hands
  off to the durable `sase run` proc, and every abort path gives the prompt back.

  '
phases:
- id: pending-launch
  title: Pending launch lifecycle and detached relaunch waits
  depends_on: []
  size: medium
  description: 'pending-launch: add the PendingLaunch record, restore helper, pending
    proc row, `,X` cancel of not-yet-submitted launches, and quit-time stash; unmount
    the bar ahead of the relaunch-cleanup/pending-kill holds; move the synchronous
    post-unmount tail (MRU write, bulk Patch resolution) off the UI thread.

    '
- id: detached-guards
  title: Hold and provider guards run after the bar unmounts
  depends_on:
  - pending-launch
  size: medium
  description: 'detached-guards: move the acceptance point ahead of the `%hold` and
    hard-disabled-provider preflights, re-key both guards per pending launch with
    non-exclusive workers, and surface their decisions without stealing focus, restoring
    or stashing the prompt on abort.

    '
- id: detached-dispatch
  title: Dispatch source preflight becomes a pending-launch stage
  depends_on:
  - detached-guards
  size: small
  description: 'detached-dispatch: post the submit immediately for `%dispatch` prompts
    (syntax errors still keep the bar) and run the source preview as the first pending-launch
    stage, restoring the prompt with the blocked-source line on failure.

    '
- id: keystroke-tag-catalog
  title: Prompt keystroke paths stop loading the project-tag catalog
  depends_on: []
  size: small
  description: 'keystroke-tag-catalog: add snapshot-only effective-tag helpers and
    convert keystroke, render, and submit-handler callers so typing and Enter never
    revalidate or rebuild the project-tag catalog on the UI thread.'
proposed_by: bbugyi200.athena.0r6
create_time: 2026-09-24 15:03:13
status: wip
bead_id: sase-185
---

- **PROMPT:** [prompts/202609/detached_prompt_submit.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/detached_prompt_submit.md)
- **BEAD:** [sase-185](https://github.com/sase-org/sase--beads/blob/main/pages/sase-185/README.md)

# Plan: Release the prompt bar at submit via detached pending launches

## Problem

Pressing Enter in `PromptInputBar` only _starts_ a chain of preflights. The bar is
unmounted at the very end, in `_submit_resolved_launch`
(`src/sase/ace/tui/actions/agent_workflow/_launch_submission.py`), right before the
durable `sase run` proc is submitted. Meanwhile `accepted_whole_bar_submit` turns
further Enter presses into no-ops, so the user is stuck in a bar that no longer
responds.

Today's chain, in order:

1. Widget: TODO-marker confirm modal, then the `%dispatch` source preflight worker
   (`src/sase/ace/tui/widgets/_prompt_input_bar_dispatch.py`). The bar stays until
   `preview_dispatch_launch` returns.
2. App: `_finish_agent_launch` → Prompt Inputs modal → `_preflight_project_tags` (warm
   peek) → `_preflight_hold_confirm` (a worker whenever `%hold` is mentioned) →
   `_preflight_provider_disables` (a `plan_launch_units` worker plus a
   `call_from_thread` round trip whenever **any** provider is hard-disabled) →
   `hold_launch_for_relaunch_cleanup` (parks the submit with the bar mounted until a
   `,x` cleanup proc settles, with a 30 s timeout; the `,X` pending-kill hold can last
   up to 180 s).
3. Unmount, followed by a synchronous tail in the same handler: `launch_toast_label` and
   `record_submit_time_vcs_replay` (each can load the project-tag catalog; the latter
   also writes the MRU file), and for bulk launches a per-Patch `detect_workflow_type`
   loop. Textual only repaints after the handler returns, so this tail also delays the
   bar's visual disappearance.

Evidence from the user's own logs on 2026-09-24 (`~/.sase/logs/tui_toasts.jsonl`,
`tui.log`, `tui_stalls.jsonl`, `~/.sase/llm_provider_disables.json`):

- Two kill-and-edit relaunches kept the bar mounted for 7 s (12:49:48 → 12:49:55) and 17
  s (13:29:34 → 13:29:51) between the "Waiting for kill/dismiss cleanup…" and "Launching
  agent…" toasts. Another hit the 30 s barrier timeout (12:01:26).
- `grok` is hard-disabled (`usage_limit`) for about six days, so every submit pays the
  provider-guard worker round trip, which can land behind one of the TUI's frequent 2–6
  s loop/pump hitches.
- Several 2–4 s hitches with `last_action=text_area_changed` have stacks through
  `_xprompt_arg_assist_project_from_text → effective_vcs_workflow_tag → expand_project_tags → load_project_tag_catalog → _catalog_signature / _build_catalog → detect_workflow_type`.
  A keystroke path is revalidating or rebuilding the tag catalog even though it checked
  `peek_project_tag_catalog()` first. An Enter that lands in such a hitch has to wait
  for it to end.

## Design

### Principle: acceptance ends the bar; the launch continues as a pending launch

A submit is **accepted** once the instant, user-editable checks pass: empty prompt,
TODO-marker confirmation, `%dispatch` directive syntax (`scan_dispatch_directive`), the
Prompt Inputs modal, and `_preflight_project_tags` against the warm catalog. At
acceptance ACE does the following, in the same handler and before starting any worker or
wait:

1. Snapshot everything the launch needs into a `PendingLaunch`.
2. Retire the prompt session and unmount the bar with
   `_unmount_prompt_bar_after_submit()`. This applies to whole-bar submits. A `keep_bar`
   single-pane submit leaves its stack mounted, as it does today. When there is no bar
   (editor, history, wait-action, and mentor-review entry points that call
   `_finish_agent_launch`), unmounting is a no-op.
3. Show the launch toast and a pending proc row, and push a launch record in a new
   `PREPARING` state.
4. Only then run the remaining stages, keyed by the pending launch id: dispatch source
   preview → `%hold` planning/confirm → provider-guard planning/panel → relaunch-cleanup
   / pending-kill hold → durable `sase run` submit.

Nothing after acceptance may depend on `prompt_session_is_live()` or on a mounted bar.

### Why a pending launch that hands off to the existing durable proc

From the moment Enter is pressed, the accepted launch is surfaced as a proc: a
placeholder row in the proc indicator and Procs tab (`ProcObserver.register_pending`).
The row's message shows the current stage, and the durable `sase run` proc's own row
takes over at submit. `sase run` remains the only out-of-process worker, and it already
re-enforces the hard-disable guard (`guard_hard_disabled_launch_units`) and project-tag
policy before it spawns anything.

We deliberately do **not** add a second durable "preflight" proc. The remaining waits
are ACE-session choreography that a detached process cannot observe or resolve without a
new cross-process protocol:

- ACE's proc observer settles the relaunch cleanup barrier.
- The `,X` pending-kill hold only ends after ACE kills the previous launch's results.
- `%hold` and provider decisions need ACE modals. A durable `sase run` skips `%hold`
  confirmation entirely when it runs off-TTY.

Rejected alternatives:

- **A separate durable preflight proc.** Adds 1.5 s or more of Python startup plus a
  callback protocol for every interactive decision.
- **Dropping the TUI provider guard and reacting to `sase run` refusals after the
  fact.** The blocked-provider panel would appear seconds later, and the refusal already
  stashes the prompt, so recovery would happen twice.
- **Deadline-bounded pre-unmount planning** (keep the bar for at most about 150 ms).
  This means two code paths for one stage and timing-dependent behavior.

### `PendingLaunch` record

Add a new module `src/sase/ace/tui/actions/agent_workflow/_pending_launch.py`; the name
is flexible.

- **Fields:** `launch_id` (uuid), `prompt`, `context` (a `PromptContext` copy),
  `keep_bar`, `extra_payload`, `bulk_patches`, `relaunch_operation`, `stage`,
  `placeholder_id`, and `cancelled`/`submitted` flags.
- **Stage values:** `DISPATCH_PREVIEW`, `HOLD_CHECK`, `HOLD_CONFIRM`, `PROVIDER_CHECK`,
  `PROVIDER_DECISION`, `WAITING_CLEANUP`, `WAITING_LAST_LAUNCH`, `SUBMITTING`.
- **Registry:** `app._pending_launches: dict[str, PendingLaunch]`.
- **Helpers:**
  - `begin_pending_launch`
  - `pending_launch_is_live(app, launch_id)`
  - `set_pending_launch_stage`, which also updates the row message
  - `finish_pending_launch`, which removes the placeholder once the durable submit has
    registered its own and attaches proc ids to the launch record
  - `cancel_pending_launch`
- `AcceptedLaunchSubmission` in `_launch_submission.py` folds into this record (or wraps
  it), and `owner_id` becomes `launch_id`.

### Restoring a prompt

Every abort path uses one helper,
`restore_pending_launch_prompt(app, launch, *, reason, explicit)`:

- **Explicit user gestures (`,X`)** always restore into a prompt bar. They use the
  existing `_mount_inflight_launch_prompt` / `_edit_and_relaunch_agent` machinery with
  the launch's `relaunch_operation`, exactly like the in-flight `,X` path. A re-submit
  therefore still respects any barrier that is open.
- **Automatic aborts** (decline, abort-all, blocked source, stage error) restore into a
  bar only when no prompt bar is mounted and no modal is on the screen stack. Otherwise
  they stash with `_schedule_failed_launch_prompt_recovery` and warn "… prompt saved to
  stash (press @ to restore)". They never overwrite a bar the user is typing in. A
  `keep_bar` pane launch therefore stashes on abort.

### Pending proc row

- Register the row at acceptance with the display name `launch <display_name>` and
  **no** exclusive scopes, so it can never collide with the durable submit's `launch:`
  concurrency key in `_submit_durable_proc`.
- Add a small `ProcObserver` update API (for example
  `update_pending(placeholder_id, message=...)`) for stage messages such as "waiting for
  kill/dismiss cleanup" and "checking providers".
- Remove the row in the same UI-thread step that calls `_submit_launch_proc` (whose own
  placeholder takes over), and on cancel or abort.

### Undo and quit

- `,X` on a `PREPARING` record cancels the pending launch before anything is submitted,
  so no kill is needed. It drops the parked barrier waiter or guard session, removes the
  row, and restores the prompt (explicit restore).
- Extend `LaunchRecordState` with `PREPARING`. `push_launch_record` must accept a record
  with no proc ids yet and attach them at submit (`_launch_records.py`,
  `_kill_last_launch.py`).
- A bulk pending launch restores its shared prompt. Re-marking the Patches is not
  restored.
- Quitting with live pending launches stashes each prompt, off the event loop, as part
  of the best-effort flush in `_flush_then_do_quit`, so `@` recovers them. Crash
  recovery is out of scope; today a crash loses the bar text too.

### Keeping the post-acceptance handler thin

Because Textual repaints only after the handler returns, everything the handler still
does synchronously after the unmount call must be in-memory work:

- `record_submit_time_vcs_replay` moves off the UI thread, either into the
  durable-submit worker or into a pump-free task (`spawn_pump_free_task`).
- `launch_toast_label` must not load the tag catalog. It uses the snapshot-only helper
  from the `keystroke-tag-catalog` phase; until that lands, keep today's behavior.
- Bulk fan-out (`_submit_bulk_resolved_launch`): per-Patch project-file resolution,
  `detect_workflow_type`, and prompt rewriting move to a thread worker that returns a
  typed per-Patch plan. The UI thread then submits the procs and pushes the record.
- `reserve_launch_timestamp_batch` may stay on the UI thread only if it measures
  sub-millisecond when warm. Otherwise reserve the timestamp in the submit step, after
  the bar is gone.

## Phase pending-launch

This phase adds the record, restore helper, row, `,X` cancel, and quit stash. It moves
the unmount point in `_submit_resolved_launch` to **before**
`hold_launch_for_relaunch_cleanup`. In this phase the hold and provider guards still run
before the unmount, unchanged.

- `_relaunch_barrier.py`:
  - Key `_RelaunchParkedLaunch` by `launch_id`.
  - `_drain_relaunch_cleanup_launch_waiters` drops a waiter when its pending launch is
    cancelled, not when the prompt session is retired.
  - Stage messages become `WAITING_CLEANUP` / `WAITING_LAST_LAUNCH`.
  - The barrier timeout keeps its 30 s release and warning toast.
- `_launch_submission.py`: accept at the top of the non-parked path. Single and bulk
  launches both go through the pending launch. Replays after settle use the stored
  snapshot and never read `_prompt_context`.
- Show one toast at acceptance: "Launching agent for X…", or, when the launch is parked,
  a toast that names the wait.
- Add tracing in `src/sase/ace/tui/util/trace.py`: `trace_event("launch.accepted", …)`
  at acceptance and `launch.submitted` with `accept_to_submit_ms` and the stages
  visited, so `SASE_TUI_TRACE=1` can measure the fix.
- Tests: update `tests/ace/tui/test_kill_and_edit_launch_barrier.py`,
  `tests/ace/tui/test_kill_and_edit_inflight.py`, and
  `tests/ace/tui/test_launch_records.py`, and add a focused pending-launch test module.
  Cover:
  - Submitting while a barrier is pending unmounts the bar immediately (no
    `PromptInputBar` after one `pilot.pause()`), shows the pending row, submits nothing,
    then submits exactly once when the barrier settles.
  - `,X` on the pending launch restores the prompt with the same relaunch operation and
    drops the waiter; a later settle launches nothing. The replacement submit is still
    held until the barrier settles.
  - The barrier timeout still releases with its warning.
  - Two overlapping barriers replay the parked launch once, and an unrelated new prompt
    launches immediately while an older launch is parked.
  - Quitting with a parked launch stashes its prompt.
  - Bulk fan-out resolves Patches off the UI thread.
  - Rewrite `test_cancelling_prompt_bar_during_hold_drops_parked_launch` and
    `test_cancelled_hold_drops_old_submit_and_new_prompt_launches` around `,X`
    cancellation, since there is no longer a bar to cancel.
- Docs: update the kill-and-edit and `,X` sections of `docs/ace.md`. Submit now returns
  immediately, a parked launch shows as a proc row, and `,X` cancels a launch that has
  not been submitted yet.

## Phase detached-guards

- Move acceptance into `_launch_resolved_prompt` (`_launch_prompt_inputs.py`), right
  after `_preflight_project_tags`. `_preflight_hold_confirm` and
  `_preflight_provider_disables` take a `launch_id` and never a prompt session.
- `_launch_provider_guard.py`:
  - The single app-wide `_provider_guard_session` becomes per-pending-launch state on
    the `PendingLaunch`.
  - `_provider_guard_context_is_live` becomes `pending_launch_is_live`, with no
    mounted-bar requirement.
  - Worker names and groups become per launch and **non-exclusive**. Today they use
    `exclusive=True, group="launch-provider-guard"`, so a second submit would cancel the
    first launch's planning.
  - `_on_provider_guard_failed_open` still fails open and submits.
- `_launch_hold_guard.py`: the same re-keying, with a per-launch, non-exclusive worker
  group.
- When a guard needs the user:
  - Push `DisabledProviderLaunchModal` or the "Arm this hold?" confirm only if no prompt
    bar is mounted and no other modal is open. A hold confirm that appears on its own
    defaults to **Cancel**.
  - If the panel can't be shown now, or a second pending launch needs a decision while
    one panel is already up, stash that launch's prompt with a warning naming the reason
    instead of stealing focus.
  - While waiting, the row stage is `PROVIDER_DECISION` / `HOLD_CONFIRM`.
- Decline, abort, abort-all, and aborting every unit call
  `restore_pending_launch_prompt` (automatic rules). The toast says the prompt was
  restored or stashed, replacing the now-false "your prompt is still here". Enable,
  soft-enable, and re-model continue to the next stage.
- The stale toasts (`_STALE_TOAST`, `_HOLD_STALE_TOAST`) now only fire for cancelled
  pending launches.
- Tests: update `tests/ace/tui/test_disabled_provider_launch_panel.py` and
  `tests/ace/tui/test_launch_hold_guard_panel.py`. Cover:
  - The bar is unmounted before any guard worker runs.
  - With no hard disable, the launch submits without a worker.
  - A blocked unit opens the panel after the unmount, and aborting restores the prompt
    into a bar.
  - With another bar mounted, a blocked launch stashes and warns, and never pushes the
    panel.
  - Two concurrent pending launches both finish planning (non-exclusive workers).
  - A hold confirm that appears on its own defaults to Cancel.
  - The relaunch entry still reaches the same panel.
- Docs: update the "Disabled-provider launch panel" section and the `%hold`
  launch-preview confirmation bullet in `docs/ace.md`.

## Phase detached-dispatch

- `src/sase/ace/tui/widgets/_prompt_input_bar_dispatch.py`:
  `_maybe_preflight_dispatch_submission` keeps the instant `scan_dispatch_directive`
  syntax-error path (the bar stays and shows the error line), but no longer runs
  `preview_dispatch_launch` before posting `Submitted`.
- App pipeline: the first stage, `DISPATCH_PREVIEW`, runs for prompts with a `%dispatch`
  directive. It calls
  `preview_dispatch_launch(prompt, payload=dispatch_payload_from_prompt_context(ctx))`
  in a per-launch worker.
  - On success: call `_record_dispatch_launch_preview(...)`, then continue.
  - On error: restore the prompt (automatic rules) and seed
    `_dispatch_preflight_override` so the restored bar's dispatch context line shows
    `source blocked: <error>`. If the prompt is stashed instead, show the error toast
    `Dispatch not submitted: <error>`.
- Keep the `_mark_dispatch_launch_unknown_for_prompt` semantics for later failures.
- Tests: update `tests/ace/tui/widgets/test_dispatch_target_picker_focus.py` and the
  dispatch launch tests. Cover:
  - The bar unmounts immediately for a `%dispatch` prompt.
  - A blocked source restores the prompt with the error line.
  - Syntax errors still keep the bar mounted.
- Docs: update the `%dispatch` target section of `docs/ace.md` if it describes checking
  at submit time.

## Phase keystroke-tag-catalog

- In `src/sase/project_tags/tags.py`, add snapshot variants built on the existing
  `expand_project_tags_with_catalog`, for example
  `effective_vcs_workflow_tag_with_catalog(prompt, catalog)` and
  `effective_find_vcs_workflow_tag_with_catalog(...)`.
- Keystroke, render, and submit-handler callers call `peek_project_tag_catalog()` once
  and use the snapshot variants. A cold catalog means no tag expansion (fall back to raw
  `#` extraction), never a load. Convert these call sites:
  - `src/sase/ace/tui/widgets/_xprompt_arg_hints.py`
    (`_xprompt_arg_assist_project_from_text`)
  - `src/sase/ace/tui/widgets/_prompt_input_bar_stack_navigation.py`
  - `src/sase/ace/tui/_agent_completion_prompt.py`
  - `launch_toast_label` in
    `src/sase/ace/tui/actions/agent_workflow/_launch_submit_helpers.py`
- Audit `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_requests.py`
  (`expand_project_tags`, `effective_vcs_workflow_tag`) and
  `src/sase/ace/tui/widgets/_file_completion_base.py` (`load_project_tag_catalog()`).
  Convert any call that runs synchronously on the UI thread; leave true background
  warmers alone.
- Catalog freshness stays with the existing background warm/refresh. Do not add a new
  refresh path.
- Tests: with a warm peeked catalog and `load_project_tag_catalog` monkeypatched to
  raise, typing `+tag …` in the prompt bar and computing the launch toast label never
  call it and still expand the tag. With a cold catalog, both fall back without loading.

## Constraints for every phase

- Before changing TUI code, read the `tui_perf.md` SASE memory note. Rules 1–4 (no
  blocking the loop or the pump, durable procs for slow user actions, re-capture state
  after awaits) and rule 11 (keystroke paths stay read-only) apply directly.
- Guard planning, tag policy, and dispatch preview stay in their existing backend
  modules. This epic changes only ACE orchestration and presentation, so nothing crosses
  into sase-core.
- No feature flag. Each phase lands a complete behavior: `pending-launch` detaches only
  the barrier waits, `detached-guards` moves the acceptance point earlier,
  `detached-dispatch` detaches the dispatch preview, and `keystroke-tag-catalog` is
  independent.
- No keymap changes are planned, so `src/sase/default_config.yml` should not need edits.
- Verify each phase with `just check` via `sase tool run`, as the lint-and-test memory
  note requires. Do not run `just check-full`.

## Acceptance

- With a pending relaunch barrier, an active hard disable, a broad `%hold`, or a
  `%dispatch` directive, `PromptInputBar` is gone after the first repaint following
  Enter and focus is back on the active list. `SASE_TUI_TRACE=1` shows `launch.accepted`
  and the unmount in the same handler.
- Every accepted launch appears as a proc row from Enter until its durable `sase run`
  row replaces it.
- No path loses a prompt: `,X` restores it, automatic aborts restore or stash it, and
  quitting stashes pending launches.
