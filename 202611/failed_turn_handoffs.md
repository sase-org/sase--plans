---
tier: epic
title: Make failed-turn staged handoffs stop hiding stranded work
goal:
  A never-started staged handoff whose originating turn failed stops counting as live
  ownership and stops blocking later work, while finalizer authorization, submitted
  execution, recovery evidence, and workspace retention remain intact.
phases:
  - id: inactive-stage-policy
    title: Rust policy for inactive staged handoffs
    depends_on: []
    size: medium
    description:
      "inactive-stage-policy: implement and bind shared inactive-stage decisions,
      correct stranded ownership, and test terminal origins, active successors,
      malformed evidence, and pending adoption without changing host side effects."
  - id: inactive-stage-host
    title: Settle failed origins and unblock later turns
    depends_on:
      - inactive-stage-policy
    size: medium
    description:
      "inactive-stage-host: collect origin and execution facts, apply idempotent stage
      settlement under existing locks, remove stale pending ownership, and exercise
      failed-monitor recovery and questions admission end to end."
proposed_by: bbugyi200.athena.sase-1vn.3
proposed_with: codex/gpt-6-astra
parent_bead: sase-1vn.3
create_time: 2026-11-27 16:41:33
status: wip
---

- **PROMPT:**
  [prompts/202611/failed_turn_handoffs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/failed_turn_handoffs.md)
- **PARENT:**
  [202611/backlog_epic_1.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_1.md)

# Make failed-turn staged handoffs stop hiding stranded work

This child epic implements phase **sase-1vn.3**, `failed-turn-handoffs`, of
`plan:202611/backlog_epic_1.md`. Its incident is **sase-1sh**: monitor `3w69pb9ezmfy`
never started after its originating phase turn failed, yet its stage kept the phase
apparently owned and its land agent waiting for five days.

The phase has no earlier-work notes. The current code still has the ownership defect;
related delivered-stage and dead-monitor fixes do not cover this case. Use two
sequential medium phases because the deterministic ownership contract and the host's
resource settlement have different failure modes. Each phase ships its own tests. The
land agent verifies their integration and supplies completion evidence for sase-1vn.3.

## Governing decisions and boundaries

- `decisions:rust-core-required`: classification and settlement eligibility live in the
  linked `sase-core` crate, exposed through `sase_core_rs`. Python observes host facts
  and performs authorized I/O; it must not grow a duplicate policy.
- `decisions:handoffs-end-through-host-completion`: a failed finalizer must never cause
  automatic adoption. Making a held stage inactive is not permission to run its command
  or launch its successor.
- `decisions:host-owned-completion`: use host final declarations for commits and the
  coordinated Rust source pin. Do not commit manually.
- `decisions:check-full-is-explicit`: use `sase tool run check` in each changed
  repository. This phase does not inherit the test-cost phase's authorization to run
  `check-full`.

The parent's decisions are final: `cost_budgets = rebaseline` and
`zombie_recovery = relaunch`. Neither needs another decision here. Do not change memory
notes, test-cost policy, zombie-epic relaunch policy, or TUI presentation. Related beads
**sase-1r4** (owned by sase-1ra), **sase-1si**, and **sase-1sj** remain outside this
scope. Preserve the separately implemented delivered-stage and stale-`followup_pending`
behavior.

Before reading or editing sase-core, use `/sase_repo` to open it and read its
`AGENTS.md`. All paths below are relative to the named repository.

## Confirmed implementation facts

In sase:

- `src/sase/axe/run_agent_exec.py` calls `dispatch_completed_staged_handoff` after the
  workflow and on workflow errors. A refused dispatch makes the originating turn fail.
- `axe/run_agent_exec_staged_dispatch.py` refuses adoption when
  `finalizer_settlement_refusal` cannot authenticate same-turn settlement. It records
  `handoff_not_adopted`, retains origin work, and invokes held monitor cleanup. This
  fail-closed behavior is intentional.
- `axe/run_agent_exec_staged_recovery.py` asks Rust for the replay decision. Automatic
  recovery of a held receipt requires explicit readoption; it does not turn a hold into
  a running successor.
- `bead/_stranded_facts_members.py` collects staged kind, receipt outcome, delivered
  state, lifecycle, and retention. Terminal workspace retention already suppresses
  ownership, but it is not a complete solution for historical stages or holds without
  unresolved retained work.
- `agent/_staged_handoff_admission.py::occupying_handoff_names` counts the staged
  filename as occupying. `monitor/staged_owner.py` also keeps a staged owner index.
  `monitor/start_stage_cleanup.py` intentionally preserves the outer record and index on
  held cleanup. Submitted proc evidence must survive this path.
- `axe/staged_handoff_reconcile.py` already provides bounded historical discovery and
  host recovery. Reuse it; do not add a second scanning service.

In sase-core:

- `crates/sase_core/src/stranded_work/classify.rs::owner_kind` treats every undelivered
  stage and every non-adopted receipt as ownership, independently of the originating
  turn having failed.
- `staged_handoff/monitor_lifecycle.rs` makes staged and held monitor membership
  nonterminal and lane-blocking. Its staged-owner predicate checks durable stage
  presence and proc absence but lacks originating-turn completion evidence.
- `staged_handoff/recovery.rs` correctly preserves finalizer proof, explicit readoption
  of held stages, receipt identity, and adopted-receipt precedence.

The relevant regressions already live in `stranded_work/tests.rs`,
`staged_handoff/tests.rs`, `crates/sase_core_py/src/stranded_work/tests.rs`, and sase's
`tests/test_stranded_staged_handoff.py`, `tests/test_staged_recovery_service.py`,
`tests/test_staged_handoff_reconcile.py`, `tests/test_staged_monitor_contracts.py`, and
`tests/agent/test_staged_handoff_reclaim.py`.

## Shared behavior contract

Keep **execution authorization**, **live ownership**, and **saved recovery evidence**
separate. A durable failed stage is evidence even after it stops claiming that work will
run.

| Observed state                                                                                       | Required behavior                                                                                       |
| ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Valid never-adopted stage; identified origin ended failed; no submitted execution or active recovery | No live handoff ownership; settle grace then `stranded/failed`; eligible to settle pending reservations |
| Matching held receipt on a terminal failed origin, including historical records without retention    | Same inactive ownership behavior; retain the reason and saved payload                                   |
| Origin running or finalizing; no durable terminal evidence                                           | Preserve pending ownership and admission protection                                                     |
| Origin ended successfully with adoptable, undelivered handoff                                        | Preserve the existing adoption/recovery window                                                          |
| Matching adopted receipt, submitted proc, or delivered successor                                     | Preserve execution/delivery evidence; do not clean it up as never-started                               |
| Active successor or explicit recovery                                                                | That live member/recovery owns the bead, even if the old origin is inactive                             |
| Malformed, mismatched, remote-only, or contradictory evidence                                        | Do not infer permission to clean up; preserve conservative existing handling                            |

Do not use age alone to infer terminal state. Distinguish confirmed absent proc state
from a failed proc-store read. A durable origin completion marker can prove the turn
ended even when a never-started monitor has no PID; a missing PID alone is insufficient.
A receipt by itself never proves a command ran.

## Phase `inactive-stage-policy`

Implement the shared decision in `sase_core::staged_handoff`, with typed wire inputs for
stage/receipt identity and validity, origin terminal outcome, observed execution,
finalization, and recovery. Return explicit inactive/pending and settlement-eligibility
results with stable reasons. Keep observation uncertainty distinct from absence. Reuse
this decision for pending-owner behavior rather than introducing separate lifecycle
rules in each caller.

Extend the stranded member contract only as needed to carry those facts or the
core-derived inactive result. Make `owner_kind` ignore stale staged and held ownership
for an inactive origin, without allowing an earlier generic pending handoff branch to
restore the same stale ownership. Preserve independent live successors, active recovery,
delegation, settlement grace, failure precedence, and blocked-ancestor propagation. The
staged monitor member associated with the failed origin must not independently recreate
ownership through synthetic `process_live` or pending membership.

Expose and register any new binding in the existing staged-handoff binding domain, and
add round-trip tests. Default omitted new fields conservatively so old producer payloads
keep parsing. Follow both repositories' wire-version rules if a schema change requires a
version increment; update all mirrors, validators, and fixtures together. Do not change
receipt outcomes merely to express liveness.

Tests must include a failed origin plus an unstarted monitor, with and without a held
receipt and workspace-retention marker; a receipt-only failed-origin fixture; settlement
grace; a live successor; recovery; live/finalizing origins; successful pending adoption;
adopted/delivered stages; malformed and mismatched state; and confirmed absence versus
unknown execution. Shuffle member order to check deterministic reduction. Assert the
phase is stranded and its waiting ancestor is blocked, rather than only asserting one
helper returns false.

Run targeted Rust tests through the repo's `just test` wrapper, then its guarded
`sase tool run check`. Never run bare cargo. This phase can land the additive contract
before the host consumes it. Any temporary epic-symbol entry must be keyed to the
still-open consumer phase or child epic, never to the closing phase.

## Phase `inactive-stage-host`

Wire the Rust contract through `src/sase/core/staged_handoff.py` and the stranded
facade/collector. Collect origin identity, terminal state, stage identity, finalizer
state, and proc/destination observations from existing persisted facts. Preserve
uncertainty. Specifically test `_process_live`'s synthetic monitor visibility so an
unstarted monitor does not appear live merely because it is listed.

Add identity-scoped, durable inactive settlement evidence under the existing
handoff/adoption locking protocol. It must identify the stage and origin and retain the
failure reason. Retain the original stage, receipt, launch payload, response, and
recovery context as recovery evidence. Prefer a separate versioned settlement marker
over overloading the adopted receipt. Readers must match the stage identity; an old
marker must never suppress a replacement stage.

The host applies settlement only after re-observing the core policy's eligibility under
the relevant locks. Use the normal terminal-origin boundary for new failures and the
existing bounded reconciler for historical stages. The inline failed-dispatch path runs
before terminal completion, so do not rely solely on writing a terminal fact there.
Share one idempotent implementation between those paths. Read-only stranded collection
must not itself mutate resources.

For an eligible never-started monitor, reuse existing cleanup to settle an owned
reserved ToolRun as never executed, release only its staged join/prepared reservation,
and remove its pending-owner/lane occupancy. Preserve workspace retention independently.
Never stop a joined run or discard an already-submitted proc; unreadable proc state must
not enter the absence branch. Repeated cleanup and restart after partial settlement must
converge without duplicate effects.

Make active handoff/monitor admission and pending-owner queries honor the
identity-matched inactive settlement. Keep saved records available for explicit
`sase agent readopt`; successful readoption still needs existing authenticated finalizer
proof and must reestablish any necessary reservations before activation. It must
atomically invalidate inactive occupancy state when execution genuinely resumes.
Automatic reconciliation must not readopt a held stage. Do not allow a fresh stage to
overwrite another stage's saved files: later agent work uses its own turn artifacts, and
the original turn remains recovery evidence.

Exercise the actual later-turn questions/admission path, identifying and removing the
stale session/lane ownership obstruction rather than merely hiding a display row. A
still-running origin's one-handoff restriction remains valid. Keep normal stranded
notification and recovery paths; if existing held-notice deduplication needs adjustment
for newly inactive historical records, make it stage-scoped and verify that repeated
reconciliation does not send duplicate notifications.

Add integration fixtures that stage a monitor, fail its origin without executing the
command, reconcile, and verify all of the following:

1. After grace the phase reports `stranded/failed`, and ancestor waiters report the
   blocked phase using the existing reducer.
2. A fresh recovery turn can stage questions and later work without inheriting the dead
   monitor's pending slot.
3. No command or successor launched during settlement; the never-run resource is
   settled, while saved evidence and unresolved workspace retention survive.
4. Reconciliation is idempotent, and explicit readoption after repaired finalizer proof
   can still activate exactly once.
5. A racing adoption, submitted proc, live successor, unknown store read, different
   stage identity, or incomplete settlement cannot cause destructive cleanup.

Run focused Python integration tests against a locally built changed Rust binding using
the repository's supported local environment workflow, then `sase tool run check` in
both changed repositories. Use `/sase_monitor` for long builds or verification, staging
before ending through `/sase_final`.

## Integration, pin, and completion evidence

The child land agent verifies the two phases together. `sase-core-revision.txt` must
name a commit containing the new Rust behavior before sase consumers land. For a final
declaration containing changes in both repositories, the configured host finalizer
commits and pushes sase-core first and writes its SHA to the pin before committing sase;
see `docs/contribute/rust-backend.md`, “The CI source revision pin.” If the core change
has already landed, use the supported revision ratchet and verify the resulting pin
contains it. Do not guess a future SHA or leave a follow-up pin bump as unfinished work.

Acceptance requires the failed-turn regression, reservation/admission regression,
preserved successful adoption/readoption coverage, and checks in both repos. Record
check failures that reproduce identically on the clean base as `PROPOSED FOLLOW-UP:`
notes, with existing task IDs where known; they do not leave an otherwise completed
phase open. Workers create no task beads.

Each worker runs `sase bead epic-symbols <its-assigned-phase>` before closing only its
assigned phase. Resolve or re-key every remaining symbol to an open owner. The child
land agent supplies the complete implementation, pin, and verification evidence for the
enclosing sase-1vn.3 completion, including its epic-symbol audit. Use the host's
delegated-phase completion path if available; if an explicit sase-1vn.3 close remains
necessary, only the agent authorized to finish that assigned phase performs it. A child
phase worker never closes an ancestor.

Do not close sase-1vn or any ancestor plan bead. Although the older parent plan also
asks to close sase-1sh, this invocation's narrower instruction is to close only the
assigned phase: leave a clear sase-1sh resolution recommendation for sase-1vn's land
agent. Its evidence must mention the separate sase-1r4, sase-1si, and sase-1sj scopes.
