---
tier: epic
title: Complete keep-work recovery for hijacked agents
goal: Recover a hijacked agent exactly once in its held workspace while preserving
  its work and obligations.
parent_bead: sase-1qz.6
phases:
- id: recovery_contract
  title: Rust recovery transaction and preview contracts
  size: medium
  depends_on: []
  description: 'recovery_contract: extend Rust hijack wires and policy for durable
    resolver/launch identity, preview decisions, and reconciliation; expose bindings
    and update the Python adapter and core pin.'
- id: recovery_context
  title: Read-only recovery preview and continuation context
  size: medium
  depends_on:
  - recovery_contract
  description: 'recovery_context: implement the shared preview adapter, pinned continuation
    prompts, budget-aware fork fallback, and hijack transcript framing.'
- id: strict_successor
  title: Same-workspace launch with mandatory inheritance
  size: medium
  depends_on:
  - recovery_contract
  description: 'strict_successor: add strict follow-up workspace handling and a recovery
    successor preparation path that requires goal, bead, finalizer baseline, repository
    obligations, and permission inheritance before provider execution.'
- id: resolution_launch
  title: Exactly-once recovery launch and settlement
  size: medium
  depends_on:
  - recovery_context
  - strict_successor
  description: 'resolution_launch: compose the CAS resolution transaction with normal
    admission, reserved successor identity, strict spawning, rollback, lineage, and
    predecessor settlement.'
- id: reconcile_wait
  title: Crash repair and successor-following waits
  size: medium
  depends_on:
  - resolution_launch
  description: 'reconcile_wait: reconcile abandoned resolutions and interrupted settlement
    from durable launch evidence, integrate stale cleanup, and make predecessor waits
    follow the recovery successor.'
- id: recovery_cli
  title: Recovery command, durable operation, and integration coverage
  size: medium
  depends_on:
  - reconcile_wait
  description: 'recovery_cli: expose recover menu, escalate/continue/Instruct, dry-run
    and JSON, add persist-recover, and prove both-repository work ownership with the
    fake provider.'
- id: land
  title: Verify and complete the assigned recovery phase
  size: small
  depends_on:
  - recovery_cli
  description: 'land: verify this child epic, remove or re-key every sase-1qz.6 epic-symbol
    exemption, and supply completion evidence for closing only the assigned original
    phase sase-1qz.6.'
proposed_by: bbugyi200.apollo.sase-1qz.6
proposed_with: codex/gpt-6-astra
create_time: 2026-11-03 08:05:20
status: done
bead_id: sase-1qz.6.1
---

- **PROMPT:** [prompts/202611/hijack_keep_work_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/hijack_keep_work_recovery.md)
- **PARENT:** [202611/agent_hijack.md](https://github.com/sase-org/sase--plans/blob/main/202611/agent_hijack.md)
- **BEAD:** [sase-1qz.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1qz/sase-1qz.6.1.md)

# Complete keep-work recovery

## Scope and existing work

This child epic implements **recover_keep** from the accepted
`plan:202611/agent_hijack.md`, under the assigned phase `sase-1qz.6`. Read that artifact
through `sase artifact read` and read the assigned bead before implementation. The
parent plan remains authoritative. Its referenced research artifact could not be
resolved during planning; the accepted plan provides the complete implementation
contract.

The phase is in progress and had no earlier-work notes when inspected. The predecessor
phases have landed: the Rust ledger/status/eligibility and escalation/continuation
policy, Python wrappers and config, stop orchestration, runner settlement, and custody
exemptions. In this checkout, those changes include `38c3a3c5fc`, `01a28287c8`,
`f163ba74c0`, and `8ac56e2a09`. Check the current tree before editing rather than
reimplementing these foundations.

The remaining work crosses Rust policy, Python launch/bootstrap, persistence, history,
wait resolution, and CLI operations. Six bounded implementation phases plus a landing
phase avoid putting all of those changes into one coding turn. `recovery_context` and
`strict_successor` can proceed independently after the wire contract lands; all later
dependencies are explicit above.

Inherited decisions are final: Enter defaults to Escalate (Switch at the top), then
Continue when escalation is unavailable; successor context defaults to `fork` with
budget fallback to `brief`. The parent authorized only `glossary:agent-hijack` memory
work, reserved for its feature landing. This child epic changes no memory notes and does
not revisit those decisions. Restart, Edit's release backend, Dismiss, recovery TUI
widgets, flag removal, and feature documentation remain with the parent's later phases.
Keep-work recovery remains available when the ingress flag is off.

## Invariants and shared interfaces

1. Recovery claims a held record by CAS before reserving external resources or
   launching. The successful transition persists the action, exact concrete model,
   reserved next session member, resolver identity, and durable launch identity.
   Concurrent callers cannot both launch. A pure calculation of the proposed identity
   may precede CAS; committing its reservation and all other effects follow the winning
   transition.
2. Keep-work launches use the same positive workspace number **and directory**. They
   transfer the held owner's claim, or freshly claim that exact workspace if the old
   claim disappeared. They never fall back to another workspace. Conflicting custody
   fails closed, including a late callback from the predecessor.
3. Continue pins the predecessor's concrete provider/model/effort, not a rotating alias.
   Escalate/Switch pins the previewed target and rechecks its availability at execution.
   Instruct can edit `%model:`; the effective concrete target, including effort, decides
   `ESCALATED` versus `CONTINUED`. Explicit CLI `-m` overrides the prompt model; absent
   that override, an edited prompt model overrides Continue's default. Persist the
   resulting concrete target once, and reject conflicting launch/fan-out directives.
4. Before any successor provider runs, the host must bind the same goal, transfer any
   bead claim, preserve the predecessor's finalizer baselines and opened-repository
   obligations, and inherit permission scope through the existing autonomy rules. A
   predecessor with no bead has no bead to transfer. Missing required evidence or failed
   inheritance aborts recovery. Never replace the predecessor baseline with a new
   snapshot of its dirty tree. Verification receipts and declarations stay with their
   original turn.
5. Normal admission may queue recovery. A durable queued request retains the pinned
   workspace and reserved identity. Queue acceptance alone does not prove that the
   successor owns the work. Settlement requires launch/bootstrap confirmation of that
   identity, custody, and required inheritance. A pending request is not an absent
   successor for reconciliation purposes.
6. `outcome: hijacked` remains on the predecessor. Update only its recovery fields with
   `state`, `action`, `successor`, and `model_changed`. Preserve unrelated fields and
   never overwrite a real non-hijack outcome. Settlement repairs are idempotent across
   the ledger, done marker, and successor metadata.
7. Rust owns state transitions, action availability, reconciliation decisions, model
   selection, and shared prompt semantics. Python gathers facts and performs filesystem,
   process, admission, and launch effects. New shared policy must have a Rust binding;
   no parallel Python state machine or fallback implementation.

Public seams should be small and shared by CLI, durable operations, and later TUI work:

- `hijack_recovery_preview(agent_or_name) -> RecoveryPreview`, with `hijack_id`, ledger
  revision, header evidence, and per-action `available`, `reason`, `label`, `detail`,
  and exact `target`. Include the selected default keep-work action. Restart and Dismiss
  are unavailable with a clear "not yet available" reason in this phase.
- A preparation function for the full continuation prompt, including an optional top
  guidance slot for Instruct. Preview and preparation cannot reserve names, mutate
  model-pool selection, claim workspaces, or advance the ledger.
- One recovery executor used by CLI and `persist-recover`, accepting the name/action,
  pinned model, optional replacement prompt, and optional expected hijack id/revision.
  Return a structured result distinguishing refusal, queued recovery, confirmed launch,
  and error; include successor identity whenever known.
- One idempotent reconciliation entry point used before mutating recovery calls and by
  stale cleanup. Bare menu and dry-run remain read-only, including stale records.

## Phase: recovery_contract

Open the linked repo with `/sase_repo` (`sase repo open sase-core`) and read its
`AGENTS.md`. The existing
`crates/sase_core/src/agent_hijack/{wire,ledger,escalation, continuation,status}.rs`
already implement most model policy. The current record/event wires do **not** carry
resolver identity or admission/launch identity, so merely adding Python dict keys would
lose those fields during a Rust round trip.

Extend the wires and transitions with backward-compatible optional resolver pid and
process-identity token, plus the durable launch identity/request evidence needed for
queued, prepared, and confirmed successors. Define the minimal recovery preview and
reconciliation fact/decision contracts here. Reconciliation observations must
distinguish absent, queued, matching prepared/live/completed, and uncertain/mismatched
successors. Use existing process and launch identity conventions, not names alone as
proof. Preserve old records; malformed or incomplete resolution evidence must keep
custody and return a diagnostic rather than permit a duplicate launch.

Keep action transitions and terminal semantics in Rust. Extend the core admission or
wait wires only where required to transport this identity; document that interface for
the dependent phases. Do not duplicate the existing escalation algorithm or directive
stripper. Add PyO3 registration/round-trip tests, Python adapter changes in
`src/sase/agent/hijack/core.py`, and update `sase-core-revision.txt` to a landed commit
containing the new bindings before Python consumers depend on them. Host finalization
owns commits; never fabricate a future core revision.

Acceptance: old v1 records round-trip, illegal transitions reject, the selected model
and reservation survive all transitions, and decision tests distinguish concurrent,
live-resolver, stale, queued, absent, confirmed, and uncertain resolution cases. Run the
linked repo's required checks and binding validation. Exported APIs awaiting later
phases may be temporarily keyed to the still-open original phase in Symvision.

## Phase: recovery_context

Implement the read-only preview adapter, gathering the existing ledger/model/lineage,
workspace, and header facts and invoking Rust policy. Reuse
`src/sase/agent/hijack/config.py` candidate enumeration and its provider disables.
Preserve the preview's exact target across execution; unavailable targets never rotate
to a different candidate. Expired or resolving records provide disabled keep-work
actions with accurate reasons. Use active and settled identity lookup as needed.

Build continuation prompts from the predecessor's canonical raw prompt and the existing
Rust continuation builder. Sanitize launch directives outside fenced examples while
preserving the task. Instruct edits must go through the same launch-safety validation
and concrete-model normalization without deleting its intentional model selection.

Use the existing continuation projection/request and Rust budget planner for a
side-effect-free preflight. `enforce_continuation_budget` writes decision artifacts, so
do not call it from preview. Size the expanded fork transcript and prompt for the chosen
model, then choose `brief` plus transcript path when the fork will not fit. Retain the
normal invocation-time budget enforcement too. Missing transcript evidence must produce
a useful brief or a typed refusal, not an unhandled fork error.

Add the hijack-specific frame next to `src/sase/history/chat_fork/failure.py`, using
Rust-owned wording/classification where shared. Wire every applicable single-agent,
session, multi-source, and versioned continuation path so the transcript says the user
stopped an unfinished agent because it was not making progress. Preserve transcript
trust boundaries and existing failure rendering for other outcomes.

Acceptance: no preview filesystem/ledger/admission writes; no model-pool rotation;
Continue remains concrete; top-rung Switch and unavailable cases match core policy;
normal fork, oversized fork, brief, guidance slot, fenced directives, model override,
and hijacked session-member framing work. Remove consumed original epic-symbol rows; do
not retain an exemption once a real production caller exists.

## Phase: strict_successor

Add `strict_workspace: bool = False` to `src/sase/turns/followup.py`'s
`launch_turn_followup`. In strict mode validate the positive workspace pair before
spawn, allow only transfer and a fresh claim on that same pair, and route further
failure to `record_not_launchable`. Existing default behavior stays compatible for
monitor/gate callers. Cover mismatched metadata as well as failed transfer.

Build a narrow recovery successor preparation/spawn seam around
`spawn_agent_session_successor` in `src/sase/agent/detached_child.py`. Resolve the exact
reserved session member, preserving solo-agent promotion. Reuse the staged successor
identity/preparation patterns in `src/sase/axe/run_agent_successor*.py` and existing
attach/bootstrap code rather than creating a second general launcher.

Add explicit strict inheritance for recovery. In particular,
`run_agent_runner_bootstrap._capture_commit_finalizer_baseline` currently falls back to
`capture_dirty_baseline`, and `commit_finalizer_inheritance.py` can log partial
repository failures. Those paths must report failure for recovery. Ensure all required
goal, bead, scope, and repository transfers are validated before provider execution,
with an acknowledgement the transaction can observe. Roll back a failed preparation or
transfer ownership safely; prove a partially booted child cannot run while the record is
returned to held. Transport lineage in `agent_meta.json` as
`{hijack_id, from_rung, to_rung, predecessor}`.

Acceptance: transfer success, exact-workspace fresh claim, invalid pair, foreign owner,
and both fallback attempts fail correctly; inject each inheritance failure; verify
original baseline fingerprints and linked-repo obligations; assert receipts and old
declarations are not copied; assert permission scope cannot accidentally widen via
ambient defaults. Add real-bootstrap coverage in addition to mocked spawn tests.

## Phase: resolution_launch

Implement `src/sase/agent/hijack/recover.py` with focused helper modules as needed.
Validate the action and effective target, then CAS held to resolving with the complete
reservation before effects. Reject stale preview revisions and return the current
resolution to repeated callers. Bind the durable admission request to the same
reservation; retries must address that request rather than submit another launch.

Use normal launch admission and a recovery-specific dispatch adapter to the strict
successor seam. A bare low-level subprocess call is insufficient to implement queuing.
Persist enough information before dispatch to survive a resolver crash. Hold the claim
while queued and revalidate the pinned model before dispatch. Cancellation or definite
prelaunch failure returns to held with an error and leaves no live or queued duplicate.
If launch outcome is uncertain, retain resolving and custody for reconciliation.

Confirm the successor identity, claim, inheritance acknowledgement, and effective model
before writing resolved recovery and predecessor metadata. Define an idempotent
settlement helper that the next phase can repair if either write is interrupted.
Preserve `outcome` and use the existing Rust status mapping for ESCALATED/CONTINUED.

Acceptance: two concurrent resolvers produce one durable request and one successor;
repeat requests do not launch again; admission holds do not release the workspace;
unavailable targets fail while held; all three keep-work forms use the same transaction;
failure before spawn returns to held; uncertain post-spawn failure does not retry
blindly.

## Phase: reconcile_wait

Reconcile a resolving record only when its resolver identity is dead and its recorded
age exceeds 60 seconds. Supply observed facts to the Rust decision function. Match the
reserved successor's launch identity, lineage, workspace, and inheritance evidence.
Adopt a proven live or already completed successor; repair partially written settlement.
A matching queued admission request remains authoritative and must not be resubmitted.
Return to held only when no queued or executing successor exists and custody is still
safe. Missing/mismatched/uncertain metadata must retain custody with an actionable
error.

Integrate reconciliation at mutating recover entry and the stale-cleanup pass without
reintroducing release races. Also repair interrupted terminal projection: a record may
have moved to `settled/` before done.json was updated, so use bounded pending-repair
tracking or authoritative record lookup rather than scanning all settled history each
scheduler tick. Make any ledger lock/move handling safe against concurrent retry.

Extend name- and session-target wait resolution to follow `recovery.successor` using the
existing handoff successor mechanisms under `src/sase/core/wait_dependency_resolution/`
and their Rust policy. Pending/expired hijacks stay parked; resumed chains follow the
actual member, including solo-agent promotion and another hijack. Add cycle/missing
successor handling consistent with current pipe waits.

Acceptance: inject crashes before admission, while queued, after creation, after claim
transfer, after acknowledgement, and between settlement writes. Include reused resolver
pids, two cleanup processes, fast-completing successors, and malformed metadata. Prove
no duplicate launch or premature release, then prove name/session waits follow success,
failure, and a subsequent recovery chain.

## Phase: recovery_cli

Add `sase agent recover NAME` and `sase ops agent persist-recover` following the
existing hijack parser/handler and `ops/commands/_agent_persist_hijack.py`
request-sidecar pattern. Register the operation in the operation names/spec and dispatch
tables as needed. Do not gate recovery on the hijack ingress flag.

Public options are alphabetized with aliases: `-a/--action`, `-n/--dry-run`,
`-j/--json`, `-m/--model`, and `-p/--prompt-file`. Implement continue and escalate;
explain the pending restart/dismiss entries until the parent's release phase adds them.
Bare invocation prints the shared preview. Dry-run of an action is read-only. `-p` on
Continue supplies the full edited continuation prompt (Instruct); validate incompatible
option combinations and file errors before claiming resolution. Define consistent JSON
results and exit codes: success/menu/queued = 0, refusal = 2, operation error = 1.

Render the accepted grouped menu with concrete targets and disabled reasons using Rich:
keep work, start over, inspect/leave. The default is Escalate/Switch or the first
enabled Continue. The durable operation transports the pinned target and revision and
invokes the same backend; closing the initiating TUI must not interrupt it. Include
useful examples and update completion/help snapshots through their existing generation
paths.

Add fake-provider integration fixtures using isolated primary and linked repositories.
For Escalate, Continue, and Instruct, begin with predecessor changes and baseline,
recover, then prove the successor observes and owns both repositories through finalizer
declaration. Assert same workspace, goal/bead/scope inheritance, lineage, and labels.
Include a queued launch and a failed inheritance case. These tests must not spawn real
providers or touch the user's live held workspaces.

## Phase: land and completion ownership

Every implementation worker runs focused meaningful tests and the required
`sase tool run check` in each changed repository; read its verification instructions.
Use `/sase_monitor` for long checks and a concrete completion handoff. Do not run
`check-full`. Read relevant memory before touching CLI, macros, ACE, tools, or docs; no
memory changes are required by this child epic.

The child landing worker reviews the integrated diff and acceptance matrix, verifies the
core revision pin, and runs the standard check for the final tree. It must inspect
`sase bead epic-symbols sase-1qz.6`. At planning time the original phase owns these
Justfile exemptions: `AgentHijackConfig`, `agent_hijack_config`,
`build_alias_candidates`, `continuation_wire_schema_version`,
`escalation_wire_schema_version`, `hijack_continuation_prompt`,
`hijack_escalation_target`, `normalize_hijack_rung`, and
`strip_hijack_launch_directives`. Remove consumed entries; resolve redundant wrappers or
re-key genuinely deferred symbols to a still-open responsible parent/later phase.
Consult `symvision.md` before resolving these. Do not close the original phase while its
exemptions remain locally or upstream according to the command.

Child coding workers close only their assigned child phase. They never close
`sase-1qz.6`, the child epic, or the parent feature epic. Completion of **sase-1qz.6**
is delegated to this child epic's land agent by the original request: after all child
work and normal host child-epic settlement are complete, run
`sase bead close sase-1qz.6 --note "<what was implemented and verified>"`. Do not force
a descendant close. If host ordering requires the child epic to settle first, supply
this exact final close step in the host completion handoff. Never close `sase-1qz` or
any ancestor plan bead as part of this phase's completion.

Record discovered follow-ups as `PROPOSED FOLLOW-UP:` notes on `sase-1qz.6`; do not
create task beads. A check failure reproduced identically on the clean base is recorded
with its evidence and any existing tracking bead and does not prevent closing the
completed phase. The final verification note should report same-workspace recovery, all
required inheritance, concurrent/crash tests, wait following, CLI/ops parity, and the
absence of stale phase exemptions.
