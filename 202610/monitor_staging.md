---
tier: epic
title: Stage monitor starts and launch after finalization
goal: Complete sase-1pj.5 so in-agent monitor starts stage durable work and launch
  only after host finalization, preserving prepared completion and existing ToolRun
  joins.
parent_bead: sase-1pj.5
phases:
- id: contracts
  title: Define staged monitor and ToolRun reservation contracts
  size: medium
  depends_on: []
  description: 'contracts: implement Rust-owned staged monitor payload validation,
    staged-owner liveness and self-wait decisions, and atomic ToolRun join reservation/activation/release.
    Add bindings, contract tests, thin Python adapters, and the required core revision
    pin. Keep public monitor starts on their existing path until activation.'
- id: start-transaction
  title: Split monitor preparation from process launch
  size: medium
  depends_on:
  - contracts
  description: 'start-transaction: persist a reconstructible staged member, reserve
    ordinary ToolRuns, launch from persisted inputs after finalization, and clean
    up unstarted resources on failure or discard. Preserve the existing public start
    path until activation. Integrate staged member reads, lane exclusion, and ToolRun
    owner observations.'
- id: prepared-joins
  title: Preserve prepared completion and joins across staging
  size: medium
  depends_on:
  - start-transaction
  description: 'prepared-joins: reserve sealed intents without binding, bind only
    at spawn, use the landed prepared-starter coverage contract, activate reserved
    joins even when the original run settles before adoption, and test green/red completion
    and resource rollback. Do not enable the CLI staging branch yet.'
- id: activation
  title: Connect staged starts to runner adoption and document the protocol
  size: medium
  depends_on:
  - prepared-joins
  description: 'activation: switch in-agent flag-on starts to staging, register the
    post-finalizer monitor adopter, preserve real final responses and transferred
    claims, connect held/discard cleanup, refuse self-waits, update the two generated
    skill sources, and prove the finalizer/check ordering and both flag states end
    to end.'
proposed_by: bbugyi200.athena.sase-1pj.5
proposed_with: codex/gpt-6-astra
create_time: 2026-10-29 14:48:29
status: wip
bead_id: sase-1pj.5.1
---

- **PROMPT:** [prompts/202610/monitor_staging.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/monitor_staging.md)
- **PARENT:** [202610/staged_handoffs.md](https://github.com/sase-org/sase--plans/blob/main/202610/staged_handoffs.md)
- **BEAD:** [sase-1pj.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pj/sase-1pj.5.1.md)

# Stage monitor starts and launch after finalization

## Scope and current evidence

This child epic implements only the monitor-staging phase of
`plan:202610/staged_handoffs.md`, assigned as `sase-1pj.5`. Read that artifact with
`sase artifact read` and the assigned bead with `sase bead read` before work. Its
DECISIONS remain final. This plan needs no memory edits and adds no reviewer choices.
The outer epic owns crash reconciliation/re-adopt (`sase-1pj.6`), workspace holds
without a successor (`sase-1pj.7`), flag removal, and broad docs/memory updates.

The phase was in progress with no earlier-work notes at planning time. Its foundation
has landed; do not rebuild it:

- `src/sase/agent/staged_handoff.py` owns admission, reservations, records, receipts,
  and handoff/declaration lock ordering.
- `src/sase/axe/run_agent_exec_staged.py` runs adoption after finalizer settlement,
  checks receipts, and records carried custody. Only the plan adopter is registered on
  the inspected tree.
- `src/sase/finalizers/prepared_coverage.py` specifies the monitor reservation payload.
  Commits `89c40e30b0` and `d30932ec87` added carry and prepared-starter satisfaction.
  `src/sase/monitor/host_completion_state.py` already refreshes context before
  installing a prepared declaration when the staged descriptor is present.
- `src/sase/monitor/start_flow.py` still binds `-f`, creates a running member, and calls
  `launch_monitor` synchronously. `start_launch.py` still reserves or joins the ToolRun
  immediately before proc submission. The CLI still writes the legacy handoff marker and
  kills the starter.

Four sequential medium phases keep each coding turn bounded and avoid concurrent edits
to the same launch transaction. The first three build and test internal APIs; the final
phase enables the complete monitor path. No intermediate commit enables an incomplete
staged `-f` or `-J` path. Host/non-agent starts and flag-off starts retain their
immediate launch behavior throughout.

## Binding behavior

The final flag-on flow is:

1. The root agent finishes edits and invokes `sase monitor start`.
2. The CLI validates, reserves the handoff slot, freezes the request and policy,
   preflights the workspace, creates a `staged` monitor member, and reserves any ToolRun
   or prepared intent. It publishes one staged record and returns success.
3. The agent runs `/sase_final` and returns its real response. No monitor command,
   supervisor, workspace transfer, or prepared binding has started yet.
4. The host settles all finalizers, including any `just fix` hook and subsequent
   fixed-point pass. Its adoption epilogue reconstructs the frozen request, binds
   prepared completion or activates the join, submits the proc, and transfers the
   workspace after ack using the existing `.go` barrier.
5. The host records adoption and actual monitor custody. Monitor completion follows the
   existing completion/follow-up contract.

Unprepared monitors allow explicit commit or carry; no declaration synthesizes carry
under the landed policy. A declared commit therefore lands before a CI-wait monitor
starts. Prepared monitors retain commit-on-green: the starter does not commit their
sealed repository obligations, green completes through the host without another model,
and red/stale/timeout takes recovery without committing the prepared work. `-J` observes
the same already-running ToolRun and never starts its command again.

Preserve finalizer-owned/helper refusals, one handoff per turn, refusal after a current
accepted declaration, and the existing flag snapshot rule: only the CLI selects the
protocol; the runner dispatches from durable artifacts. Staged records remain outside
`PENDING_HANDOFF_MARKERS`. Do not add a new flag or remove the legacy branch here.

## contracts

Open `sase-core` through `/sase_repo`, read its `AGENTS.md`, and follow
`docs/contribute/rust-backend.md`. Shared lifecycle, reservation, and admission policy
belongs in `sase_core`; Python gathers facts, persists local artifacts, and calls thin
bindings. Do not introduce a Python fallback.

1. Define a versioned monitor launch payload validator/codec alongside the existing
   staged-handoff domain. Preserve legacy staged-record readability; validate the new
   payload explicitly at the monitor staging/adoption boundary rather than retroactively
   rejecting historic minimal monitor records. Freeze the normalized request,
   command/execution argv, label, policy and digest, request fingerprint, monitor/member
   identity, starter identity and artifacts, lane, workspace and claim context,
   continuation parents/protocol/checkpoint, and resource references. Reconstructing it
   must need neither the exited CLI's environment nor a live `StartIdentity` object.
   Reference durable paths and existing ToolRun launch envelopes instead of duplicating
   secrets or full repository contents.
2. Define the shared decisions needed for `staged` membership: it is nonterminal, blocks
   another lane start, has no supervisor, and does not occupy additional runner
   capacity. Preserve current running/terminal behavior. Add only the necessary Rust
   predicates/wire support and Python mirrors; this is not a general monitor domain
   migration or a UI redesign.
3. Add atomic join reservation, activation, and release in the Rust ToolRun store
   (`tool_run/handoff_wire.rs`, `tool_run/store/handoff.rs`, and owning bindings).
   Reservation is identified by staged ID plus monitor ID, validates the starter agent
   and existing state/stop/join ownership, and prevents another joiner from taking the
   run. Activation converts that same reservation to the ordinary monitor join; matching
   replay succeeds, foreign ownership refuses. Preserve executing proc ownership. A run
   terminal before initial reservation is still refused; a run that settles after
   successful reservation can be activated and its result consumed. Release is
   identity-scoped and safe to repeat, including after settlement. Add compatible
   optional wire/store state and retain ordinary join behavior.
4. Define the prepared-intent reservation ownership check in the existing continuation
   domain, reusing `PreparedReservationWire`. Persist ownership separately from the
   sealed intent under its existing store lock, keyed by intent and stage identity. All
   binding paths must honor it: matching adoption may bind, another stage or an
   immediate start cannot steal the intent, and matching cleanup may release it. Python
   owns the file/lock operations; Rust owns identity and transition legality. Retain the
   already-landed prepared-starter coverage and sealed digest semantics.
5. Add the minimal Rust decision support for two staging facts: a positively identified
   staged monitor remains a pending owner before its proc exists, and a caller cannot
   synchronously wait on its own pending staged monitor's ToolRun. Distinguish owner
   identity from agent-name coincidence. No wall-clock grace period may substitute for
   durable stage ownership. Unknown/corrupt facts are not proof that a command ran or
   that a reservation may be stolen.
6. Test serialization/binding round trips, historic records, staged lifecycle facts,
   reservation contention, foreign starter, stop and terminal cases, matching replay,
   terminal-after-reserve activation, and identity-scoped release. New binding consumers
   require `sase-core-revision.txt` to name a landed commit containing them; coordinate
   host finalization/pin ratcheting using the existing repo workflow.

Public CLI behavior remains unchanged. If new Python adapter exports have no live
consumer until later phases, use only narrowly justified epic-symbol scaffolding
attached to this still-open child epic or its actual consuming phase; read
`symvision.md` before doing so.

## start-transaction

Refactor the existing transaction, reusing its rollback and process barrier machinery.
Primary seams are `monitor/start_flow.py`, `start_launch.py`, `start_lane.py`,
`start_claim.py`, `start_runtime.py`, `member.py`, `models.py`, `store_lane.py`, and
`tool_handoff.py`.

1. Extract preparation from spawning. Keep the existing `start_monitor` facade for
   immediate callers, composed from shared preparation and launch pieces. Add an
   internal staging entry that accepts an already-admitted staged identity. Under the
   lane lock, resolve identity, replay, inherited model/effort, outcome policy and
   fingerprint once; preflight claims; allocate the monitor identity/suffix; create the
   member in `staged` state; persist the frozen launch payload. No proc submit, `.go`,
   claim move, start-intent publication, or command execution occurs here.
2. Use the handoff reservation before any mutating preparation (including promotion,
   member creation, and ToolRun reservation). Preserve the handoff-before-declaration
   lock order; do not acquire those locks in reverse while holding the lane lock. Prefer
   acquiring the handoff slot, releasing its locks, doing lane work, then publishing
   through the reservation-aware writer. Record enough resource identity to unwind a
   failure between these steps.
3. Move normal ToolRun reservation to staging using existing tool-wrap and
   `reserve_handoff_run` machinery. Persist the selected execution strategy and reserved
   ID and reuse it at launch: never resolve a different command or reserve a duplicate
   run at adoption. Preserve the existing documented fail-open fallback for an
   unrecordable ordinary monitor, without printing an invalid run ID. Prepared
   completion must retain its existing evidence eligibility requirements. Raw commands
   remain supported without an invented ToolRun.
4. Add launch-from-persisted-payload under the lane lock. Revalidate mutable claim and
   reservation ownership, but retain the frozen request, cwd, policy, member identity,
   and ToolRun. Reuse `submit_proc_request`, `after_ack`, claim rollback, `.go`, and
   `persist_monitor_start_intent_after_ack`. Stamp running state only when launch is
   acknowledged. Persisted starter context must replace assumptions about CLI env when
   the runner or a later adopter calls this entry point.
5. Make staged members queryable and lane-blocking without treating their missing PID as
   a lost supervisor. In `tool/owner.py` and `tool/liveness.py`, supply the staged
   ownership facts to the Rust decisions from contracts: the short-lived CLI exiting
   must not settle the reserved run as `launch_failed`. Verify capacity counts only the
   starter before adoption and only the transferred owner afterward.
6. Provide one idempotent cleanup entry for unstarted staged resources. On preparation
   failure, discard/user kill, or held finalization, settle an owned reserved ToolRun as
   never executed, release only this stage's join/intent reservation, and settle or
   annotate its staged member consistently. Never stop an already-running joined run. A
   held handoff retains the outer staged record and diagnostics; expose a clear result
   for outer adoption-recovery to refresh canceled reservations before retry. Never
   silently launch a canceled ToolRun. Full crash recovery and re-adopt remain the outer
   phase's work.
7. Test real persistence and reconstruction across separate calls, lane conflicts,
   replay identity, nested cwd and explicit lane behavior, ToolRun ID stability,
   launcher-exit liveness, ack failure/rollback, and cleanup at each resource boundary.
   Existing immediate-start tests must continue to pass. Internal tests may invoke
   stage/adopt directly; public CLI activation waits for the last phase.

## prepared-joins

Build on the transaction APIs, without reimplementing the landed carry and prepared
coverage engine.

1. For `-f`, validate the intent and exact command before publishing the stage, using
   the existing Rust completion validation. Persist `payload.prepared_intent_ref` and
   `payload.prepared_reservation` exactly as required by
   `finalizers/prepared_coverage.py`:
   `{staged_id, monitor_id, request_fingerprint, command}`, with command as the exact
   normalized argv. Keep the intent in `prepared` state. Do not bind and then roll back
   merely to simulate reservation. The reservation must prevent a competing consumer
   from binding that same intent; use the existing locks and stage identity rather than
   an unaudited Python policy.
2. At spawn, bind that same monitor/fingerprint/command once through
   `bind_prepared_completion`, using persisted starter artifacts explicitly. Failure
   before ack unwinds the binding and unstarted resources. Mutation of sealed repo
   observations, identity, command, or policy must refuse eligibility/activation.
3. Exercise the actual starter finalizer and host completion paths. Confirm prepared
   satisfaction avoids starter commit and hooks that would change the sealed tree;
   confirm `install_prepared_declaration` still uses a fresh host-issued context with
   the staged descriptor present. Green commits exactly once with no model; failure,
   timeout, or stale observations commit no prepared work and enter recovery. Keep
   ordinary unprepared carry and explicit commit behavior unchanged.
4. Wire `-J` preflight to the atomic Rust reservation and activation APIs. Keep current
   option incompatibilities (`-J` with `-f`, explicit command, weight, or explicit
   agent). At stage time reserve the existing run; at adoption activate the join and
   launch only its observer. Update `tool/join_worker.py` so a correctly activated
   reservation can consume a run that finished during finalization, even though ordinary
   new joins on settled runs remain refused. Preserve output, exit code, stop handling,
   and ownership attribution.
5. Cover prepared reservation/binding/release, green/red/stale outcomes, malformed or
   missing intents, joined run completion between stage and adoption, competing joiners,
   foreign agents, stopped runs, and submit failure. Assert the original joined command
   executes once and cancellation releases the observer reservation without canceling
   the executing command. Keep these as direct API integration tests until activation.

## activation

1. In `main/monitor/start.py`, select staging only for an in-agent root start with the
   pinned `staged_handoffs` flag on. Use existing validation and guards before side
   effects, then publish through `agent/staged_handoff.py` with workspace `inherit` and
   the correct carry/prepared descriptor. The staged path neither writes legacy
   pending/in-flight kill markers nor calls the kill helper. Flag-off and non-agent
   starts preserve their existing behavior, including epic-launch callers.
2. Print a line beginning `Staged`, the monitor/member and any valid reserved ToolRun
   ID, and explicit instructions that nothing starts until the turn ends and the agent
   must finish through `/sase_final`. Preserve machine-readable `--json` output as valid
   JSON: include staged status/identity in its object and put any instructional prose on
   stderr. Do not say a proc is running or invite a self-follow/wait while it is staged.
   Keep legacy output stable.
3. Register a monitor adopter in `axe/run_agent_exec_staged.py` that accepts persisted
   payload and run metadata and calls the prepared launch entry. Reuse the shared
   successful receipt and custody pipeline; return `StagedAdoptionOutcome` with the real
   monitor artifact custodian. Separate reusable metadata/checkpoint handling from the
   kill-specific parts of `run_agent_exec_monitor.py`: save the ordinary final response,
   include monitor details as extra context, and do not reset kill state or manufacture
   an interrupted chat on normal adoption.
4. Connect cleanup to staging failure, the existing discard/user-kill path, and the
   existing held-finalization/adoption-failure path. The held record remains readable
   for phase `sase-1pj.6`. Expose stable launch/cleanup seams and leave implementation
   notes for that phase and `sase-1pj.7`; do not implement their reconciler, new CLI, or
   workspace-hold policy here. Confirm normal runner exit respects the moved claim and
   the monitor follow-up still waits for the starter terminal marker.
5. Call the shared self-wait decision before polling in `tool/control_wait.py`.
   `sase tool wait RUN` for this caller's own pending staged monitor must return
   promptly with a useful `/sase_final` message. Cover both owned and joined run IDs;
   unrelated callers and ordinary runs keep their behavior.
6. Edit only `src/sase/macros/skills/sase_monitor.md` and the prepared-monitor section
   of `sase_final.md` for this phase's instruction changes. Teach both command outcomes,
   stage before declaring, no waiting on a staged run, reservation versus binding, and
   how to explicitly commit before a CI wait. Preview with `sase skill init --diff`; do
   not deploy skills or edit memory/user docs.
7. Add an integration test with an event/barrier proving the monitor check cannot start
   before the commit finalizer's `just fix` hook and all remaining finalizers finish.
   Also cover dirty primary plus linked obligations carried to a follow-up, explicit
   commit before CI wait, prepared green/red, joined completion during finalization,
   failed finalization with zero proc launches, real response capture,
   root/helper/finalizer guards, and flag-off marker/kill compatibility. Reuse existing
   monitor, staged-handoff, and finalizer harnesses instead of live host launches.

## Verification and completion ownership

Every coding phase reads `lint_and_test.md`, runs targeted tests for its contract, then
`sase tool run check` in each changed repo. Cross-repo changes include core binding
tests and the pinned-binding validation. Use supported `just` recipes in sase-core,
never bare cargo. Run `just fix` before final verification, and use `/sase_monitor` if a
check needs a handoff. `just check-full` is not requested. Read `ace.md` before changing
`src/sase/ace/`; read `tui.md` before any actual UI change and avoid expanding this work
into presentation redesign.

The final acceptance evidence must show:

| Boundary                                | Required observation                                                                                   |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Stage returns                           | Durable staged member; stable reserved IDs; no proc, command, bind, or claim move                      |
| CLI exits                               | Reserved ToolRun remains pending; missing proc does not imply launch failure                           |
| Starter finalizes                       | Carry remains owned; prepared repos remain sealed; explicit commits finish first                       |
| Adoption succeeds                       | One monitor proc; correct ToolRun; bind/join activation; transferred claim and real response           |
| Prepared outcome                        | Green commits once without a model; red/stale/timeout enters recovery without committing prepared work |
| Joined run settles early                | Same run/result consumed; zero reruns of its command                                                   |
| Finalization/launch fails or user kills | No unauthorized command; unstarted reservations cleaned; held evidence retained where required         |
| Both flag states                        | Staged branch works; existing immediate and legacy kill paths remain compatible                        |

Do not create task beads. Record discovered follow-up as `PROPOSED FOLLOW-UP:` on the
assigned phase for its land agent. A check failure independently reproduced identically
on the clean base is recorded with the evidence and any existing task ID; it does not
keep a completed phase open.

Each child phase closes only its own assigned bead after `sase bead epic-symbols` has no
entries keyed to that closing phase (remove resolved entries or re-key justified ones to
the still-open consuming phase/epic). Children do not close `sase-1pj.5` or any other
ancestor.

The child epic's land agent owns the final integrated verification and completion of the
delegated `sase-1pj.5` scope. It must inspect later landed work before duplicating
tests, check `sase bead epic-symbols sase-1pj.5`, resolve or re-key every leftover, and
close `sase-1pj.5` with a note naming the verified ordering, prepared outcomes, joins,
cleanup, flag parity, and check results once the child work is complete. This is not
authorization for child phase workers to close an ancestor. No agent working this child
plan closes the outer epic `sase-1pj`.
