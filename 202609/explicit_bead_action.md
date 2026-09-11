---
tier: epic
title: Require explicit bead decisions when creating stitches
goal: Prevent commits from closing unfinished beads by requiring an explicit close
  or keep decision for bead-associated stitch creation, including host finalization
  and checkpoint recovery.
phases:
- id: core_contract
  title: Shared bead decision contract
  depends_on: []
  size: medium
  description: 'core_contract: add a strict Rust bead-action policy and Python bindings,
    plus authenticated finalizer context for the assigned bead.'
- id: integrate_policy
  title: Enforce and carry the decision through every commit path
  depends_on:
  - core_contract
  size: medium
  description: 'integrate_policy: replace the opt-out CLI, enforce preflight and resume
    rules, carry declared choices through host finalization, and migrate callers,
    skill sources, documentation, and focused tests together.'
- id: verify_lifecycle
  title: Verify lifecycle and recovery across the integrated system
  depends_on:
  - integrate_policy
  size: medium
  description: 'verify_lifecycle: exercise real phase and task beads with local commit
    remotes, test interrupted close recovery and declaration propagation, and fix
    integration defects before landing.'
proposed_by: bbugyi200.athena.0jp
create_time: 2026-09-11 15:33:34
status: wip
bead_id: sase-zq
---

- **PROMPT:** [prompts/202609/explicit_bead_action.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/explicit_bead_action.md)
- **BEAD:** [sase-zq](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zq/README.md)

# Explicit bead decisions for stitch creation

## Problem and scope

The user reports phase beads closing before their work is finished. The affected command
is `sase stitch create`: it owns `-B`, and its post-commit hook invokes
`sase bead close`. Interpret the request's later reference to adding this decision to
`sase bead close` as referring to that stitch workflow. Direct `sase bead close ID`
already expresses an explicit close request and keeps its existing interface.

Current behavior is implemented in `src/sase/workflows/commit/bead_hooks.py`. The
resolver treats an assigned `in_progress` bead as eligible unless `do_not_close_bead` is
true; it does not exclude phase or plan beads. `docs/commit_workflows.md` incorrectly
says it does. `src/sase/main/stitch_create_handler.py` gets the association from
`SASE_BEAD_ID` and only writes an opt-out into the payload when bare `-B` is present.

`CommitWorkflow.run()` performs repository reconciliation, bead synchronization, commit
hooks, and dispatch before its final close step. `workflow_resume.py` replays the
checkpoint payload. The host finalizer's
`src/sase/finalizers/commit_repair.py::run_stitch_create()` supplies no bead decision,
and finalizer repository declarations currently accept only a commit message and
repository action. Fixing the parser alone would therefore leave both a policy bypass
and a broken host completion path.

This is an epic because the change crosses the separately released Rust binding, Python
CLI and workflow, and authenticated host declaration/recovery contracts. Keep the Python
behavior switch and all its producers in one phase so no released CLI rejects the host's
own commands. The first phase adds unused capabilities; the second enables a complete
feature. Each phase includes its own focused tests; the last phase verifies interactions
through real subprocesses and bead stores.

## Chosen UX

Replace the Boolean opt-out with `-B/--bead-action {close,keep}`:

```bash
# Intermediate work: preserve the assigned bead's status.
sase stitch create -M .sase/commit_message.md -B keep

# The assigned work is complete and verified: close after the commit succeeds.
sase stitch create -M .sase/commit_message.md -B close

# Normal conflict recovery reuses the saved bead ID and action.
sase stitch create --resume

# An older checkpoint needs a decision before recovery can continue.
sase stitch create --resume -B keep
```

Use verbs rather than `true/false` or `yes/no`: a Boolean value on the old negatively
worded option would be easy to invert. `keep` means leave the status unchanged,
including when already closed; `open` would misleadingly imply a reopen operation. One
valued option avoids competing positive and negative flags. Do not offer an `auto`
value, implicit close, or implicit keep for an associated agent. Silently keeping would
prevent accidental closure but would not satisfy the requested explicit decision.

The public option is syntactically optional, with a conditional runtime requirement when
a bead is associated. Do not set argparse `required=True`, add a positional decision, or
prompt interactively. This conditional requirement follows the user's specific
instruction over the general CLI memory's prohibition on required options. Explain it in
help, including both examples and the fact that the association comes from the agent
context rather than `-b/--bug-id`.

Bare `-B`, invalid values, and the removed `--do-not-close-bead` must fail with a clear
migration diagnostic showing `-B keep` and `-B close`. Retain the old long spelling only
as an error tombstone if helpful; do not preserve an executable legacy branch. Repeated
bead-action options must not silently take the last value. Sort the touched public
options alphabetically and supply the short/long pair. Preserve the established
argument-error versus workflow-error exit conventions.

| Context                                   | Omitted action           | `keep`                                     | `close`                                                 |
| ----------------------------------------- | ------------------------ | ------------------------------------------ | ------------------------------------------------------- |
| Assigned bead, primary commit or PR       | Reject before mutations  | Commit; leave status unchanged             | Commit, then explicitly close an eligible bead          |
| Assigned bead, proposal                   | Reject before mutations  | Create proposal; leave status unchanged    | Reject: proposals cannot complete the bead              |
| Assigned bead, linked/external/SDD commit | Reject before mutations  | Commit; leave status unchanged             | Reject: only the owning primary repository may close it |
| No assigned bead                          | Existing commit behavior | Allowed no-op for uniform internal callers | Reject: there is no assigned bead to close              |
| Assigned bead already closed              | Still require a decision | Preserve the close                         | Idempotent; no duplicate close note or event            |

All bead types use the same explicit decision rule. Phase work can close its own phase
after completion; it must never infer completion from its first commit or close its
parent epic. Retain existing lifecycle validation, including refusal to close unfinished
descendants. Do not add `--force` or bypass those checks. A new `close` request for a
bead in a state other than `in_progress` or `closed` is an actionable error; unreadable
bead or repository identity also prevents closing. `keep` must not require successful
status lookup simply to preserve the status.

## Shared implementation contract

Use a typed `BeadAction` with exactly `close` and `keep`, serialized as the
`bead_action` payload key. Preserve the distinction between absent and explicit values;
reject Boolean, null, empty, and unknown values when a choice is required. Reject the
old internal `do_not_close_bead` input on new operations with migration guidance rather
than allowing it to restore an implicit default.

Put deterministic policy in `sase-core/crates/sase_core`, exposed through `sase_core_rs`
and a thin Python adapter. The policy accepts facts collected by the host: assigned
bead, canonical commit method, repository scope, action, and the status needed to
validate an explicit close. It returns a typed disposition or diagnostic. Python retains
environment/filesystem reads, process execution, rendering, and checkpoint persistence.
Do not introduce a Python fallback or move unrelated bead/finalizer logic as part of
this change.

The core contract must also support validating choices from finalizer repository
decisions against host-issued bead context. Add an optional typed `assigned_bead`
context value containing the canonical bead ID and owning primary repository obligation
ID. It participates in the context digest and accepted context snapshot; omit it when
there is no association. Update Rust and Python wire records, converters, exports, and
versioned fixtures together. Follow the existing wire-version convention for this
additive field, preserving canonical serialization and digest behavior for contexts
without an assigned bead.

The host resolves repository identity, including external repositories, from its
workspace/repository records. A failed primary-repository lookup must not grant close
permission. Preserve bead IDs in commit footer tags for both actions.

## Phase core_contract

Open `sase-core` with `/sase_repo`; if the project's inventory cannot resolve that name,
use `sase repo open gh:sase-org/sase-core -r "Implement explicit bead-action contract"`.
Use only the returned checkout. Read its `AGENTS.md`.

1. Implement the pure policy and its public wire types near the existing bead and commit
   domain modules. Encode the UX table above as behavior tests, including strict
   missing-action handling and positive primary-repository identification.
2. Add the authenticated assigned-bead context to the existing finalizer wire contract.
   Expose the validation entry points through `crates/sase_core_py` and cover JSON round
   trips, context-digest changes when association changes, and rejection of a
   declaration that applies to a different assignment.
3. Verify both Rust and PyO3 consumers. Run the core repo's `just check` or
   `./scripts/check.sh`; `cargo test -p sase_core` alone is insufficient. Respect its
   Python >= 3.12 requirement.
4. Let host finalization and release-plz manage commits and release versions. Do not
   manually bump crate release versions. Record the exported API and release required by
   the integration phase. The current Python dependency range is
   `sase-core-rs>=0.34.0,<0.35.0`; update it only to the actual compatible release
   containing this API when integrating. Local coordinated testing can use the project's
   established binding-build workflow.

Acceptance: the policy and binding tests pass, no old call site changes behavior yet,
and the integration phase has an available core API with no fallback needed.

## Phase integrate_policy

### CLI, workflow, and recovery

1. Update `src/sase/main/parser_stitch.py` and `src/sase/main/stitch_create_handler.py`.
   Normalize aliases before applying policy. Carry the explicit action into the workflow
   rather than inferring it from the method, agent type, or bead status. Keep message
   files on rejection.
2. Add shared preflight at the workflow boundary, not just the CLI. Reject missing or
   invalid choices before origin reconciliation, bead sync, hooks, staging, branch/PR
   reservation, checkpoint creation, or VCS dispatch. Direct `CommitWorkflow` callers
   must obey the same rule. The post-commit close helper must independently refuse to
   close without explicit `close`.
3. Replace the implicit resolver in `bead_hooks.py` with the Rust policy adapter. `keep`
   produces a clear informational result, not a warning that encourages the agent to
   close unfinished work. For `close`, validate predictable failures before committing
   and recheck lifecycle eligibility at the close step because state can change while
   committing. Use the existing non-cascading close operation with `resolution=done`.
4. An explicitly requested close that fails must return workflow failure with the bead
   error, retain the checkpoint, and say that the commit may already exist. Do not print
   overall success or delete the checkpoint first. Resume must retry pending completion
   without creating another commit. Distinguish an intentional keep, an idempotent
   existing close, and an actual close error instead of relying on the current ambiguous
   Boolean helper result.
5. Persist `bead_id` and `bead_action` in the checkpoint before commit side effects. A
   normal `--resume` uses those saved values, unaffected by today's environment. An
   explicitly repeated matching action is allowed; a conflicting action is rejected
   rather than silently changing an existing decision. For a legacy checkpoint with a
   bead but no canonical action, stop before any resume mutation and require
   `--resume -B close|keep`. Validate and save that supplied choice, replacing any
   legacy opt-out field, before proceeding. Do not infer a decision from the old field
   or current environment. Checkpoints without beads retain normal recovery behavior.
6. Keep `close_bead` completion tracking idempotent, including a crash after the bead
   closes but before the checkpoint records it. Historical closes are not reopened
   automatically. Update close notes to state the explicit requested action and commit
   evidence; remove claims that closure was automatic and do not invent verification
   results.

### Host finalizer and producers

1. Publish the assigned bead in the authenticated finalizer context and its
   human-readable presentation. Use the host's run association, not a model-authored
   bead ID or path. Refresh/reject stale context when the association changes. Persist
   the host binding through accepted declarations and recovery, alongside the existing
   repository identities.
2. Add `bead_action` to each `builtin@commit` repository decision when that context has
   an assigned bead. Require an agent-authored value for every such repository decision.
   The manifest template must use an unmistakably unset placeholder such as null that
   submission rejects; never prefill `close` or `keep`. Repositories that cannot close
   the bead still explicitly select `keep`. Without a bead, retain the existing
   declaration shape; optional `keep` remains harmless and `close` is invalid.
3. Validate this through the core policy during submission, before any commit, and when
   loading accepted declarations. Relevant modules include `declaration.py`,
   `declaration_store.py`, `declaration_manifest.py`, `declaration_context_evidence.py`,
   and `commit_declaration.py`. Old associated declarations missing a choice must enter
   the existing declaration-recovery path and obtain a new explicit decision; never
   synthesize completion.
4. Thread the accepted choice through `commit_types.py`, `commit_dispatch.py`,
   `commit_repair.py`, `commit_dispatch_followup.py`, and checkpoint recovery. Pass
   `--bead-action` in host-created stitch commands. Follow-up commits and
   repaired/retried attempts must preserve the decision; include it in retry
   fingerprints so changing a previously missing/invalid decision can be retried. Use
   the saved bead binding rather than reassociating a retry from ambient environment
   variables. Do not introduce a separate direct bead-close path in the finalizer.
   Include the action and pending/completed close outcome in durable operation evidence.
   Audit `commit.py`, reconciliation, and marker-based recovery so a commit hash or
   previously written result marker cannot turn a failed or interrupted explicit close
   into finalizer success. Before accepting completion for `close`, verify the assigned
   bead is closed through the existing read API and reconcile the saved operation; a
   published commit alone is insufficient.
5. Update `sase final defer` and deferral manifests as needed: a deferred repository
   uses explicit `keep`; accepting a commit deferral must not close its bead. Preserve
   existing deferral adjudication. A turn with no commit obligations still needs no
   commit declaration solely because a bead exists; intentional no-code completion
   continues to use `sase bead close`.
6. Inventory all stitch producers and direct workflow callers with `rg`, including
   `src/sase/axe/run_agent_exec_plan_sdd.py`, generated workflow command builders, and
   the `src/sase/scripts/sase_git_commit` wrapper. Mechanical plan, metadata, and
   publication commits explicitly use `keep`. The wrapper forwards the authored
   arguments and does not manufacture a decision for an agent.

### Documentation and focused verification

Update `src/sase/xprompts/skills/sase_git_commit.md` and `sase_final.md` together with
declaration-recovery instructions and the actual phase/task completion prompts that
describe committing or closing. Teach that committing intermediate work uses `keep`, and
`close` requires completing and verifying the bead's full scope. Retain the rule that
epic phase workers never close the parent epic.

Update `docs/commit_workflows.md`, the stitch argument table in `docs/configuration.md`,
relevant lifecycle examples in `docs/beads.md`, and CLI completion snapshots. Remove the
stale task-only description. Search active sources for bare `-B`, `--do-not-close-bead`,
and assertions that commit success automatically completes a bead; avoid rewriting
historical records.

Generated skills must come from source templates. Preview with `sase skill init --diff`
or `--dry-run`; do not hand-edit installed provider skills or deploy from this unlanded
tree. Include their canonical regeneration in the landing sequence so CLI and installed
instructions switch together.

Extend existing suites rather than relying on new tests that only mirror parsing:
`tests/main/test_stitch_create_cli.py`, `test_stitch_create_cli_resume_flag.py`,
`test_stitch_parser.py`, `tests/test_commit_bead_hooks.py`, workflow dispatch/resume
tests, finalizer declaration and reconciliation tests, wrapper tests, skill-source
tests, and completion snapshots. Assert missing-action rejection invokes no mutating
dependency, keeps `-M`, and cannot be bypassed by direct workflow entry. Cover all
method aliases and both action spellings. Inspect declaration templates to ensure they
cannot accidentally submit a default choice.

Run `just check` after integration; read current `lint_and_test.md` through
`/sase_memory_read` first. Run relevant binding tests against the integrated dependency.
Keep required verification failures in scope until resolved.

## Phase verify_lifecycle

Use temporary repositories, local bare remotes, isolated bead stores, and local
agent-context fixtures. Do not close real user beads, publish test PRs, or modify global
skills during tests.

1. Reproduce the reported failure with an in-progress epic phase. A first stitch without
   a choice must fail before mutation; `keep` must commit while leaving the phase in
   progress and dependencies blocked; a later explicit `close` must close only that
   phase after successful commit. Verify its parent remains open.
2. Exercise task and plan/epic lifecycle eligibility, closed-bead idempotence,
   unfinished descendants, unreadable bead state, failed hooks, failed dispatch,
   proposal attempts, and commits from linked, external, and SDD repositories.
   Assertions should inspect bead events/notes and repository state, not just mocked
   helper calls.
3. Test conflict and publication recovery with both saved actions and a changed
   `SASE_BEAD_ID`. Test legacy checkpoints requiring a supplied action and conflicting
   overrides. Inject close failure after a successful commit, then resume and verify no
   duplicate commit, close event, or note. Cover a crash between successful close and
   checkpoint save.
4. Exercise `sase final context` through submission and the real stitch command builder
   for a bead-associated turn. Missing/null choices are refused; `keep` survives
   repair/follow-up; an eligible `close` reaches the same workflow policy. Test multiple
   repositories, changed association after context issue, old accepted declarations,
   deferrals, and an unassociated turn. Inject a successful commit plus failed bead
   close into host recovery and assert it cannot be accepted as complete from commit
   evidence alone.
5. Verify generated skill previews and CLI completion reflect the same contract,
   including the `--resume -B keep` legacy-checkpoint recovery example. Fix any
   uncovered integration defect and rerun the affected tests and required gates.

## Landing and completion criteria

The core API is additive and unused until integration. The Python change replaces the
unsafe policy in one coordinated switch. Do not add a feature flag that can restore
implicit close, nor keep a backward-compatible executable legacy branch. This is the
requested policy correction, not a configurable beta behavior.

Core release precedes the Python dependency update. Host finalization owns commits and
landing. Before landing the combined epic, run `just check-full` through `/sase_monitor`
with the required TESTING/TESTED handoff and complete the core repo's full gate. Do not
manually alter release-plz versions. After the source templates land on the canonical
branch, regenerate provider skills from that clean revision with
`sase skill init --force` and apply the managed output using the generated-skills
procedure. Coordinate this with activation of the new CLI.

Completion requires demonstrated absence of implicit closure across manual stitches,
host finalization, internal commits, and resume, plus passing required verification.
Every associated commit has an explicit saved choice; only `close` can close an eligible
bead; failed explicit closure remains recoverable and visible. Direct `sase bead close`
remains the explicit standalone completion path. Do not repair past bead statuses
automatically: deciding which historical work is incomplete is outside this
implementation's scope.
