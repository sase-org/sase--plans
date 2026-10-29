---
tier: epic
title: Stage plan handoffs after host finalization
goal:
  Complete sase-1pj.1 so flag-enabled plan proposals preserve edits and publish approval
  only after host finalization succeeds.
parent_bead: sase-1pj.1
phases:
  - id: record-contract
    title: Staged record contract, storage, guards, and rollout flag
    size: medium
    depends_on: []
    description:
      "record-contract: implement Rust-owned staged-handoff contracts and guards, a thin
      Python storage adapter, reciprocal legacy-marker exclusion, and the
      staged_handoffs beta flag."
  - id: adoption-epilogue
    title: Normal completion adoption and authentic planner chats
    size: medium
    depends_on:
      - record-contract
    description:
      "adoption-epilogue: adopt staged plans after successful finalizer convergence
      through a persisted-context adopter API, preserve inline continuation outcomes,
      and retain the model's real final response."
  - id: failure-boundaries
    title: Provider failure, failed finalization, and lifecycle boundaries
    size: medium
    depends_on:
      - adoption-epilogue
    description:
      "failure-boundaries: finalize staged turns after provider failures without
      retrying ordinary work, hold failed handoffs, discard on explicit user kill, and
      audit marker readers and workspace release."
  - id: proposal-command
    title: Enable staged plan proposals and dual-state instructions
    size: medium
    depends_on:
      - failure-boundaries
    description:
      "proposal-command: route flag-enabled plan proposals through staging, preserve
      archival and legacy behavior, and update plan/final skill sources and the
      inherited authorized macros.md guidance."
  - id: acceptance-matrix
    title: Verify commit-before-approval and failure isolation
    size: medium
    depends_on:
      - proposal-command
    description:
      "acceptance-matrix: prove primary-plus-linked repository ordering, declaration
      recovery, finalizer convergence, clean final:none, protocol pinning, and legacy
      compatibility in integration tests."
proposed_by: bbugyi200.athena.sase-1pj.1
proposed_with: codex/gpt-6-astra
create_time: 2026-10-28 20:47:02
status: wip
---

- **PROMPT:**
  [prompts/202610/staged_plan_foundation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/staged_plan_foundation.md)
- **PARENT:**
  [202610/staged_handoffs.md](https://github.com/sase-org/sase--plans/blob/main/202610/staged_handoffs.md)

# Scope and completion ownership

This child epic implements the `plan-staging` section of
`plan:202610/staged_handoffs.md`, tracked by `sase-1pj.1`. Read that artifact with
`sase artifact read` and read the assigned bead before implementation. The phase was in
progress with no earlier implementation notes when this plan was authored. The checkout
still used only legacy markers; the flag and staged record module did not exist. Recheck
what has landed before starting each phase.

The work crosses a durable contract, a host completion state machine, and command
publication. Five sequential phases keep each implementation bounded and prevent the CLI
from exposing staging before normal and failed completion are both handled. Each phase
is direct implementation work for one coding agent. This is a subdivision of the
assigned phase, not a redesign of the parent epic.

The child epic's **land agent** verifies the aggregate result and closes `sase-1pj.1`
with its verification evidence. Individual child phase workers close only their own
assigned beads. Nobody working this child epic closes `sase-1pj`, the ancestor epic.
Before every close run `sase bead epic-symbols <closing-bead>` and remove its entries or
re-key them to a still-open owner. The initial audit found no entries for `sase-1pj.1`.

Record discovered work as `PROPOSED FOLLOW-UP:` notes on the assigned phase; do not
create ordinary task beads. The required flag is created through `sase flag new`, as the
parent epic explicitly specifies. Host finalizers own all commits and publication.

# Binding contract

With `staged_handoffs` enabled in the proposing CLI's inherited `SASE_FEATURE_FLAGS`
snapshot:

1. `sase plan propose` validates and archives the proposal, writes a staged record,
   prints a line beginning `Staged`, and exits successfully without killing anything.
2. The agent finishes through `/sase_final` and its normal final response.
3. The host runs finalizers to their fixed point, including existing declaration and
   failed-close recovery mechanisms.
4. A fixed host epilogue adopts the plan only after successful completion. It publishes
   the existing plan/epic approval flow, or performs the existing synchronous auto
   result. It is not a selectable finalizer and cannot be removed by `%final:none`.

The runner dispatches on durable artifacts, never the live flag. Other handoff kinds
remain on their existing protocol until their parent-epic phases implement them. Keep
the flag disabled by default; dogfooding and global installation remain the user's
choice.

The staged artifact is `<artifacts>/.sase_handoff_staged.json`, containing
`schema_version`, UUID `staged_id`, `kind` (`plan|questions|pipe|gate|monitor`),
`created_at`, kind-specific `payload`, `successor_workspace` (`inherit|none`),
`carry_allowed`, and `dirty_obligations_at_stage`. The plan payload retains at least
`plan_file` and `original_file`, and any immutable creator facts adoption cannot recover
from durable run metadata. Plans always use `none` and `false`, including tales.

The receipt `<artifacts>/.sase_handoff_adopted.json` identifies `staged_id`,
`adopted_at`, outcome, and any gate or successor references. A held handoff preserves
its stage and archived plan and records `handoff_not_adopted` plus a specific reason; it
must never leave a success receipt. Preserve successful receipts for the later
reconciler. A receipt for another staged ID cannot authorize anything.

Do not add this file to `PENDING_HANDOFF_MARKERS` or broaden `has_pending_handoff`
globally: finalizer skipping must remain limited to the legacy kill protocol. Give
callers explicit staged-state queries where they need them.

Shared schema validation, lifecycle decisions, and guard policy belong in `sase_core`.
Python owns host observations, invocation, artifact-path plumbing, and publication. Open
`sase-core` with `/sase_repo`, use the printed path, and read its `AGENTS.md` and the
main repo's `docs/contribute/rust-backend.md`. Do not implement a Python fallback.

# record-contract

Implement a compact Rust module for the staged-handoff wire types and deterministic
guards, with PyO3 bindings registered in the appropriate domain and a thin local
`sase.core` facade. Import by module path, with no new root/prelude aliases. Keep this
contract separate from the finalizer context wire: `FinalizerHandoffWire`, carry
decisions, synthesized carry, and prepared monitor semantics belong to parent phase
`handoff-wire`, which depends on this phase's completion.

Add `src/sase/agent/staged_handoff.py` as the common host-facing API. It gathers
observations, calls the Rust policy, and exposes read/write/discard and receipt helpers.
Use existing durable atomic JSON and locking primitives. Validate schema versions,
kind/workspace/carry invariants, IDs, and required plan payload data. Treat malformed
existing state as a refusal/held failure, not as absence. Keep dirty-obligation
measurement read-only and attributable to this turn; it is evidence, not permission to
omit later obligations.

Enforce one pending handoff per turn across both protocols. Extend
`src/sase/agent/pending_handoff_write.py` so `handoff_guard` and the legacy writer see
staged records. Serialize the final check and write for staged and legacy writers under
one lock so concurrent commands cannot both win. Acquire the staging reservation before
proposal archive/link side effects; release it on pre-stage failure. Define a consistent
lock order with the declaration lock and test stage-versus-submit races. Do not leave a
crash-created reservation indistinguishable from a valid staged record.

Staging is refused in finalizer-owned turns and native helpers. Retain the Claude helper
guard in `src/sase/llm_provider/_claude_helper_guard.py` and existing root-only checks.
Refuse staging after an actual accepted, current final submission. The helper
`final_submission_is_current` also returns true when no declaration is required: test
for a real accepted submission before using it, so clean turns can still stage. Provide
repair guidance that the handoff must be staged before the final declaration, which must
then be refreshed through `/sase_final`.

Read `sase_flags.md`, create `staged_handoffs` only through `sase flag new`, and paste
its emitted registry definition into `src/sase/feature_flags/registry.py`. The enabled
sentence describes staging then host completion; the disabled sentence describes the
existing marker-and-kill behavior; the removal condition names completion of the parent
epic's migration and acceptance checks. Record the flag bead ID on the phase for
`retire-kill-path`; this child epic does not remove it.

Tests cover the Rust wire/binding round trip, invalid records, atomic exclusion in both
directions (including concurrent writers), accepted-submission ordering, clean turns
without submissions, finalizer-owned/helper refusals, discard, and receipt identity.
Bump `sase-core-revision.txt` past the host-created Rust commit using the established
ratchet/finalizer ordering; never pin an uncommitted SHA or leave Python calling a
binding absent from the CI pin. Verify the binding from the pinned build.

# adoption-epilogue

Add a host adopter registry keyed by kind, registering only `plan` now. The adapter
accepts the staged record and explicit persisted run metadata, rather than recovering
creator identity from ambient CLI environment. Keep the runner-only inline continuation
context separate so parent phase `adoption-recovery` can later invoke the adopter
outside the runner. Persist any needed creator facts at stage time.

In `_run_execution_loop_bound` in `src/sase/axe/run_agent_exec.py`, dispatch a staged
record after workflow/finalizer success and before ordinary terminal bookkeeping or
claim release. Reuse `handle_plan_marker` and `_handle_plan_via_gate_turn` in
`run_agent_exec_plan.py` through an extracted shared path where necessary. Preserve
`None` as an inline continuation and strings as terminal outcomes. Do not let a handled
stage fire again on the next loop iteration; scope receipts and completion facts to the
original artifacts directory and staged ID before successor state moves.

Require affirmative finalizer settlement for the same turn, including the empty
selected-finalizer success case. Plans with remaining unsatisfied repository obligations
must be held; `%final:none` cannot bypass dirty-plan protection. Commits and accepted
typed deferrals are the existing legal outcomes; this phase does not invent carry.
Preserve the controller's re-observation of repositories after later finalizers mutate
them. Do not adopt at the first successful commit callback.

Preserve lifecycle phase `handoff`, existing `done.json` outcome meanings, successor
relationships, and completion-notification suppression. Set
`agent_meta.handoff_protocol` from the artifact actually consumed (`staged` or
`killed`). Persist success receipts only after successful adoption. Recognize an already
successful receipt before dispatch so an in-process repeat does not publish twice;
cross-crash exactly-once publication remains parent `adoption-recovery` work.

Keep the ordinary provider's actual final response before declaration recovery can
replace the returned invocation result. Save it as the planner chat for both pending
approval and synchronous auto paths; add `format_plan_as_response` output as an extra
section. Inspect `run_agent_exec_plan.py`, `run_agent_exec_gate.py`,
`plan_gate_turn.py`, and `run_agent_exec_finalize_chat.py` for overwrite and duplicate
chat paths. Legacy killed turns retain their synthetic response. Do not invent a model
response for a provider that failed without returning one.

Tests use staged fixtures before CLI activation: gate publication follows finalizer
settlement, `%final:none` works when clean, dirty obligations remain protected,
auto-approval preserves continuation semantics, normal final text survives recovery,
receipts prevent repeat dispatch, and unimplemented staged kinds fail closed.

# failure-boundaries

Provider invocation currently enters `run_finalizers` only after
`invoke_with_instructions` returns normally (`src/sase/llm_provider/_invoke.py`). A
staged provider error must still reach the ordinary completion controller with the
original provider/options and a recorded provider-failure result. Refactor this boundary
without rerunning the ordinary prompt, masking its error evidence, or recursively
running finalizers when their own invocation fails. A dirty plan with no accepted
declaration gets the existing one-shot declaration-recovery turn. Recovery turns remain
forbidden from staging another handoff.

Check staged state before `handle_workflow_error` in
`src/sase/axe/run_agent_exec_retry.py` can sleep, select a fallback, reprepare the
workspace, or start another provider attempt. A staged record must also prevent
automatic restart/healing from re-running the ordinary task. Preserve current retry
behavior when there is no staged record.

`run_finalizers` invokes failed-close recovery but deliberately re-raises the original
failure. Do not equate launching that recovery with a successful fixed point. Adopt only
on proven successful settlement; otherwise persist `handoff_not_adopted`, keep the stage
and archived proposal, write a failed run result, and notify with the held kind, staged
ID, and concrete reason. Leave durable context usable by the later
re-adopt/reconciliation phase. Partial multi-repository success must preserve existing
commit receipts and must not activate the plan.

An explicit user kill wins over staging: discard the staged record in the existing
user-kill path and never adopt it. Preserve external-kill/provider-crash provenance
separately; do not classify all signals as a successful handoff. A process that dies
before recovery can run retains evidence for the later reconciler.

Audit these marker consumers and add targeted coverage for their staged behavior:

| Consumer                                                                                        | Required behavior                                                                                                                                      |
| ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `finalizers/controller_context.py`, `controller_run.py`, `declaration_recovery.py`              | Staged turns never use the legacy skip path; guards still refuse recovery-owned staging.                                                               |
| `workspace_provider/vcs_release.py`                                                             | Keep the claim until the post-workflow adoption boundary; a VCS release step must not release it between finalizers and adoption.                      |
| `axe/run_agent_runner_signals.py`, `run_agent_runner_lifecycle.py`                              | Distinguish user kill, unresolved stage, adopted successor claim, and failed finalization; retain dirty failed work through the existing failure hold. |
| `axe/run_agent_runner_finalize.py`, `run_agent_runner_scratch.py`                               | Use the real adoption outcome for completion and cleanup; preserve evidence needed by a held run.                                                      |
| `llm_provider/postprocessing.py`                                                                | Keep the actual response and error evidence; do not suppress finalization because a stage exists.                                                      |
| `bead/stranded_facts.py`, `agent/auto_restart/inputs.py`, healer helpers found by marker search | Show unresolved staging/held failure correctly and do not restart or discard it as an ordinary abandoned run.                                          |

Audit both helper calls and literal marker names. Keep policy that is shared across
frontends in Rust, with host observations supplied by Python. Do not solve the audit by
making staged records legacy markers. Broad carry-custody hardening and automatic
post-crash adoption belong to later parent phases.

Tests cover provider failure before declaration, recovered versus unrecovered dirty
plans, exhausted declaration recovery, commit/finalizer failure, partial repository
success, user kill, and prevention of retry/reclone/release on unresolved staging.

# proposal-command

Change `src/sase/main/plan_propose_handler.py` only after the host paths above land.
Read the flag through the existing snapshot-aware feature API in the CLI. Outside an
agent, keep the existing rejection. With the flag off retain the legacy validation,
archive, marker, and kill behavior. With it on, use the shared guarded staging
transaction and persist the plan payload instead of a legacy marker. Return/exit 0 and
print a message beginning `Staged` that says nothing starts until the turn ends and
instructs the root to finish with `/sase_final`. Do not wait for approval.

Preserve tier/decision validation, both formatting validation passes, bead/parent
associations, auto-tier behavior, source consumption, archived-plan validation,
proposal-time link derivation/publication, and rollback on preparation errors. A second
handoff or accepted declaration must be refused before scratch/archive/link mutations.
Fix `_derive_links_for_archived_plan`'s obsolete no-finalizer docstring. Confirm
archived plans and link mutations retain their existing host-owned sidecar commit path
rather than becoming new planner repository obligations.

Update only the generated skill **sources** `src/sase/macros/skills/sase_plan.md` and
`sase_final.md`. Teach the dual-state rule: if the command reports `Staged`, complete
through `/sase_final`; otherwise the legacy command mechanically ends the turn. Stage
before declaring, do not mutate after successful declaration, and never wait for one's
own stage. Keep native helpers exempt and preserve the bead-closure rules. Preview with
`sase skill init --diff`; do not deploy global provider skills.

The parent decision `memory_final_exemption = yes` is final and authorizes the matching
dual-state correction in `sase/memory/macros.md`. Use `/sase_memory_write`, read the
note through `/sase_memory_read`, update that existing paragraph, and regenerate its
managed projections using `sase memory init --no-commit`. Check the generation diff
first and keep the inherited memory allowlist intact.

The parent's requested core-template rewrite would regenerate `sase/memory/sase.md`, but
its explicit authorized-note list contains only `macros.md` for this decision. That
mismatch is recorded as a `PROPOSED FOLLOW-UP:` on `sase-1pj.1`. Leave that
template/core-note change for the ancestor land agent to resolve with the allowed scope;
do not silently widen authorization or re-ask the already accepted decisions. The other
accepted memory decisions concern parent `docs-memory` work, not this foundation. No new
memory decision is introduced by this child plan.

Tests cover both flag states, no-agent rejection, pre-side-effect guards, formatting and
archive failure, preserved parent/bead stamping, frozen CLI flag snapshots, and the
source instruction examples. No new public CLI subcommand is needed.

# acceptance-matrix

Build integration tests around the existing finalizer protocol harness and plan-chain
fixtures, using temporary primary and linked repositories and fake provider/gate
boundaries. At the gate-publication callback assert that both expected commits are
already present. Avoid timing sleeps or tests that assert only helper call counts.

Cover these observable cases:

- Dirty primary plus linked repo: propose, declare, normal return, commit both, then
  create approval. Rejection or timeout afterward leaves the commits standing.
- The same edits without a declaration: one recovery turn supplies it, then commits
  precede approval; failure leaves no gate and preserves the archived plan/stage.
- A later finalizer changes the tree: the commit controller reconverges before any plan
  publication; failure to reconverge holds the handoff.
- Clean `%final:none`: adoption still occurs. Dirty `%final:none`: unsatisfied
  obligations cannot activate a plan.
- Provider error after stage does not retry the ordinary task; user kill discards the
  stage; a failed commit never publishes approval. Partial commit success keeps its
  receipts and leaves activation held.
- A flag flip after stage cannot switch runner behavior. Flag-enabled non-plan commands
  still use their existing markers until their later phases land. A second handoff fails
  before publication under either protocol.
- The planner chat contains its actual final message plus the plan preview; both pending
  review and auto-approved continuation keep correct metadata/outcomes.
- Proposal archival and artifact-link writes preserve their current host-owned commit
  route and do not pollute planner obligations.

Extend the named parent-plan regressions with explicit enabled and disabled variants:
`test_handoff_skips_generic_controller`,
`test_controller_handoff_skip_journals_phase_skipped`,
`test_handoff_skips_declaration_recovery`, `test_finalizer_owned_turn_handoffs.py`, and
`plan_chain_golden/test_marker_and_loop_golden.py`. Retain legacy expectations for
disabled cases. Fix only issues inside this child's scope; record parent-phase work as
follow-up evidence.

# Verification and landing

Each coding phase reads `lint_and_test.md`, runs focused relevant tests, formats, and
runs `sase tool run check` for each changed repository. Rust changes also require its
binding tests and the main repo's pinned-binding verification. Follow the core repo's
recipes rather than bare cargo. Use `/sase_monitor` when a required check needs a
handoff; wait for the monitor-start command itself to exit. Never run `check-full`
without a new explicit instruction to do so.

Keep finalizer receipts and verification evidence on the phase notes. A failure that
reproduces identically on the clean base is a `PROPOSED FOLLOW-UP:` with concrete
evidence (and an existing task ID if known), not a reason to strand the completed phase.
Do not rerun unchanged passing verification without a new reason.

The child land agent verifies the staged flag remains beta/off, the core revision pin
contains required bindings, all five phases are complete, and the parent phase's
acceptance matrix is satisfied. Record the flag ID and the public staged/adopter API for
parent `handoff-wire`, `questions-pipe`, `gate-staging`, `monitor-staging`, and
`adoption-recovery`. Check and resolve epic symbols for the child plan and `sase-1pj.1`,
then close `sase-1pj.1` with the verified result and finish the child epic through its
normal host-owned completion path. Keep `sase-1pj` open.

PR-mode successor visibility remains a parent non-goal and is already recorded as a
follow-up on `sase-1pj.1`. Carry, prepared monitor intents, other entry-point
conversions, crash reconciliation/re-adopt, kill-path retirement, broad user docs, and
decision/glossary updates remain owned by their original parent phases.
