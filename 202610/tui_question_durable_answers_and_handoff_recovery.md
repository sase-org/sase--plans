---
tier: tale
title: Answer TUI questions out of process and recover interrupted gate handoffs
goal:
  An ACE restart can no longer strand a gate follow-up, and an interrupted handoff never
  stalls silently.
size: medium
decisions:
  auto_resume:
    ask:
      Should the gate-turn reclaim routine resume an interrupted follow-up launch on its
      own, once, before asking a human?
    choices:
      once:
        Routine submits one supervised --resume proc per gate; notifies if impossible or
        already tried
      notify:
        Routine never launches; it sends one notification with the resume command for a
        human
    default: once
    why:
      Interrupted launches with no successor are what --resume is for; sessions must not
      stall silently
    answer: once
proposed_by: bbugyi200.athena.15w
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-29 14:00:42
status: wip
---

# Answer TUI questions out of process and recover interrupted gate handoffs

## What happened (agent session `15t`, project bob-cli)

`15t` stalled. It has been idle since 12:56 EDT on 2026-10-29, with no successor
running:

1. `15t--1` asked a question gate (`be5cbcdf-ed37-4f0f-8ce6-fd5997a5ed2c`: "authorize
   the host to commit these Task Card theme changes?"). The user answered **Authorize
   commit** in ACE at 12:54:47.
2. The ACE question modal (`_submit_question_response_task` in
   `src/sase/ace/tui/actions/agents/_notification_question_modal.py`) runs
   `execute_user_question_response` **inside the TUI process** as a session-worker
   thread. That covers selected-command execution, `settle_gate_turn`, and the follow-up
   launch. Neutral gates (`submit_gate_execution_task`) and plan gates (`ae6afe9685`,
   epic `sase-zr`) already answer through `_submit_durable_proc`, which runs a separate
   `sase gate answer --no-detach --json` process. Questions are the remaining in-process
   kind.
3. The gate turn `15t--gate-0` settled, and `launch_or_record_followup` persisted
   attempt stage `launching` (`live: true`, `owner_pid` = the TUI pid) at 16:56:15Z.
4. At 16:56:26Z ACE's dev auto-update logged _"restart wait expired; restarting with 1
   TUI task: answer question be5cbcdf… still active"_. `restart_after_update_when_ready`
   (`src/sase/ace/tui/update_restart.py`) only waits `_RESTART_WAIT_SECONDS = 60`, and
   the launch can itself wait up to 60 s in `wait_for_starter`. ACE re-exec'd **in
   place, with the same pid**, and the launch thread died between "launching" and
   "launched". `done.json` was never rewritten with follow-up fields, and no failure
   notification was sent.
5. At 12:57:26 the `gate_turn_reclaim` routine (`reconcile_incomplete_gate_handoffs` →
   `_diagnose_one`) classified the gate in `diagnose` mode. The Rust classifier never
   allows a launch in that mode, so it only persisted `gate_followup_outcome: failed` /
   `gate_followup_error: "a stale handoff attempt can be resumed after ruling out a launched successor"`.
   It notified nobody and advanced its cursor past the gate, so the gate is never looked
   at again.
6. Because the re-exec kept the pid, the persisted attempt kept looking live
   (`gate_followup_attempt_is_live` is just "live flag + pid running"). While that ACE
   process lived, `sase gate answer … --resume` would have classified as
   `in_progress`/`wait` and done nothing, and `%w:15t` waiters stayed pending
   (`turn_followup_handoff_agent` in
   `src/sase/core/wait_dependency_resolution/_artifact_state.py`). The settle-hold
   workspace claim, also owned by that pid, stayed held for about 50 minutes until that
   ACE process exited and axe `stale-cleanup` released it.

A read-only dry run of `classify_gate_handoff(mode="resume")` against the gate's current
metadata (the owner pid is now gone) returns `launch_allowed: true` with no successor
evidence. So the documented resume command would work today. That is an operational step
for the user, not part of this plan.

## Goals

- An ACE update restart, or any other ACE exit, can no longer kill a question answer
  mid-settlement. Question answers run out of process like every other gate kind.
- A persisted follow-up attempt whose launcher is gone reads as dead, even when the pid
  was reused by an in-place re-exec. `--resume` then works and `%w:` waiters stop
  hanging.
- An interrupted handoff never stalls silently. The reclaim routine resumes it (under
  `auto_resume = once`) or notifies with the exact resume command.

## Non-goals

- Resuming session `15t` itself. The user does that.
- Changing the Rust gate-follow-up classifier in `sase-core`. Its `resume` mode already
  allows the launch. This plan only changes which mode Python asks for and which
  liveness it reports.
- Migrating other in-process answer paths: the mobile gateway's
  `execute_user_question_response` call, `launch_approval_actions`, and sudo. The
  recovery changes in steps 2 and 3 still cover them. If you confirm another ACE gate
  kind that still settles in-process, file it with `/sase_new_task` rather than widening
  this tale.
- Changing the 60 s update-restart wait.

## Changes

### 1. Answer turn-backed questions through ACE's durable proc queue

In `src/sase/ace/tui/actions/agents/_notification_question_modal.py`, make
`_submit_question_response_task` follow `submit_gate_execution_task`
(`_notification_gate_execution.py`) for neutral, non-legacy question bundles:

- Before submitting, resolve the bundle and normalize/validate the modal's answers in
  the TUI, so malformed answers still fail fast with today's messages. Reuse the helpers
  `execute_user_question_response` already uses (`_resolve_question_bundle`,
  `_questions_from_request_root`, `_normalize_user_question_response` in
  `src/sase/user_question_actions.py`). If they need to be public, expose them through a
  small public wrapper rather than importing privates across modules.
- Submit through `app._submit_durable_proc(...)`. Use argv
  `sase_argv("gate", "answer", "--id", <request_id>, "--kind", "question", "--no-detach", "--json")`,
  `operation=GATE_ANSWER`, and
  `request=durable_request_payload(option_ids=[QUESTION_OPTION_ID], input_data=<normalized>, feedback=<non-blank global note or None>, source="tui")`.
  Use a
  `durable_fingerprint(GATE_ANSWER, "question", request_id, QUESTION_OPTION_ID, "", "tui")`,
  concurrency key `notification-gate:<notification id>`, and a display name like
  `answer question <request_id>`. Running the CLI path also gives the answer
  `settle_gate_turn(..., wait_out_contention=True)` and `with_follow_up_stage_tracking`,
  which the in-process path lacks.
- Keep the user-visible behavior of today's `_finish_question_response_task`. On
  success, call `on_response_written`, toast the result, refresh the notification count,
  and run the question side effects from `_run_question_side_effects` (dismiss the
  notification and `mark_already_handled`). Map an already-answered/`gate_cancelled`
  failure to the same warning ("Question was already answered") and anything else to an
  error toast. Call `schedule_gate_decision_receipt_refresh` the same way the neutral
  gate path does.
- Legacy (non-neutral) question bundles only write a JSON file and have no settlement,
  so they keep the existing in-process path. If `_submit_durable_proc` is unavailable,
  keep the synchronous fallback.
- Confirm that the `sase gate answer --kind question` operation-request path accepts the
  normalized `{"answers": [...], "global_note": ...}` object as `input_data` for the
  `submit` option, the same object `execute_user_question_response` passes to
  `execute_gate_selection`. Add whatever minimal adapter the CLI needs if it does not.

### 2. Make an orphaned follow-up attempt read as dead

The launcher holds the per-gate `.gate_followup.lock` (an `flock` taken in
`with_gate_followup_lock`) for the whole settle-and-launch, and it writes every
`live: true` mark under that lock. Python opens files close-on-exec, so an in-place
re-exec releases the lock while the pid survives. Use the lock as the liveness proof:

- In `src/sase/gate_turn/attempt_liveness.py`, give
  `gate_followup_attempt_is_live(meta, *, artifacts_dir: str | None = None)` a lock
  probe. Keep the existing flag and pid checks. When an artifacts dir is known, open
  `<artifacts_dir>/.gate_followup.lock` without creating it and try
  `flock(LOCK_EX | LOCK_NB)`. If the file is missing or the lock is acquired (release it
  immediately), the attempt is **not** live. If the lock is held, it is live. Move the
  `FOLLOWUP_LOCK_NAME` constant here, or somewhere both modules can import, to avoid a
  `handoff.py` ↔ `attempt_liveness.py` cycle. Document that a lock holder must never
  call the probe, because it would see its own lock.
- `classify_gate_handoff` in `src/sase/gate_turn/handoff.py` takes a required keyword
  `holds_followup_lock: bool`. When it is `True`, `_attempt_snapshot` reports any
  persisted attempt as `live: False`, because holding the exclusive lock proves no other
  launcher is running. Otherwise it uses the probe with `meta.get("artifacts_dir")`.
  Callers:
  - `launch_or_record_followup` (`handoff_launch.py`) and `_diagnose_one` (`reclaim.py`)
    pass `True`; both run under the lock.
  - `gate_turn_followup_needs_attention` (`projection.py`) passes `False`.
- In `turn_followup_handoff_agent` (`_artifact_state.py`), pass the gate member's
  artifacts dir to `gate_followup_attempt_is_live`. Take it from `meta["artifacts_dir"]`
  when present, or thread it from the caller. If it is unknown, keep today's behavior.

### 3. Reclaim routine: resume or notify instead of failing silently

In `_diagnose_one` (`src/sase/gate_turn/reclaim.py`), after classification:

- The reconciler's `meta` dict is built from the `GateTurnRecord` and lacks
  `gate_selected_option_ids`, `cl_name`, `gate_followup_attempt_started_at`, and the new
  markers below. Read the gate's `agent_meta.json` for those.
- **Interrupted attempt** means: the attempt stage is `preparing` or `launching`, no
  `gate_followup_error_type` is recorded, the decision's `recovery == "resume"` and
  `resume_eligible`, and successor evidence is unambiguous with no attached, running, or
  completed successor.
- If an interrupted attempt started less than 120 s ago (a just-spawned successor may
  not be indexed yet), return `None` before `apply_decision`. That defers the gate, so
  the project cursor does not advance past it.

> [!decision] auto_resume = once For an interrupted attempt with no
> `gate_followup_auto_resume_attempt_id` equal to the current attempt id, persist
> `gate_followup_auto_resume_attempt_id` and `gate_followup_auto_resume_at` under the
> lock. Then submit one supervised proc, mirroring `submit_detached_answer` in
> `src/sase/notification_gates/cli_answer_submit.py`: `submit_proc_request` with argv
> `["sase", "gate", "answer", "--id", <gate_id>, "--kind", <kind>, "--no-detach", "--json"]`,
> `operation=GATE_ANSWER`, and payload
> `{"option_ids": <gate_selected_option_ids>, "retry": "resume", "source": "gate-reconcile"}`.
> Use a proc so the launch is not bound by the chop's 75 s / 2 m budget. The resume path
> collects successor evidence again under the lock, adopts a late successor if one
> appeared, and already notifies on launch failure. Count it as a new `resumed` summary
> field. If the attempt was already auto-resumed, or the submission raises, fall through
> to the notify rule below.

> [!decision] auto_resume = notify Never submit a resume proc. Every interrupted attempt
> goes straight to the notify rule below.

- **Notify rule.** When the decision `needs_attention` and the failure was not already
  reported, call
  `notify_handoff_failure(full_meta, error=<decision reason>, stage=<attempt stage or "interrupted">)`
  and persist `gate_followup_attention_notified_attempt_id` so later passes do not
  repeat it. It counts as already reported when the marker equals the attempt id, or
  when a `gate_followup_error_type` shows the launcher's own exception path already
  notified. Its body already includes `resume_command_for(meta)`.
- Report the new count in `sase_chop_gate_turn_reclaim.py`'s summary payload.

### 4. Docs and config text

Read the `docs_writing` reference memory before editing `docs/`. Then update:

- `docs/reference/automation/notification-gates.md`, in the paragraph on detached
  answers and "A failed or interrupted handoff…". Say that ACE question answers also run
  as a durable proc, and describe what the reclaim routine now does with an interrupted
  handoff under the chosen `auto_resume` branch.
- The `gate_turn_reclaim` description in `src/sase/default_config.yml`, and wherever
  `docs/reference/automation/scheduler-default-routines.md` mirrors it, to mention the
  resume-or-notify step.

## Tests

- Question modal (extend the existing `tests/ace/tui` question-notification tests). A
  neutral bundle submits exactly one `_submit_durable_proc` call with the argv, payload,
  fingerprint, and concurrency key above, and never calls
  `execute_user_question_response` in-process. A successful completion runs
  `on_response_written` and the dismissal side effects. An already-answered failure
  shows the warning. A legacy bundle still uses the in-process path. Invalid answers
  fail before submission.
- Liveness. Re-exec case: `live: true`, `owner_pid = os.getpid()`, lock file present but
  unheld → not live. Lock held through `with_gate_followup_lock` on another open file
  description → live. Lock file missing → not live.
  `classify_gate_handoff(..., holds_followup_lock=True)` in `resume` mode on a persisted
  `launching` + live-flag attempt returns `launch_allowed: true`. The `_artifact_state`
  waiter no longer reports `<pending-turn-followup>` for the re-exec case.
- Reconciler, with `submit_proc_request` and `notify_handoff_failure` mocked:
  - An interrupted attempt older than 120 s with no successor gets one resume proc with
    the payload above, and the marker is persisted. A second pass submits nothing and
    notifies once (`once` branch).
  - A younger attempt is deferred and the cursor does not advance past it.
  - An attempt with a recorded `error_type` is never auto-resumed or re-notified.
  - Under the `notify` branch, an interrupted attempt produces exactly one notification
    and no proc.
- Existing gate-turn reclaim/handoff tests keep passing after the new required
  `holds_followup_lock` keyword is threaded through.

## Verification

Read the `lint_and_test` reference memory and run the verification it prescribes. That
is the guarded `just check` through `sase tool run`, not `just check-full`.

## Acceptance criteria

- Answering an ACE question on a turn-backed gate creates a durable `Gate answer` proc,
  and the TUI thread does no gate execution or settlement.
- In the re-exec case, a gate whose launch died reads as not live everywhere: the
  `--resume` classification, `%w:` wait resolution, and the reclaim routine.
- An interrupted handoff is resumed automatically once (`once`) or produces exactly one
  notification carrying the resume command (`notify`). It never only flips to `failed`
  silently.
