---
tier: tale
title: Preserve accepted gate decisions during cleanup
goal:
  Prevent false expiry and completion of accepted gates and retain actionable AXE error
  diagnostics.
size: medium
proposed_by: bbugyi200.athena.0m0
create_time: 2026-09-16 13:14:05
status: wip
---

# Preserve accepted gate decisions during cleanup

## Problem and evidence

AXE's September 16, 2026 digest at
`~/.sase/axe/error_digests/digest_20260916_130123.txt` reports
`housekeeping/gate_shell_reclaim: reclaim_errors` at 12:21:00 EDT, with only the
placeholder `<no python traceback: subprocess error>`.

The underlying run is `20260916T122011_098018`; its log under
`~/.sase/axe/lumberjacks/housekeeping/chops/gate_shell_reclaim/runs/` says:

> gate shell reclaim failed: 0la.f0--gate: GateError: gate decision is already accepted

The job exited zero and returned structured status `check_error`. The same failure
recurred in run `20260916T130122_471451`.

The affected plan gate is `c7c0bdf3-89c8-4024-b819-27249cc1ffb8`, belonging to
`0la.f0--gate`, for `chezmoi_headless_sudo_guards.md`. Its operational bundle lives at
`~/.sase/interaction_requests/plan/c7c0bdf3-89c8-4024-b819-27249cc1ffb8/`.

1. The gate was created September 15 at 12:20:11 EDT with a 24-hour review timeout.
2. The reviewer accepted `approve` and `commit` at 12:21:01. A durable
   `decision_receipt.json` records this selection.
3. Both option commands finished at 12:21:30, and the journal recorded
   `attempt_completed`. Subsequent terminal preparation failed to archive the plan. The
   bundle's error record reports `plan_archive_failed`, caused by an operational
   workspace materialization failure: the loaded `sase_core_rs` Git object-sharing wire
   was stale, **expected 2, got 1**.
4. `response.json` and `cancellation.json` were absent at investigation time, and
   `sase gate show 0la.f0--gate -j` reported a pending shell holding a workspace claim.
   Acceptance had already dismissed the review notification. A separate `plan-archive`
   failure notification was emitted and is now dismissed.
5. The next day's reclaim pass saw no response, reached the review deadline, and called
   `cancel_gate`. Cancellation correctly refused the accepted decision.

Relevant notifications: AXE `7483e109-dab7-4c79-b282-2a5248cfc185` at September 16
13:01:23 EDT; archive failure `6605bbbb-1707-4b31-abf7-74459f6901b3` at September 15
12:21:30 EDT. Their identities are provenance, not implementation dependencies.

The acceptance split was introduced by `c8152f4978`; cleanup and its callers still
assume that an accepted decision necessarily has a completed response. Isolated, mocked
calls against the current implementation reproduced three consequences:

- During the timeout grace window, reclaim raises the observed error.
- After the grace window, reclaim directly settles the same accepted gate as `lost`.
- `cancel_gate_shell` interprets `already_answered` from an acceptance receipt as a
  completed response and incorrectly settles the shell as `answered`.

A fresh Python process in the investigating checkout reports matching Git object-sharing
schemas, 3/3. This establishes that today's checkout can load a compatible binding; it
does not establish the binding loaded by yesterday's executor or any existing long-lived
process. The initiating version mismatch must remain a meaningful error.

## Intended behavior

An accepted decision whose execution is unfinished remains accepted across review
deadline and grace expiry. Cleanup must neither cancel it, declare it lost because of
that deadline, nor claim it completed. Completed responses and genuine cancellations
remain authoritative terminal evidence. Accepted unfinished execution must be visible
for inspection, including an existing execution error when present.

An acceptance or completion racing with cleanup is a normal state transition, not a
`reclaim_errors` failure. Unexpected I/O, invalid state, or lock failures remain
diagnosable. AXE digests must retain bounded, sanitized output for structured
`check_error` results even when the subprocess exits zero.

## Implementation

### 1. Add a shared receipt-aware lifecycle decision

Open `sase-core` through
`sase repo open sase-core -r "Implement receipt-aware gate reclaim policy"`, and read
that checkout's instructions. Put deterministic lifecycle classification in
`crates/sase_core/src/gate_decision/`, expose it through `crates/sase_core_py`, and add
a thin adapter beside `src/sase/core/gate_decision_facade.py`. Python collects
filesystem facts and performs I/O; Rust decides the disposition. Keep acceptance
identity and its existing receipt format compatible.

The policy must distinguish completed response, cancellation, accepted unfinished
execution, unaccepted pending review, expired review, and expired grace. Receipt
evidence takes precedence over either review deadline. Verify receipt identity against
the gate/request and report malformed or contradictory evidence explicitly rather than
silently treating it as an unanswered gate eligible for cleanup.

Use this policy from `src/sase/gate_shell/reclaim.py` and
`src/sase/gate_shell/cancel.py`. In particular, handling
`GateError.code == "already_answered"` must reread evidence and distinguish a receipt
from an actual response; do not match exception message text. Explicit cancellation of
accepted unfinished execution should return a clear refusal without settlement,
workspace release, or follow-up launch. A concurrently completed response can still
settle as answered.

Serialize the destructive expiry decision with acceptance using the existing short,
bounded `.acceptance.lock`. Both deadline branches, including the current direct `lost`
branch, need this protection. Reuse or factor the lock-owned cancellation transition in
`notification_gates/executor.py`, avoiding recursive acquisition. Persist the chosen
expiry outcome before unlocking so acceptance cannot slip between a check and
settlement, and recovery preserves timeout versus grace-expired loss. Keep slow shell
settlement, archive work, and follow-up launches outside this lock. Preserve normal
cleanup of genuinely unanswered gates and missing bundles.

### 2. Expose accepted unfinished execution without fabricating success

Add an explicit accepted-unfinished count/disposition to reclaim's summary and log
messages; keep it separate from reclaim failures and pass-budget deferrals. The reclaim
pass should inspect and defer such gates, never replay their selected commands.

Extend the existing `sase gate show` human and JSON projections with additive
acceptance/completion details so the affected gate can be recognized without manually
opening its bundle. Reuse the shared Rust classification. Preserve existing terminal
status fields and inspect existing execution error records with bounded reads. Make
clear that a recorded error is historical evidence, not proof that a current retry is
dead. A receipt by itself does not prove whether an executor is alive.

The incident must display as accepted, execution incomplete, with the recorded
`plan_archive_failed` cause. The existing archive failure notification and supported
`sase gate answer ... --resume` path remain the operator's recovery mechanisms. This
plan does not add an automatic replay engine or change the journal's option-command
completion semantics. In particular, `attempt_completed` currently precedes archive
preparation and cannot be treated as proof of a completed gate response.

### 3. Preserve diagnostics for structured AXE check failures

In `src/sase/axe/chop_runner_script_result.py`, the `result_status == "check_error"`
branch currently supplies only the structured reason and `NO_PYTHON_TRACEBACK`. Capture
diagnostics with the existing `capture_chop_subprocess_diagnostic` helper and pass them
to both `finalize_script_chop_run` and `ChopRunOutcome`, including the actual exit code
of zero and output byte count. Reuse the existing Rust sanitization and size limits,
error-store propagation, and digest rendering.

Keep structured counters numeric and the stable reason useful for grouping. The
diagnostic excerpt should preserve the affected gate and actual exception separately
from that reason. It must survive pruning of the original job log. Missing/empty logs
and genuine Python tracebacks retain their existing handling.

### 4. Regression coverage and verification

Add Rust policy and PyO3 boundary tests, plus focused Python behavior tests in the
existing gate and AXE suites:

- A receipt with no response before, during, and after deadline/grace expiry never
  cancels, settles, releases a claim, or launches a successor; repeated passes are
  harmless and account for accepted unfinished work.
- Acceptance and response publication racing with cleanup are handled using fresh
  evidence, including the grace-expired path. Cancellation winning the lock prevents
  later acceptance. Use deterministic synchronization, not timing sleeps.
- Explicit cancel with only a receipt refuses; explicit cancel with a real response
  preserves existing answered-settlement behavior.
- Ordinary pending timeout, grace expiry, terminal cancellation/response, missing
  bundle, malformed receipt, and real lock/I/O failures retain correct behavior.
- Extend `tests/test_plan_gates_execution.py`'s archive-failure case: options complete,
  archive preparation raises, receipt survives, response is absent, and reclaim plus
  gate inspection preserve the accepted incomplete state and original error.
- Extend `tests/test_axe_chop_output_contract.py` for the accepted-unfinished counter.
- Extend `tests/test_axe_chop_subprocess_diagnostics.py` with a zero-exit script that
  writes a structured `check_error` and logs the incident's error. Verify the detail
  reaches the run record, AXE error store, and digest after log pruning; redaction and
  bounded capture still apply.

Run the core repository's `just check`, including its PyO3 tests. Build/install the
matching binding for Python integration as required by repository instructions. Run the
focused Python suites and SASE's `just check`; use `just check-full` through
`sase_monitor` when the repository's landing/broadening rules require it. Before an
implementation that adds CLI options or changes TUI performance, consult the respective
reference memories; this design needs neither new CLI options nor TUI refresh work.

## Operational follow-through and acceptance criteria

Keep this investigation and implementation separate from executing the old plan's
approved commands. After deployment, inspect the affected gate again and check the
binding versions in the process that would perform recovery. If its accepted execution
is still incomplete, present the supported resume command for that same gate and
selection; preserve its original inputs and review intent. Do not hand-edit receipt,
response, cancellation, notification, or claim files to repair it. Existing gates
already terminalized by the old cleanup require review of their evidence before any
repair; the fix must not reopen arbitrary terminal gates.

Acceptance criteria: the three reproduced lifecycle errors are covered and corrected;
the archive failure remains visible without a false timeout/loss/completion; and a
future structured AXE error digest contains the underlying exception and source run
instead of only `reclaim_errors` and the traceback placeholder.
