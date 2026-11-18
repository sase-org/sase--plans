---
tier: epic
title: Resume interrupted chop launch batches without duplicates
goal: Scheduler and service-host stops preserve every accepted chop launch, resume
  its original identities once, and respect explicit clan cancellation.
parent_bead: sase-1tc.3
phases:
- id: journal
  title: Define and persist the crash-safe chop batch contract
  depends_on: []
  size: medium
  description: 'journal: implement Rust batch wire types and reconciliation rules,
    registered PyO3 bindings, and a thin atomic Python store with process-safe ownership.
    See Phase journal.'
- id: ordinary_launch
  title: Journal ordinary launches at each spawn boundary
  depends_on:
  - journal
  size: medium
  description: 'ordinary_launch: persist accepted concrete plans before dispatch and
    record ordinary clan and standalone launches incrementally; add cooperative pause
    hooks without changing ordinary launch semantics. See Phase ordinary_launch.'
- id: typed_launch
  title: Attach typed admission to the durable chop batch
  depends_on:
  - ordinary_launch
  size: medium
  description: 'typed_launch: link each batch to its existing typed admission bundle
    before dispatch, reopen that bundle on replay, and distinguish scheduler pause
    from permanent cancellation. See Phase typed_launch.'
- id: recovery
  title: Resume unfinished batches and honor clan kills
  depends_on:
  - typed_launch
  size: medium
  description: 'recovery: reconcile pending batches before routine guards, thread
    shutdown through workers, preserve lifecycle and once-per state, record explicit
    clan cancellation, and emit durable notices. See Phase recovery.'
- id: acceptance
  title: Prove process-stop recovery and record disposable acceptance
  depends_on:
  - recovery
  size: medium
  description: 'acceptance: exercise process-death windows, repeated restart, cancellation
    and notification behavior with isolated subprocess tests, perform the disposable
    stop/resume demonstration, and record evidence for sase-1tc.3. See Phase acceptance.'
proposed_by: bbugyi200.athena.sase-1tc.3
proposed_with: codex/gpt-6-astra
create_time: 2026-11-17 19:16:06
status: wip
bead_id: sase-1tc.3.1
---

- **PROMPT:** [prompts/202611/chop_launch_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/chop_launch_recovery.md)
- **PARENT:** [202611/backlog_epic_cmdsub_retries_launches.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_cmdsub_retries_launches.md)
- **BEAD:** [sase-1tc.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tc/sase-1tc.3.1.md)

# Interrupted chop batches resume their original launches

## Scope and evidence

This is the implementation child epic for **sase-1tc.3**, the `chop_launch` phase of
`plan:202611/backlog_epic_cmdsub_retries_launches.md`. It addresses task **sase-1c2**.
The parent review's `mirror_fix = retire_mirror` decision remains final and belongs to
its separate beads.db phase. This plan changes no memory notes and introduces no new
reviewer choices.

The original incident launched two of 61 members of `toobig-68`. Restarting the service
killed the routine before it dispatched the other 59. The partial clan then inhibited
the next job invocation. The approved scope requires durable remaining work, idempotent
restart, explicit user-kill cancellation, visible interruption/resume notices, and a
demonstration away from live athena routines.

Inspection baseline: sase `07aedfc8ca`; linked sase-core `c5e60d78`. The phase had no
earlier implementation or remaining-work notes. Its prerequisite retry fix is the sase
HEAD commit. Relevant existing behavior:

- `axe/chop_proposal_planning.py` produces concrete clan/member names and wait
  references without reserving them. Replanning after restart can choose a new clan
  token, so recovery must preserve the original plan.
- `axe/chop_proposal_launch.py` dispatches ordinary clans through one
  `launch_agents_from_cwd` call, then calls `_record_batch_results` after return. Its
  exception path can record partial returned results but cannot survive process death.
  Standalone proposals have an in-process loop as well.
- `agent/multi_prompt_launch_execution.py::spawn_segments_into` has per-slot execution
  records, name reservations, and clan preflight. The existing `on_agent_spawned`
  callback has no result identity. Add a distinct optional progress contract instead of
  overloading that counter callback.
- Typed proposals already have a durable bundle and Rust admission journal via
  `_maybe_dispatch_typed_chop_proposals`, `axe/chop_typed_admission.py`, and
  `agent/launch_admission*`. Their bundle is not a general chop-startup resume
  mechanism, and scheduler shutdown must not permanently cancel its units.
- `axe/lumberjack.py` runs jobs in a `ThreadPoolExecutor`; SIGTERM only flips
  `_running`. `service/host_support.py` retains the approximately ten-second stop bound.
  A launch loop in a worker does not observe `_running` today.
- `axe/_chop_lifecycle_runner.py` finalizes launched runs and removes linkage;
  `axe/_state_chops.py` prunes run history. Pending launch work must outlive both.
- `axe/_state_inflight.py` warns that stopping drops remaining launches. Its wording and
  detection must distinguish recoverable new journals from legacy unjournaled runs.

A side-effect-free baseline reproduction constructed four concrete planned proposals,
replaced only the typed dispatch branch and external batch launcher, and raised
`SystemExit(143)` after the launcher simulated two spawns. Output:

```text
simulated_stop=143; proposed=4; fake_spawned=2; batch_progress_callbacks=0; unlaunched=2
```

No agents, service commands, or durable runtime files were created by that reproduction.
It proves the missing incremental callback boundary; it is not a claim that SIGTERM
itself raises `SystemExit` in a routine. Acceptance below must additionally exercise
actual subprocess termination.

Five sequential phases keep the cross-repository contract, two dispatch paths, runtime
integration, and destructive-to-disposable-process verification within separate
coding-agent scopes. There is no parallel file ownership. Inspection found live work on
macro provenance, targeted log queries, and the artifact viewer; no live agent
identified itself as working on this launch recovery scope. Recheck before editing
because that observation can change.

## Shared design contract

### Durable identity and ownership

Add a versioned batch manifest keyed by routine, job and original run ID. Keep
unfinished manifests under a dedicated job-local pending-batch directory, outside the
prunable run-history payloads. Enumerate that directory for recovery; do not scan the
entire agent archive or rerun scripts to reconstruct proposals. Retain
terminal/cancelled evidence with the corresponding bounded run history only after
pending dispatch ownership has ended.

The manifest contains the accepted concrete proposal plan, stable source indexes and
IDs, exact agent names, clan identity/generation when reserved, declaration or join
roles, dependency references, prompt and launch environment, routing, model/effort,
once-per keys, and the run's preflight/checkpoint/finalization context. Include enough
typed launch data to reopen the original bundle. Store only the explicit proposal launch
inputs, never a dump of ambient environment. Freeze the accepted plan before the first
launch; no new name allocation or script rerun on resume. Finish preflight/reservation
for the batch before spawning; persist concrete reservation evidence before relying on
it.

Rust in `sase_core::axe_chop` owns the versioned wire, validation, transitions, resume
selection, counts, and cancellation decisions. Python owns filesystem I/O, process
observation, notifications, and calls to launchers through a thin binding adapter. Use
additive bindings and a separate journal schema version; do not unnecessarily change
existing chop/admission wire versions.

The Python store serializes read/transform/write with both an in-process mutex and a
dedicated cross-process lock. Durable writes use same-directory temp file, file fsync,
atomic replace, and directory fsync for creation/rename durability. Only one executor
can own a batch; a second routine/manual caller leaves a live owner alone. Do not hold a
store-update lock throughout a slow launch: cancellation must be able to persist while a
member is launching. Use a separate executor ownership lock, establish lock order with
admission/name locks, and test it.

### Replay and the spawn boundary

For ordinary launches persist pending/dispatching intent before process creation and
launched evidence immediately after each result, before moving to the next member.
Reconcile a crash between spawn and acknowledgement against the durable name registry,
spawn artifacts/receipts, and existing chop linkage. A matching member counts as already
launched even if completed or dismissed. A PID alone or a name reservation without spawn
evidence does not prove a launch occurred. Preserve artifact/name reservation identity
across recovery, and use it when joining an existing clan generation rather than
redeclaring that clan.

An existing unrelated owner of the same name is a visible conflict, never an excuse to
allocate a new identity. An uncertain spawn must be reconciled before retry; recover
provably unspawned reservations after the old owner dies, and retain an actionable
blocked result when evidence is genuinely ambiguous. Test both sides of the
spawn/acknowledgement window so this is not implemented as "mark launched before spawn"
or "retry every unacknowledged member."

For typed proposals the existing admission journal/receipts remain the sole per-unit
authority. The outer batch records the original bundle reference and mirrors progress
for chop history; it does not create a competing dispatch ledger. Never rewrite a bundle
on resume or rerun a terminal conditional unit. Preserve skipped-first clan declaration
promotion, wait rewiring, holds, queue capacity and existing uncertain-dispatch
behavior.

### Pause, cancellation and recovery order

SIGTERM requests a pause. Finish and acknowledge the current launch if it returns within
the existing stop budget; start no further member. If SIGKILL interrupts that launch,
the journal and spawn reconciliation carry the uncertainty. Keep the current service
stop deadline unchanged. Signal handlers set state only; workers perform journaling and
notification work at safe boundaries.

Resume is continuation of the already accepted run. Before a routine's normal tick
lifecycle sweep, run-every filtering, script execution, and inhibit guards, reconcile
its pending batches. Reuse the same recovery entry point for a manual invocation
encountering unfinished work. Do not evaluate the partial clan as a new-run inhibitor;
keep normal guards intact for genuinely new runs. Disabled jobs are held visibly rather
than silently resumed or deleted. Dry-run paths remain read-only and never dispatch
pending work.

An explicit **clan-generation kill** writes durable batch cancellation before signalling
members or deleting their artifacts. Recovery permanently discards that batch's
remaining launch intent. Killing an individual member must not cancel its siblings; a
service/routine stop, member completion, dismissal, or missing PID must not be
interpreted as a clan kill. Carry the explicit container selection through TUI cleanup
expansion to a shared backend operation; all applicable clan/bulk kill entry points use
that operation. The shared operation also cancels the linked typed admission bundle, so
its detached coordinator cannot continue after the clan was killed. Cancellation may
allow the one already-crossing launch to settle, but must win before the next dispatch.

Checkpoint and once-per bookkeeping remains associated with the original run. Paused
members retain their claims; definitive skips/failures/cancellation release only the
relevant unlaunched claims. Finalizers and registry/history cleanup cannot pronounce the
batch complete while dispatch work remains.

Log interruption, resumption and cancellation against the original run. Use the existing
notification store with stable batch/event dedupe keys: one notification per
interruption episode and one per resume episode, naming routine/job/run, clan(s), and
launched/remaining counts. On hard kill, restart detects and reports the previously
unreported interruption. Repeated polls or a second restart of a completed batch must
neither relaunch nor repeat settled notices. Persist enough event identity to recover a
crash during notification delivery.

## Phase journal

Implement the shared contract and its thin store; do not activate startup recovery yet.

1. Open sase-core with `/sase_repo`, read its `AGENTS.md`, and add focused wire and
   transition/reconciliation modules under `crates/sase_core/src/axe_chop/`. Use
   explicit observations for matched spawn evidence, reservation-only state,
   foreign-name conflict, live owner, pause and clan-generation cancellation.
2. Register additive bindings in `crates/sase_core_py/src/axe/`, add round-trip binding
   coverage, and expose the thin sase facade under `src/sase/core/`. Avoid Python copies
   of the state machine. Coordinate the containing core revision pin through host
   completion or `just ratchet-core-revision`.
3. Add the batch store under `src/sase/axe/` with path helpers, bounded pending
   enumeration, atomic writes and separate mutation/executor ownership. Reserve
   original-run context and typed bundle reference fields in the schema now.
4. Test validated round trips; unknown schema/corrupt data refusing dispatch;
   restart-safe writes; stale and live ownership; duplicate acknowledgement; terminal
   skips; cancellation priority; and concurrent update exclusion. Cancellation/phase
   changes must not be inferred from process liveness alone.

Done when Rust and binding tests prove the transitions, Python store tests prove
durability/ownership, and both changed repositories pass the required check workflow.
Record the callable API and lock order for later phases. If intentional inter-phase
symbols need an epic whitelist, key it to the still-open child epic, never this phase
when it closes.

## Phase ordinary_launch

Integrate the journal into standalone and normal multi-prompt clan launches.

1. At the accepted-plan boundary in `chop_runner_script_result.py` /
   `chop_runner_script_result_launch.py`, persist full original run context and the
   concrete accepted plan before any dispatch. Preserve dry-run behavior, all-member
   preflight, proposal collision handling and existing dependencies.
2. Thread optional stop/progress hooks through the relevant launcher facade,
   `multi_prompt_launcher.py`, `multi_prompt_launch_execution.py` and
   `launch_executor.py` as needed. Report the execution record, source index and slot
   identity immediately at the spawn boundary; do not wait for list return. Keep the
   existing spawn counter callback and non-chop callers compatible.
3. Reuse stable reservations and per-spawn artifacts/linkage to cover the
   spawn-before-acknowledgement gap. Add proposal identity to ordinary chop linkage
   where needed. Journal write failure stops further dispatch visibly. Preserve full
   preflight and clan generation on a suffix-only replay.
4. Add an explicit paused outcome carrying partial results. It is not a generic launch
   failure and must not release still-pending once-per claims. Accept the cooperative
   pause hook now; routine signal wiring arrives in `recovery`.
5. Add targeted tests for k/n progress, stopping before the first and after the last
   member, an exception with partial results, a lost acknowledgement, standalone
   dependencies, clan declaration/join and ordinary callers without callbacks. Cover
   fan-out source-slot mapping rather than assuming `zip` is sufficient; preserve
   existing rejection if a proposal shape is unsupported.

Done when ordinary launches journal each actual member before advancing, and direct
replay through the new adapter uses the same names and launches only provably missing
members. Existing multi-prompt naming/preflight regressions must still pass.

## Phase typed_launch

Bind typed admission to the batch without changing its existing semantics.

1. Split typed prepare/persist from dispatch in
   `chop_proposal_launch.py::_maybe_dispatch_typed_chop_proposals`. Persist the stable
   bundle identity and digest in the outer manifest before its first dispatch. Make
   interrupted bundle creation discoverable/idempotent; reopening must not overwrite
   journal state or freshly expand mutable prompt inputs.
2. Reopen the existing bundle on recovery and use its admission lock, replay, dispatch
   fingerprints and receipts. Reconcile outer counts/launch rows from this authority,
   including outcomes produced by a detached coordinator.
3. Add a cooperative pause input distinct from `cancelled` to the admission driver where
   necessary. Scheduler pause must not call `_cancel_open_units`, publish permanent
   cancellation, or start a replacement coordinator after the routine has begun
   stopping. Explicit user cancellation keeps its terminal semantics. A still-live
   coordinator prevents duplicate adoption.
4. Test stop/reopen with launched, skipped and waiting units; skipped first clan member;
   dependency rewiring; a dispatch receipt lost before outer progress; original bundle
   reuse; and repeated recovery with a live coordinator. Reuse
   `test_axe_chop_proposal_launch_clan_dispatch.py`,
   `test_axe_chop_lifecycle_typed_admission.py` and admission stop/failure tests.

Done when the same bundle can finish the batch after a routine stop, skipped units never
re-evaluate, and ordinary/direct typed launches retain their old public behavior. Expose
the unified resume entry point consumed by the next phase.

## Phase recovery

Activate recovery, cancellation and visible lifecycle handling.

1. Wire a routine-local thread-safe stop signal through `run_configured_chop_once`,
   script-result dispatch, both launch paths and the worker loop. Prevent a new worker
   launch after shutdown is requested; allow the current dispatch to acknowledge.
   Preserve the host's stop timeout.
2. Reconcile pending batches at routine startup/tick before lifecycle cleanup or new
   work, and on manual encounter through the same shared entry point. Resume saved
   inputs and the original run ID. Do not consume fresh trigger events, rerun job
   scripts, or allocate another clan. Hold disabled jobs and live owners; ensure
   maintenance policy still prevents new dispatch.
3. Make `_chop_lifecycle_runner.py`, run-history pruning and once-per cleanup account
   for outstanding manifests. A partial batch whose initial agents finish remains
   resumable. Completion merges old and new launch records once.
4. Implement the shared explicit clan-generation cancellation operation and thread it
   through confirmed TUI clan/marked/bulk cleanup paths before member expansion loses
   the selected container identity. Read `ace`, `tui` and applicable child memory before
   UI edits. Reuse shared kill entry points for any existing non-TUI caller; do not add
   a new CLI surface for this feature. Persist the cancellation before signals/artifact
   deletion, and make typed coordinator cancellation observe it. Update affected help
   text if needed.
5. Emit deduplicated run logs and notifications, and correct `_state_inflight.py` to
   describe resume for journaled batches while preserving the warning for older
   unjournaled runs. Legacy history is readable but is not automatically turned into
   executable launch intent.
6. Test recovery before clan guards/run-every; old members already terminal;
   routine/manual ownership races; shutdown propagation; disabled/maintenance behavior;
   cancellation before cleanup; individual-member isolation; a reused clan name in
   another generation; and one notice per durable event.

Done when an unfinished batch resumes automatically without being inhibited by its own
clan, and a confirmed clan kill permanently prevents its continuation.

## Phase acceptance

Prove the integrated behavior, repair defects within this scope, and leave the closure
evidence to the child epic's land agent.

1. Build a deterministic subprocess harness in the test suite using an isolated SASE
   state root and disposable job. Exercise production journal/recovery and routine
   signal wiring. Replace only the actual provider/agent workload with harmless
   processes and durable fake launch receipts. Never call `systemctl`, restart the real
   scheduler/service, or run live `toobig_split`.
2. Use explicit pipe/file handshakes, not timing guesses. For both ordinary and typed
   dispatch, terminate after k of n successful spawns; restart in a fresh process;
   verify n distinct original identities and exactly n total spawns. Restart again and
   assert zero new launches. Include k=0 and k=n boundaries.
3. Cover SIGTERM finishing the current launch and stopping before the next, and SIGKILL
   during a deliberately held launch after spawn but before its outer acknowledgement.
   Prove identity reconciliation prevents duplication. Also kill before spawn after
   reservation and prove missing work is not skipped.
4. Exercise explicit clan kill while paused and while recovery is active,
   deletion/dismissal of launched artifacts after cancellation, typed skipped units, a
   still-live coordinator, and recovery notifications. Confirm retained cancelled intent
   never revives when runtime files are cleaned up.
5. Perform and record a separate manual invocation of this disposable harness:
   total/k/remaining counts, original clan/run identity, termination mode, observed
   notifications/logs, first restart result and second restart result. Run the clan-kill
   variant too. Store evidence on this phase and on `sase-1tc.3`; no real provider calls
   are required.
6. Run focused Rust/binding and Python regressions and the required repository checks.
   Relevant existing suites include `test_axe_chop_clan_launch.py`,
   `test_multi_prompt_launcher_launch.py`, `test_axe_lumberjack_tick.py`,
   `test_axe_inflight_launches.py`, chop lifecycle/once-per/typed admission tests, and
   changed clan cleanup tests. Verify the sase core pin contains every new binding.
   Record any identical clean-base failure precisely.

Done when the three parent acceptance requirements are demonstrated: only n-k remaining
members launch after interruption, a second restart launches none, and a user-killed
clan's pending batch is discarded. Record
`TASK RESOLVED: sase-1c2 — <tests and disposable demonstration evidence>` on
`sase-1tc.3`; close only the assigned acceptance phase, not any ancestor/task.

## Rules for every implementation phase

- Read the assigned phase, this design and its notes before editing; check which earlier
  work has already landed. Recheck `sase agent list` and active plan beads for
  overlapping live ownership. If overlap is real, record a `PROPOSED FOLLOW-UP:` on the
  assigned phase and coordinate with the owning epic through a bead note rather than
  editing its files blindly.
- Open linked repos only through `/sase_repo`. Shared behavior belongs in Rust;
  registered bindings and tests accompany it. Let host completion move a containing
  `sase-core-revision.txt` or use `just ratchet-core-revision`; never fabricate a SHA or
  globally install a wheel.
- Read `lint_and_test` and applicable domain memories. Run focused regressions,
  formatting, then `sase tool run check` in each changed repository. Do not run
  `just check-full`, weaken assertions, add suppressions/xfails, raise timeouts, or
  change flake baselines. Use `/sase_monitor` for long final checks and finish the
  staged handoff through `/sase_final`.
- Do not create task beads or edit memory. Record discovered work as
  `PROPOSED FOLLOW-UP:` notes on the assigned phase. A failure reproduced identically on
  the clean base is evidence for such a note, not a reason to strand a completed phase.
  Cite an existing tracking task when known.
- Before closing the assigned phase run `sase bead epic-symbols <phase-id>` and resolve
  or re-key all remaining entries to a still-open epic/later phase. Close only that
  phase with a verification note, and finalize through `/sase_final`. No phase worker
  closes its parent or an ancestor plan.

## Landing responsibility

The child epic's land agent verifies the complete contract and the recorded disposable
demonstration, resolves any child-epic integration defects, checks all phase symbols,
and follows normal child-epic closure rules. Only after that may it close **sase-1tc.3**
with the combined verification evidence; first run `sase bead epic-symbols sase-1tc.3`
and resolve/re-key any leftovers. It must not close **sase-1tc**, any higher ancestor,
or task **sase-1c2**. The outer epic's lander owns task closure and the backlog
continuation, and receives the `TASK RESOLVED` / `PROPOSED FOLLOW-UP` notes as evidence.
