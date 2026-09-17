---
tier: epic
title: 'Gate decision integrity: owned execution, durable failure outcomes, truthful
  completion'
goal: An accepted gate decision can never be superseded or cancelled while its execution
  owner is live; every post-acceptance failure (option command, terminal preparation
  or archive, side effects, follow-up, or owner death) leaves a redacted, durable,
  receipt-scoped failure outcome; attempt_completed is journaled only after the response
  is published; resume retries only the failed work; poll_gate and every waiting requester
  receive a failure result instead of a false pending or already_answered; and each
  failure publishes one deduped notification carrying resume, restart and cancel recovery.
parent_bead: sase-zr.7.1
phases:
- id: core_execution_policy
  title: Execution owner, failure outcome and liveness policy in sase-core
  depends_on: []
  size: medium
  description: 'core_execution_policy: in sase-core, add the execution owner, acceptance
    id, execution-facts and failure-outcome wires; make decide_gate_decision_acceptance
    reject conflicts while the owner is live and supersede only after a failed outcome
    or a proven-dead owner; add cancellation-over-receipt precedence and accepted_failed/accepted_owner_lost
    dispositions with cancel/supersede permissions to decide_gate_lifecycle; add claim_gate_decision_execution;
    cover it with Rust and PyO3 binding tests; land it so release-plz publishes it.'
- id: failure_journal
  title: Receipt-scoped journal, truthful attempt completion and durable failure outcomes
  depends_on:
  - core_execution_policy
  size: medium
  description: 'failure_journal: in sase, adopt the new core revision, stamp every
    receipt and journal lifecycle event with an acceptance id, record a redacted attempt_failed
    outcome for command, terminal_prepare, side_effects and follow_up failures, journal
    attempt_completed only after response.json is published, make resume retry only
    archive/terminal preparation or failed side effects without re-running completed
    commands, and keep legacy early-completed journals resumable.'
- id: owner_conflict
  title: Verifiable owner, live-owner conflict rejection and post-failure supersede/cancel
  depends_on:
  - failure_journal
  size: medium
  description: 'owner_conflict: in sase, record the execution owner on every receipt,
    collect lock/pid/proc liveness and failure facts for the core policy, reject a
    conflicting answer while the owner is live, supersede or cancel only after a failed
    outcome or proven-dead owner, re-own the receipt under the acceptance lock before
    execution, and teach lifecycle, reclaim, gate show and gate cancel the new dispositions.'
- id: failure_surfacing
  title: Failure results for requesters and deduped recovery notifications
  depends_on:
  - owner_conflict
  size: medium
  description: 'failure_surfacing: in sase, give poll_gate a failed status, record
    owner_lost on the poll and reclaim paths, keep waiting requesters honest through
    their deadlines, update sase gate wait and the other requesters, publish one deduped
    GateExecutionFailed notification per failure with resume/restart/cancel recovery,
    dismiss it on success, supersede or cancel, add a minimal ACE fallback for the
    new action, and run the combined verification.'
proposed_by: bbugyi200.apollo.sase-zr.7.1
create_time: 2026-09-17 06:47:18
status: wip
bead_id: sase-zr.7.1.1
---

- **PROMPT:** [prompts/202609/gate_decision_integrity_1.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/gate_decision_integrity_1.md)
- **BEAD:** [sase-zr.7.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zr/sase-zr.7.1.1.md)

# Gate decision integrity: owned execution, durable failure outcomes, truthful completion

## Context

This epic implements phase `decision-integrity` (bead `sase-zr.7.1`) of epic `sase-zr.7`
(plan `plan:202609/sase_zr_close_out.md`). That phase is too large for one agent: it
changes a shared sase-core contract that has to be released before Python can use it,
then changes the gate executor, the acceptance policy and every requester. Before
starting any phase, read the close-out plan section "Phase decision-integrity" and the
original contract with `sase artifact read plan:202609/prompt_gate_approval.md "<why>"`.
Their constraints still apply:

- Shared policy belongs in sase-core, with no Python fallback.
- `response.json` stays the execution-complete record.
- Completed commands are never silently re-run, and no duplicate coder is launched.
- `%auto` stays synchronous and never spawns a second coder.
- Measure against targets; do not add brittle wall-clock assertions.
- Never touch real user gates or send real notifications or Telegram messages outside
  test sandboxes.

Repositories:

- the current `sase` checkout;
- `sase-core`, opened with `sase repo open sase-core`. Use only the path it prints, and
  read its `AGENTS.md`. release-plz owns versions, so never hand-edit a version. Verify
  with `just check`, never with `cargo test -p sase_core` alone.

Read `lint_and_test` before finishing any sase phase. Read `tui_perf` before touching
anything under `src/sase/ace/` beyond the one-entry dispatch fallback in
`failure_surfacing`.

Sibling phases of `sase-zr.7` that depend on this epic:

- `approval-projection` (`sase-zr.7.2`) will derive TALE/EPIC APPROVED from the receipt
  and surface the failed outcome as a status. It consumes this epic's lifecycle
  dispositions.
- `approval-projection` will also move epic-launch preparation out of
  `apply_side_effects` ordering. The side-effects resume added here must stay correct
  when that happens.
- `ace-fast-refresh` (`sase-zr.7.3`) will add the ACE resume/restart/cancel modal for
  the failure notification and the plan-gate `partial_attempt` path.

Do not implement their work here.

Also coordinate with:

- `sase-10z`, which tracks existing secret-input leaks, including the stdout/stderr
  copies in `errors/*.json`. The new failure outcome must not widen them, and this epic
  does not fix them.
- `sase-11v` (mobile) and `sase-11w` (Telegram). Surfacing the failure there is
  deferred. The new notification action must degrade to "unsupported" on those surfaces,
  not break them.

### Verified current behavior (sase master `4f5717828b`, sase-core `575f97a`, 2026-09-17)

- `src/sase/notification_gates/decision.py::accept_gate_decision` supersedes a
  conflicting receipt whenever `journal.incomplete_attempt(...)` is not `None`, and a
  running attempt is also incomplete. `execution_owner` is written only when
  `SASE_PROC_ID` is set, and nothing reads it.
- `src/sase/notification_gates/executor.py::execute_gate_selection` runs these steps in
  order: journal `attempt_completed`, then `adapter.prepare_terminal_response` (plan
  archive), then write `response.json`, then `apply_side_effects`. A terminal-prepare
  failure therefore leaves an attempt that looks complete. Failures only write
  `errors/*.json` through `command_runner.record_execution_error`.
- `cancel_gate` refuses whenever `decision_receipt.json` exists.
- `poller.poll_gate` ignores the receipt. `wait_for_gate` and
  `agent/launch_request_response._wait_for_terminal_gate` re-raise `already_answered`
  when their deadline passes with only a receipt on disk.
- `gate_shell/lifecycle.py` and sase-core `decide_gate_lifecycle` rank a valid receipt
  above a cancellation.
- `gate_shell/reclaim.py` already defers `accepted_unfinished` without error. The chop
  logs it at info level. Keep that.
- `record_execution_error` returns nothing and records no stage or attempt.
- `executor.py` is 663 lines, and toobig reports FYI above 700 and a warning above 850.
  Move attempt and failure logic into new modules rather than growing it.
- Detached answers run under the proc supervisor with `SASE_PROC_ID` set
  (`procs/supervisor.py::_child_environment`).
- Liveness helpers already exist:
  - `sase.core.process_identity`: `process_identity_token`, `process_identity_matches`,
    `identity_from_previous_boot`;
  - `sase.ace.hooks.processes.is_process_running`;
  - `sase.procs.store.get_proc`;
  - `sase.procs.identity.supervisor_is_alive`.
- `durability.file_lock` uses `fcntl.flock`. flock locks belong to the open file
  description, so a separate non-blocking probe detects a holder even inside the same
  process.
- Notification `upsert_notification` dedups on `(sender, dedup_key)`. A +1 does not
  undismiss the row. `mark_dismissed` / `mark_many_dismissed` dismiss rows by id.
- Only gate-shell refs work with `sase gate cancel`.
- CI builds against `sase-core-revision.txt`. The `sase-core-rs` floor lives in
  `pyproject.toml` (currently `>=0.34.37,<0.35.0`), and `just install` builds from the
  opened sase-core checkout when it is ahead of that floor.

## Design decisions (apply to every phase)

1. **Acceptance id.** Every minted receipt (fresh or superseding) carries a host-minted
   `acceptance_id` (uuid4 hex).
   - Every journal lifecycle event for that decision carries the same id.
   - A failure is "current" only for events scoped to the current receipt's
     `acceptance_id`, so a stale failure from a superseded decision can never make a
     live decision supersedable.
   - Legacy receipts have no `acceptance_id`. They are matched against legacy events
     that have no `acceptance_id`.
2. **Current failure rule.** A pre-response failure is current when the last lifecycle
   event for the current acceptance is a failure.
   - Lifecycle events: `attempt_started`, `attempt_resumed`, `attempt_failed`,
     `attempt_completed`, `attempt_superseded`.
   - For legacy acceptances only, an `option_failed` with no later start/resume also
     counts as a failure.
   - A post-response failure (`side_effects`, `follow_up`) is current when no later
     `stage_completed` for that stage exists.
   - The journal reducer lives in Python beside the existing `incomplete_attempt`. It is
     fact collection, like `collect_gate_lifecycle_facts`. The decisions that use it
     live in Rust.
3. **Liveness is decided in Rust from raw facts that Python collects.**
   - Python collects:
     - whether `.response.lock` is currently held (non-blocking probe);
     - whether the owner's host matches;
     - whether the owner pid is running;
     - whether its identity token matches, or is unverifiable;
     - whether the owner is from a previous boot;
     - for legacy string `execution_owner` proc ids: the proc status and whether its
       supervisor is alive.
   - Rust rules, in order:
     1. Lock held means live.
     2. For a recorded pid owner:
        - host mismatch → unknown;
        - previous boot, not running, or definite identity mismatch → dead;
        - otherwise → live.
     3. For a legacy proc id only:
        - missing or terminal proc → dead;
        - active proc with a live supervisor → live;
        - active proc with a dead supervisor → dead.
     4. For no owner at all (legacy only): dead, because the executor holds
        `.response.lock` for its entire run.
   - Unknown is treated as live.
4. **Permission rule (Rust).** Supersede and cancel are permitted only when a
   pre-response failure is current or the owner is dead. A live or unknown owner means a
   conflicting answer fails promptly with `gate_decision_conflict`, and cancel keeps
   raising `already_answered`. An identical resubmission always replays.
5. **Claim before executing.** While holding `.response.lock`, the executor takes
   `.acceptance.lock` (bounded, 5 s) before appending any attempt event. It then:
   - verifies the on-disk receipt still has its `acceptance_id`, and otherwise aborts
     with `gate_decision_conflict` ("superseded while waiting") without running
     anything;
   - appends `attempt_started` / `attempt_resumed` / `attempt_superseded`;
   - re-owns the receipt through the Rust `claim_gate_decision_execution`.

   Attempt planning (`partial_attempt`, `no_partial_attempt`) runs before the claim and
   writes nothing, so a negotiation error never clears a failure or re-owns a receipt.
   Lock order is always `.response.lock` → `.acceptance.lock`. Code holding the
   acceptance lock only probes the response lock non-blockingly.

6. **Cancellation outranks a valid receipt.** A cancellation can coexist with a receipt
   only after a permitted post-failure or dead-owner cancel, so lifecycle precedence
   becomes:
   1. response;
   2. unreadable or mismatched receipt (error);
   3. cancellation;
   4. receipt (`accepted_failed` / `accepted_owner_lost` / `accepted_unfinished`);
   5. deadlines.
7. **Failure outcome record.**
   - The record is an `attempt_failed` journal event plus the existing `errors/*.json`
     record, with no parallel store.
   - It carries: `outcome_id` (uuid4 hex), `acceptance_id`, `attempt_id` (empty string
     for a failure after acceptance but before any attempt opened), `stage` (`command` |
     `terminal_prepare` | `side_effects` | `follow_up`), `code`, `message`, `at_unix`,
     and the relative `error_record` path.
   - `message` is bounded to 1000 characters and scrubbed of submitted secret-typed
     string values.
   - For `command_failed` and `invalid_command_output`, the message is a fixed summary
     (option id plus exit status). It never includes stdout or stderr.
   - Owner death is recorded with code `execution_owner_lost` and the stage of the last
     `stage_started` (default `command`).
   - `BaseException` is recorded with code `execution_interrupted` and then re-raised.
   - These codes are never recorded as failures: `partial_attempt`,
     `no_partial_attempt`, `gate_decision_conflict`, `gate_cancelled`,
     `already_answered`, and a `lock_timeout` raised by the claim.
8. **Recovery semantics.**
   - Pre-response failures (`command`, `terminal_prepare`) offer resume, restart and
     cancel. Resume replays completed option results and retries only what failed; for
     `terminal_prepare` that means only the archive/terminal preparation. Restart runs a
     fresh attempt. Cancel is permitted by rule 4.
   - Post-response failures offer resume only:
     - `side_effects`: re-run `apply_side_effects` under the response lock. Skip any
       launch whose id `response.json` already records (`task_launch_task_id`,
       `epic_launch_monitor_id`, `epic_launch_task_id`).
     - `follow_up`: the existing `_resume_answered_shell` settle with `resume=True`.
   - `%auto` is unchanged. It has no notification id, so it publishes no failure
     notification, and its creator still removes the bundle on failure.
9. **No feature flag.** These are correctness fixes that leave master consistent after
   every phase. Legacy receipts and journals stay readable through compatibility
   parsing, not a selectable old branch.
10. **Epic symbols.** A phase that adds a public symbol consumed only by a later phase
    whitelists it with `--epic-symbol` keyed to a later phase or this epic's bead. The
    last phase must leave `sase bead epic-symbols` empty for every phase it closes.

## Phase core_execution_policy (sase-core)

Work in `crates/sase_core/src/gate_decision/{wire,policy,tests}.rs` and the binding in
`crates/sase_core_py/src/lib.rs`. Keep schema version 1: all fields are additive and
optional, and `deny_unknown_fields` already makes an old binding reject a new request
loudly.

### Wire changes

- `GateExecutionOwnerWire { proc_id: Option<String>, host: String, pid: u32, process_identity: String }`
  - `process_identity` may be empty when unverifiable.
- `GateDecisionReceiptWire` gains:
  - `acceptance_id: Option<String>`;
  - `owner: Option<GateExecutionOwnerWire>`.

  Keep the legacy `execution_owner: Option<String>` readable. New receipts leave it
  unset.

- `GateDecisionAcceptanceRequestWire` gains:
  - `acceptance_id: Option<String>`, required when minting;
  - `owner: Option<GateExecutionOwnerWire>`;
  - `existing_execution: Option<GateExecutionFactsWire>`.

  Keep accepting `execution_owner`.

- `GateExecutionFailureWire { outcome_id, acceptance_id: Option<String>, attempt_id, stage: GateExecutionStageWire, code, message, at_unix, error_record: Option<String> }`
  - The stage enum is snake_case: `command`, `terminal_prepare`, `side_effects`,
    `follow_up`.
  - Reject an empty `code` or `outcome_id`.
- `GateExecutionFactsWire` has:
  - `response_lock_held: bool`;
  - `owner_host_matches: Option<bool>`;
  - `owner_process_running: Option<bool>`;
  - `owner_identity_matches: Option<bool>`;
  - `owner_from_previous_boot: bool`;
  - `legacy_proc_status: Option<String>`
    (`pending|running|settling|success|error|killed|missing`);
  - `legacy_proc_supervisor_alive: Option<bool>`;
  - `current_failure: Option<GateExecutionFailureWire>`, for pre-response failures;
  - `post_response_failure: Option<GateExecutionFailureWire>`.

### Policy changes

- Add a pure `execution_owner_liveness(receipt, facts) -> live|dead|unknown`
  implementing design decision 3. Expose the result in outputs as a snake_case string.
- `decide_gate_decision_acceptance`:
  - An identical fingerprint returns `Replayed`, as today.
  - A different fingerprint with `current_failure` present, or a dead owner, returns a
    new status `Superseded`. The outcome's new receipt carries the request's
    `acceptance_id` and `owner`. The outcome also includes:
    - `superseded_receipt`: the old receipt;
    - `owner_lost: bool`, which tells the host to record an `execution_owner_lost`
      outcome for the old acceptance first.
  - Otherwise return `Err(gate_decision_conflict)`. The message states whether the owner
    is live or unknown, plus the owner pid, host and proc id when known.
  - A missing `existing_execution` counts as live, which keeps today's strict conflict.
  - Minting (`Accepted` or `Superseded`) without an `acceptance_id` is
    `invalid_gate_decision_request`.
- `GateLifecycleRequestWire` gains `execution: Option<GateExecutionFactsWire>`.
  `decide_gate_lifecycle`:
  - applies the precedence in design decision 6;
  - classifies a valid receipt as `accepted_failed` (current failure), then
    `accepted_owner_lost` (dead owner), and otherwise `accepted_unfinished` (including
    when `execution` is absent);
  - adds these fields to `GateLifecycleDecisionWire`:
    - `cancel_permitted: bool`: true for `pending`, `expired_review`, `expired_grace`,
      `accepted_failed`, `accepted_owner_lost`;
    - `supersede_permitted: bool`: true for `accepted_failed`, `accepted_owner_lost`;
    - `owner_liveness: Option<String>`;
    - `failure: Option<GateExecutionFailureWire>`: echoes `current_failure` for
      `accepted_failed`, and `post_response_failure` for `answered`.
  - Add the `accepted_failed` and `accepted_owner_lost` disposition constants.
- New
  `claim_gate_decision_execution(request { schema_version, receipt, acceptance_id, owner }) -> receipt`:
  - It errors with `gate_decision_conflict` when the receipt's `acceptance_id` differs.
    A legacy receipt without an id matches only when the request's `acceptance_id` is
    absent.
  - Otherwise it returns the receipt with `owner` replaced. The identity fingerprint is
    unchanged.
  - Add the `_from_json` variant and the PyO3 function `claim_gate_decision_execution`.
    Update the module doc list near the other gate-decision entries.

### Tests

Extend `gate_decision/tests.rs`:

- each liveness rule;
- conflict while live or unknown;
- supersede after a failure and after a dead owner, including the `owner_lost` flag;
- replay while failed;
- legacy receipts (string `execution_owner`, no `acceptance_id`) round-tripping and
  classifying;
- cancellation outranking a valid receipt;
- permission flags for every disposition;
- the failure echo for `answered`;
- claim with a matching id, a mismatched id and a legacy receipt;
- invalid failure wires.

Extend `gate_decision_bindings_round_trip_json_shapes` (and its function-list assertion)
for the new fields and for `claim_gate_decision_execution`.

### Landing and handoff

- Run `just check` in sase-core, which includes the binding tests.
- Commit with a `feat(gate-decision): ...` Conventional Commit so release-plz publishes
  it. Do not edit versions.
- Record the landed sase-core commit sha and, if already published, the release version
  in a bead note for `failure_journal`.

## Phase failure_journal (sase)

### Adopt the new core

- Run `sase repo open sase-core` so the checkout includes `core_execution_policy`.
- Ratchet `sase-core-revision.txt` to a sase-core commit that contains it.
- Raise the `sase-core-rs` floor in `pyproject.toml` only if a published release
  contains it; check the package index. Otherwise leave the floor to the release-branch
  reconciler and say so in the bead note.
- Run `just install` and confirm the workspace binding exposes
  `claim_gate_decision_execution`.
- Add thin facade functions and constants in `src/sase/core/gate_decision_facade.py`:
  - `claim_gate_decision_execution`;
  - the new disposition constants and `ACCEPTED_FAILED`/`ACCEPTED_OWNER_LOST` mirrors in
    `gate_shell/lifecycle.py`.

### Acceptance id

In `decision.py`, mint `acceptance_id` (uuid4 hex) for every request that may mint a
receipt. Keep the existing `incomplete_attempt` supersede rule for now, because
`owner_conflict` replaces it, but make the superseding request carry a fresh id.

### Journal (`notification_gates/journal.py`)

- Extend `append_journal_event` with `acceptance_id`, `stage`, `message`, `outcome_id`
  and `error_record` fields. It must still never write raw input.
- Add the new events:
  - `attempt_resumed`;
  - `stage_started` / `stage_completed` for `terminal_prepare`, `side_effects`,
    `follow_up`;
  - `attempt_failed`.
- Treat `attempt_completed` as terminal in `incomplete_attempt` only when
  `response.json` exists, because legacy bundles journaled it before a failed archive.
  That keeps them resumable.
- Give `IncompleteAttempt` `acceptance_id` and `failed_stage`. Make `describe()` say
  "all options completed; terminal preparation failed" when that applies.
- Add `current_execution_failure(bundle_path, receipt, *, response_exists)`, which
  returns the pre-response failure (design decision 2) or `None`.
- Add `current_post_response_failure(...)`.
- Both return a small frozen dataclass with `to_wire()` that matches
  `GateExecutionFailureWire`.

### Failure recording (new `notification_gates/failure_outcome.py`)

- Add
  `record_failure_outcome(bundle_path, *, acceptance_id, attempt_id, stage, error, selected, resolved_inputs, source) -> ExecutionFailure`,
  implementing design decision 7.
- Extend `command_runner.record_execution_error`:
  - add optional `stage`, `attempt_id` and `outcome_id` fields;
  - return the written path, which is stored relative to the bundle as `error_record`.
- Add a public scrub helper in `executor_inputs.py` that wraps
  `_submitted_secret_strings` / `_scrub`.
- Leave a single `on_recorded` seam where `failure_surfacing` will publish the
  notification. Keep it a plain module-level call with no registry.

### Executor

Move attempt planning, opening and resume logic into a new
`notification_gates/attempts.py` so `executor.py` stays under 700 lines. Then:

- Keep the receipt returned by `accept_gate_decision`.
- Wrap everything under `.response.lock` after acceptance in a stage tracker that
  records a failure outcome for the current stage and re-raises. Pre-attempt
  revalidation failures use stage `command` and `attempt_id=""`.
- Split `_begin_attempt` into a pure plan step (which raises the negotiation errors) and
  an open step that appends events tagged with `acceptance_id`. Resume appends
  `attempt_resumed`.
- Keep `option_failed` and add `attempt_failed` (`stage=command`).
- Run the stages in this order:
  1. `stage_started(terminal_prepare)`;
  2. `prepare_terminal_response` (on failure, record `terminal_prepare`);
  3. write `response.json`;
  4. `attempt_completed` (moved here);
  5. settle the notification;
  6. `stage_started(side_effects)`;
  7. `apply_side_effects`;
  8. `stage_completed(side_effects)`, or on failure record `side_effects`.
- The `FileExistsError` branch appends `attempt_superseded` for its own attempt before
  returning `already_completed`.
- When `retry == "resume"`, `response.json` exists and a current `side_effects` failure
  exists: re-run only `apply_side_effects`, with the launch-id guard from design
  decision 8. In every other response-exists case, return `already_completed` as today.

### Follow-up stage

- In `notification_gates/cli_answer.py` (inline path) and
  `plan_approval_actions.py::_execute_neutral_plan_approval_response`:
  - wrap `settle_gate_shell` in `stage_started(follow_up)`;
  - on success, record `stage_completed(follow_up)`;
  - when it raises, record a `follow_up` failure and re-raise.
- In `_resume_answered_shell`:
  - when a current `side_effects` failure exists, run
    `execute_gate_selection(..., retry="resume")` before settling;
  - after a successful settle, record `stage_completed(follow_up)`.
- Settlement's own follow-up launch failures stay in `gate_followup_error` metadata.
  Leave them unchanged.

### Tests

New `tests/test_gate_execution_failure_outcomes.py`, plus extensions to
`tests/test_plan_approval_actions_archive.py`,
`tests/test_plan_archive_approval_recovery.py`,
`tests/test_notification_gate_execution.py` and `tests/test_gate_cli_answer.py`. Cover:

- **Archive failure, then resume.** Patch `archive_approved_plan` to raise, and have the
  option command increment a counter file. After the failure:
  - the attempt stays open with every option completed;
  - `attempt_failed(terminal_prepare)` is present and `attempt_completed` is absent;
  - an identical answer raises `partial_attempt`.

  Then resume and check that the counter is unchanged, the archive is retried, and
  `response.json` exists with `attempt_completed` after it.

- **Side-effect failure.**
  - The outcome is recorded after `attempt_completed`.
  - A plain identical answer returns `already_completed`.
  - Resume re-runs only the side effects.
  - A recorded launch id is not relaunched.
- **Legacy early `attempt_completed`** with no response: the attempt is resumable.
- **Command failure.** The outcome has `stage=command` and the fixed message, and
  neither the journal nor the outcome contains a secret-typed input value that the
  command echoed.
- **Pre-attempt revalidation failure** (monkeypatched) is recorded.
- **`KeyboardInterrupt`** is recorded as `execution_interrupted` and re-raised.
- **Negotiation errors** record nothing.
- **`current_execution_failure` scoping.** Include a superseded acceptance whose stale
  failure is ignored.
- **`follow_up`** failure is recorded and cleared by a successful shell resume.

Run `just fix`, then `just check`.

## Phase owner_conflict (sase)

### Owner identity (new `notification_gates/execution_owner.py`)

- `current_execution_owner()` returns `proc_id` from `SASE_PROC_ID` (omitted when
  unset), `host` from `socket.gethostname()`, `pid` from `os.getpid()`, and
  `process_identity` from `process_identity_token`.
- `collect_execution_facts(bundle_path, receipt, *, response_exists)` builds
  `GateExecutionFactsWire` (design decision 3):
  - Probe `.response.lock` with a new `durability.lock_is_held(path)`. It takes
    `LOCK_EX|LOCK_NB` on a separate descriptor, releases immediately, and never blocks.
  - Check the owner pid only for a same-host owner.
  - Read the proc store only for a legacy string `execution_owner`.
  - Attach the `failure_journal` reducers' failures.

### Acceptance (`decision.py`)

- Record `owner` on every mint.
- Send `existing_execution` with any existing receipt.
- Delete the Python `incomplete_attempt` supersede rule.
- Handle `Superseded` under the acceptance lock, in this order:
  1. if `owner_lost`, record an `execution_owner_lost` outcome for the old acceptance;
  2. append a `decision_superseded` journal event (old and new fingerprints and
     acceptance ids);
  3. append `attempt_superseded` for the old open attempt;
  4. write the receipt non-exclusively.
- Surface the conflict message unchanged, as a `GateError`.

### Executor claim

Implement design decision 5 in `attempts.py`, using `claim_gate_decision_execution` and
the bounded acceptance lock.

### `cancel_gate`

Under the acceptance lock, when a receipt exists:

1. Collect the facts and classify.
2. If `cancel_permitted`:
   1. record `execution_owner_lost` if the owner is dead and no failure is current;
   2. append `attempt_superseded` (code `cancelled`) for the open attempt;
   3. write `cancellation.json`;
   4. settle the notification as cancelled.
3. Otherwise raise `already_answered`, with a message that says the decision is still
   executing.

### Lifecycle, reclaim, show and cancel CLI

- `gate_shell/lifecycle.py`:
  - pass execution facts into the classifier;
  - export the two new dispositions;
  - make `resolve_already_answered_race` treat them like `accepted_unfinished`, since
    `cancel_gate` only refuses while the owner is live.
- `gate_shell/reclaim.py`:
  - count the new dispositions in a new `accepted_failed` summary field;
  - never settle, cancel or error for them;
  - make the chop script log them at info level;
  - keep every repeated pass error-free.
- `cli_show._acceptance_payload`: report every `accepted_*` disposition with:
  - `disposition`;
  - owner (`proc_id`, `host`, `pid`);
  - `owner_liveness`;
  - the failure outcome, when present.
- `gate_shell_handler.handle_gate_shell_cancel`: when a live accepted gate cannot be
  cancelled, print "decision accepted and still executing; nothing cancelled". Keep the
  exit codes.

### Tests

Extend `tests/test_gate_decision_acceptance.py`, `tests/gate_shell/test_reclaim.py`,
`tests/main/test_gate_shell_handler_cancel.py`, `tests/test_gate_cli_show.py` and
`tests/test_notification_gate_durability.py`. Add
`tests/test_gate_execution_ownership.py`. Cover:

- **Conflict during execution.** While the first command is parked on the existing
  `_BLOCKING_COMMAND` barrier, a different answer raises `gate_decision_conflict` and
  runs no command. After release, the receipt and `response.json` agree.
- **New answer after a recorded failure** is accepted and executes the new branch, and
  the journal shows `decision_superseded`.
- **Cancel after a failure** succeeds, and the gate shell settles `stopped`.
- **Owner death during execution.** Run `execute_gate_selection` in a subprocess and
  SIGKILL it after the command starts. Then:
  - `lock_is_held` is false;
  - the lifecycle is `accepted_owner_lost`;
  - a different answer supersedes, with an `execution_owner_lost` outcome recorded;
  - cancel is permitted.
- **Owner death after `response.json`** stays `answered`.
- **Identical waiter.** An identical waiter blocked on `.response.lock` aborts with a
  conflict when the receipt is superseded while it waits.
- **Negotiation errors** never re-own the receipt.
- **Legacy receipts:**
  - a string proc id whose proc is terminal counts as dead;
  - no owner with the lock free counts as dead;
  - no owner with the lock held counts as live.
- **Same-process holder.** `lock_is_held` detects a holder in the same process.
- **Reclaim** passes over failed and live gates report no errors.

Run `just fix`, then `just check`.

## Phase failure_surfacing (sase)

### Poller (`notification_gates/poller.py`)

- `GatePollStatus` gains `"failed"`, and `GatePollResult` gains `failure`.
- `poll_gate` returns, in order:
  1. `responded`, plus `failure` when a post-response failure is current;
  2. `cancelled` / `timed_out`;
  3. for a receipt, the result of a new `observe_execution_failure(bundle_path)` helper:
     - `accepted_failed` returns `failed`, with a payload of the outcome plus a receipt
       summary;
     - `accepted_owner_lost` records the `execution_owner_lost` outcome under a bounded
       acceptance lock, re-checking under the lock, and then returns `failed`. On
       `lock_timeout` it still returns `failed` without recording;
  4. otherwise `None`.
- Add one shared deadline helper used by `wait_for_gate` and
  `agent/launch_request_response._wait_for_terminal_gate`:
  - `failed` returns immediately.
  - When the deadline cancel is refused because a live owner holds an accepted decision,
    keep polling until `responded` or `failed`. The review deadline no longer applies,
    and failure recording plus dead-owner detection guarantee an outcome. Never re-raise
    `already_answered` there.
  - A requester-initiated cancel (`cancelled()` callback) that is refused still raises,
    as today.

### Requesters

- `notifications/cli_wait.py`:
  - status `failed` exits with code 5;
  - the JSON projection gains `failure`, including the resume/restart commands;
  - the human summary shows the stage, code and message;
  - update the `sase gate wait` epilog in `main/parser_gate.py`, and any doc that lists
    its exit codes.
- `launch_request_response`: add a `failed` `LaunchRequestStatus` carrying the failure
  message.
- `xprompt/workflow_hitl_gate.py`: `failed` becomes a reject whose message names the
  failure. It is never an approval.
- `scripts/_bead_task_triage_gates.py::gate_state`: `failed` is still open, so it never
  creates a duplicate gate.
- `cli_show`: project `failed`.

### Failure notification (new `notification_gates/failure_notification.py`)

Called from the `failure_journal` seam after every recorded outcome, and best-effort: it
logs and never raises into an error path.

- Publish only when the envelope has a `notification_id`.
- Build a `Notification` with:
  - `id = uuid5(<module namespace>, f"{gate_id}:{outcome_id}")`;
  - `sender="gate"`;
  - `action="GateExecutionFailed"`;
  - `dedup_key=f"gate-execution-failed:{gate_id}:{attempt_id}:{outcome_id}"`;
  - tags `gate`, `execution`, `error`.
- `action_data` (strings only):
  - `request_id`, `request_kind`, `bundle_path`;
  - `attempt_id`, `outcome_id`, `stage`, `code`;
  - `recovery_actions`: `resume,restart,cancel` pre-response, `resume` post-response;
  - `error_report_path`;
  - the gate-shell ref, when the gate is shell-backed.
- Notes:
  - a redacted summary;
  - the exact `sase gate answer --id <id> --kind <kind> --resume` and `--restart`
    commands;
  - `sase gate cancel <shell ref>`, only for shell-backed gates.
- Publish with `upsert_notification`. Before publishing, dismiss older failure rows for
  the same gate (ids derived from the journal's outcome ids) so at most one failure row
  per gate is visible.
- Add `dismiss_execution_failure_notifications(bundle_path)`. Call it:
  - after a successful `response.json` publication;
  - after `stage_completed` for a post-response stage;
  - when an attempt is claimed for resume or restart;
  - on supersede;
  - on a permitted cancel.
- `GateExecutionFailed` must not be a privileged gate action. `pending_actions` must
  register nothing for it, so mobile and Telegram stay unsupported (deferred to
  `sase-11v`/`sase-11w`).

### Reclaim

For `accepted_owner_lost`, call `observe_execution_failure`. A killed detached answer
then gets one durable outcome and one notification even when nothing polls. This reuses
the poll-path helper and adds no reconciliation machinery.

### Minimal ACE fallback

Map `GateExecutionFailed` in `ace/tui/actions/agents/_notification_modal_flow.py` to the
existing `handle_view_error_report` flow (it opens `error_report_path`). Add badge and
icon entries in `ace/tui/modals/notification_modal_constants.py`, so the row is
actionable rather than "Unsupported". The resume/restart/cancel modal belongs to
`ace-fast-refresh`. Leave a docstring pointer there.

### Tests

Extend `tests/test_notification_gates.py`, `tests/test_gate_wait_cli.py`,
`tests/test_plan_approval_launch_reliability_integration.py`,
`tests/gate_shell/test_reclaim.py` and the launch-request and workflow-HITL tests. Add
`tests/test_gate_execution_failure_notifications.py`. Use the `gate_home` fixture; never
use the real notification store. Cover:

- `poll_gate` failed results for a recorded failure and for a dead owner (recorded once
  across repeated polls);
- `wait_for_gate` and the launch waiter at their deadline with a live owner: they keep
  waiting, then return `responded` once the barrier releases, or `failed` when the
  command fails;
- `sase gate wait` exit code 5 and its JSON;
- the HITL reject on failure;
- dedup:
  - the same outcome published twice yields one row;
  - a resumed-then-failed-again attempt shows exactly one visible row;
- dismissal after a successful retry, a supersede and a cancel;
- no notification for `%auto` or for notification-less gates;
- the notes and `action_data` contain no secret input values;
- the ACE dispatch opens the error record for the new action;
- reclaim records owner-lost exactly once.

### Verification

1. Run `just fix` and `just check`.
2. Run `just check-full` through `/sase_monitor` (`TESTING`/`TESTED`) as the combined
   gate for this epic, and record any unrelated failures with evidence.
3. Resolve every remaining `sase bead epic-symbols` entry.

## Landing note for this epic's land agent

- Re-verify the goal end to end against the close-out plan's "Phase decision-integrity"
  requirements and tests list.
- Confirm that:
  - the sase-core change is released, or the CI revision pin covers it, with the floor
    rule respected;
  - `sase bead epic-symbols` is empty.
- Then close `sase-zr.7.1` with a note summarizing what was verified.
- Do not close `sase-zr.7` or `sase-zr`.
- Carry forward any `PROPOSED FOLLOW-UP` notes from this epic's phases per the usual
  triage.
