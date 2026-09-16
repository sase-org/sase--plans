---
tier: epic
title: 'Close out sase-zr: decision integrity, honest status, and fast TUI gate refresh'
goal: 'The narrowed sase-zr close-out holds end to end: an accepted gate decision
  can never be superseded while its execution runs, any post-acceptance failure is
  durable and recoverable instead of invisible, TALE/EPIC APPROVED derives from the
  receipt while PLAN COMMITTED waits for archive success, the TUI paints gate responses
  on exact rows promptly from every source, Telegram rejects strangers and TTY-only
  options, and every deliberately deferred audit gap is tracked by a ready task bead
  so the land agent can close sase-zr.

  '
parent_bead: sase-zr
phases:
- id: decision-integrity
  title: Conflict rejection while running, durable failure outcomes, truthful attempt
    completion
  depends_on: []
  size: large
  description: 'decision-integrity: in sase-core and sase, record a verifiable execution
    owner on every accepted receipt, reject conflicting answers while the owner is
    live, permit supersede and cancel only after a durable failed outcome or a proven-dead
    owner, record a redacted durable failure outcome for command, archive, side-effect
    and post-acceptance GateError failures, journal attempt_completed only after terminal
    preparation succeeds, give poll_gate and waiting requesters a failure result,
    and publish one deduped failure notification with resume, restart and cancel actions.'
- id: approval-projection
  title: Receipt-derived approval labels and honest commit status
  depends_on:
  - decision-integrity
  size: medium
  description: 'approval-projection: derive TALE/EPIC APPROVED from the acceptance
    receipt on every load path, show PLAN COMMITTED only after archive success and
    roll it back on failure, surface the failed outcome as a distinct status, and
    publish response.json, shell terminal state and the refresh pulse before post-terminal
    epic launch preparation.'
- id: ace-fast-refresh
  title: Exact, off-loop ACE refresh and actionable failure recovery
  depends_on:
  - decision-integrity
  - approval-projection
  size: medium
  description: 'ace-fast-refresh: route gate receipts and watcher observations to
    exact shell, planner and family row deltas, fix the notification-cache disappearance
    race, make the acceptance pulse target the exact agent directory, move the remaining
    synchronous count refresh and journal reads off the UI thread and message pump,
    and add the plan-gate partial_attempt retry path plus failure-notification resume/restart/cancel
    actions.'
- id: telegram-auth
  title: Authenticated Telegram updates and TTY-only pre-rejection
  depends_on: []
  size: small
  description: 'telegram-auth: in sase-telegram, reject every update whose effective
    chat or callback sender is not the configured chat before any handler runs, hide
    requires_tty options from gate keyboards and pre-reject them before submission,
    and tag submissions with source telegram.'
- id: verify-close
  title: Corrected docs, targeted latency evidence, and combined verification
  depends_on:
  - ace-fast-refresh
  - telegram-auth
  size: medium
  description: 'verify-close: correct the notification and Telegram inbound docs,
    record targeted before/after TUI gate-response latency evidence on an isolated
    fixture, run every changed repo''s checks plus sase''s combined check-full through
    sase_monitor, remove any epic scaffolding, and hand the narrowed-contract re-verification
    of parent sase-zr to the land agent.'
proposed_by: bbugyi200.apollo.07
create_time: 2026-09-16 14:25:04
status: wip
bead_id: sase-zr.7
---

- **PROMPT:** [prompts/202609/sase_zr_close_out.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sase_zr_close_out.md)
- **BEAD:** [sase-zr.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zr/sase-zr.7.md)

# Close out sase-zr: decision integrity, honest status, and fast TUI gate refresh

## Context and scope decision

Epic `sase-zr` shipped all six phases (indexed gate-shell lookup, fast durable decision
acceptance via `decision_receipt.json` under `.acceptance.lock`, ACE durable submission,
Telegram durable submission with early callback acknowledgement, and a supervised
Telegram long-poll receiver). Its land agent never ran; the 2026-09-16 landing audit
(`sase bead show sase-zr`, note #2) found unmet plan requirements, and a first
completion epic covering every gap was proposed as `plan:202609/sase_zr_completion.md`
and **rejected by the owner**.

On 2026-09-16 the owner narrowed the remaining scope: Telegram press-to-ack latency is
now acceptable in practice, and the remaining user-facing pain is that the TUI is slow
to reflect gate status changes and responses even when the gate is answered from the TUI
itself. This epic therefore keeps only:

1. the decision-safety holes that make the epic's core "durable acceptance" promise
   false (a running attempt can be superseded; failure after acceptance is invisible; an
   archive failure strands the gate);
2. honest, receipt-derived status labels;
3. exact, prompt TUI refresh on gate responses; and
4. a small, contained Telegram authentication/TTY fix (a genuine security hole: any
   Telegram account that reaches the bot can trigger handlers including agent launch).

Every other audit gap was deliberately deferred by the owner and is already tracked by a
ready task bead — do **not** re-plan or partially implement them here:

- `sase-11v` — mobile/fleet durable gate submission (audit gap 8).
- `sase-11w` — Telegram receiver/actions hardening: upgrade restart, crash-loop alerts,
  durable update claiming, truthful completion, bounded retries (gaps 9-10 remainder).
- `sase-11x` — bounded gate-shell lookup fallback and settlement-pending repair (gap 6).

Also descoped by the owner: the committed reusable latency probe and the full
cross-surface barrier suite from the original plan (targeted phase tests and the
verify-close evidence replace them).

The `parent_bead: sase-zr` link hands landing back to `sase-zr`: this epic's land agent
re-verifies and closes `sase-zr` against the narrowed contract described in the landing
note below.

Before starting, read the original contract with
`sase artifact read plan:202609/prompt_gate_approval.md "<why>"`. Its constraints still
apply:

- Shared policy belongs in sase-core; there is no Python fallback.
- `response.json` stays the execution-complete record.
- Completed commands are never silently re-run, and no duplicate coder is launched.
- `%auto` stays synchronous and must not spawn a second coder.
- Measure against targets; do not add brittle wall-clock assertions.
- Never touch real user gates or send real Telegram messages.

Repositories: the current `sase` checkout, `sase-core` (`sase repo open sase-core`) and
`sase-telegram` (`sase repo open sase-telegram`). Use only the paths those commands
print and read each repository's instructions. Release a sase-core binding before Python
consumers use it, and ratchet the `sase-core-rs` pin only after a compatible artifact
exists. Never hand-edit Rust release versions. Run `just install` / `just rust-install`
so the workspace binding matches the pin: the audit was misled by a venv at 0.34.35
against the `>=0.34.36` floor. Read `lint_and_test` before finishing, and `tui_perf`
before any TUI change.

Related in-flight work to coordinate with, not duplicate:

- Epic `sase-117.5` (in progress) delivers production settlement-notification targeting
  for the post-settlement EPIC APPROVED to EPIC CREATED transition. The ace-fast-refresh
  phase targets the _acceptance-receipt_ moment; reuse its exact targeting machinery
  where it exists and do not fork or conflict with its handoff changes.
- Task `sase-zc`: the shell-handoff and plan-propose pulse writers compute a wrong,
  positionally derived pulse path. Only touch those call sites if ace-fast-refresh
  introduces a shared exact pulse-path helper; then note the effect on `sase-zc`.
- Task `sase-10z` tracks secret-input leaks; the new failure-outcome record must not
  widen them.
- Task `sase-10y`: the shared artifact-link write lane is currently wedged by a dirty
  hidden plans clone. If a typed link write fails with that error, record the relation
  as a prose bead note instead of retrying.

The audit verified these already-met items; do not redo them: indexed exact gate-shell
lookup (sase-core 682dbec, released in v0.34.25), the decision-acceptance policy and
binding (sase-core 809f45e, v0.34.26), inherited gate ids not matching, `%auto`
synchronous, and the receiver launch fix (sase-telegram 8586f91) with a live receiver on
athena since 2026-09-14.

File and line references below were verified on sase master `e17d4e0c0a` on 2026-09-16.
Confirm them before editing.

## Phase decision-integrity: conflict rejection, durable failure outcomes, truthful completion

The audit reproduced each problem against temporary state; all were re-confirmed in
current source:

1. **A running execution can be superseded.** In
   `src/sase/notification_gates/decision.py::accept_gate_decision` (~lines 185-206), a
   conflicting receipt is superseded whenever `incomplete_attempt(...)` is not `None` —
   and a still-running attempt is also "incomplete". A second, different answer during
   execution replaces the receipt, waits on `.response.lock`, then returns "already
   completed", leaving a receipt that disagrees with `response.json`. `execution_owner`
   is recorded only when `SASE_PROC_ID` is set, and nothing reads it.
2. **Failure is invisible after acceptance.** Acceptance dismisses the notification; a
   later command failure leaves only `errors/*.json`. `poll_gate`
   (`notification_gates/poller.py`) still reports pending and at its deadline a waiting
   requester gets `already_answered`. Cancel is refused because a receipt exists. The
   reclaim routine logs an error every pass until its grace period ends, then marks the
   gate "lost".
3. **Archive failure strands the gate.** `attempt_completed` is journaled before
   `adapter.prepare_terminal_response` runs, so a terminal-prepare (archive) failure
   leaves an attempt that looks complete: a different answer is rejected as a conflict
   and cancel is refused; the only exit is an identical CLI resubmission that re-runs
   every command.
4. **Side-effect failure after `response.json`** is recorded as an error but nothing
   marks the execution failed.

Required outcome:

- **Owner on every receipt.** Record a verifiable execution owner on every accepted
  receipt: the supervising proc id when present, otherwise a host, pid and start-time
  identity.
- **Owner liveness is a policy input in sase-core.** Extend
  `decide_gate_decision_acceptance` (Rust wire, policy, binding, then the Python
  adapter) so it distinguishes a live owner from a dead one or a terminally failed
  attempt. A conflicting answer while the owner is live fails promptly with
  `gate_decision_conflict`. A dead owner or a durably recorded failed outcome permits
  supersede and cancel — this keeps today's recover-by-resubmission behavior, now safe.
- **Durable, redacted failure outcome.** Write a durable outcome record beside the
  receipt (or as journal terminal events) for option-command failures,
  terminal-prepare/archive failures, side-effect failures after `response.json`, and
  `GateError`s raised after acceptance. Reuse `record_execution_error`
  (`notification_gates/command_runner.py`) and the journal rather than inventing a
  parallel store. It carries the attempt id, failed stage (`command`,
  `terminal_prepare`, `side_effects`, `follow_up`), error code and message, and a
  timestamp — never raw input values (see `sase-10z`).
- **`attempt_completed` is truthful.** Journal it only after terminal preparation
  succeeds, or add an explicit terminal-prepare stage. `resume` then skips completed
  option commands and retries only archive/terminal preparation; `restart` stays
  available; completed commands are never silently re-run.
- **Requesters see the failure.** `poll_gate` and every waiting requester get a failure
  result carrying the outcome — never a false pending or `already_answered`.
- **Failure re-raises a notification.** A failed outcome publishes one deduped,
  actionable failure notification per gate and attempt, with resume, restart and cancel
  actions, through the existing notification store and gate-action plumbing. A later
  successful attempt dismisses it. (ACE surfacing and the retry modal land in
  ace-fast-refresh; other surfaces are deferred with `sase-11v`/`sase-11w`.)
- **Reclaim stops crying wolf.** The reclaim routine must not repeatedly error-log a
  gate whose receipt is owned by a live execution.
- **Deliberately descoped** relative to the rejected completion plan: no lease subsystem
  and no new reclaim-side receipt-reconciliation machinery. Dead-owner detection on the
  answer, poll and cancel paths, plus the durable failure outcome, covers recovery; a
  receipt whose owner died simply becomes supersedable and its next poll reports the
  failure.
- **`%auto` unchanged.**

Tests: extend `tests/test_gate_decision_acceptance.py`,
`test_notification_gate_durability.py`, `test_plan_approval_actions_archive.py`,
`test_plan_archive_approval_recovery.py`, `test_gate_cli_answer*.py` and the poller and
reclaim suites, plus the Rust policy tests and the PyO3 binding tests. Cover at minimum:
a conflicting answer while the first command is blocked on a barrier (rejected); a new
answer or cancel after a recorded failure (accepted); an archive failure then `resume`
with no command re-run; a side-effect failure; owner death after acceptance and after
the response; failure-notification dedup and dismissal after a successful retry;
`poll_gate` failure results.

## Phase approval-projection: receipt-derived labels and honest commit status

Findings (verified in current source):

- `TALE APPROVED`/`EPIC APPROVED` come from `plan_approved` metadata written to
  `agent_meta.json` by `persist_plan_approved`
  (`src/sase/ace/tui/actions/agents/_notification_plan_persistence.py`), which
  `plan_approval_actions.py` triggers only after option commands finish; no status
  loader reads the acceptance receipt, so a fresh load shows the old status until
  execution completes.
- Commit-only choices write their metadata before the archive runs and never roll it
  back, so an archive failure still shows `PLAN COMMITTED`.
- `notification_gates/adapters.py` prepares the epic launch before shell settlement, and
  `gate_shell/settlement.py` calls `touch_shell_refresh_pulse` (~line 220) only after
  `launch_or_record_followup` returns — so the externally visible response and refresh
  pulse wait on follow-up launch.

Required outcome:

- At acceptance, project the accepted decision into the shared status projection
  (through the index or metadata projection owned by sase-core) so a fresh load shows
  `TALE APPROVED`/`EPIC APPROVED` from the receipt. Keep decision status separate from
  execution status; the approved label must not free a worker's resources or make an
  active family disappear.
- Show `PLAN COMMITTED` and other execution-dependent labels only after archive success;
  roll back or supersede them on failure. Surface the decision-integrity failed outcome
  as a distinct, visible status the approved label cannot hide.
- Reject, feedback and cancel keep their branch-specific labels. Old in-flight bundles
  without receipts keep working.
- On command success, publish `response.json`, the shell terminal/index state and the
  refresh pulse before post-terminal epic launch preparation; extract that preparation
  from `apply_side_effects` ordering as the parent plan requires. Preserve the invariant
  that the shell's decision record is visible before a successor resolves its fork
  context, and preserve gate-followup locks, fingerprints, claims and live-creator
  suppression.

Tests: with archive and launch barriers held, the approved label is visible and the
committed label absent until archive success; on archive failure the label rolls back to
the failure status; the pulse and terminal state precede follow-up launch; cover
receiptless legacy bundles.

## Phase ace-fast-refresh: exact, off-loop refresh and failure recovery

This phase is the owner's headline complaint: gate responses made _from the TUI_ are
slow to appear. Read `tui_perf` first. Findings (verified in current source,
`src/sase/ace/tui/actions/agents/` unless noted):

- `_notification_utils.py::_request_gate_decision_refresh` refreshes via
  `request_notification_agents_refresh`, which resolves only the notification's agent —
  the planner row for plan approvals — so the shell and family rows get no exact delta;
  for generic gates it falls through to broad completion-notification dirs.
- `_schedule_notification_snapshot_refresh` replaces the notification cache without
  checking what disappeared, and the watcher poll (`_notification_polling.py`) uses that
  cache as its "previous" list, so externally answered gates' disappearance is missed.
- The acceptance-time pulse written by `decision.py::_touch_gate_shell_refresh_pulse`
  matches no agent directory in `actions/event_refresh/_artifact_paths.py`, so it
  triggers a full rebuild instead of an exact delta.
- `_notification_gate_execution.py` still calls the synchronous
  `_refresh_notification_count` (~line 188) and its `on_complete` reads the journal on
  the UI thread; `_notification_sudo.py::_refresh_after_sudo` uses the same synchronous
  count refresh.
- `_notification_plan_gate.py`'s `on_complete` has no `partial_attempt` branch, and the
  generic "answer it again" hint points at an already-dismissed notification.

Required outcome:

- Route receipt and watcher observations — approvals from the TUI, CLI, Telegram and
  mobile alike — to exact shell, planner and family row deltas through the existing
  artifact-delta queue and pump-free helpers. Detect disappearances before the cache is
  replaced. Make the acceptance pulse target the exact agent directory.
- Keep the five-second full-load floor and idle cadence. Protect newer state from older
  in-flight snapshots, including rows that are folded, filtered, promoted or off-tab.
- Move the remaining disk, journal and subprocess reads off Textual's event loop and
  serial message pump, following `tui_perf` (pump-free tasks, `patch_row`, coalescing
  guards).
- Reuse `GateRetryModal` (`src/sase/ace/tui/modals/gate_retry_modal.py`) for the
  plan-gate `partial_attempt` branch, and surface the decision-integrity failure
  notification with working resume, restart and cancel actions even after the original
  review notification was dismissed.
- Coordinate with epic `sase-117.5` and task `sase-zc` as described in Context.

Tests: the receipt watcher with a stale in-flight snapshot; folded and off-tab rows; the
disappearance race; plan-gate `partial_attempt`; failure-notification actions; and
confirm the j/k p95 16 ms navigation benchmark and the quiet-tick reload/file-open
counters still pass.

## Phase telegram-auth: authenticated updates and TTY-only pre-rejection

All work is in sase-telegram (`src/sase_telegram/scripts/sase_tg_inbound.py`,
`src/sase_telegram/inbound.py`, `formatting.py`). Findings from the audit:

- `_dispatch_one_update` checks no chat or user identity: any Telegram account that
  messages the bot reaches `_handle_text_message` (including agent launch) plus photo
  and callback handling.
- `requires_tty` options — sudo `approve` in particular — are rendered by
  `render_gate_keyboard` and submitted; the submission fails and the gate stays pending
  with its Telegram controls removed, including Deny.
- Submissions carry no `source`, so receipts record `cli`.

Required outcome:

- Reject every update whose effective chat — and sender, for callbacks — is not the
  configured chat, before any handler runs; log the rejection without replying to
  strangers.
- Hide `requires_tty` options from Telegram keyboards and also reject them before
  submission, matching `cli_answer._reject_detached_tty_options`. Extend the sudo
  keyboard test to assert no approve button appears.
- Add `"source": "telegram"` to the submission payload.

Test with fake transports and isolated stores: foreign-chat text, photo, document and
callback updates are rejected; the sudo keyboard has no approve button; the receipt
records source `telegram`. Never send real Telegram messages. The broader receiver and
completion-truthfulness hardening is deferred to `sase-11w`; do not start it here.

## Phase verify-close: docs, targeted evidence, combined verification

- **Fix the docs.** In `docs/notifications.md` (fast decision acceptance section):
  describe approved-versus-committed status semantics accurately, document failure
  outcomes and their recovery actions, document old in-flight bundle handling, and move
  the key table that the section split away from its paragraph back under its heading.
  In sase-telegram `docs/inbound.md` and `README.md`: the real stop procedure is
  disabling Telegram (or removing its credentials) — the job tick re-arms the receiver
  and it survives `sase axe stop` and TUI updates — and until `sase-11w` lands, a
  package upgrade requires manually retiring the running receiver so a current-code one
  starts.
- **Record targeted latency evidence.** On an isolated fixture (no real gates), capture
  matched before/after numbers for the two paths this epic changes: acceptance receipt
  to ACE row paint, and response publication/settlement to refresh-pulse visibility. Use
  the phase tests' timers or `SASE_TUI_TRACE=1` spans; record p50/p95 and method in the
  phase bead note. The committed reusable probe from the original plan is descoped by
  the owner — do not build it.
- **Run the checks.** Run `just check` in every changed repo, then sase's combined-tree
  `just check-full` through `sase_monitor`. Record any pre-existing unrelated failures
  with evidence.
- **Clean up.** Remove any temporary epic scaffolding (`sase bead epic-symbols`).

## Landing note for this epic's land agent

After this epic closes, re-verify parent `sase-zr` against the **narrowed** contract and
close it:

1. The narrowed contract is the original `sase-zr` plan minus the items the owner
   deferred on 2026-09-16, which must each be tracked by a ready-or-triaged task bead:
   `sase-11v` (mobile/fleet durable submission), `sase-11w` (Telegram receiver and
   completion hardening), `sase-11x` (bounded lookup fallback), plus the descoped
   committed latency probe and full cross-surface barrier suite.
2. The `sase-zr` landing-audit note (#2) already triaged the original phases'
   `PROPOSED FOLLOW-UP` notes; do not repeat that triage — only check for drift since
   2026-09-16.
3. Close `sase-zr` with a note stating the narrowed-scope decision and the deferral bead
   ids, and mark its plan file `202609/prompt_gate_approval.md` `status: done`.
