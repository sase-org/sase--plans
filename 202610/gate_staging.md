---
tier: epic
title: Stage gate requests before host publication
goal:
  Complete sase-1pj.4 so agent-created gates publish only after host completion, with
  synchronous auto gates and preserved legacy behavior.
parent_bead: sase-1pj.4
phases:
  - id: gate-contract
    title: Durable gate staging contract
    depends_on: []
    size: medium
    description:
      "gate-contract: define the Rust-owned staged gate payload and workspace
      disposition, with Python projection and resource snapshot support."
  - id: gate-publisher
    title: Publish staged gates from persisted creator context
    depends_on:
      - gate-contract
    size: medium
    description:
      "gate-publisher: split gate publication from preparation and register the
      post-finalizer gate adopter with replay and explicit custody."
  - id: gate-cli
    title: Stage custom and sudo gates through one entry helper
    depends_on:
      - gate-publisher
    size: medium
    description:
      "gate-cli: activate the shared staged entry helper for custom and sudo gates,
      protect own-gate waits, and update their skill sources."
  - id: launch-cli
    title: Stage launch requests and agent-origin run requests
    depends_on:
      - gate-cli
    size: medium
    description:
      "launch-cli: retain launch preview and continuation validation while returning
      staged descriptors from launch request and agent-origin run."
  - id: workflow-completion
    title: Bring workflow review into host completion
    depends_on:
      - gate-cli
    size: medium
    description:
      "workflow-completion: provide a host-owned completion boundary for workflow HITL
      so review staging precedes finalizer settlement without relaxing CLI admission."
  - id: workflow-hitl
    title: Suspend workflow execution on staged review
    depends_on:
      - workflow-completion
    size: medium
    description:
      "workflow-hitl: route agent, bash, and Python HITL reviews through staging,
      preserve waiting state, and prove publication ordering and follow-up behavior."
proposed_by: bbugyi200.athena.sase-1pj.4
proposed_with: codex/gpt-6-astra
create_time: 2026-10-29 14:37:27
status: wip
---

- **PROMPT:**
  [prompts/202610/gate_staging.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/gate_staging.md)
- **PARENT:**
  [202610/staged_handoffs.md](https://github.com/sase-org/sase--plans/blob/main/202610/staged_handoffs.md)

# Stage gate requests before host publication

## Scope and ownership

This child epic implements the `gate-staging` section of
`plan:202610/staged_handoffs.md`, owned by phase bead `sase-1pj.4`. Read both through
`sase artifact read` and `sase bead read` before implementation. The parent epic's
DECISIONS are final. This child changes no memory notes; the parent owns the approved
memory work and flag retirement. Do not change the selected decisions or add an approval
choice for them.

The initial inspection found no previous-work notes on `sase-1pj.4`. Its prerequisite
phases `sase-1pj.1` and `sase-1pj.2` are closed. The checkout has the shared staging
record, admission reservation, finalizer handoff descriptor, carry support, and adoption
dispatcher. The gate producer conversion is absent: `create_gate_turn` still creates the
member, moves its workspace claim, and publishes the notification immediately. The
inspected gate transaction, launch request, and HITL histories contain no earlier
gate-staging implementation. Recheck the current tree before implementing each phase.
`sase bead epic-symbols sase-1pj.4` initially reported no entries.

Six bounded coding phases separate the wire contract, publication transaction, CLI entry
points, launch result types, workflow completion boundary, and workflow suspension. The
`launch-cli` and `workflow-completion` phases may proceed independently after
`gate-cli`; all other dependencies are listed above. Each phase includes its tests. The
land agent integrates their evidence and closes this child epic through its normal
host-owned completion, then closes **only `sase-1pj.4`** as the containing work item. It
must not close `sase-1pj` or any higher ancestor. Child phase workers close only their
assigned child phase.

## Binding behavior

With the pinned `staged_handoffs` flag enabled, an agent-origin pending gate is
validated and durably staged while its provider remains able to finish. Its descriptor
contains `request_id`, gate kind, staged identity, and `status: staged`; it has no
notification ID, gate member name, or claim that the published bundle exists. The
command explains that nothing starts until the turn ends and directs the root through
`/sase_final`. Host finalizers settle before publication and activation. The real
provider response remains the turn's chat.

Keep the existing flag-off marker-and-kill path and non-agent creation behavior. The CLI
chooses the protocol; the adopter dispatches from persisted state without consulting the
flag again. Effective auto resolution remains synchronous, writes no staged record, and
leaves the caller running. A gate whose requested auto policy evaluates to a human
decision must stage. LaunchApproval cannot auto-execute. Preserve plan/question host
publication paths and their `before_auto_settle` behavior: adding staging must not make
their adopter recursively stage another gate.

All agent entry points share one preparation/staging/output/handoff helper. Do not turn
every low-level `create_gate_turn` call into an ambient-flag-dependent stage operation.
Provide an explicit immediate publisher for host callers and an explicit staged result
variant for entry points. Never manufacture a `GateCreationResult` for an unpublished
gate.

Reuse the reservation and write helpers in `src/sase/agent/staged_handoff.py`. Refuse a
second staged or legacy handoff before publishing anything, including on the legacy gate
creation path. On staged calls, admission must precede intent markers, resource snapshot
writes, lane promotion, member creation, claim mutation, or notifications. Release
reservations on preparation failure. Keep root/helper, finalizer-owned-turn, and
accepted-current-submission guards. CLI-time staged calls write neither
`.sase_gate_pending` nor a gate-intent marker. An inherited workspace maps to
`successor_workspace: inherit` and `carry_allowed: true`; a released workspace maps to
`none` and `false`. Rust owns this rule.

## gate-contract

1. Open `sase-core` with `/sase_repo`, read its `AGENTS.md`, and read
   `docs/contribute/rust-backend.md` in sase. Extend the existing staged-handoff domain
   under `crates/sase_core/src/staged_handoff/` and its PyO3 bindings under
   `crates/sase_core_py/src/staged_handoff/`. Existing outer wire version 1 already
   permits `kind: gate` with an object payload. Add a versioned typed gate payload and
   validation/building function without needlessly replacing the outer wire or
   invalidating legacy marker formats. New shared decisions and invariants belong here;
   Python remains the adapter and filesystem observer.
2. The persisted payload must contain the normalized gate spec and stable request ID,
   the resolved auto evaluation/policy snapshot, resource snapshot metadata, and enough
   creator identity to publish without the original CLI environment: project and project
   file, source artifacts/timestamp, agent/session identity, workspace/claim identity,
   and relevant metadata. Persist only the required identity and policy fields, not the
   entire process environment. Validate required fields and consistency, including spec
   request ID and workspace mode versus the outer handoff disposition.
3. Add Python projection and rehydration alongside the gate-turn code and the thin
   `src/sase/core/staged_handoff.py` adapter. A normalized spec must round-trip all
   options, groups, branches, operations, resource modes, presentation, timeout,
   continuation, and turn policy. Keep `turn_row_managed` host-internal; user JSON must
   not acquire permission to assert that a row already exists.
4. Capture `source` resource bytes into private, durable staging storage after
   admission. Support binary attachments as well as text commands/previews. Publication
   must not reopen mutable relative source paths or depend on the CLI working directory.
   Preserve the original source identity used by editable resources with
   `edit_target: origin`; a staging copy must not silently become the user's edit
   target. Validate commands at preparation time and preserve hash/executable semantics.
   Keep these snapshots separate from the public interaction-request bundle.
5. Test complete round trips, malformed payloads, mismatched request IDs,
   inherit/release disposition, Unicode and binary resources, source removal/change
   after preparation, and editable-resource origins. Add binding round-trip tests.
   Ratchet `sase-core-revision.txt` past the landed core commit before sase depends on
   the new binding. Follow existing host-owned finalization; do not commit manually.

This phase provides the contract and preparation building blocks; it does not activate
CLI staging before a gate adopter is installed.

## gate-publisher

1. Refactor `src/sase/gate_turn/transaction.py` so its ordered publication transaction
   accepts explicit creator context. Preserve the existing immediate public interface as
   a compatibility wrapper. Resolve the creator from persisted run identity for staged
   adoption, including any session promotion, instead of `_resolve_creator` accidentally
   adopting the re-adopter's environment. Revalidate claim ownership at the actual move;
   never steal a workspace based only on a stale recorded PID.
2. Preparation validates the spec through existing `validate_gate_spec` and resolves
   `effective_auto_state`. Publication consumes that frozen result. Refactor the narrow
   seams in `notification_gates/service_creation.py` and `service_evaluation.py` so
   staging plus publication does not evaluate/log autonomy twice or change policy when
   the ambient flag/autonomy environment changes. Keep the ordinary `create_gate` API
   and its journal recovery behavior working.
3. The publisher alone creates the gate member, transfers/releases its workspace claim,
   materializes the bundle, publishes the notification, and pulses watchers. Keep lane
   locking, request-ID replay, compensation, and `before_auto_settle` intact. A retry
   after publication must find the original member and gate, not allocate another suffix
   or move the claim twice.
4. Register `kind: gate` in the existing `src/sase/axe/run_agent_exec_staged.py` adopter
   registry, preferably using a focused sibling module. Publish from the staged payload,
   then reuse `handle_gate_marker` with `persist_chat=False`. Return
   `StagedAdoptionOutcome` with the actual gate-member artifacts identity as custodian
   when it retains the workspace; include stable gate references in the existing
   adoption receipt. Do not derive custody from a command string or replace the model's
   final response with a synthetic gate summary.
5. Tests must exercise the real dispatcher settlement gate: no publication while
   finalizers are unsatisfied; publication follows successful commit or valid carry;
   finalizer failure leaves the stage held; publication failure follows existing held
   receipt behavior; two publication attempts yield one gate and one claim transfer.
   Repeat adoption from persisted metadata with the original agent environment absent.
   Preserve auto-settlement and plan/question caller regressions.

General crash reconciliation and the re-adopt command remain owned by `sase-1pj.6`.
Provide the reusable adopter and deterministic identity that phase needs, without
implementing a second recovery framework.

## gate-cli

1. Add the shared entry helper using the contract and publisher. It distinguishes
   staged, synchronous auto, and legacy-created results. Reserve before side effects,
   validate and freeze the pending request, atomically write its staged record, release
   on failure, and return normally on success. Legacy handoff continues to print and
   flush before killing. The shared helper owns the repeated output/handoff protocol.
2. Route `src/sase/main/gate_handler.py` and `src/sase/sudo/cli.py` through it. Sudo
   normalization, target resolution, manifest hash, risk badges, and preview generation
   remain preparation-time work; no sudo command executes during staging. Remove the
   eager `begin_gate_intent` call from the staged sudo branch while retaining correct
   legacy failure cleanup. Ordinary custom gates without a turn keep their contract.
3. Provide a single honest staged descriptor projection. Human-readable status starts
   with `Staged`. Keep explicit JSON output parseable as one JSON value: put the
   human-readable instruction on stderr for JSON consumers and expose staged status and
   instruction in the object. Do not print a fake notification ID or published
   member/bundle path.
4. In `src/sase/notifications/cli_wait.py`, reject waiting for the caller's own staged
   request by matching its current staged kind/request ID before trying to load a
   nonexistent bundle. Explain that it starts after the turn ends. Preserve waits for
   other gates and the existing agent turn-gate wait restriction.
5. Update only the source templates `src/sase/macros/skills/sase_gate.md` and
   `sase_sudo.md`: on `Staged`, finish through `/sase_final`, stage before declaring,
   and never wait for one's own handoff; otherwise retain the legacy termination
   instructions. Preview with `sase skill init --diff`; do not deploy generated skills.
6. Test both flag states for custom and sudo creation, inherited and released
   workspaces, human/JSON descriptors, invalid spec/resource failures, two concurrent
   creation attempts, existing staged/legacy markers, finalizer-owned/native-helper
   calls, and staging after an accepted current declaration. Assert zero gate members,
   notifications, claim moves, and gate-intent markers before adoption. Auto-resolution
   must finish inline with no leftover stage or reservation.

## launch-cli

1. Route the agent branch of `src/sase/agent/launch_request.py` through the shared
   helper. Keep normalization, inherited tab, prompt/typed-plan previews, slot limits,
   selected project, submission timestamp, and requester-continuation validation before
   the durable stage. Same-session successor conflicts still fail before publication.
   Reject invalid automatic launch resolution during preparation; never discover it
   through the current post-publication `notification_id is None` assertion.
2. Give `LaunchRequestCreationResult` in `launch_request_types.py` an explicit staged
   representation or a typed result union. Published fields must remain required for the
   published variant and absent for the staged variant. Audit callers of
   `.notification_id`, `.preview_path`, `.response_dir`, and `gate_turn_creation`;
   terminal response readers continue to operate only on published requests.
3. Update `src/sase/main/launch_handler.py` and the agent-origin branch of
   `src/sase/main/query_handler/_launch.py` (`sase run`) to use the common descriptor
   and handoff flow. Keep human-origin launches and approved dispatch behavior
   unchanged. Do not wait for approval or spawn the requested child before host
   publication and approval. The validated typed launch plan must survive staging
   without re-expansion into a different preview at adoption.
4. Update `src/sase/macros/skills/sase_run.md` with dual-state instructions and a staged
   descriptor example. Preview generated output without deploying it.
5. Test `launch request` and agent-origin `run` in both flag states; invalid max slots,
   continuation conflicts, auto requests, and second handoffs leave no published gate.
   Exercise staged JSON/text output, no spawn before approval, typed-plan preservation,
   and approve/reject/timeout/stopped requester continuation after adoption.

## workflow-completion

This phase fixes a concrete ordering issue before activating workflow HITL staging.
`workflow_executor_steps_prompt.py` currently invokes `invoke_agent`, which already runs
finalizers in `llm_provider/_invoke.py`, and only afterward calls the HITL handler.
Simply replacing the latter's gate creation with staging can hit the accepted-current-
submission guard and leaves finalizer evidence bound to a context without the handoff.
Script steps can also request HITL without invoking a provider at that step.

1. Introduce a narrow, host-owned workflow completion seam that can prepare a configured
   review from completed step output, bind its staged handoff to a refreshed finalizer
   context, and run the ordinary finalizer controller before the runner's adoption
   epilogue. For reviewed prompt steps, separate obtaining the provider result from
   final completion so output preparation and review staging precede finalizer
   settlement. The normal non-HITL `invoke_agent` path remains unchanged.
2. This is host orchestration, not a provider command that may bypass admission. Keep
   CLI refusal after an accepted current submission unchanged. Define any needed
   host-preparation admission in Rust with explicit originating invocation/step
   identity, completed-provider evidence where applicable, and the existing one-handoff
   and finalizer-owned restrictions. Do not add an environment-variable bypass, clear a
   valid submission to evade the check, or accept a stale declaration as proof for the
   changed context. If the existing accepted declaration cannot cover the refreshed
   handoff context, use ordinary declaration recovery or legal host-synthesized carry.
3. For bash/Python reviews, pass the same explicit workflow/step identity and use a host
   completion pass even when there was no provider invocation at that step. A prior
   step's successful finalizer result is insufficient evidence for later script edits.
   Carry is legal only for the retained workspace; release requires current repository
   completion. Keep prepared-monitor semantics untouched.
4. Preserve real provider response persistence, lifecycle phase transitions, error
   handling, and the finalizer fixed-point guarantee. Expose a focused seam consumed by
   the next phase; do not create a parallel finalizer implementation or activate review
   staging in the workflow handler yet.
5. Tests cover a reviewed prompt with an accepted submission, a clean prompt with no
   submission, a script-only review that dirties a repository, and a script after an
   already-finalized prompt. Fresh context and evidence must name the stage; finalizer
   failure prevents publication. Normal invocation, explicit CLI post-declaration
   refusal, and finalizer-owned rejection remain covered. Pin and verify any core
   binding changes as in `gate-contract`.

## workflow-hitl

1. Route agent-context `create_workflow_hitl_shell_gate` and `TUIHITLHandler.prompt`
   through the shared gate staging helper and the preceding host completion seam. Keep
   non-agent mechanical waits and flag-off behavior. Preserve option sets and
   continuation policies for accept, edit, feedback, rerun, reject, timeout, stopped,
   and failed outcomes.
2. Represent successful staging as suspension, not `HITLResult(action="reject")`,
   success/completion of the reviewed step, or an ordinary workflow error. Thread the
   explicit suspension result through prompt, bash, Python, embedded, and control-flow
   execution paths that can reach HITL. Keep `WAITING_HITL`/`waiting_hitl` state and
   persisted output/checkpoint intact, stop subsequent ordinary steps, and return
   control normally so the root can finalize and the runner can adopt. Account for
   finally/post-step cleanup: required cleanup must finish before finalizer settlement,
   and must not accidentally mark the workflow accepted or rejected.
3. Avoid polling or loading an unpublished bundle, running the rejected-workflow retry
   path, or overwriting the real response. The gate follow-up retains the original
   workflow/step identity and receives its existing typed review result. No new general
   workflow resume engine is required; preserve the established follow-up contract.
4. Add integration tests for agent, bash, and Python reviews through workflow execution,
   including a nested/control-flow review and finally cleanup. Assert suspension, no
   next step, no notification before settled finalizers, and correct approval/reject
   follow-ups after publication. Exercise a source artifact changed by a finalizer to
   prove the reviewer sees the frozen staged input. Cover both flag states and keep
   existing `tests/test_workflow_hitl_gates.py` behavior.

## Verification and landing

Use the existing suites under `tests/gate_turn/`, `tests/agent/test_staged_handoff.py`,
`tests/core/test_staged_handoff.py`, `tests/test_staged_handoff_acceptance_matrix.py`,
the launch request/CLI tests, `tests/test_sudo_gate_request.py`, and workflow HITL tests
as the starting points. New tests should prove side-effect ordering and durable replay,
not just mock the staging helper at every caller.

Every coding phase reads `lint_and_test.md`, runs appropriate focused tests, and runs
`sase tool run check` for each changed repository. Use `/sase_monitor` for commands that
cannot finish in the turn; run `just fix` before final verification. Do not run
`just check-full` without a separate explicit instruction. Follow the linked core's
build rules and do not invoke bare cargo. Any source skill edits require a read-only
`sase skill init --diff` preview. Memory, TUI, new CLI options, and tool-source changes
require their respective memory/skill prerequisites if actually needed.

The land agent verifies all five producers together: custom gate, sudo, launch request,
agent-origin run, and workflow HITL. Prove commit-before-publication, legal carry and
release behavior, synchronous auto, preserved real response, own-stage wait refusal,
second-handoff refusal before effects, and idempotent publication. Review the off-state
regressions and plan/question publisher compatibility. General recovery, abandoned carry
handling, final rollout, and parent memory updates remain with their parent phases.

Before closing each child phase, run `sase bead epic-symbols <assigned-phase>` and
resolve its entries or re-key still-needed symbols to a still-open owner. Before closing
the containing phase, run `sase bead epic-symbols sase-1pj.4` and do the same. Close
with `sase bead close sase-1pj.4 --note "<verified behavior and check evidence>"` only
after this child's implementation is integrated. This is the child epic land agent's
responsibility; child phase workers provide evidence and never close ancestors.

Do not create task beads. Record discovered follow-ups on the assigned phase as
`PROPOSED FOLLOW-UP: <summary — evidence>`. A check failure reproduced identically on
the clean base is recorded with that evidence and any existing task ID and does not keep
a finished phase open. The land agent triages proposals. No implementation or bead
closure is implied by this planning turn.
