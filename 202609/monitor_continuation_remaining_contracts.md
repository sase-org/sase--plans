---
tier: epic
title: Repair the remaining monitor continuation contracts
goal:
  Preserve exact frozen context, protected instructions, single-delivery recovery, and
  existing-record semantics while integrating continuation ancestry with retention and
  proving the complete route.
parent_bead: sase-zl.13
phases:
  - id: frozen_context
    title: Route and render successors from immutable result context
    size: medium
    depends_on: []
    description:
      "frozen_context: bind ordinary and recovery successors to the exact result graph
      and project facts, evidence, intents and checkpoints from persisted context."
  - id: protected_budget
    title: Budget from trusted provenance and explicit checkpoint coverage
    size: medium
    depends_on:
      - frozen_context
    description:
      "protected_budget: replace Markdown-based reduction discovery with attributed
      projection data and validate actual resulting bytes before provider invocation."
  - id: atomic_recovery
    title: Fence manual resume against concurrent receiver adoption
    size: medium
    depends_on:
      - frozen_context
    description:
      "atomic_recovery: atomically revalidate and supersede undelivered branches,
      preserve acknowledgments, and refuse ambiguous receiver ownership."
  - id: record_semantics
    title: Preserve persisted protocol semantics across rollout changes
    size: medium
    depends_on:
      - protected_budget
      - atomic_recovery
    description:
      "record_semantics: select the protocol for new starts with the rollout flag while
      keeping existing versioned runs on their immutable capture, policy and delivery
      contracts."
  - id: ancestry_retention
    title: Protect and recover referenced continuation ancestry
    size: medium
    depends_on:
      - frozen_context
    description:
      "ancestry_retention: integrate the run-retention planner with active continuation
      dependencies and durable portable refs without broad UI or repeated archive scans."
  - id: acceptance
    title: Prove repaired production routes and combined compatibility
    size: medium
    depends_on:
      - record_semantics
      - ancestry_retention
    description:
      "acceptance: exercise integrated capture-through-invocation and recovery paths,
      publish reproducible evaluation evidence, and complete released-package, visual
      and full verification gates."
proposed_by: bbugyi200.athena.sase-zl.13.land
create_time: 2026-09-13 06:00:37
status: wip
---

- **PROMPT:**
  [prompts/202609/monitor_continuation_remaining_contracts.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/monitor_continuation_remaining_contracts.md)
- **PARENT:**
  [202609/monitor_continuation_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/monitor_continuation_landing_repairs.md)

This is remaining work from the landing of `sase-zl.13`. Read its audit
`file:explicit:af7cbd77c889b65ed26818c9` first: it records five executed failures, all
ten phase commits and nineteen notes, intervening changes, verification limits, and all
four proposed-follow-up dispositions. Governing requirements remain
`plan:202609/monitor_continuation_landing_repairs.md` and
`plan:202609/monitor_continuations.md`. Preserve the original parent audit's twenty
follow-up outcomes in `file:explicit:b356c51cb45677f60961e909`.

The inspected primary was `817c5c5679`; fetched base advanced to `bad80a9324`
(RefreshPanelModal). Open core through `sase repo open`; configured-name resolution may
still require `gh:sase-org/sase-core` because of existing task sase-zo. Read and write
only the returned path. Core inspected at `a64c40d`; paths beginning `crates/` below are
relative to that checkout. Recheck current base and core drift first.

Keep the implemented capture/replay packages, immutable content files, frozen Rust
policy, delivery journal, host controller/receipt recovery, selected diagnostics, CLI
controls, and existing monitor goldens. Repair the defects below rather than repeating
completed phases. Shared identity, validation, graph/coverage selection, delivery
transitions and retention decisions belong in Rust with PyO3 adapters; Python owns I/O,
process effects and rendering. Preserve the atomic_json import boundary, current
queue-capacity semantics, explicit bead decisions, typed launch workspace/remote
targeting, pooled-provider policy, and cached history/UI paths.

## frozen_context

`monitor/followup.py` loads the frozen result but passes mutable command/status/time
arguments to `compose_followup_prompt`; that builder constructs a second result and
forks the starter/family. The audit's real-start/resume probe retained the frozen
delivery ID while rendering a changed command and exit 99 instead of frozen exit 1.
Existing follow-up tests even expect the starter fork. Correct the actual launch route
and normal fork-source resolver, not only isolated replay helpers.

Carry the exact immutable monitor-result node and its parent graph to the receiver,
separately from family attachment. Bind result facts, selected evidence, resolved
branch, intent revision and checkpoint to one persisted projection input. Render
ordinary follow-up, direct/family inspection and manual resume from that source, with
deliberate historical evidence aging. Missing or corrupt enabled-route essential records
must produce recoverable refusal, never mutable reconstruction that looks like a
successful frozen delivery. Preserve explicit legacy inspection.

Make replay expose structured attributed block/provenance metadata to downstream
budgeting. Include authored checkpoint contents and explicit coverage on the actual
resumed prompt; setting checkpoint_ref in monitor metadata alone is not sufficient.
Preserve literal hostile headings/directives, materialized local user and gate
instructions, and full command arguments. Do not append raw bytes outside the
centralized evidence selector.

Acceptance: drive normal start/capture/settlement/dispatch and preprocessing with fake
effects; mutate metadata and aliases after freeze and prove rendered facts and ancestry
remain exact. Start the 100-handoff test from only the final actual successor target,
with every protected constraint/decision once and linear new canonical storage. Exercise
none/file with zero diagnostic/raw bytes and explicit legacy conflicts.
Changed-checkpoint resume must expose the checkpoint body and coverage while retaining
the same command result.

## protected_budget

`llm_provider/continuation_budget.py` currently finds removable sections using Markdown
regexes. An authored `## Selected diagnostics` section loses all protected instructions
under pressure (6,551 bytes becomes 164). Checkpoint eligibility is also inferred from
headings rather than the authored coverage record. This is a correctness failure even
when input headings are innocent examples.

Use the structured provenance from frozen_context to identify protected and reducible
spans. Put shared coverage/reduction validation in Rust; make rendering and I/O thin
consumers. Checkpoints may replace only explicitly covered material, never infer
coverage from a displayed ref or heading. Preserve objective, user/gate constraints,
unresolved decisions, next action and newest result. Keep omission manifests and refs
visible. Do not introduce summarization model calls.

Apply selected reductions to the actual prompt, avoid overlapping-span byte
double-counting, and remeasure UTF-8 bytes after projection before invocation. Oversized
protected content must refuse durably. Preserve selected/fallback provider and model
budgets, reserves and transport contracts; do not consume pooled alias selection twice.
Unknown limits remain explicitly uncertain.

Acceptance: reproduce the audit's authored-heading failure as a regression; include fake
checkpoint headings, fences, Unicode, overlapping sections, uncovered constraints,
threshold/no-op prefix preservation, real covered ancestry, actual smaller/larger routes
and fallback. A compact result must fit its effective budget; a refusal must remain
uninvoked and recover with an adequate checkpoint or route.

## atomic_recovery

`resume_monitor` reads records before allocating a branch and
`_supersede_active_records` later persists transitions from those stale objects. The
audit injected real adoption immediately after allocation: the old acknowledged delivery
became needs_attention with acknowledged_by=null and a new branch spawned.

Define the atomic resume/adoption decision in the Rust delivery contract and apply it
under the existing bounded Python store/admission locks. Reload current state before
superseding, and preserve a concurrent acknowledgment. Fence the old receiver before
admitting a replacement; serialize concurrent different and identical revisions against
the same result. Lack of a discoverable receiver is not proof of non-delivery: use
existing launch receipts/process identity to prove it uninvoked, or record
needs_attention without another automatic spawn. Keep I/O, spawn and waits outside locks
and document lock order.

Ensure starter/monitor/family capacity and workspace ownership remain continuous until
acknowledgment or explicit disposition. Preserve stopped/lost inspection-only rules,
stop-versus-adoption semantics and host-completion original-workspace rules. Do not
rerun the monitored command or retry ambiguous external finalizers.

Acceptance: deterministic barrier tests for acknowledgment between read and supersede,
concurrent same/different revisions, crashes around reservation/spawn/ adoption, stale
receiver visibility, stop races and repeat resume. Assert provider invocation counts
across processes/deliveries, not just stable key strings or the count of calls in one
mocked launcher. The acknowledged old branch must never be erased. Include this epic's
concurrent-start full-lane flake in regression triage.

## record_semantics

The current rollout test covers runs started and settled in the same flag state. The
audit started a v1 monitor with the flag on, resumed it from an off-flag process, and
observed a spawned successor with no delivery key and no delivery disposition.
`followup.py`, `settlement.py` and capture wrappers consult current global flags.

Persist the selected protocol at new-start admission. Use that record identity through
handoff capture, settlement, reconciliation, resume and receiver adoption. Disabling
`monitor_continuation_records` changes new writer/launcher selection; existing v1 runs
retain their validated immutable policy and delivery semantics. Do not manufacture v1
capture for old legacy runs merely because a later process has the flag enabled. Handle
corrupt/incomplete v1 state as recovery, not fallback.

Retain the existing sunset flag sase-102 and its removal policy unless its actual
conditions are satisfied through the feature-flag workflow. If any new partial
user-reaching behavior requires a beta scaffold, follow the mandatory epic-beta workflow
and remove that scaffold before landing. Do not deploy generated skills from an unlanded
source revision.

Acceptance: on→off and off→on in separate process snapshots (clear inherited test flag
snapshots before overriding them), ordinary success and failure, frozen
none/continue/complete, capture failure, terminal crash and manual resume; old records
keep their semantics and stopped/lost never auto-dispatch.

## ancestry_retention

Intervening `b9684d76e6` added run-directory retention. Its protection collector skips
artifact trees and its planner does not follow continuation dependencies. The audit's
preview protected a live monitor yet selected the old run referenced by its starter path
and exact parent ID for deletion. Meanwhile portable capture registration is
best-effort, called only for checkpoints, and its returned ref is not used as the
graph's canonical portable locator.

Integrate continuation dependency closure with the existing retention/index path.
Protect necessary parents, results, intents, checkpoint/evidence blobs and pending
delivery data of live/recoverable continuations. Register and resolve immutable portable
content through existing artifact APIs where reconstruction after local cleanup is part
of the contract. Distinguish required storage failure from optional debug capture; never
allow missing essential evidence to look successfully retained. Avoid permanent blanket
retention of every transcript or an unbounded rescan on each UI refresh. Coordinate
touched files with active retention epic sase-zw; this continuation integration remains
work of this child.

Acceptance: real captured old ancestry referenced by a new live/pending monitor is
excluded from deletion; after a safe terminal disposition unrelated old data can be
reclaimed; retained portable refs hydrate required ancestry after local files are
absent. Cover reference/index failures, concurrent publication/cleanup and required
checkpoint registration failure. Use temporary trees for deletion tests; no production
cleanup is requested.

## acceptance

The existing evidence `file:explicit:5a2f9ca53bca3cb178cd9a5a` records useful 163-test
and 100-handoff results, but does not prove the five failures above or a final
successful full gate. Extend realistic existing fixtures into complete
capture→command→freeze→dispatch→preprocess→adopt→provider/host-completion paths. Retain
the detached supervisor/kill-group/timeouts and real host controller tests: 33 focused
controller/delivery/policy tests passed in the landing audit despite the remaining
defects. Tests must not replace production contracts with permissive callbacks or feed
every ancestor directly to the renderer.

Run the failure matrix over success, stage failures, timeout, cancellation, legacy
boundaries, long ancestry, evidence policies, checkpoint budget recovery, concurrent
resume, partial multi-repo finalization and retention. Preserve weighted admission,
queue-budget zero/absence compatibility, current import boundaries, explicit bead
actions and pooled fallback routing. Independent command reservations/monitor weights
belong to sase-zm.5 and are excluded.

Publish an indexed artifact with exact commands, tested revisions, corpus bounds, prompt
bytes, retrieval volume, successor/provider counts, elapsed time and work correctness,
separating normalization, evidence and checkpoint effects. Provider, cache and reasoning
usage may be unknown in fake-provider tests; record that honestly and make the measured
provider-session experiment decision without making sessions a dependency. Recheck the
existing 90x40/120x40 monitor states and capture the governing navigation p95/idle-read
evidence using tui_perf and perf_runbook procedures. Preserve current refresh-modal and
indexed-history integration.

Release-plz owns Rust versions. Current published minimum 0.34.23 was independently
imported in an isolated environment and contains the four earlier missing policy/
delivery APIs, so those proposals are resolved. Any new required APIs must be built,
published and verified in the actual compatible minimum wheel before ratcheting Python
requirements; preserve later core functionality. Verify local wheel and clean
published-package paths, including installed plugin compatibility instead of skips.

Each phase changing tracked SASE files runs `just check`; core changes run full core
`just check` including PyO3 tests with Python >=3.12. Use sase_monitor for long
commands. Complete combined `just check-full` through a TESTING/TESTED monitor with
failure-handling continuation; complete every post-pytest gate after any budget repair.
Run dedicated visual verification for changed goldens. Triage concrete failures under
sase_new_task, preserving all audit dispositions: Models flake→sase-si; gateway
bootstrap→active sase-xe.16.11; dirty-plan attachment→active sase-yy.8; concurrent
monitor starts→this epic. No unverified generic flake task.

Record every phase's implemented behavior and exact test result in its close note;
literal placeholder close notes are not acceptance evidence.

No phase closes sase-zl.13 or sase-zl, runs their post-close Symvision steps, or marks
their linked plans done. The child plan's parent_bead is the landing handoff. Its land
agent must recheck normal descendant/plan readiness and drift before resuming the parent
landing; never force a successful nested close.
