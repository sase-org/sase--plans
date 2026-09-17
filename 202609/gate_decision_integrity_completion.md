---
tier: epic
title: Complete gate decision integrity after landing audit
goal: 'Finish the gate-decision integrity contract left incomplete by sase-zr.7.1.1
  so every accepted execution has one verifiable owner, every failure and terminal
  transition is durably receipt-scoped, and every requester receives actionable, deduplicated
  recovery information without duplicate execution.

  '
phases:
- id: core-contract-completion
  title: Complete and validate the shared gate-decision policy contract
  depends_on: []
  size: medium
  description: 'core-contract-completion: extend the Rust wire and policy contract
    with superseded receipt, owner-loss, liveness, and failure evidence; reject malformed
    acceptance and failure facts; expose the completed contract through PyO3; and
    publish a core release that downstream Python can adopt.'
- id: terminal-transition-integrity
  title: Serialize and journal every terminal ownership transition
  depends_on:
  - core-contract-completion
  size: medium
  description: 'terminal-transition-integrity: adopt the completed core contract,
    put supersede, cancel, owner-loss, poll, reclaim, resume, and restart decisions
    behind one bounded acceptance lock with a final recheck, and journal each owner_lost,
    decision_superseded, and attempt_superseded transition exactly once while restoring
    drift-deleted regression coverage.'
- id: failure-recovery-surface
  title: Finish the requester and recovery-notification contract
  depends_on:
  - terminal-transition-integrity
  size: medium
  description: 'failure-recovery-surface: publish deterministic actionable execution-failure
    notifications, route them through ACE and every waiting requester, preserve them
    until recovery is actually claimed or succeeds, document the failed status and
    exit behavior, and close the remaining end-to-end and plan-gate recovery test
    gaps.'
proposed_by: bbugyi200.apollo.sase-zr.7.1.1.land
parent_bead: sase-zr.7.1.1
create_time: 2026-09-17 19:54:34
status: wip
bead_id: sase-zr.7.1.1.5
---

- **PROMPT:** [prompts/202609/gate_decision_integrity_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/gate_decision_integrity_completion.md)
- **BEAD:** [sase-zr.7.1.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zr/sase-zr.7.1.1.5.md)

# Complete Gate Decision Integrity After Landing Audit

## Why this follow-on exists

The land audit for parent epic `sase-zr.7.1.1` reviewed its linked plan, every child and
note, all four implementation commits, the corresponding `sase-core` commit, the current
source, and non-epic mainline drift since the work began. The existing phases landed
useful foundations, but several explicit requirements from
`plan:202609/gate_decision_integrity_1.md` are absent. The parent epic therefore cannot
close truthfully yet.

This plan contains only the unfinished implementation and regression work. It does not
repeat the parent epic landing, symbol cleanup, close, or plan-status steps; the parent
link returns control to its land agent after this child lands.

## Verified starting point — preserve, do not redo

- Main commits `934be032`, `26797a95`, and `1d14218a` already provide the receipt
  journal, execution-owner helpers, live-owner conflict checks, failed poll outcome, and
  initial failure notification path.
- Core commit `b4c3ca63` is already an ancestor of the currently pinned core revision.
  Extend the current `sase-core` head and publish a normal release; never rewind the
  dependency pin to the historical epic commit.
- `response.json` remains the completion point, `attempt_completed` remains
  post-publication only, redaction remains mandatory, and a failed accepted decision
  must never trigger a second option-command launch implicitly.
- `sase bead epic-symbols sase-zr.7.1.1` is already empty. Do not introduce a new
  whitelist exemption for this work when wiring, privatizing, or deleting the symbol is
  possible.
- External mobile and Telegram recovery UI remains outside this scope. The durable
  notification and action payload must be complete enough for those clients to adopt
  later without changing gate semantics.

## Cross-phase invariants

1. `acceptance_id` is the stable attempt identity. Failure, ownership, supersession,
   cancellation, notification, and recovery facts must be scoped to it and must not leak
   across a later receipt.
2. All shared lifecycle decisions belong in `sase-core`; Python is a thin persistence,
   locking, CLI, and presentation adapter. There is no Python policy fallback.
3. A mutation that depends on owner liveness must acquire the bounded acceptance lock,
   reread receipt and journal state, evaluate current liveness once, persist the
   transition, and only then release the lock. Timeout is an explicit conflict, never an
   unlocked best effort.
4. Recovery notification identity is deterministic per gate and acceptance attempt.
   Replays update or reuse the same notification, and a new acceptance does not inherit
   a stale failure from the old receipt.
5. Tests use fake gates, fake notification stores, and controlled processes. They do not
   create real user gates, deliver real notifications, or depend on wall-clock sleeps
   when an observable synchronization point can be used.

## Phase 1 — `core-contract-completion`

Complete the domain contract in the linked `sase-core` repository and expose it through
the existing binding. The Rust types and policy functions are authoritative for every
frontend.

- Extend `GateDecisionAcceptanceOutcomeWire` so a supersession result carries the prior
  `superseded_receipt` and the `owner_lost` fact used to justify takeover.
- Extend lifecycle input/output with the evidence required by the original plan: current
  and post-response execution failure, owner liveness, failure echo, and the resulting
  cancel/supersede permissions. Do not make Python reconstruct these decisions from
  booleans.
- Require a nonempty `acceptance_id` wherever an accepted receipt or receipt-scoped
  execution fact is evaluated. Validate failure code/outcome fields and reject empty or
  structurally inconsistent payloads instead of treating them as valid facts.
- Make live-owner conflicts carry enough structured owner and liveness evidence for
  CLI/requester diagnostics while keeping secrets and raw command output out of wire
  errors.
- Update Rust unit tests, JSON compatibility fixtures, PyO3 exports, type stubs, and
  Python binding tests. Include missing-ID, malformed-failure, live/dead/unknown owner,
  post-response failure, supersession receipt, and old-receipt isolation cases.
- Run the core repository checks, publish the normal core release, and record the
  released version/commit on the phase bead for the next phase. The downstream phase
  must consume that release rather than relying on an unshipped local checkout.

## Phase 2 — `terminal-transition-integrity`

Adopt the released core contract in the main repository and make every ownership or
terminal transition atomic, durable, and idempotent.

- Ratchet the core revision and package floor from the current mainline state, then
  thread the richer acceptance/lifecycle outcome through the facade and CLI-show
  projection. Acceptance output must include owner identity and evaluated liveness.
- Move the response-lock probe to the shared durability/locking layer if needed so
  decision, executor, poll, cancel, and recovery paths use one implementation.
- For supersede, cancellation after failure/dead owner, poll-detected owner loss,
  reclaim, resume, and restart: acquire the bounded acceptance lock; reread the current
  receipt and journal; re-evaluate owner liveness; and append the required `owner_lost`,
  `decision_superseded`, and `attempt_superseded` facts exactly once. Never append a
  supersession fact for a refused live-owner request.
- Ensure concurrent pollers or recovery claimants cannot duplicate owner-loss facts,
  cannot both claim execution, and cannot operate on a receipt that changed while they
  waited for the lock. Lock timeout and unknown-liveness cases must remain explicit
  non-mutating conflicts.
- Preserve the post-response distinction: a response can remain the terminal answer
  while a later archive, side-effect, or follow-up failure remains a current recovery
  obligation. Plain replay must not dismiss that failure merely because `response.json`
  exists.
- Dismiss or supersede the current failure notification only when the matching recovery
  attempt is claimed or the failed stage completes successfully. Successful follow-up
  stage tracking must clear the matching current failure without touching a newer
  acceptance.
- Restore the two phase-2 regression tests removed by mainline commit `85d6fc74`:
  terminal failure/recovery reducer assertions and acceptance-scoped
  `current_execution_failure` isolation. Preserve all unrelated record-before-admit
  behavior introduced by that commit.
- Add deterministic race tests for simultaneous owner-loss polls, supersede versus
  completion, cancel versus live owner, stale receipt recovery, bounded-lock timeout,
  and a subprocess owner killed with `SIGKILL`. Assert both returned outcomes and the
  exact durable journal sequence.

## Phase 3 — `failure-recovery-surface`

Finish the user-visible recovery contract across notifications, ACE, command-line
polling, launch approval, workflow HITL, and plan approval.

- Derive the execution-failure notification ID deterministically with UUIDv5 from gate
  and acceptance identity. Use the contract tags `gate`, `execution`, and `error`;
  include the gate-shell reference, redacted failure message, `error_report_path`, and
  exact resume, restart, and cancel commands. Re-publishing the same current failure
  must update/reuse one notification; a later receipt must not collide with it.
- Route `GateExecutionFailed` through the ACE notification action flow so View Error
  Report works from `error_report_path`. If the report is absent or the action payload
  is old/malformed, show a safe actionable fallback instead of an unsupported-action
  warning or traceback.
- Make `sase gate wait`, launch approval, workflow HITL, plan approval, and other
  waiters surface the durable failure message and recovery commands. A requester
  cancellation refused because a live owner still holds the acceptance must be reported
  as a conflict, not silently converted back into polling.
- Keep machine-readable poll/wait output structured and stable. Update the human
  summary, parser epilog/exit-code documentation, notification documentation, and
  configuration documentation for the failed outcome and exit code 5.
- Add exact-contract tests for deterministic ID/tags/actions, redaction, dedupe, receipt
  replacement, dismissal timing, missing-report ACE fallback, and every requester
  message/exit path. Add the originally required generic and plan-specific
  archive/terminal-preparation recovery cases in
  `tests/test_plan_approval_actions_archive.py` and
  `tests/test_plan_archive_approval_recovery.py`; these are parent-epic acceptance
  coverage, not a deferred follow-up.
- Run focused tests during implementation, then the repository governed check. Finish
  with the full check lane through the SASE monitor and preserve unrelated flake
  evidence according to the normal task-triage policy.

## Completion evidence

The child epic is complete only when the released Rust policy and Python callers agree
on the richer wire contract; concurrency tests prove unique ownership and journal
transitions; failure notifications are deterministic, actionable, and lifecycle correct;
every requester exposes the same failed outcome; the drift-deleted and plan-gate
regressions are restored; and both repository check suites pass (apart from separately
triaged unrelated flakes). Record the exact core release, main commits, focused test
commands, governed checks, and any independent flake records in phase notes so the
parent land agent can re-audit without inference.
