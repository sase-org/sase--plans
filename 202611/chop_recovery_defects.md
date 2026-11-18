---
tier: epic
title: Make chop batch recovery correct against production evidence
goal: Interrupted chop batches resume exactly the missing members with real chop linkage,
  a live owner or explicit clan kill always wins, and every hold, failure and episode
  is visible.
parent_bead: sase-1tc.3.1
phases:
- id: evidence
  title: Reconcile spawn evidence by run linkage and replay with the original launch
    context
  depends_on: []
  size: medium
  description: 'evidence: match chop-agent records by run and proposal index instead
    of cl_name, classify foreign owners, replay with chop linkage env, origin, clan
    generation and rewired waits, merge resumed launches into the original run, and
    make the test fakes production-shaped. See Phase evidence.'
- id: ownership
  title: Single executor ownership, crossing cancellation and consistent claims
  depends_on:
  - evidence
  size: medium
  description: 'ownership: hold batch ownership during first-run ordinary and typed
    dispatch, settle a launch that crosses a clan kill without failing the run, stop
    typed dispatch on outer cancellation, keep once-per claims for journaled pending
    members, and stop lifecycle finalization while any dispatch work remains. See
    Phase ownership.'
- id: visibility
  title: Visible holds, bounded retention, complete clan-kill coverage and per-episode
    notices
  depends_on:
  - ownership
  size: medium
  description: 'visibility: surface every recovery hold and failure, stop silent per-tick
    retry loops, retire settled manifests with run history, cancel every clan of a
    batch from all TUI panel kill paths, and give each interruption/resume episode
    its own durable notice. See Phase visibility.'
- id: acceptance
  title: Re-prove recovery with production-shaped evidence and record corrected acceptance
  depends_on:
  - visibility
  size: medium
  description: 'acceptance: rerun the subprocess harness with real chop linkage records
    and name registry evidence, add the routine/manual race and crossing-cancel cases,
    repeat the disposable demonstration, and record corrected TASK RESOLVED evidence
    on sase-1tc.3. See Phase acceptance.'
proposed_by: bbugyi200.athena.sase-1tc.3.1.land
proposed_with: claude/opus
create_time: 2026-11-18 07:18:59
status: done
bead_id: sase-1tc.3.1.6
---

- **PROMPT:** [prompts/202611/chop_recovery_defects.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/chop_recovery_defects.md)
- **PARENT:** [202611/chop_launch_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202611/chop_launch_recovery.md)
- **BEAD:** [sase-1tc.3.1.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tc/sase-1tc.3.1.6.md)

# Make chop batch recovery correct against production evidence

## Why this epic exists

This is remaining work for epic **sase-1tc.3.1** (`plan:202611/chop_launch_recovery.md`,
read it first). Its five phases landed at sase `c6b6da1d2c`, `558fbd127f`, `3cf982bdff`,
`b8c9f6c6aa` and `07479640e2`, with the Rust contract at sase-core `6cd9a0c2` (pinned by
`sase-core-revision.txt`). The land agent's verification found the following defects in
that work. Each was confirmed by reading the code at `07479640e2`; line numbers refer to
that commit. They are epic defects, not follow-ups. The shared design contract and
decisions of the parent plan stay final. This plan changes no memory notes.

The acceptance harness hid the most serious defect.
`tests/axe/chop_batch_process_worker.py` patches `get_chop_agent_records` to return
`SimpleNamespace(cl_name=<agent name>)`, and its fake launcher falls back to the planned
name when `SASE_CHOP_PROPOSAL_INDEX` is missing.
`tests/axe/test_axe_chop_batch_recovery.py::test_collect_observations_marks_recorded_spawn_as_matched`
uses the same false shape. Production records never look like that. The earlier
`TASK RESOLVED: sase-1c2` note on `sase-1tc.3` therefore rests on evidence the
production code cannot produce.

### Verified defects

1. **Spawn evidence matches the wrong field**
   (`src/sase/axe/chop_batch_recovery.py:75-129`). `collect_spawn_observations` compares
   member `agent_name` against `_ChopAgentRecord.cl_name`. In production `cl_name` is
   the project name or VCS ref (`src/sase/agent/launch_cwd_single.py:121-128`, recorded
   at `src/sase/agent/launch_spawn.py:613-628`). Its prefix rules can also false-match,
   for example agent `sase-fix` against project `sase`. As a result a member spawned
   before its acknowledgement is never `matched_spawn`. Any visible agent name,
   including the member itself and an unrelated owner, becomes `reservation_only`, so it
   is relaunched. `foreign_conflict` and `live_owner` are never emitted. Records already
   carry `run_id`, `proposal_index` and `proposal_id`
   (`src/sase/axe/chop_agents.py:51-70`).
2. **Replay does not reproduce the original launch** (`chop_batch_recovery.py:515-551`).
   - `_dispatch_member` passes only the stored proposal env plus
     `SASE_AGENT_PLANNED_NAME`. It never adds `build_chop_launch_env(...)` or
     `origin="generated"`, unlike the first run
     (`src/sase/axe/chop_proposal_launch_impl.py:1028-1052, 1112-1136`). Resumed agents
     therefore get no chop-agent record and no run linkage, and a second crash has no
     evidence to reconcile.
   - The stored `clan_generation` is never used, so a joiner can join the newest
     generation and a declarer can declare a new one.
   - The stored prompt is the planning-time `plan.prompt`, so a dependent of a skipped
     member still waits on the skipped name. The first run rewires that wait
     (`chop_proposal_launch_impl.py:1093-1108`).
   - The original run's `entry.launches` is never updated with resumed launches, so
     lifecycle finalization pronounces the run finished when only the first k agents
     finish.
3. **The first-run executor never owns its batch.** Only `chop_batch_replay.py:51` and
   `chop_typed_admission.py:138` call `acquire_chop_batch_ownership`. Recovery runs
   before the `active_script_chop_run` overlap guard
   (`src/sase/axe/chop_runner_script.py:~92`, `src/sase/axe/lumberjack.py:~315`). A
   routine tick or manual run can therefore adopt a batch that another process is still
   launching. It then relaunches the same names and emits false interrupted/resumed
   notices.
4. **A clan kill that crosses a launch fails the run**
   (`chop_proposal_launch_impl.py:777-890`). `cancel_clan_generation` moves
   `dispatching` members to `cancelled`, and Rust then rejects `acknowledge_launch` on
   the cancelled batch (`batch.rs` `require_batch_dispatchable`). `_journal_launch`
   raises before `launch_recorded_fn`, so the run records `action_failed` and loses the
   spawned agent. The typed path has the same failure in `_mirror_typed_launch`
   (`src/sase/axe/chop_typed_admission.py:320-354`), which leaves the admission unit
   dispatching with no receipt.
5. **Typed dispatch ignores outer cancellation** (`chop_typed_admission.py:357-377`).
   `_ensure_typed_member_dispatching` skips a cancelled member silently and dispatch
   continues. Only the bundle cancel marker stops it, and `_cancel_linked_typed_bundle`
   swallows any failure to write that marker (`chop_batch_recovery.py:320-339`).
6. **Once-per claims contradict the retained journal**
   (`src/sase/axe/chop_runner_script_result_launch.py:160-166`). A launch exception
   releases the claim of every unlaunched proposal, but the batch stays open and
   recovery replays those members. A newer run can claim the same key and launch a
   duplicate.
7. **Lifecycle finalization only waits for `paused`**
   (`src/sase/axe/_chop_lifecycle_runner.py:52-62`). An `open` batch that still has
   pending members, such as one retained after a launch exception, is finalized while it
   is resumable.
8. **Recovery holds and failures are invisible.**
   - `_recover_ordinary_batch` catches only `ChopBatchError`, so any other launch error
     aborts recovery for the remaining jobs.
   - `Lumberjack._recover_pending_batches` (`lumberjack.py:~315-327`) and the manual
     caller (`chop_runner_script.py:~511-519`) discard results and swallow every
     exception.
   - Disabled, maintenance, live-owner and error holds therefore produce no log line or
     notice, and a deterministic launch failure retries silently on every tick.
9. **Manifests never retire.**
   - `delete_chop_batch` has no production caller.
   - `list_pending_chop_batches` (`src/sase/axe/chop_batch_store.py:231-260`) applies
     its 256-file cap before filtering for unfinished batches. Once 256 newer runs
     accumulate, an older unfinished batch drops out of recovery and clan cancellation.
   - Each tick also rereads up to 256 manifests per job.
10. **Clan-kill coverage gaps.**
    - `_bulk_kill_panel_agents`
      (`src/sase/ace/tui/actions/agents/_kill_action_flow.py:208-272`) and its
      collapsed-panel sibling expand clan containers before any cancellation, so a panel
      kill never cancels.
    - `_batch_clan` (`chop_batch_recovery.py:197-210`) returns only the first clan of a
      batch, although `plan_chop_proposals` supports several clans.
    - A batch stopped before any launch has no recorded generation, so
      `_generation_matches` lets a kill of an older generation that reuses the same clan
      name cancel it.
    - The cancellation notice always reports `launched=0`, because it reads `counts`
      from a manifest, which has no counts.
11. **Notices are not per episode** (`chop_batch_recovery.py:132-194`).
    - The dedupe key `chop-batch:<event>:<routine>:<job>:<run>` has no episode identity,
      so a second interruption only adds "+1" to the first one's notice.
    - Nothing persists delivery, so a crash between a completed replay and
      `_emit_batch_notice` loses both notices.
12. **Minor (fix in `visibility`).** The typed bundle is written before the outer
    manifest (`_persist_typed_outer_batch`), so a crash between the two leaves a bundle
    that recovery cannot find. It loses no launches. Make the order or the discovery
    consistent with "interrupted bundle creation discoverable/idempotent".

Correct and to be preserved:

- Plans persist before the first dispatch.
- Per-member acknowledgement happens at the spawn boundary.
- Pause is distinct from failure.
- Typed resume reopens the original bundle by digest and never calls
  `_cancel_open_units` for a scheduler pause.
- Recovery runs before lifecycle, run-every and guard evaluation, and dry-run never
  dispatches.
- `_state_inflight.py` distinguishes journaled from unjournaled runs.
- Individual-member kills never cancel siblings.
- Lock order: admission lock, then agent-name lock, then batch mutation lock, then batch
  ownership lock. No lock is held across a launch.

## Rules for every phase

- Read the parent plan `plan:202611/chop_launch_recovery.md`, its notes and this plan
  before editing, and check what earlier phases of this epic already landed. Recheck
  `sase agent list` and the active plan beads for overlapping live ownership.
- Shared state-machine or reconciliation behavior belongs in sase-core
  `crates/sase_core/src/axe_chop/batch.rs`. Open the repo with `/sase_repo`, add Rust
  and binding round-trip tests, and move the pin through host completion or
  `just ratchet-core-revision`. Never hand-write a SHA, and keep the wire additive with
  `CHOP_BATCH_SCHEMA_VERSION` unchanged unless an old reader would misread the data.
- Before editing TUI files, read the `ace` and `tui` reference memories. Before
  finishing, read `lint_and_test`.
- Tests must use production-shaped evidence: real `_ChopAgentRecord` rows written by
  `record_chop_agent_launch_from_env` with `cl_name` set to a project name that differs
  from the agent name. Never patch a fake whose shape production cannot produce.
- Run focused regressions, then `sase tool run check` in each changed repository, using
  `/sase_monitor` for long runs. Do not run `just check-full`, weaken assertions, add
  xfails or suppressions, raise timeouts, or edit flake baselines.
- Do not create task beads or edit memory. Record discovered out-of-scope work as
  `PROPOSED FOLLOW-UP:` notes on the assigned phase.
- New inter-phase symbols needing a Symvision epic whitelist are keyed to this child
  epic, never to a phase that is about to close. Before closing a phase, run
  `sase bead epic-symbols <phase-id>` and resolve every entry, or re-key it to a
  still-open bead.
- Close only the assigned phase, with a verification note. No phase closes this epic,
  `sase-1tc.3.1`, `sase-1tc.3`, or task `sase-1c2`.

## Phase evidence

1. Rewrite `collect_spawn_observations` around run linkage:
   - A member is `matched_spawn` only when a chop-agent record for the batch's routine,
     job and `run_id` has `proposal_index == member.source_index`, or a matching
     non-empty `proposal_id`. Remove every `cl_name` and prefix comparison.
   - A name that a live or visible agent owns without that linkage is
     `foreign_conflict`. Never relaunch it under a new identity.
   - Use the durable agent-name registry (`src/sase/agent/names/_registry.py`) to tell a
     provably unspawned reservation (`reservation_only`, which may relaunch after its
     reserver died) from a reservation whose reserving process is still alive
     (`live_owner`).
   - A PID alone is never launch proof. Keep the observation list complete so the replay
     precondition `observations_are_complete=True` stays honest.
2. Make replay rebuild the first-run launch context from the manifest:
   - Add
     `build_chop_launch_env(lumberjack_name=routine, chop_name=job, prompt=…, run_id=<original run>, proposal_index=member.source_index, proposal_id=…)`
     on top of the stored env, and pass `origin="generated"`.
   - Pass the recorded clan identity and generation so a `join` member joins that exact
     generation and a resumed `declare` member never declares a new one. If the launcher
     API cannot express this today, add an explicit optional parameter through the
     launcher facade and leave non-chop callers unchanged.
   - Rewire `wait_on` exactly as the first run does when an earlier member was skipped.
     Derive it from journal statuses and the stored wait references, without rerunning
     the job script.
   - Store whatever prompt input is needed in the manifest so this rewiring is possible.
     Prefer existing fields, and keep any Rust wire addition additive.
3. Merge resumed launches into the original run history exactly once. Use the same
   launch descriptor shape as `_launch_descriptor`, keyed by index, name and pid, so
   that finalization, checkpoints and once-per bookkeeping see every member of the run.
4. Replace the fakes in `tests/axe/chop_batch_process_worker.py`,
   `tests/axe/chop_batch_process_harness.py` and
   `tests/axe/test_axe_chop_batch_recovery.py` with production-shaped evidence:
   - The fake launcher must call `record_chop_agent_launch_from_env` from the env it
     actually received, and must fail if a replayed launch lacks the chop linkage env.
   - Add regression tests where `cl_name` is a project name that does not equal, and is
     a prefix of, member names.
   - Add tests for a spawn before acknowledgement whose agent is already dismissed, a
     foreign owner of the planned name, a live reserver, and a resumed join that keeps
     the original clan generation.

Done when production-shaped tests prove that only provably missing members relaunch and
that resumed agents carry full chop linkage for the original run.

## Phase ownership

1. Hold `acquire_chop_batch_ownership` for the whole first-run dispatch, on both the
   ordinary path (`launch_chop_proposals`) and the typed path, and release it in a
   `finally` after the final journal write. Respect the documented lock order and never
   hold the mutation lock across a launch. Recovery and a manual run that find a live
   owner must leave the batch alone and emit no interruption notice.
2. Settle a launch that crosses a clan kill. When cancellation lands while a member is
   between `begin_dispatch` and acknowledgement, the spawned agent is real.
   - Record it as launched evidence: publish it through `launch_recorded_fn`, keep its
     once-per claim, and start no later member.
   - The run must not become `action_failed`.
   - Make the Rust transition own this settlement, for example by letting
     `cancel_clan_generation` leave in-flight members `dispatching` and letting
     `acknowledge_launch` settle a `dispatching` member on a cancelled batch, so counts
     stay truthful. Cover it with Rust and binding tests.
   - Apply the same rule to typed `_mirror_typed_launch`, so the admission unit gets its
     receipt.
3. Before each typed unit dispatch, check the outer manifest. A cancelled batch or
   member stops dispatch, as it does for ordinary launches. A failure to write the
   bundle cancel marker in `_cancel_linked_typed_bundle` must be logged and surfaced
   instead of swallowed. The outer cancellation remains authoritative.
4. Make once-per claims follow journal state. On a launch exception, release claims only
   for members the journal records as skipped, blocked or cancelled. Members still
   pending or dispatching keep their claims for recovery. Apply the same rule to the
   pause and recovery-completion paths.
5. In `_chop_lifecycle_runner.py`, skip finalization while `pending_batch_for_run(...)`
   reports any unfinished dispatch work, whether the batch is `open` or `paused`. Merge
   the old and new launch records once when the batch completes.
6. Add tests for:
   - A manual run racing a routine tick (and the reverse), with no duplicate launch and
     no false notice.
   - A cancellation crossing an ordinary launch and a typed launch.
   - Claim retention after a launch exception, followed by recovery.
   - Finalization waiting on an `open` batch with pending members.

Done when exactly one executor can dispatch a batch, a crossing kill settles truthfully
without failing the run, and claims and finalization agree with the journal.

## Phase visibility

1. Surface every recovery outcome:
   - Catch every exception per batch in the recovery entry point and convert it to a
     `blocked` result, so one job's failure never aborts recovery for the others.
   - Log disabled, maintenance, foreign-conflict and error holds to the original run's
     output. Raise one deduplicated notice per hold episode through the existing
     notification store. Silent holds for a normal live owner are fine.
   - Mark a member that fails to launch deterministically `blocked`, with an actionable
     reason, after a bounded number of attempts, so recovery stops retrying it on every
     tick.
   - Both callers (`lumberjack.py` and `chop_runner_script.py`) must stop discarding
     results.
2. Retire settled manifests:
   - Once a batch is complete or cancelled and its dispatch ownership has ended, move
     its manifest out of `pending_batches/`, keeping the evidence with the run's bounded
     history per the parent contract, or delete it when that history is pruned. Give
     `delete_chop_batch` (or its replacement) a production caller.
   - Enumerate unfinished batches without letting settled manifests consume the bound.
     Keep the enumeration bounded and cheap per tick.
   - Keep TUI cancellation off any unbounded scan.
3. Close the clan-kill gaps:
   - `_bulk_kill_panel_agents` and the collapsed-panel path must call the shared
     `cancel_chop_batches_for_clan` for each selected clan container before expansion.
   - Cancellation must match every clan a batch contains, not only its first.
   - A batch with no recorded generation must not be cancelled by a kill whose
     generation provably predates the batch. Persist a generation, or another comparable
     identity, before the first dispatch if needed.
   - The cancellation notice must report real launched and remaining counts from
     `summarize_chop_batch`.
   - Cancellation failures must be visible in the TUI instead of swallowed.
4. Make notices per episode and durable:
   - Persist a per-batch episode identity and delivered-notice markers in a sidecar next
     to the manifest, or an additive wire field. Include the episode in each dedupe key,
     so every interruption and every resume gets exactly one notice and a second restart
     repeats none.
   - On restart, detect and deliver a notice that a crash prevented.
   - Emit the interruption notice for a graceful SIGTERM pause at the safe boundary
     where the pause is journaled, rather than only on the next restart.
5. Make interrupted typed bundle creation discoverable. Either persist the outer
   manifest's bundle reference before the bundle body, or let recovery find an orphaned
   bundle for the run idempotently.
6. Add tests for each item, including:
   - A held disabled job producing exactly one notice.
   - A second clan in one batch.
   - A panel kill.
   - Notices across two separate interruption episodes.
   - A crash between replay and notice delivery.
   - A retired manifest no longer counting against the bound.

Done when no hold or failure is silent, settled manifests leave the pending set, every
TUI clan-kill path cancels, and notices follow episodes exactly once.

## Phase acceptance

1. Extend the deterministic subprocess harness in `tests/axe/` so it runs production
   evidence end to end:
   - Use real chop-agent records, written from the launch env the fake launcher
     received.
   - Use the real agent-name registry.
   - Use `cl_name` equal to the project name.
   - Keep the explicit pipe and file handshakes, an isolated SASE state root, a
     disposable job, and harmless processes in place of real agents.
   - Never call `systemctl`, restart the real scheduler or service, or run live
     `toobig_split`.
2. Cover these cases on both the ordinary and the typed path:
   - k-of-n termination with k=0, 0<k<n, and k=n.
   - SIGKILL after spawn but before acknowledgement, where the spawned agent was already
     dismissed.
   - SIGKILL after reservation but before spawn.
   - SIGTERM finishing the current launch.
   - A second restart launching zero.
   - A routine tick and a manual run racing.
   - A clan kill while paused, while recovery is active, and while a launch is crossing.
   - A multi-clan batch.
   - Notices across two interruption episodes.

   Assert the following:
   - n distinct original identities.
   - Exactly n spawns in total.
   - Resumed agents' chop linkage points to the original run and proposal index.
   - The original run's history lists all n launches once.

3. Repeat the disposable manual demonstration with this production-shaped harness, for
   both a stop/resume and a clan kill. Record the total, k and remaining counts, the
   original clan and run identity, the termination mode, the observed notices and logs,
   and the results of the first and second restart.
4. Run focused Rust, binding and Python regressions, then the required checks in every
   changed repo. Include `test_axe_chop_clan_launch.py`,
   `test_multi_prompt_launcher_launch.py`, `test_axe_lumberjack_tick.py`,
   `test_axe_inflight_launches.py`, the chop lifecycle, once-per and typed-admission
   suites, and the changed clan cleanup tests. Verify that the sase-core pin contains
   every new binding or transition.
5. Record
   `TASK RESOLVED: sase-1c2 — <production-shaped tests and demonstration evidence>` on
   `sase-1tc.3`. State that it supersedes note #2 there, which relied on test fakes with
   an impossible evidence shape. Put the same evidence on this phase. Close only this
   phase.

Done when the three parent acceptance requirements hold under production-shaped
evidence:

- Only the n−k remaining members launch after an interruption.
- A second restart launches none.
- A user-killed clan's pending batch is discarded.

## Landing

This child epic's land agent verifies the phases above, resolves its own epic-symbol
entries, and closes this epic. Its `parent_bead` link (`sase-1tc.3.1`) then resumes the
interrupted landing of `sase-1tc.3.1`. That landing handles the `sase-1tc.3.1` close,
its remaining `--epic-symbol` entries, the Symvision pass, the parent plan's
`status: done`, and the `sase-1tc.3` parent phase. Nothing in this epic closes
`sase-1tc.3.1`, `sase-1tc.3`, `sase-1tc`, or task `sase-1c2`.
