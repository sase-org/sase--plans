---
tier: epic
title: Durable hijack latches across agent handoffs
goal: Complete sase-1t2.8 with durable, cancellable hijacks that suppress successors
  at safe agent-turn boundaries.
parent_bead: sase-1t2.8
phases:
- id: latch_store
  title: Latch ledger contract and durable application primitives
  size: medium
  depends_on: []
  description: 'latch_store: add additive Rust latch transitions and reconcile policy,
    bindings, and Python target-indexed request/observe/apply/cancel/moot primitives;
    keep public eligibility unchanged.'
- id: spawn_guards
  title: Prevent successor launches from bypassing a hijack
  size: medium
  depends_on:
  - latch_store
  description: 'spawn_guards: carry predecessor identity through follow-up and detached
    launch paths, enforce pre/post-spawn checks, preserve HijackLatchedError through
    every fallback, and prove lock order and child cleanup.'
- id: runner_boundaries
  title: Apply latches at adoption, retry, and run completion
  size: medium
  depends_on:
  - spawn_guards
  description: 'runner_boundaries: enforce latches under staged adoption and successor-delivery
    locks, at in-process and spawned retries, and at no-handoff completion; retain
    suppressed payloads and prevent replay.'
- id: owner_recovery
  title: Reconcile dead owners and support explicit forced stops
  size: medium
  depends_on:
  - runner_boundaries
  description: 'owner_recovery: integrate identity-checked latch reconciliation with
    stale sweeping, implement explicit force after 120 seconds, and preserve uncertain-effect
    and suppressed-handoff evidence through recovery.'
- id: pending_tui
  title: Present pending hijacks and durable cancel or force actions
  size: medium
  depends_on:
  - owner_recovery
  description: 'pending_tui: add pending overlays on members and containers, durable
    cancel/force controls, accepted-latch completion handling, moot notifications,
    help, and deterministic visual coverage before enabling eligibility.'
- id: activate_cli
  title: Enable finalizer latching and expose the CLI contract
  size: medium
  depends_on:
  - pending_tui
  description: 'activate_cli: return latched decisions only for supported finalizing
    or staged agent turns, route the orchestrator to request_latch, add mutually exclusive
    cancel/explain/now flags and exit 3, and document the delivered behavior.'
- id: land
  title: Verify the complete latch flow and prepare phase closure
  size: medium
  depends_on:
  - activate_cli
  description: 'land: extend the real-pipeline canary through finalizing and recovery,
    verify race and replay invariants, run required checks, resolve epic symbols,
    and report closure evidence for sase-1t2.8 to its host completion owner.'
proposed_by: bbugyi200.athena.sase-1t2.8
proposed_with: codex/gpt-6-astra
create_time: 2026-11-16 04:09:02
status: wip
bead_id: sase-1t2.8.1
---

- **PROMPT:** [prompts/202611/hijack_latch_boundaries.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/hijack_latch_boundaries.md)
- **PARENT:** [202611/agent_hijack_dependability.md](https://github.com/sase-org/sase--plans/blob/main/202611/agent_hijack_dependability.md)
- **BEAD:** [sase-1t2.8.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t2/sase-1t2.8.1.md)

# Durable hijack latches across agent handoffs

## Scope and evidence

This child epic implements only `latch_core` from the approved
`plan:202611/agent_hijack_dependability.md`, attached to `sase-1t2.8`. Read that
artifact with `sase artifact read` before implementation. Its decisions remain final.
The research reference named by that plan returned `missing` during planning; the
approved plan supplies the authoritative contract reproduced here.

The bead was in progress with no earlier-work notes. The inspected base was
`8254254e2f`, which includes the real-pipeline canary. Earlier work already supplies
session occupant resolution, the v2 decision wire, target-level deduplication,
transactional immediate stops, recovery, TUI decision caching, and stale-stop repair.
Reuse those implementations. The current Rust ledger has no latch transitions;
`eligibility.rs` still refuses finalizing turns. Python has no latch module, the
orchestrator accepts only `immediate`/`llm`, and the CLI reduces exit values other than
0 or 2 to 1. This phase has not already landed.

The scope needs separate coding agents because the ledger, launch transaction, runner
control flow, stale-owner recovery, and Textual presentation have independent failure
modes. Every phase is a bounded implementation unit. The dependency chain is
intentional: shared files and lock contracts settle before their consumers change. No
phase before `activate_cli` changes public eligibility. This avoids advertising latches
before all enforcement and user controls work, without introducing a flag.

Executing-gate policy/settlement belongs to `sase-1t2.10`; monitor policy and its stop
adapter belong to `sase-1t2.11`. Waiting/processless eligibility and the parent epic's
global chaos/static inventory work remain outside this child epic. Provide reusable
latch APIs for those consumers, but do not enable their decisions here. Suppressing a
staged gate or monitor before adoption is in scope: it is still an agent-turn boundary.
Continue uses ordinary hijack recovery; a separate action to replay the suppressed
handoff remains outside scope.

No memory edits are planned. The parent decision authorizes only
`glossary:agent-hijack`, and the parent land phase already owns that final update. Do
not edit other memory or re-ask the decision. Follow the current sase-core `AGENTS.md`
release convention: versions and changelogs are release-tool owned, despite the older
parent plan's instruction to update changelogs.

## Contracts shared by every phase

### Ownership, durability, and recovery

Rust owns state transitions, eligibility, reconciliation, elapsed-time force
eligibility, and continuation evidence semantics. Python owns filesystem writes, locks,
process observations, signals, snapshots, and launches. Use thin bindings; do not
implement a second Python policy table.

Keep ledger schema version 1 and facts/decision version 2. New Rust fields must have
serde defaults and old records must still advance. Add:

- State `latched`; event `latch`: `requested` to `latched`.
- Event `apply_latch`: `latched` to `stopping`, then the existing confirmed stop to
  `held` once effects and evidence are durable.
- Event `cancel_latch`: `latched` to terminal `cancelled`.
- Allow `finished_first` from `latched`, preserving the real terminal outcome.
- Fields `latched_at`, `applied_at`, `boundary`, `boundary_outcome`,
  `suppressed_followup` as `{kind, ref}`, `forced`, `owner_pid`, and `owner_identity`.

Use the existing target key, `sha256(project_file + "::" + session-or-agent name)`, and
target index. Persist the originating member/artifact and process identity too: an old
request must not silently attach to a later turn with a recycled PID or name. Concurrent
requests return one operation and one hijack ID. Resolve the occupant again inside the
transaction; use the existing one-retry/`occupant_changed` behavior instead of acting on
stale selection.

Expose `request_latch`, `observe_hijack_latch(target_key, boundary)`,
`apply_latch_at_boundary`, `cancel_latch`, and `moot_latch` from
`agent/hijack/latch.py`, with small implementation modules as needed. Observation
returns a structured verdict; callers must distinguish no latch, pending, applied,
cancelled, moot, and ownership uncertainty. Keep a latch from being mistaken for a
claim-transfer error. `HijackLatchedError` is a dedicated exception, never a
`WorkspaceClaimError`, and is explicitly re-raised before broad catches.

Pending requests do not stop the owner or interrupt its finalizer. At application, the
shared helper records `done.json` as `hijacked` while retaining the real outcome in
`boundary_outcome`, holds the exact claim, records the suppressed handoff, collects the
existing repository evidence/snapshots, and durably reaches `held`. Serialize
application ownership and make interrupted steps repairable: retries must converge on
the same marker, snapshots, and record. Do not expose `held` before required evidence is
durable. Never release while a writer survives. Preserve existing pins and restore prior
state on pre-application failure.

Keep suppressed payloads readable for recovery, bounded and labeled untrusted in the
continuation brief. Do not execute or replay their contents during recovery. Active
custody continues to block ordinary successors after `latched` becomes `stopping` or
`held`; the already-authorized recovery successor must still launch exactly once using
its existing resolution/admission identity.

### Locking and launch linearization

Document and test this parent-approved order in the hijack package:

`adoption -> gate/monitor lane -> gate follow-up -> hijack ledger/target index -> patch_lock`.

Define one consistent sub-order for target and per-record locks. Current
`cas_advance_record` moves terminal records and releases the target slot while holding
the record lock; account for that when adding compound transactions. Do not acquire the
same file lock recursively. `patch_lock` stays innermost and is never taken twice in one
process. Use locked variants or an explicit lock context where existing claim/transfer
helpers acquire it themselves.

The spawn transaction must have a single winner relative to latch registration. Perform
a pre-spawn check and re-check in the child claim callback before admitting the child,
with the target/ledger lock acquired before the claim lock. Do not acquire the ledger
lock from inside an already-held `patch_lock`. Propagate latch refusal through the Rust
spawn callback so its existing failure cleanup terminates the child. Verify that cleanup
and claim retention complete before reporting suppression. If a legitimate successor
demonstrably won first, preserve its real result and moot the old latch; never stop an
unrelated new occupant.

## Phase: latch_store

Open `sase-core` with `/sase_repo` and read its `AGENTS.md`. Work in
`crates/sase_core/src/agent_hijack/{wire,ledger,reconcile,continuation}.rs` and
`crates/sase_core_py/src/agent_hijack/`, adding a focused latch policy module if needed.
Implement the additive schema, transitions, terminal classification, and pure
reconciliation: matching live owner keeps pending; identity-proven dead owner with no
successor applies; a real outcome with a proven launched successor finishes first.
Unknown liveness or uncertain successor evidence retains custody with an actionable
error. PID presence alone and permission-denied probes do not prove death.

Include the 120-second force eligibility rule with injected time in the core contract.
Bind and round-trip the new APIs. Extend continuation evidence with the suppressed
kind/path and forced-interruption warning, retaining defaults for v1 records and the
existing prompt budget.

In sase, add thin wrappers and durable primitives using `ledger.py`, the existing target
index, `_stop_claim.py`, `_stop_evidence.py`, and custody helpers. Extract the shared
settlement/evidence work needed by the immediate orchestrator and latch application
without changing immediate stop behavior. Add `cancelled` to terminal handling and
ensure cancellation/moot release only their own target reservation. Keep public
`agent_hijack_decide` behavior unchanged.

Tests cover every legal/illegal transition, legacy v1 defaults, duplicate requests,
cancel versus apply, moot, restart/readback, interrupted application replay, snapshot
failure retention, identity mismatch, and the pure reconciliation/force matrix. Check
the bindings against the workspace wheel and advance `sase-core-revision.txt` past the
host-committed core change before downstream Python binding calls land.

## Phase: spawn_guards

Integrate guard identity and the typed exception in `src/sase/turns/followup_launch.py`,
`agent/launch_spawn.py`, and the relevant detached-child/follow-up wrappers. Pass
explicit predecessor target and artifact identity where necessary; PID or workspace
number alone is insufficient. Ordinary unrelated launches with no predecessor are
unaffected.

Cover every attempt in `launch_turn_followup`, including transfer, fresh claim on the
same workspace, pool claim, and workspace zero. Explicitly re-raise `HijackLatchedError`
ahead of `RuntimeError`, `WorkspaceClaimError`, and generic exception handlers. At the
real spawn funnel, put a pre-check before process creation and a serialized post-check
inside the claim callback using the lock contract above. Preserve the original typed
failure across PyO3; if the existing spawn binding erases it, fix and test that
transport in the linked core checkout.

Barrier tests must place a latch before spawn and between spawn and callback. Assert one
retained predecessor claim, termination of the losing child, no provider work after
refused admission, and no fallback launch anywhere else. Test the opposite winner and
ordinary follow-ups. Test the explicitly authorized hijack recovery successor so these
guards cannot deadlock Continue. Add an assertion-style lock trace test covering
registration, callback, apply, cancel, and terminal cleanup.

## Phase: runner_boundaries

Add checkpoints in these concrete funnels, preserving their existing outcomes:

| Boundary             | Implementation and required outcome                                                                                                                                                                                                                 |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Staged adoption      | `axe/run_agent_exec_staged_dispatch.py::dispatch_completed_staged_handoff`, under the adoption lock before any adopter. Apply and persist `{kind: staged kind, ref: staged payload path}`.                                                          |
| Adoption replay      | Already-adopted receipts, reconciliation, and staged successor delivery must recognize suppressed progression and never replay it. Distinguish a hijacked dispatch from generic failed adoption.                                                    |
| In-process successor | Audit `run_agent_successor_inprocess.py`, staged restoration, pipe, plan, and questions. Enforce the checkpoint before mutating member identity whenever a caller can bypass central dispatch.                                                      |
| Run end              | In `axe/run_agent_exec.py` and its finalization plumbing, a settled turn with no handoff moots the request, keeps its real outcome, and holds nothing solely for the latch.                                                                         |
| Retry                | In-process retry entry in `run_agent_exec.py`/`run_agent_exec_retry.py` and `run_agent_retry_spawn.py` stop progression after an applied latch. Explicitly re-raise the latch exception so spawned-retry failure cannot become an in-process retry. |
| Detached successor   | `run_agent_successor_delivery.py` and staged successor restoration check the source identity before delivery and use the guarded spawn transaction.                                                                                                 |

Persist suppression evidence before acknowledging the boundary. Ensure ordinary runner
shutdown, auto-dismiss/release, failed-adoption repair, and next-loop finalization do
not overwrite the hijacked result or unpin custody. Preserve `boundary_outcome` instead
of replacing the finalizer's own successful/failed result with a guessed value. Replayed
suppression uses the same hijack ID.

Use deterministic barriers to test each boundary, apply versus cancel, and latch versus
adoption. Assert exactly one application and at most one successor; the latch winner has
zero successors. Test crashed/replayed dispatch, inline and detached paths, no-handoff
moot, and successful completion that wins before registration. Leave public eligibility
unchanged until `activate_cli`.

## Phase: owner_recovery

Read `ace.md` before integrating `ace/scheduler/stale_running_sweep.py`. Add latch
reconciliation through the Rust policy from `latch_store`, reusing process identity,
adoption receipts, successor manifests, and admission evidence. Include latched records
whose owner has disappeared from the normal live-row scan. Repair partially applied
latches through the shared helper and existing stopping-record machinery. Never let an
already-adopted successor be suppressed again by stale repair.

Implement the explicit force operation for a still-pending latch aged at least 120
seconds. Reject younger, cancelled, moot, or mismatched operations with specific codes
and next actions; revalidate under the transaction lock. Reuse identity-checked
termination, fatal intent persistence, stop retry limits, and `stop_blocked` custody
from the immediate orchestrator. Set `forced: true` and retain the same hijack ID. Never
run this operation automatically from a timeout or stale sweep.

Carry “effect uncertain — finalizer interrupted; inspect the snapshot” through recovery
preview and the continuation brief. Verify Continue from a boundary-applied or forced
latch retains the workspace and does not replay the suppressed handoff. Tests cover
owner death before/after each durable application step, surviving writers, PID reuse,
unknown liveness, successor proof/mismatch, the 119/120-second boundary, force versus
cancel/application, and repeated sweeps with no duplicate effects.

## Phase: pending_tui

Read `ace.md`, `tui.md`, `tui_perf.md`, and `tui_screenshot.md`. Extend
`models/agent_hijack_presentation.py`, row/status formatting, and the existing
`_hijack_decision_cache.py`/`_hijack_action.py` flow. Pending members and collapsed
session containers display `⏏ HIJACK PENDING · after <boundary>` from the durable
record. Mirror that record through `_agent_status_apply.py`; avoid deriving pending
status from process liveness alone.

On a pending row, `,k` opens Cancel hijack and, only when eligible, Hijack now with the
interruption warning. Route both actions through the existing durable proc submission
and target-key deduplication. Extend the request sidecar and
`ops/commands/_agent_persist_hijack.py` as needed. An accepted latch is successful
submission, remains pending on refresh, and never appears as a generic stop error.
Cancellation and application races report the observed durable result.

Notify a moot transition once with “finished before the hijack; nothing was held”. Track
observed pending IDs and their terminal results through the background loader; do not
scan terminal history from a render callback. Keep all disk I/O and slow work off both
the UI thread and serial message pump. Revalidate selection before applying delayed
responses. Update `?` help, footer/kill-modal parity, and `default_config.yml` only
where its descriptions or bindings change.

Test actions and rendering with real loader-shaped records, member/container parity,
refresh survival, delayed selection, force-age changes, and one-shot moot notices.
Capture deterministic pending and forced-choice visual fixtures, inspect the report and
PNGs, and verify them with the normal and UTC check runs required by the snapshot
memory. Public eligibility still refuses finalizing turns at this point.

## Phase: activate_cli

With all mechanisms in place, update Rust eligibility and facts together. Return
`latched`, `eligible: true`, `adapter: null`, and an additive boundary hint for agent
turns in `provider_done`/`finalizing`/`handoff` or finalizer `declaring`/`executing`,
and for staged-but-not-adopted agent handoffs. Gather the staged/receipt facts durably
in `agent/hijack/facts.py`; do not infer them from row text. Keep locality, capability,
numbered-custody, and successor-attachability constraints. A done marker with a
genuinely pending handoff must be distinguished from a fully completed turn. Retain
unrelated and later-phase refusals/redirects.

Update `stop_agent_for_hijack` to request or coalesce a latch with exit 3, returning the
active record and actual occupant. Route cancel and force to the same backend operations
the TUI already uses. In `main/parser_agent_lifecycle.py` and `agents/cli_hijack.py`,
expose alphabetized `-c/--cancel`, `-e/--explain`, and `-n/--now` as mutually exclusive
options, compatible with `-j` and `-r`.

Preserve existing JSON keys and meanings. Add `member`, `latched`, and boundary details;
keep `mode`, `code`, `next_action`, and `redirect_op`. Accepted latches must exit 3 in
both text and JSON modes, without falling into the current generic-error exit 1 branch.
Preserve `ok`'s existing exit-0 meaning; the explicit `latched` field and exit 3
identify acceptance. Successful cancellation returns 0 with an explicit cancelled
status; a race that has already applied or finished reports that state without
pretending to cancel. Explain remains read-only and returns 2 if no target resolves.
Cover invalid flag combinations and repeated requests.

Update `docs/reference/tui/agent-statuses.md` and
`docs/reference/agents/agent-sessions.md` for finalizer latching, cancellation, moot,
the 120-second force bound, and suppressed-handoff evidence. Update existing hijack
reference text if it still promises immediate-only behavior. Follow `docs_writing`;
regenerate CLI/config/keymap references with `just fix`. Do not describe gate or monitor
adapters as available until their owning phases deliver them.

Run Rust decision/legacy-record tables, binding round trips, Python fact/orchestrator
and CLI tests, and TUI/CLI parity tests for identical finalizing, staged, pending,
cancelled, moot, and forced cases. Ratchet the core pin to this phase's landed core
commit. No accepted mode may lack an enforcing mechanism.

## Phase: land

Extend `tests/fakey/test_agent_hijack_canary.py` with a deterministic finalization
barrier and a real staged handoff. Launch through the normal pipeline, request a hijack
via the durable TUI path or real CLI, observe pending, release finalization, and assert
the intended finalizer effects complete before HIJACKED. Verify one ledger record,
retained numbered workspace, preserved suppression payload, no ordinary successor, and
Continue settling in the same workspace. Add the no-handoff variant that ends moot with
the real outcome and no latch-owned hold. Retain the existing immediate-hijack canary as
a regression test.

Review the completed checkpoint inventory against the current tree. Record its covered
call sites and narrow recovery exemptions for the parent chaos phase, without
implementing that phase's repository-wide static guard here. Confirm cancel, force,
dead-owner recovery, replay, exception propagation, and lock tests cover real control
flow rather than merely mocked call counts.

Run required verification and record the final core revision, tests, and evidence on
this child's land bead. Before any phase closes, run `sase bead epic-symbols` for that
phase and resolve every entry or re-key it to a still-open owner. The planning run found
no entries for `sase-1t2.8`; check it again when implementation is complete.

Closure intent: the child epic's completion owner should settle this child epic and then
`sase-1t2.8` only after all descendants and acceptance work are complete. Phase workers
close only their own assigned beads; this plan is not permission for a phase worker to
close an ancestor. Provide the completion evidence to the host or authorized ancestor
land owner when that restriction applies. Never close `sase-1t2` or another ancestor
plan bead from this work.

## Verification and completion rules

Each phase reads the current `lint_and_test.md`, runs focused meaningful tests, then
`sase tool run check` in every changed repository. In sase-core use its `just test`
wrapper for targeted Rust tests and its wrapped check gate; never bare cargo. Build or
install changed bindings only into the workspace test environment using the documented
local flow. Follow host-owned commit/pin sequencing; do not write a pin to an
uncommitted Rust tree or install globally.

Use `/sase_monitor` for long checks/builds and snapshot captures as required, after
`just fix` or formatting. Wait for staging commands to exit, then finish through
`/sase_final`; never end with an unmanaged command running. No `check-full` is requested
by this plan. Visual verification is separate from the default check and requires
inspection before final completion.

Do not create task beads. Put discovered follow-up work on the assigned phase as
`PROPOSED FOLLOW-UP: <summary — evidence>`. A failure reproduced identically on a clean
base is recorded with reproduction evidence and any existing tracking bead; it does not
leave a completed phase open. Preserve the parent's memory restrictions.

Acceptance is the full `sase-1t2.8` contract: durable pending state survives process and
UI restart; finalizers finish before safe-boundary application; cancellation and moot
release only latch-owned reservations; every listed handoff/retry/spawn race has one
winner and no extra successor; dead owners converge; force is explicit and available
only after 120 seconds; CLI and TUI agree; recovery keeps the same workspace and exposes
suppressed/uncertain evidence; old records still work.
