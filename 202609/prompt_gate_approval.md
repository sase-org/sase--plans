---
tier: epic
title: Make gate approval and notification dismissal respond promptly
goal:
  Publish durable tale and epic approval decisions and refresh ACE and Telegram promptly
  without waiting for archive publication or successor launch, while preserving
  execution recovery and exactly-once follow-up ownership.
phases:
  - id: bounded-gate-resolution
    title: Measure approval stages and replace full-history gate lookup
    depends_on: []
    size: medium
    description:
      "bounded-gate-resolution: In sase-core and sase, instrument the approval
      boundaries from submission through paint and replace find_gate_shell_by_gate_id's
      full-history scan with an indexed exact gate-id lookup exposed through the Rust
      binding. Maintain the lookup on gate creation and marker mutation, handle old
      indexes off the interactive path, and test that lookup work stays bounded as
      unrelated history grows. Resolve the exact owning shell, not a successor
      inheriting its gate id. Preserve project scoping and newest-real-shell behavior.
      Run focused Rust/PyO3 and Python tests and each changed repository's required
      checks."
  - id: durable-approval-publication
    title: Separate durable decision acceptance from slow execution
    depends_on:
      - bounded-gate-resolution
    size: medium
    description:
      "durable-approval-publication: Implement shared acceptance and execution policy in
      sase-core with thin sase orchestration. Validate and durably reserve one decision
      and its supervised proc before publishing approval and dismissing the
      notification. Keep response.json's existing execution-complete contract and
      required host archive receipts; project accepted plan decisions separately from
      terminal shell state. Route ACE-facing plan APIs, CLI, mobile, and auto approval
      through the same policy. Move archive/network/launch work behind the acceptance
      boundary, publish terminal shell state before follow-up, and make crash recovery
      and retry ownership explicit. Preserve branch inputs, source, archive protocol,
      wait and coder choices, cancellation, and partial attempts. Prove acceptance
      remains visible while archive/launch barriers are blocked, and prove duplicate or
      racing submissions cannot duplicate work."
  - id: ace-immediate-projection
    title: Apply decision and notification changes through ACE's fast path
    depends_on:
      - durable-approval-publication
    size: medium
    description:
      "ace-immediate-projection: Update neutral plan and generic gate submissions in
      sase to use the shared durable operation and consume its acceptance result.
      Refresh the exact gate shell, planner, and family plus the cached notification row
      and indicator promptly, including approvals from Telegram/CLI/mobile. Bypass the
      broad agent-load cadence only for bounded decision deltas, reuse existing
      coalescing and pump-free refresh helpers, and protect new state from older
      snapshots. Keep all disk, subprocess and lock work off Textual's event loop and
      serial message pump. Verify folded/filtered/off-tab rows, stale overrides,
      repeated actions, failure recovery and input responsiveness."
  - id: telegram-prompt-actions
    title: Decouple Telegram acknowledgements and cleanup from gate execution
    depends_on:
      - durable-approval-publication
    size: medium
    description:
      "telegram-prompt-actions: In the sase-telegram repository, replace synchronous
      resolve_gate_response execution and settlement in inbound handlers with the shared
      durable submission API. Acknowledge callbacks promptly, remove or disable accepted
      decision keyboards, persist completion/error delivery and keyboard-cleanup
      retries, and continue processing later updates while gate execution runs.
      Reconcile externally accepted gates independently of slow handlers and outbound
      PDF/message delivery. Test authentication, duplicate callbacks, restart recovery,
      rate limits, input/feedback flows and cross-surface dismissal without real
      Telegram sends. Preserve the existing polling entry point for the receiver phase
      to integrate."
  - id: telegram-continuous-receiver
    title: Remove Telegram's periodic polling delay
    depends_on:
      - telegram-prompt-actions
    size: medium
    description:
      "telegram-continuous-receiver: Measure the effective inbound cadence and integrate
      a supervised, single-owner long-poll receiver with sase-telegram's existing chop
      entry point. Preserve --once, disabled-credential behavior and axe stop/config
      lifecycle. Keep local cleanup independent of the network poll, durably claim
      actionable updates before advancing offsets, and preserve ordering for mutable
      question/input progress and non-gate launches. Test receiver restart, two
      competing pollers, offset replay, rate limits, idle CPU and prompt cleanup during
      a long poll. Reuse SASE supervision rather than adding an unsupervised process or
      new queue service."
  - id: integrated-latency-verification
    title: Verify latency, recovery, and coordinated rollout
    depends_on:
      - ace-immediate-projection
      - telegram-continuous-receiver
    size: medium
    description:
      "integrated-latency-verification: Exercise both approval tiers across ACE,
      Telegram, CLI and mobile using the matched sase-core binding and the full combined
      implementation. Produce before/after stage timings and bounded-work evidence,
      including blocked archive/launch workers and large history. Run just check in
      every changed repository and sase's combined-tree just check-full through
      sase_monitor. Verify restart, duplicate, cancellation, partial-command and
      archive-failure behavior. Document rollout order, the Telegram receiver
      lifecycle/configuration, latency measurements and any remaining network limits.
      Remove temporary epic scaffolding before landing."
proposed_by: bbugyi200.athena.0js
create_time: 2026-09-12 05:06:11
status: wip
---

- **PROMPT:**
  [prompts/202609/prompt_gate_approval.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/prompt_gate_approval.md)

# Prompt gate approval publication

## Problem and scope

Approving a tale or epic should immediately end the human-review wait: its agent node
should show `TALE APPROVED` or `EPIC APPROVED`, and its SASE notification should
disappear. Archive publication, workspace preparation and launching the next agent may
take longer, but must remain visible as durable execution work. Telegram must
acknowledge the press and remove obsolete controls promptly, and one slow action must
not hold up subsequent callbacks or cross-surface cleanup.

This is a six-phase epic because the fix spans a shared Rust contract and index, Python
process orchestration, Textual state reconciliation, and a separate Telegram transport
with its own polling lifecycle. The ACE and Telegram phases can run in parallel only
after the shared contract is implemented. Every phase must preserve a working existing
path until its dependent consumers are ready.

The repositories are the current `sase` checkout, `gh:sase-org/sase-core`, and
`gh:sase-org/sase-telegram`. Open other repositories with `sase repo open` and use only
its returned paths. Do not assume a sibling checkout or a particular numbered workspace.
Read each opened repository's instructions. No implementation files were changed while
authoring this plan.

## Evidence from investigation

1. `src/sase/gate_shell/store.py:find_gate_shell_by_gate_id` calls `list_gate_shells`,
   queries all projects with full history and no record limit, converts and sorts all
   gate records, then selects one id. Its missing-index or query-error fallback scans
   the entire artifacts tree. A read-only no-match probe using the current source with
   the installed SASE interpreter measured **0.482 seconds for import and 7.032 seconds
   for lookup**. This is one sample, not a percentile or a full approval trace; the
   positive-id path performs the same initial enumeration. CLI, plan approval, mobile
   and Telegram use this lookup before gate execution.
2. `notification_gates/executor.py:execute_gate_selection` holds `.response.lock` across
   selected commands, adapter terminal preparation, response publication and adapter
   side effects. Notification dismissal already occurs just after response publication;
   adding another terminal dismissal alone will not fix the wait before publication.
3. `plan_approval_actions.py:prepare_plan_terminal_response` persists approval metadata
   and then requires `archive_approved_plan` for commit-bearing tale selections before
   `response.json` exists. `_plan_archive_approval.py` acquires an operational workspace
   lease, materializes the SDD store, commits, performs synchronous verified
   publication, and may reset/replay. It explicitly allows two-second sync-worker lock
   waits. Existing `host_v2` archive fields are required by successor preparation and
   must remain trustworthy.
4. `notification_gates/adapters.py:apply_side_effects` can prepare an epic launch after
   response publication but before the caller settles the gate shell.
   `gate_shell/settlement.py` writes terminal/index state before its own follow-up, but
   touches the refresh pulse only after follow-up launch returns. Thus slow work both
   before and after the response can delay different visible surfaces.
5. `ace/tui/actions/agents/_notification_plan_gate.py` submits neutral plans as a
   session worker and refreshes on whole-operation completion. Neutral generic gates use
   a durable `sase gate answer` proc; shell-backed CLI answers default to another
   detached proc, so audit the extra process layer and its misleading completion
   boundary. `_notification_gate_execution.py` still calls the synchronous
   notification-count refresh at completion.
6. ACE's event refresh intentionally defers general agent reloads and has
   `AGENTS_LOAD_MIN_INTERVAL_SECONDS = 5.0`. Notifications already have an immediate
   watcher poll, and the code has exact artifact-delta refresh helpers. Reuse those
   helpers. Check disappearance detection as well as dismissal flags and gate-shell
   identity: a planner-only delta can leave its family projection stale. Do not globally
   lower the full-load interval or rebuild all rows.
7. In `sase-telegram`, `scripts/sase_tg_inbound.py` fetches once with
   `get_updates(timeout=0)`, saves the whole batch's offset before processing, processes
   updates serially, and cleans up externally handled keyboards at the end.
   `_execute_gate_callback_response` calls `_resolve_response` before answering the
   callback or removing the keyboard. `inbound.py` executes the shared gate and settles
   its shell synchronously. API retries sleep in the same path. A source comment
   describes five-second chop ticks; the effective deployment cadence must be measured
   rather than assumed from that comment.
8. The existing workspace virtualenv returned agent-scan wire schema 7 while the current
   Python caller expected 8. The installed SASE interpreter completed the probe
   successfully. Set up a matching binding before drawing conclusions from tests; do not
   add a Python fallback or weaken schema validation.

## Shared decision and execution contract

### Fast acceptance

Introduce an additive, typed decision receipt in the existing gate/proc lifecycle, not a
second queue or a frontend-only optimistic truth. Its policy, validation of transitions,
stable identity and projections belong in `sase-core`; Python owns plugin calls,
subprocess execution and Textual/Telegram I/O. Expose the contract through
`sase_core_rs` and a thin public adapter used by every surface.

The receipt must bind the verified request/content hash, normalized selected option ids,
input identity, feedback identity, source, acceptance timestamp, execution attempt and
durable proc id. Reuse operation-request sidecars for actual inputs, and preserve
existing secret redaction and file permissions. Preserve coder prompt/model,
commit/run-coder combinations, wait specifications, epic launch mode and launch origin.
Public traces contain ids and durations, not prompts, input secrets or Telegram
credentials.

Under a bounded per-gate acceptance lock, verify that the reviewed bundle and inputs are
still valid, arbitrate cancellation and competing answers, and durably record enough
work to recover after process death. The accepted decision must never exist without
recoverable execution ownership. Reuse the proc request reservation and execution
journal, with deterministic operation identity and concurrency keys. Include every
semantically relevant input in identity; do not deduplicate only by selected option
names. Identical duplicate submissions return the original receipt; conflicting answers
fail promptly. Coordinate acceptance with the existing execution lock so older direct
answer paths cannot race it.

For validated tale `approve` and epic `approve`, publish the accepted decision, update
the exact shell/planner index projections, dismiss the human-review notification and
mark the shared action handled, then notify observers. None of these operations may wait
for workspace leases, Git, network operations, a full history scan or successor startup.
Use the authoritative receipt to repair a crash between its publication and
notification/index projection.

Keep decision status separate from execution status. `TALE APPROVED` and `EPIC APPROVED`
mean the decision is durably accepted. The execution proc can still be
queued/running/failed; the gate is not falsely marked completed and no premature
`done.json` is written. Pure commit choices must not display `PLAN COMMITTED` before
commit success. Generic gates with arbitrary commands must show submitted/executing
promptly, and their declared success status only after command success.
Reject/feedback/cancel keep their branch-specific labels. Keep activity buckets,
workspace claims and runtime accounting tied to actual execution state; an approved
label must not free an executing worker's resources or make its family disappear from
active work.

### Durable completion and recovery

Keep `response.json` as the execution-complete record expected by existing consumers. Do
not publish an incomplete `host_v2` response, mark an unpublished archive as archived,
switch verified publication to best effort, or launch a commit-dependent coder before
archive requirements pass. The supervised worker performs the existing commands, archive
and launch preparation after acceptance. The accepted reviewed bytes must remain stable;
later edits cannot silently change a decision already owned by a worker.

On command success, publish the response and shell terminal/index state and refresh
pulse before slow post-terminal follow-up work. Preserve the invariant that the shell's
decision record/chat and index are visible before a successor can resolve its fork
context. Extract epic launch preparation from any adapter ordering that prevents this
publication. Preserve existing gate-followup locks, fingerprints, claim ownership,
live-creator suppression, and resume behavior.

An archive/command/launch failure must produce a durable failure outcome and visible
recovery action. It must not disappear merely because the review notification was
dismissed. Do not silently re-run completed commands, reset an accepted answer to
pending, launch a duplicate coder, or lose a requested commit. Preserve journal-based
partial-attempt resume/restart semantics. Reconcile an accepted receipt after crashes
before spawn, after response publication, and during follow-up. Prompt normal completion
must not depend on the slow reclaim chop. `%auto` keeps its synchronous creator
semantics while using the same decision rules and notification publication; it must not
spawn a second coder.

Keep existing synchronous APIs synchronous where callers require an execution result,
and expose a separate submission result for interactive callers. CLI `--no-detach` must
still await completion; detached mode must report submission accurately. Avoid nesting a
second detached gate-answer proc inside ACE's already supervised execution proc.
Preserve already-terminal legacy bundles; if old runtime branches must remain selectable
during migration, follow `sase_flags` sunset policy rather than inventing an
environment-variable fallback.

## Bounded gate lookup and publication

Add an exact gate-id lookup to the Rust artifact index and binding, with a real indexed
identity rather than parsing every serialized record in SQL or Python. Index only real
gate-shell membership when selecting the owner; inherited gate ids on descendants must
not match. Maintain the field on creation, upsert, deletion and index migration. A
verified creation receipt/direct pointer can serve a bounded fallback for bundles that
already record their shell location.

An outdated/missing index must not trigger an all-project rebuild during a click. Use a
verified exact pointer when available; otherwise schedule durable repair and return an
honest pending/submitted result while the owned work resolves it. Do not silently skip
shell settlement when resolution is temporarily unavailable. Test old/new indexes,
missing targets, multiple projects, promoted families and successors inheriting ids.
Record rows decoded and artifacts opened, in addition to elapsed time, so a warm-cache
benchmark cannot hide linear work.

## ACE behavior

Use shared durable submission for plan and generic gate actions. Close the modal and
show submitting promptly; apply approved status and notification dismissal once the
authoritative receipt returns. A failed validation/reservation keeps the review
actionable. Persisting work must survive quitting ACE.

Marshal acceptance effects onto the UI thread, re-resolve the current identities and
selection, and patch the shell/planner/family rows and notification cache. Use the
existing artifact-delta queue, `request_notification_agents_refresh`, notification
snapshot loader and pump-free task helpers. Schedule urgent exact decision deltas on
receipt/watcher change without waiting for the broad five- second floor. Extend shared
loader projections so a reload reproduces the same state and a stale in-flight snapshot
cannot restore `TALE`/`EPIC` or the dismissed notification. Cover approvals received
while another tab is active or the row is folded, filtered, promoted or off-screen.
Bound urgent work and coalesce repeats; keep ordinary archive-scaled refreshes and idle
polling at their existing cadence. Check the existing j/k latency benchmark and
quiet-tick reload counters: urgent gate updates must preserve the p95 16 ms navigation
target and near-zero idle file opens instead of trading approval latency for background
load.

## Telegram behavior and receiver lifecycle

After authenticating and decoding a callback, send a bounded quick acknowledgement that
does not claim execution success. Validate/submit through the shared API, then remove or
disable the accepted decision controls and clear input/feedback progress. Report
rejection or submission failure explicitly and leave a retryable review. Slow commands,
workspace work and successor launch belong to the shared supervised proc, never the
polling process. Completion errors need a durable delivery record, rather than a late
callback popup that may already have expired.

Fix keyboard cleanup durability: `_dismiss_gate_callback` currently removes the
transport record even if the API edit fails. Retain a cleanup tombstone until Telegram
acknowledges the edit; retry network/rate-limit failures without re-executing the gate.
Cross-surface acceptance must drive the same cleanup and must run independently of
outbound PDF conversion, message retries or a slow incoming action. Use the existing
shared pending-action store and per-message transport identity, including all keyboards
attached to a gate.

Replace the idle gap between one-shot polls with one supervised long-poll receiver per
configured bot. Keep the existing chop entry point as an idempotent ensure- receiver
operation, so normal installations adopt the receiver without a hidden manual config
edit; preserve `--once` as a true one-shot path for diagnostics and tests. The receiver
must use the existing SASE supervision/lifecycle machinery, have a stable single-owner
key/lease, respond to stop/config changes, and leave no duplicate `getUpdates` consumers
after restart. Confirm its integration with the axe chop lifecycle and
disabled/incomplete-credential behavior. Check the actual deployment schedule during
verification. Do not compensate by spawning a new interpreter and scanning all state
several times a second.

The long poll waits only for Telegram updates; it cannot block local decision cleanup or
durable action submission. Use a persistent client/event loop or equivalent isolated I/O
owner so slow retries cannot stall the receiver's control path. Respect Telegram rate
limits and retain unsent effects. Advance offsets only after each actionable update is
durably claimed or queued; duplicate deliveries must converge on the same operation. Do
not parallelize mutable question/input progress or introduce duplicate non-gate agent
launches while removing the old batch-at-most-once shortcut. Existing slash-command,
photo, question, feedback and gate input behavior must remain intact.

## Acceptance and verification

Instrument click/update arrival, validation, lookup, durable reservation, acceptance
publication, notification mutation, ACE paint, Telegram ack/edit, response publication
and follow-up launch separately. Use monotonic elapsed times within a process and
durable ids to correlate processes. First collect a matched-environment baseline and
then repeat the same fixture/workload after the change. Do not exercise real user gates,
push dummy plans or send test Telegram messages; use isolated stores and fake transports
unless the user authorizes a live end-to-end test.

Targets on a healthy local host are p95 below 250 ms for an exact shell lookup, and
below one second from receipt of a valid review submission to durable plan acceptance
plus local notification dismissal. ACE should paint the accepted status and remove the
indicator entry within another 250 ms of observing that receipt. Telegram should issue
its callback acknowledgement within 250 ms of receiving the update and issue keyboard
cleanup within one second of acceptance; report API/network time separately. Long
polling should remove the multi-second application polling floor. These are measured
acceptance targets, not brittle wall-clock assertions in ordinary CI.

Use synchronization barriers in regression tests: block archive publication or successor
startup indefinitely within the test, then assert the decision and notification are
already updated and other UI/callback work proceeds. Release the barriers and verify one
correct archive and successor. Add deterministic ordering and bounded-read assertions,
plus a repeated performance run reporting p50/p95/max with host load and fixture sizes.
At minimum cover:

- Tale approve-only, approve+commit, commit-only, epic approve and launch-skip,
  rejection, feedback, generic command success/failure and `%auto`.
- Invalid/tampered plans, mismatched request hashes, invalid inputs, secret redaction,
  accepted-plan edits and submitted coder/wait configuration.
- Simultaneous ACE/Telegram/CLI/mobile answers, cancellation races, duplicate delivery,
  partial attempts and process death at each acceptance/response seam.
- Archive contention/failure, unavailable workspace, failed follow-up spawn, resume
  after restart and no duplicate commands, archives or successors.
- External approvals with a stale ACE load in flight; family/planner/gate rows,
  folded/off-tab lists and notification-modal/count consistency.
- Telegram callback while another gate is blocked, external dismissal during long
  polling, API timeouts/rate limits, failed keyboard edits, restart before completion
  delivery, offset replay and two competing receiver processes.
- Large unrelated history, old index migration and targeted lookup unavailable.

Extend existing suites rather than creating a parallel test framework: relevant coverage
includes `tests/gate_shell`, `tests/plan_shell`, `test_gate_cli_answer*.py`,
`test_notification_gate_execution.py`, `test_notification_gate_durability.py`,
`test_plan_approval_actions_archive.py`, `test_plan_archive_approval_recovery.py`,
`test_plan_approval_launch_reliability_integration.py`,
`tests/ace/tui/test_notification_plan_gate.py`, event-refresh/artifact-delta and
family-status tests, Rust artifact-index/notification/gate-followup tests, and Telegram
inbound/pending-action/client tests.

Read `lint_and_test` and `tui_perf` before implementation. Run `just check` in each
changed repo; core verification must include the PyO3 binding tests, not only
`cargo test -p sase_core`. Run the combined sase tree through `just check-full` using
`sase_monitor` as required. Validate help/docs for any changed CLI behavior, update
`default_config.yml` if configuration changes, and use the generated-skills workflow
only if skill source changes prove necessary.

Release the core binding before Python consumers and then Telegram; ratchet dependency
requirements only after compatible artifacts exist. Do not hand-edit Rust release
versions. Keep foundational phases additive; if partial behavior would be user-visible,
use the required temporary beta epic scaffold and remove it after combined verification.
Document old in-flight bundle handling and the receiver adoption/stop procedure. No new
memory notes or unrelated feature work are required by this plan.
