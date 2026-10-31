---
tier: epic
title: Recover staged handoffs without repeating adoption
goal: Make every staged handoff replayable after runner death, reconcile completed
  turns, and let held handoffs be explicitly re-adopted after their obligations are
  satisfied.
parent_bead: sase-1pj.6
phases:
- id: recovery-contract
  title: Define durable recovery identity and replay decisions
  depends_on: []
  size: medium
  description: 'recovery-contract: add Rust-owned recovery contracts and replay decisions,
    persist the original execution context, and serialize adoption across runner,
    manual recovery, and housekeeping callers. Preserve existing receipt formats.'
- id: successor-replay
  title: Recover pipe and inline successors exactly once
  depends_on:
  - recovery-contract
  size: medium
  description: 'successor-replay: persist successor identity before continuation side
    effects, replay pipe adoption without allocating another successor, and provide
    the common detached continuation path used when the original runner is gone.'
- id: approval-replay
  title: Replay plan and question approval handoffs
  depends_on:
  - recovery-contract
  - successor-replay
  size: medium
  description: 'approval-replay: bind plan and question gates to staged identity,
    recover original creator and approval context without ambient agent variables,
    and replay both human-gated and auto-resolved continuations.'
- id: gate-replay
  title: Complete partially published gates on replay
  depends_on:
  - recovery-contract
  size: medium
  description: 'gate-replay: resume partial gate publication under its existing request
    and member identities, repairing claim and notification effects without duplicating
    the gate or changing its frozen creator and policy.'
- id: monitor-replay
  title: Close monitor spawn and held-reservation crash windows
  depends_on:
  - recovery-contract
  size: medium
  description: 'monitor-replay: recover monitor launch from existing fingerprint and
    proc identities, including completed commands, and retain or safely restore held
    reservations and prepared completion evidence across retries.'
- id: recovery-service
  title: Adopt from persisted state and preserve finalizer progress
  depends_on:
  - recovery-contract
  - successor-replay
  - approval-replay
  - gate-replay
  - monitor-replay
  size: medium
  description: 'recovery-service: expose one host recovery entry point for all kinds,
    prove settlement from the original run and current obligations, preserve successful
    multi-repository commit receipts, and replay held handoffs after valid repair.'
- id: readopt-cli
  title: Expose re-adoption and actionable held notifications
  depends_on:
  - recovery-service
  size: small
  description: 'readopt-cli: add sase agent readopt for one unambiguous originating
    turn, render recovery results and refusals, and include the exact command in held
    handoff notifications and focused user documentation.'
- id: reconcile-stages
  title: Reconcile abandoned finalized stages and verify crash recovery
  depends_on:
  - recovery-service
  - readopt-cli
  size: medium
  description: 'reconcile-stages: add bounded scheduler housekeeping for finalized
    staged turns whose runner is gone, coordinate it with claim cleanup, and verify
    concurrent recovery and crash boundaries across all handoff kinds.'
proposed_by: bbugyi200.athena.sase-1pj.6
proposed_with: codex/gpt-6-astra
create_time: 2026-10-31 08:56:17
status: wip
bead_id: sase-1pj.6.1
---

- **PROMPT:** [prompts/202610/staged_handoff_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/staged_handoff_recovery.md)
- **PARENT:** [202610/staged_handoffs.md](https://github.com/sase-org/sase--plans/blob/main/202610/staged_handoffs.md)
- **BEAD:** [sase-1pj.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pj/sase-1pj.6.1.md)

# Recover staged handoffs without repeating adoption

## Scope and completion ownership

This is a child epic implementing **only** the adoption-recovery phase of
`plan:202610/staged_handoffs.md`, tracked by `sase-1pj.6`. Read that artifact through
`sase artifact read` and read the assigned bead before implementing.

The parent phase has no earlier-work notes. Exploration found substantial foundations
already landed, but no general persisted adoption service, recovery CLI, or staged
handoff reconciler. The remaining work crosses Rust contracts, runner continuations,
gate publication, monitor process submission, finalizer evidence, and scheduling. It
needs the bounded phases above rather than one tale. Each phase owns its implementation
and regression tests; the child epic's land agent integrates and closes the original
phase when complete.

Honor the outer epic's final decisions. Its memory authorizations remain limited to the
named decision records, `macros.md`, and the two glossary strands. This child epic
requires **no memory edits** and introduces no new reviewer decisions. Flag retirement,
kill-path removal, and general carry abandonment policy belong to the outer epic's other
phases. Preserve whatever legacy compatibility remains when implementation runs; do not
restore a flag branch that another approved phase has already removed.

Workers close only their own assigned child phase. They never close `sase-1pj.6` or
`sase-1pj`. The child epic's land agent owns the verified completion of `sase-1pj.6`; it
must not close `sase-1pj` or any other ancestor. Handle the child epic's own closure
through the ordinary land workflow. Discovered work is a `PROPOSED FOLLOW-UP:` note on
the worker's assigned bead, not a new task bead.

## Existing behavior to preserve

- `src/sase/axe/run_agent_exec_staged_dispatch.py` checks the adoption receipt first,
  proves finalizer settlement, persists origin custody, calls the kind adapter,
  transfers custody, and writes the success receipt. There is no lock spanning that
  entire sequence. A receipt currently preserves an inline `None` loop outcome but does
  not reconstruct the continuation state for a fresh caller.
- `run_agent_exec_staged_adopters.py` supports all five kinds, but every adopter takes
  `StagedInlineContinuation`, and plan/questions/pipe still need live runner context.
  Plan and question handlers mint new session UUIDs; pipe can allocate a new suffix.
  These are unsafe replay boundaries after the side effect and before the receipt.
- `gate_turn/transaction.py:publish_gate_turn` already reuses a member by request ID
  under a lane lock. Extend this transaction instead of creating another publisher.
- `monitor/start_stage_adopt.py` reconstructs a frozen launch payload under the lane
  lock and reuses a running member. Audit the earlier proc submission window and the
  case where the command finished before the outer adoption receipt was written.
- `run_agent_exec_staged_settlement.py` authenticates the sealed finalizer plan and
  same-turn identity, checks instance results, and re-observes repository work. Its
  `success` and legal `deferred` cases already include carry and prepared coverage.
- `agent/_staged_handoff_receipts.py` and `agent/_staged_handoff_store.py` own existing
  receipt/storage helpers. Do not fork those contracts into each recovery caller.
- The current Rust `staged_handoff` module owns schemas, receipt authorization,
  admission, gate payloads, and monitor lifecycle predicates. New shared recovery policy
  belongs there; Python gathers facts and performs host I/O.

Relevant regression suites include `tests/test_axe_run_agent_exec_staged.py`,
`test_questions_pipe_staged_adoption.py`, `test_gate_staged_adoption.py`,
`test_staged_failure_boundaries.py`, `test_staged_custody_adoption.py`,
`test_staged_handoff_acceptance_matrix.py`, `tests/monitor/test_monitor_start_stage*`,
and `tests/test_finalizers_commit_reconciliation*`.

## Recovery invariants

1. One logical adoption is identified by the **originating turn plus `staged_id`**. A
   matching successful receipt wins. A malformed or mismatched receipt is a refusal, not
   missing state. `handoff_not_adopted` records a reason and remains retryable through
   explicit recovery; it never authorizes activation.
2. A host-owned adoption lock serializes the normal runner, reconciler, and CLI. Read
   and revalidate after acquiring it. Document lock ordering with existing handoff,
   gate-lane, monitor-lane, and claim locks; avoid recursively acquiring the current
   receipt writer's lock. No successful receipt may be overwritten by a racing hold.
3. Persist enough origin context before the first activation side effect to recover
   without the caller's `SASE_AGENT*`, current working directory, model defaults, or
   live `LoopState`. Include stable origin identity, workspace/project ownership,
   prompt/continuation references, selected model/provider/policy, and chosen child
   identities. Reuse existing authenticated metadata and immutable artifacts rather than
   copying unrelated environment or credential values.
4. Receipt absence never proves that publication or spawn did not happen. Resolve a
   stable request/member/proc/successor identity first. Unknown liveness or an
   incomplete publication is held or resumed under the same identity; it never causes a
   second arbitrary process launch. Completed monitors and successors count as existing
   effects and must not be relaunched.
5. An adopted receipt is written only after the logical gate, monitor, or continuation
   is durably owned. Record destination refs and enough continuation state to recover an
   inline result. A crashed runner cannot consume `None`; recovery must deliver the
   already-identified successor through the normal host launch path exactly once. Do not
   convert every live inline continuation into a new process.
6. Automatic adoption requires a proven-dead originating runner, successful finalizer
   settlement (including legal typed deferrals), and no terminal adoption receipt or
   explicit hold. A successful inline receipt with an undelivered continuation permits
   delivery-only recovery of that recorded child, never another adopter call. Unknown
   liveness is not death. Recheck kill/discard and ownership under the lock. Automatic
   recovery never reruns finalizers or the originating provider.
7. Explicit re-adoption also uses current evidence. A clean working tree alone cannot
   erase a failed non-commit finalizer, authenticate a stale declaration, discharge
   unpublished commits, or grant carry. Partial repository success remains recorded;
   only the still-unfinished authorized finalizer work may be repaired.
8. Preserve original chats, archived plans, carried/prepared evidence, and the workspace
   claim until the actual next owner is established. Recovery updates projections and
   lifecycle metadata using existing host APIs, without rewriting historical provider
   failure as if it never occurred.

## recovery-contract

Open `sase-core` with `/sase_repo` and read its `AGENTS.md`. Add focused typed recovery
facts and decisions to `crates/sase_core/src/staged_handoff/`, exported and bound
through `crates/sase_core_py/src/staged_handoff/`. Cover origin/stage identity, receipt
outcomes, runner liveness, kill/discard evidence, automatic versus explicit recovery,
and observed destination identity. Keep filesystem probing and spawning outside Rust
policy functions.

Define a versioned recovery-context/attempt contract that carries only facts not already
recoverable from canonical artifacts. Make its persistence atomic and bound to the
original stage. Additive fields or a separate companion record should preserve v1 staged
records and receipts. Old artifacts with insufficient evidence get a precise refusal; do
not manufacture context from the recovering shell. If a wire version must change, update
every mirror, fixture, and validator together.

Introduce the shared adoption lock and receipt transition rules. Make the existing
runner dispatcher use them, preserving outcomes and custody behavior. Add an internal
adapter input/result contract that allows persisted origin context with optional live
continuation; later phases convert the kind adapters. Persist the recovery context
before finalization/activation can leave a dead runner with no reconstructable state. Do
not activate an incomplete new outside-runner path in this phase.

Rust and binding tests cover the recovery decision table and malformed evidence. Python
tests cover lock contention, receipt monotonicity, stale/mismatched IDs, and unchanged
runner behavior. Update `sase-core-revision.txt` past the landed core commit through the
host-owned cross-repository completion path before introducing dependent Python calls.

## successor-replay

Own `run_agent_exec_pipe.py`, the shared successor preparation helpers it calls, and
pipe-specific adapter changes. Persist the chosen child suffix, artifacts identity,
prompt and continuation refs, model/provider, workspace/custody destination, and launch
identity before changing runner state or making the successor runnable.

Build a common continuation delivery helper for subsequent approval adapters. The live
runner applies a durable continuation to its `LoopState`; an outside-runner caller
launches that same continuation through the existing host launch/runner-slot machinery.
Use stable launch identity and inspect durable launch/proc evidence so a crash after
submission cannot duplicate the provider turn. An already-started or terminal child is
returned as the destination, not restarted. Do not add another general job scheduler.

Handle receipt replay with inline `None`: restore the intended destination/state when
still in the originating runner, or complete detached delivery when the origin is dead.
Persist whether the successor has actually started separately from logical adoption; a
crash after the success receipt but before the next provider invocation must remain
discoverable as pending continuation delivery. The housekeeping phase consumes this
state without invoking the adopter again. Preserve `%fresh`, explicit name tokens,
models, continuation ancestry, chat links, and carried repository ownership. Test
crashes after child allocation, after durable launch submission, after start, and before
outer receipt, plus repeated and concurrent replay.

## approval-replay

Own the plan/question adapters, `run_agent_exec_plan.py`, `run_agent_exec_questions.py`,
`plan_gate_turn/create.py`, and `question_gate_turn/create.py`. Capture or derive a
stable session/request identity from the originating stage before publication; a retry
must not call `uuid4()` for a replacement approval request. Thread explicit persisted
creator data through the staged path, retaining compatible live/legacy wrappers where
still required.

Persist the effective approval inputs/policy at the original transition so recovery does
not newly enable auto approval. Freeze the question base prompt, prior rounds, parent
gate linkage, plan feedback round, and relevant coder options. Preserve the parent
epic's existing reviewed decisions; do not ask them again.

Use gate request replay for human approvals. Use `successor-replay` for accepted plans,
auto-answered questions, and feedback continuations that previously returned inline.
Recover existing destinations after publication or answer settlement, even if the gate
is now terminal. Test tale and epic plans, auto and human paths, rejection/timeout,
multi-round questions, original response retention, and replay from a neutral shell.

## gate-replay

For generic gates, audit each publication boundary in `gate_turn/transaction.py`,
`publisher.py`, and `run_agent_exec_staged_gate.py`: member creation, claim transfer,
bundle creation, notification publication, and custody updates. Existing request-ID
replay must finish missing work and validate identity/payload instead of blindly
assuming a partially created member is fully published. Preserve frozen auto policy,
original creator, resource snapshots, and retained/released workspace modes. Exercise
the shared gate path for ordinary, sudo, launch, run, and workflow HITL requests.

Test crashes at each durable boundary, including a terminal gate found before the outer
receipt, notification retry, failed claim transfer, and concurrent publication. Use the
real durable gate store with fake external publication, retaining its request ID
deduplication contract. Add a payload mismatch refusal test.

## monitor-replay

For monitors, use the frozen payload's stage ID, monitor ID, request fingerprint,
reserved ToolRun and existing proc identity. Harden `start_stage_adopt.py` and the
submission boundary in `start_launch.py` so a process submitted before member state or
the outer receipt is written can be found and adopted. A terminal matching process must
also be reused. Where the generic proc submit API needs an idempotency contract, add the
smallest Rust-owned extension rather than a monitor-specific second process store.

Separate explicit discard from a held failed-finalizer attempt. Audit
`cleanup_staged_monitor_handoff` and `start_stage_cleanup.py`: held cleanup currently
releases staged resources, so recovery must either safely reacquire the same logical
reservation under its locks or report a precise unrecoverable conflict. It must not
reuse a canceled ToolRun, lose prepared intent evidence, or start an extra command.
Preserve prepared verification's bind-on-launch and commit-on-green behavior.

Add crash tests at spawn boundaries, with the actual durable stores and a fake process
boundary rather than only a mocked adopter. Cover repeated and concurrent attempts,
completed effects, failed claim transfer, prepared monitors, joins, held resource
recovery, and payload/fingerprint mismatch. Never restart an ambiguous command merely
because the starter died.

## recovery-service

Expose one host entry point accepting an originating artifacts identity and a recovery
mode. Resolve original metadata and the new persisted context, acquire the shared lock,
gather current facts, ask Rust policy, and dispatch the now-replayable adapters. The
runner wrapper supplies live continuation when present; other callers do not fabricate
an `AgentExecContext` with defaults from their own environment.

Preserve the existing finalizer proof checks. Refactor shared settlement policy into
Rust as needed, keeping Python observation and existing authentication. Missing
finalizer files, malformed results, stale contexts, absent real provider response, and
changed workspace ownership produce typed held/refused results with evidence intact.

Distinguish two cases: a fully settled turn can adopt immediately without running any
finalizer; a held failed-finalizer turn cannot activate until its remaining obligations
have authenticated completion evidence. Reuse the existing finalizer repair/checkpoint
and commit reconciliation paths (`commit_execution.py`, `commit_checkpoint_recovery.py`,
`commit_unpushed_resume.py`) to retain progress and settle only unfinished work. Do not
make re-adoption an implicit provider rerun or a blanket finalizer retry. After repair,
refresh/revalidate the original run's settlement through host APIs; never manually stamp
old failed results as success. If repair needs an unavailable declaration or other
unsatisfied work, return a precise refusal and the supported repair action.

Add a two-repository integration fixture where A commits and B fails. Verify that A's
commit/checkpoint identity and publish evidence survive, activation stays held, repair
only processes B or its pending checkpoint, and subsequent re-adoption creates one
destination. Include failure of a non-commit finalizer, legal carry/typed deferral,
prepared coverage, changed dirt after settlement, and a crash after commit before any
adopter call. Successful receipt replay must not re-check successor-owned dirt as if it
still belonged to the origin.

## readopt-cli

Add `sase agent readopt <agent>` using the existing agent selector conventions and the
shared recovery service. The selector must resolve exactly one originating turn; a
session with multiple eligible turns is an actionable ambiguity error. The command
checks for a matching success receipt first, refuses an active/unknown owner or unsafe
evidence, and otherwise retries a held adoption whose obligations now have valid proof.
No force option may bypass settlement, workspace ownership, or identity checks.

Use normal exit conventions: success and already-adopted are zero; held/refused state is
nonzero and names the cause. Show handoff kind, original agent identity, and destination
refs on success. Add `-j/--json` for a structured result and keep parser
commands/options alphabetized. Read `cli_rules.md` before implementing.

Update the held notification path in `run_agent_runner_finalize.py` to name the exact
re-adopt command and origin, including failures discovered by recovery. Deduplicate
unchanged notifications. Document the distinction between repairing incomplete
finalization and re-adopting a satisfied handoff in the existing agent handoff docs,
after reading `docs_writing.md`. Tests cover help, selector ambiguity, active turns,
already adopted, held failure, repaired success, JSON, and notification text.

## reconcile-stages

Add a bounded staged-handoff reconciliation job to existing scheduler housekeeping,
using the repository's builtin chop registration and script conventions and
`src/sase/default_config.yml`. Follow the existing artifact index and cursor patterns;
do not add reconciliation to a TUI read path or rescan all archived artifacts every
tick. Choose the housekeeping lane alongside gate-turn reclaim, with an always/periodic
trigger because runner death need not produce a file event. Register packaging and job
inventory entries needed for the script to run.

Discover pending staged records from durable evidence, including terminalized origin
rows and staged turns that predate the index update. Provide bounded reconciliation or
index backfill so such records are not permanently missed. Candidate selection and
eligibility are Rust-owned; Python gathers liveness and artifact facts. Re-read each
candidate under the same adoption lock. Skip live or unknown runners, discarded/killed
stages, explicit holds, missing/failed settlement, and matching completed receipts.
Adopt eligible finalized origins without invoking the originating provider or finalizer
controller. Also discover adopted inline continuations whose delivery is still pending;
complete only that recorded delivery, including the crash after receipt but before the
next provider invocation. Fully delivered success receipts remain no-ops.

Coordinate ordering with dead-process claim cleanup so a recoverable inherited workspace
is not released or reassigned before adoption. Preserve existing carry holds and defer
broader stranded-work policy to the outer carry-hardening phase. If an ACE scheduler
file needs modification, first read `ace.md`; shared decisions still belong in Rust. One
malformed candidate must not abort the pass. Bound rows/time and use a persisted cursor
or existing equivalent; repeated failures must not flood notifications.

Complete the crash matrix below, including runner-versus-CLI-versus-reconciler races.
Use subprocess termination at real durable boundaries for representative gate and
monitor cases so an exception handler cannot hide the crash window. Add a scheduler test
proving the configured job reaches the service and its next pass is a no-op.

## Acceptance and verification

| Situation                                                           | Required observable result                                                               |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Finalization committed, runner dies before adoption                 | One destination; no provider or finalizer rerun                                          |
| Gate published, outer receipt absent                                | Same gate/member/notification and repaired receipt                                       |
| Monitor submitted or already finished, receipt absent               | Same logical monitor and command execution; no second proc                               |
| Pipe/auto continuation prepared, runner dies                        | Same child identity delivered through host launch once                                   |
| Inline success receipt written, runner dies before successor starts | Delivery-only recovery starts the recorded child once; adopter is not repeated           |
| Crash after destination ownership, before custody/receipt update    | Original and destination custody reconcile without releasing the workspace               |
| Matching success receipt replay                                     | No publication/spawn; live inline continuation or detached delivery is handled correctly |
| A repository succeeds and another fails                             | Successful receipt/commit remains; only remaining work is repaired; no early activation  |
| Failed finalizer or stale/malformed proof                           | Explicit refusal with held state and actionable command                                  |
| Manual recovery races scheduler or runner                           | One winning logical adoption; receipt cannot regress to held                             |
| Explicit user kill or discard                                       | No later activation or resurrection                                                      |
| Historical stage lacks sufficient recovery context                  | Clear held/refused result; no invented creator or duplicated work                        |

Each coding phase reads `lint_and_test.md`, runs focused tests for its boundary, then
`sase tool run check` in every changed repository as required by that repository.
Cross-repository phases run the core/binding checks and move the CI pin past the host's
core commit. Use `/sase_monitor` for long verification. Do not run `check-full` unless
explicitly instructed. Keep both flag states covered while both exist. Do not weaken
assertions to accommodate a known base-tree failure.

The child epic's land agent verifies the complete matrix and entry points, the pinned
Rust bindings, consistent docs/help, preserved original final responses, and no pending
phase symbols. Run `sase bead epic-symbols` before closing each assigned phase and again
for `sase-1pj.6`. Resolve symbols or re-key them to a genuinely still-open owner; do not
leave exemptions pointing at closed phases.

When all child work has landed and the acceptance evidence is complete, close
`sase-1pj.6` with
`sase bead close sase-1pj.6 --note "<verified recovery behavior and checks>"`. If a
check failure reproduces identically on the clean base, record a `PROPOSED FOLLOW-UP:`
with reproduction and any existing task ID, then close the completed phase as
instructed. Never close the outer epic `sase-1pj` or another ancestor.
