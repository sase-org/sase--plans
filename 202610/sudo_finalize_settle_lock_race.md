---
tier: tale
title: Sudo finalize survives gate follow-up lock contention
goal:
  A detached sudo finalize whose answer is durable never fails, notifies, or suggests a
  rerun because another settler holds the gate lock.
size: medium
decisions:
  gate_cli_scope:
    ask:
      Should `sase gate answer` (fresh and --resume) also wait out follow-up lock
      contention?
    choices:
      sudo_and_gate_cli:
        Closes the same reclaim race for every gate kind answered via the CLI or TUI
        proc
      sudo_only:
        Only sudo changes; other gate kinds keep the 5s wait and can still fail this
        race
    default: sudo_and_gate_cli
    why:
      Both run as CLI procs after the answer is durable, so the same race and fix apply
    answer: sudo_and_gate_cli
proposed_by: bbugyi200.athena.138
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-23 12:15:09
status: wip
---

# Fix: detached sudo finalize fails on benign gate follow-up lock contention

## Problem

The detached `sase sudo finalize` proc for `bob-cli-an.3`'s sudo gate
(`sudo-193ab068-d2b6-4f0c-b767-3e9e47911d88`, proc `cwn6h82jrg98`) ended with status
`error` and exit code 2. In fact every privileged command ran with exit 0, the gate
answer was durably recorded, the gate shell settled as `answered`, and the follow-up
agent `bob-cli-an.3--1` launched normally. The only thing that failed was the finalize
proc's own shell-settlement call. It lost a race for the per-gate follow-up lock to the
scheduled `housekeeping/gate_turn_reclaim` chop. The proc then reported a hard failure
and published a misleading `ViewErrorReport` notification. That notification tells the
user to rerun the privileged commands with `sase sudo answer <id> --run`, which is wrong
for a gate that is already answered.

## Evidence and timeline (2026-10-23, UTC)

From the sudo bundle (`journal.jsonl`, `errors/`, `execution-state.json`), the proc
runtime (`operation-result.json`, `settlement.json`), the gate artifacts
(`agent_meta.json`, `done.json`), and the chop run record
(`~/.sase/axe/lumberjacks/housekeeping/chops/gate_turn_reclaim/runs/20261023T115714_825070.json`):

- 13:26 and 13:32: the first two approvals failed with `input_too_large` (a 66 KB ledger
  against a 64 KiB cap). HEAD commit `35e593bcc0` fixed that. Those failures are not
  part of this bug.
- 15:57:14: the `gate_turn_reclaim` chop starts as pid 1980033. Its pass lasts 39s.
- 15:57:30.9: the user re-approves. The finalize proc starts and the runner executes all
  6 commands.
- 15:57:35.9: inside the finalize proc, `apply_approved_receipt` →
  `execute_gate_selection` writes `response.json`. From this point the answer is
  durable.
- About 15:57:36–38: the reclaim chop reaches this still-non-terminal gate turn and
  classifies it `DISPOSITION_ANSWERED` (because `response.json` exists). It calls
  `settle_gate_turn(record, gate_state="answered", reason="gate answered")`. That call
  takes `.gate_followup.lock` and holds it through `launch_or_record_followup`, which
  launches `bob-cli-an.3--1` (gate `stopped_at` is 15:57:38, the attempt starts at
  15:57:40, and `done.json` `finished_at` is 15:57:51.9). The lock is held for about
  14s.
- 15:57:38.4: the finalize proc reaches `settle_shell` → `settle_gate_turn`, which uses
  the default `FOLLOWUP_LOCK_TIMEOUT_SECONDS = 5.0`.
- 15:57:43.47: `GateError("lock_timeout", ..., "held by pid 1980033")`. `_run_finalize`
  then calls `_record_finalize_failure`, which writes the journal error and an
  undismissed `ViewErrorReport` notification. Next comes `recover_dead_attempt` (a no-op
  because the proc is live). Finally the proc exits 2 with outcome `lock_contention`,
  and the proc store marks it `error`. `_retire_after_settlement` never runs, so
  `execution-state.json` stays at `startup_state: settling` and the handoff dir under
  `~/.sase/sudo/exec/<gate>/` is leaked.

## Root cause

`settle_gate_turn` is idempotent against a concurrent settler. Once it gets the lock it
sees `_is_terminal_meta` and takes the `settle_already_terminal_handoff` / `diagnose`
path, which will not launch a second follow-up. But callers that settle **after the
answer is already durable** use the 5s default lock wait. That is shorter than the time
a legitimate concurrent settler holds the lock while it launches the follow-up agent
(about 14s here). So benign contention with the reclaim chop turns into a hard failure.
The code already recognizes this hazard for one caller:
`CREATOR_LIVE_FOLLOWUP_LOCK_TIMEOUT_SECONDS = 60.0` exists because "the gate's selected
commands may already have produced irreversible side effects, so it waits out benign
contention instead of failing the run on it." The sudo finalize path is in exactly that
situation, but it does not get the longer wait. Even when settlement cannot finish, the
finalize proc's own job (run the commands and durably record the answer) is complete.
The reclaim chop converges any still-unsettled answered shell:
`reclaim_pending_gate_turns` settles non-terminal answered shells, and
`reconcile_incomplete_gate_handoffs` resumes incomplete handoffs. Reporting `error`, or
offering a rerun, is therefore wrong.

## Changes

### 1. Let post-answer settlement wait out contention (`src/sase/gate_turn/`)

- `settlement.py`: add a keyword-only `wait_out_contention: bool = False` parameter to
  `settle_gate_turn`. Choose the lock timeout as
  `CREATOR_LIVE_FOLLOWUP_LOCK_TIMEOUT_SECONDS if (creator_live or wait_out_contention) else FOLLOWUP_LOCK_TIMEOUT_SECONDS`.
  Update the docstring to say `wait_out_contention=True` is for callers that settle
  after the answer is durable and side effects may have run. The reclaim chop and other
  callers keep the 5s default, because the chop has a per-pass time budget.
- `handoff.py`: keep the constant's name and value, but broaden the comment on
  `CREATOR_LIVE_FOLLOWUP_LOCK_TIMEOUT_SECONDS` so it also covers post-answer settlements
  such as detached sudo finalize.

### 2. Make sudo settlement post-answer-safe (`src/sase/sudo/answer_ops.py`)

`settle_shell` is called only after a durable sudo answer. Its call sites are
`apply_approved_receipt`, `cli._deny`, and
`detach_finalize._finalize_existing_response`; confirm with grep that there are no
others.

- Pass `wait_out_contention=True` to `settle_gate_turn`.
- Make `settle_shell` return `Literal["settled", "deferred", "absent"]`. It returns
  `"absent"` when no gate turn exists (the current early return). If `settle_gate_turn`
  raises `GateError` with code `lock_timeout`, `settle_shell` logs a warning that names
  the gate and the error message (which includes the holder pid), then returns
  `"deferred"` instead of re-raising. Any other exception still propagates unchanged.
- `apply_approved_receipt`: record the settle result. Still call
  `dismiss_sudo_finalize_failed`, because the answer is durable either way. Then add
  `"settlement": <result>` to the returned payload. Do this with an optional
  `settlement` kwarg on `answer_payload` that is emitted only when given, so other
  `answer_payload` callers keep their exact shape.
- `print_answer`: when `settlement == "deferred"`, append a short note to the
  human-readable line, for example "shell settlement deferred to gate reclaim".

### 3. Detached finalize treats deferred settlement as success (`src/sase/sudo/detach_finalize.py`)

- With step 2 in place, `_run_finalize` no longer sees a `GateError` for this race. It
  falls through to `_retire_after_settlement`, so the handoff dir and execution state
  are cleaned up, and `finalize` exits 0 with `success=True`. Make the operation-result
  message distinguish the two cases: "answered" versus "answered; shell settlement
  deferred".
- `_finalize_existing_response`: carry the `settle_shell` result into its payload the
  same way.
- Keep the existing failure handling for every other `GateError`.

### 4. Correct the failure notification when the answer is already durable (`src/sase/sudo/finalize_notifications.py`)

`publish_sudo_finalize_failed` tells the user to rerun the commands even when
`response.json` already exists. If a post-answer failure other than lock contention
still reaches `_record_finalize_failure`, rerunning privileged commands is the wrong
advice. When `bundle_root / RESPONSE_FILENAME` exists:

- Replace the recovery note with one that says the answer is recorded and the commands
  must not be rerun. Point the user at resuming the answered shell's handoff instead.
  Check `sase gate answer --help`. The generic resume path is `resume_answered_shell`
  via `--resume`, with the kind and id flags shown in that help. Verify it works for a
  sudo shell and does not re-execute commands. If it does not, name whichever command
  does resume the handoff without rerunning, and cover it in a test.
- Keep the existing note when `response.json` is absent.

### 5. Generic gate CLI settle sites

> [!decision] gate_cli_scope = sudo_and_gate_cli In
> `src/sase/notification_gates/cli_answer_handle.py` (the fresh-answer
> `settle_gate_turn` inside `with_follow_up_stage_tracking`) and
> `src/sase/notification_gates/cli_answer_resume.py` (`resume_answered_shell`), pass
> `wait_out_contention=True`. Both run after `execute_gate_selection` has made the
> answer durable, and the TUI runs `sase gate answer` as a proc, so the longer wait does
> not block any UI thread. Do not change the reclaim chop, mobile, plan-approval,
> user-question, launch-approval, cancel, or transaction call sites.

> [!decision] gate_cli_scope = sudo_only Skip this section. Only the sudo path changes.

## Tests

Use the existing fixtures in `tests/_sudo_gate_helpers.py`,
`tests/test_sudo_ledger_settlement.py`, and `tests/gate_turn/test_inflight_creation.py`.
Do not let any test sleep for real 5s or 60s. Monkeypatch the timeout constants or the
lock helper on the `sase.gate_turn.settlement` module, because it imports the constants
by name.

1. `tests/gate_turn/test_inflight_creation.py`: extend
   `test_settle_gate_turn_uses_a_longer_lock_timeout_for_live_creators`, or add a
   sibling test, showing that `wait_out_contention=True` selects
   `CREATOR_LIVE_FOLLOWUP_LOCK_TIMEOUT_SECONDS` and the default still selects
   `FOLLOWUP_LOCK_TIMEOUT_SECONDS`.
2. Regression test for the race that actually happened, in a sudo test module: create a
   sudo gate turn and durably answer it. Hold `.gate_followup.lock` from a helper thread
   that settles the shell as the reclaim chop would, for longer than the patched short
   default but shorter than the patched long wait. Then run the finalize or
   `apply_approved_receipt` path. Assert all of the following:
   - It succeeds with `settlement == "settled"`.
   - No `ViewErrorReport` sudo row exists.
   - The shell is `answered`.
   - The follow-up launched exactly once (stub the launcher and count calls).
3. Deferred path: hold the lock past the patched long wait. Assert all of the following:
   - `settle_shell` returns `"deferred"`.
   - The finalize handler returns 0 with `success=True` and `settlement == "deferred"`.
   - No finalize-failure notification is published.
   - The execution state is cleared and the handoff dir is removed.
   - A later `settle_gate_turn` (as the next reclaim pass would run) settles the shell
     normally.
4. `settle_shell` still propagates any `GateError` that is not `lock_timeout`.
5. Notification text: when `response.json` exists, `publish_sudo_finalize_failed` omits
   the "--run reruns the selected commands" advice and names the resume command. When it
   is absent, the existing advice stays. The current
   `test_finalize_failure_notification_dedupes_and_settlement_dismisses` must keep
   passing.
6. If `gate_cli_scope = sudo_and_gate_cli`: show that `sase gate answer`'s fresh and
   `--resume` settle calls pass `wait_out_contention=True`, for example with a recording
   monkeypatch of `settle_gate_turn`.

## Docs

Update `docs/reference/automation/sudo.md` (read the `docs_writing` reference memory
first), in the paragraph that starts "If a detached finalize proc fails after the runner
has already written a ledger". Say that once the answer is recorded, contention with a
concurrent gate settler is waited out and, if it persists, is deferred to gate reclaim
rather than reported as a failure. Also say that when the answer is already recorded,
the notification points to resuming the handoff and never to rerunning the commands.
Mention the additive `settlement` JSON field next to the existing `--json` outcome text.

## Out of scope

- Do not hand-edit any `~/.sase` state for this incident. That includes the stale
  undismissed `ViewErrorReport` row for `sudo-193ab068-…`, its leftover
  `execution-state.json`, and its handoff dir. The user can dismiss the notification.
- Rust core: lock timing and sudo settlement are Python host glue here. No `sase-core`
  change or `sase-core-revision.txt` bump is needed.
- Do not change the reclaim chop's 5s wait or its error counting.

## Verification

Read the `lint_and_test` reference memory, then run `just check`, for example through
`sase tool run check` as that memory and the guarded-recipe rules direct. Do not run
`just check-full`.
