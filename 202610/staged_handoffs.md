---
tier: epic
title: Staged handoffs end through host completion
goal: "Turn-ending commands (plan propose, questions, pipe, gate create, sudo request,
  launch request, run, monitor start) stage their handoff instead of killing the agent
  runner. Every root turn ends normally through /sase_final, the host runs the selected
  finalizers to their fixed point, and only then does a fixed host-owned step adopt the
  staged handoff. Agents can commit file changes and propose a plan (or start a monitor)
  in the same turn, and no handoff ever silently drops uncommitted work.

  "
decisions:
  memory_decision_records:
    ask:
      Record the change as a new decision record that partly supersedes the three
      kill-era records?
    memory:
      - decisions
      - decisions:gates-never-block
      - decisions:single-turn-agents
      - decisions:host-owned-completion
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: true
  memory_final_exemption:
    ask:
      Rewrite the core final-declaration text and macros.md so handoff turns no longer
      skip /sase_final?
    memory:
      - macros.md
    default: true
    corrects: Intentional plan/monitor/pipe/questions handoffs skip declaration.
    answer: true
  memory_glossary_kill:
    ask:
      Update the gate turn and monitor glossary strands that say starting one kills the
      agent?
    memory:
      - glossary:gate_turn
      - glossary:sase_monitor
    default: true
    corrects:
      Creating a gate turn from inside an agent hands off and kills that agent's turn
    answer: true
phases:
  - id: plan-staging
    title: Staged-handoff foundation and plan proposals
    depends_on: []
    size: large
    description:
      "plan-staging: create the staged_handoffs beta flag, the staged-handoff record and
      its guards, the runner's post-finalizer adoption epilogue with fail-closed
      semantics, and make `sase plan propose` stage instead of killing the runner."
  - id: handoff-wire
    title: Handoff descriptor and carry disposition in the finalizer wire
    depends_on:
      - plan-staging
    size: large
    description:
      "handoff-wire: add a handoff descriptor to the sase-core finalizer context, a
      carry repository decision legal only for same-workspace handoffs, the
      prepared-intent rule, host-synthesized carry, commit-finalizer support, and the
      revision pin move."
  - id: questions-pipe
    title: Stage questions and pipe handoffs
    depends_on:
      - plan-staging
      - handoff-wire
    size: medium
    description:
      "questions-pipe: make `sase questions` and `sase pipe` stage with carry allowed,
      adopt them through the existing runner handlers with the real final response, and
      update their skills."
  - id: gate-staging
    title: Stage gate, sudo, launch, run, and workflow HITL gates
    depends_on:
      - plan-staging
      - handoff-wire
    size: large
    description:
      "gate-staging: split gate-turn creation into CLI-time validate/stage and
      post-finalizer publish for every gate entry point, keeping %auto gates inline and
      moving the one-handoff guard ahead of publication."
  - id: monitor-staging
    title: Stage monitor starts and spawn after finalization
    depends_on:
      - plan-staging
      - handoff-wire
    size: large
    description:
      "monitor-staging: split monitor start into CLI-time preflight/stage and
      post-finalizer spawn, integrate prepared (-f) completion and -J joins, and close
      the just-fix race between monitor checks and the commit finalizer."
  - id: adoption-recovery
    title: Idempotent adoption, crash reconciliation, and re-adopt
    depends_on:
      - plan-staging
      - questions-pipe
      - gate-staging
      - monitor-staging
    size: large
    description:
      "adoption-recovery: make adoption idempotent for every kind, reconcile staged
      handoffs whose runner died after finalization, and add a re-adopt command for
      handoffs held by failed finalization."
  - id: carry-hardening
    title: Hold workspaces whose carried work has no successor
    depends_on:
      - handoff-wire
      - questions-pipe
      - gate-staging
      - monitor-staging
    size: large
    description:
      "carry-hardening: when carried work reaches a branch with no successor (stopped or
      lost monitor, cancelled question, terminal gate branch, failed adoption), hold the
      workspace and surface it as stranded instead of releasing it."
  - id: retire-kill-path
    title: Remove the flag and the kill-path machinery
    depends_on:
      - adoption-recovery
      - carry-hardening
    size: large
    description:
      "retire-kill-path: delete the staged_handoffs Off branch and close its flag bead,
      remove the runner kill and its coping code, update provider adapter text, and
      teach the instruction readout classifier the new protocol."
  - id: docs-memory
    title: Final skills, docs, and memory
    depends_on:
      - retire-kill-path
    size: medium
    description:
      "docs-memory: rewrite the generated skills and user docs for the single
      end-of-turn protocol, review the epic land prompt, and land the decision record,
      glossary, and final-declaration memory changes."
proposed_by: bbugyi200.athena.research.5f.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-28 20:28:55
status: wip
---

- **PROMPT:**
  [prompts/202610/staged_handoffs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/staged_handoffs.md)

# Plan: Staged handoffs end through host completion

## Why

Today every in-agent handoff writes a marker (`.sase_plan_pending`,
`.sase_questions_pending`, `.sase_monitor_pending`, `.sase_gate_pending`,
`.sase_pipe_pending`) and calls `kill_agent_runner_group()` (`src/sase/main/utils.py`).
`should_skip_finalizers()` (`src/sase/finalizers/controller_context.py`) then skips
every finalizer because a marker exists, so **any edit made before the handoff is never
committed**. Marker adoption runs only on the killed path (`_handle_killed_iteration` in
`src/sase/axe/run_agent_exec.py`) and writes a synthetic chat response because the
model's real reply died with the provider. This already lost work (the bob-cli backlog
landers' plan-status edits, the `sase-1ou.land` sase-core fix; `bob-cli-dw.land`
declined to propose its successor so its edits would commit), and handoffs end about
half of all turns.

The research of record is
`research:202610/staged_handoffs_through_host_completion/staged_handoffs_through_host_completion.md`
(read it with `sase artifact read <ref> "<reason>"`). The user accepted all of its
requirements. This plan follows its recommended solution, including the four adjustments
it makes to the original proposal:

1. **Reuse the declaration and `builtin@commit` for repository decisions, but the
   handoff itself is not a finalizer.** Adoption is a fixed, host-owned epilogue that
   runs after the finalizers settle; `%final:none` cannot remove it, retries cannot
   repeat it, and a later finalizer dirtying the tree cannot race it.
2. **Plans with dirty repositories must commit** (or file a typed deferral). Only
   same-workspace handoffs (monitor, questions, pipe, workspace-keeping gates) get a new
   host-defaulted **`carry`** disposition. Prepared verify monitors keep
   commit-on-green.
3. **Every root turn runs `/sase_final`**; a manifest is submitted only when the context
   says one is required. No empty acknowledgment on clean turns. Native helpers still
   never declare.
4. **All eight entry points** move, not just gates, plans, and monitors.

## Shared design (binding on every phase)

### Turn-end protocol (flag on)

```text
finish file work
→ turn-ending command validates and STAGES (no kill); prints a line starting "Staged"
  that says nothing starts until the turn ends and to finish with /sase_final
→ /sase_final (context shows the staged handoff and allowed dispositions;
               submit only if submission_required)
→ real final message → provider exits normally
→ host: run_finalizers to a fixed point (declaration recovery / failed-close recovery
        as today) → adoption epilogue → gate / monitor / successor
```

### Staged record and receipt

- One record per turn: `<artifacts>/.sase_handoff_staged.json`. It is **not** a member
  of `PENDING_HANDOFF_MARKERS` (`src/sase/agent/pending_handoff.py`), so every
  marker-presence skip stays off for staged turns. Fields: `schema_version`, `staged_id`
  (uuid), `kind` (`plan|questions|pipe|gate|monitor`), `created_at`, kind-specific
  `payload` (everything adoption needs, at minimum what the legacy marker carried),
  `successor_workspace` (`inherit|none`), `carry_allowed`, and
  `dirty_obligations_at_stage` (measurement for the flag period).
- Adoption writes a receipt `<artifacts>/.sase_handoff_adopted.json` (`staged_id`,
  `adopted_at`, outcome, refs such as gate id, monitor id, successor name). A held
  handoff records `handoff_not_adopted` with the failure reason and keeps the staged
  record.
- Put the read/write/discard helpers and guards in one module (for example
  `src/sase/agent/staged_handoff.py`); later phases extend it, not fork it.

### Kind table

| Kind                              | `successor_workspace`                          | `carry_allowed`  | Dirty-repo dispositions       |
| --------------------------------- | ---------------------------------------------- | ---------------- | ----------------------------- |
| plan (tale and epic)              | none                                           | false            | commit or typed deferral      |
| questions, pipe                   | inherit                                        | true             | commit or carry               |
| gate / sudo / launch / run / HITL | inherit iff the gate keeps its workspace claim | true iff inherit | commit, or carry when inherit |
| monitor                           | inherit                                        | true             | commit or carry               |
| monitor with `-f` prepared intent | inherit                                        | n/a              | prepared (commit on green)    |

Tale plans stay commit-only even though approval inherits the workspace (planners almost
never edit files; simpler is better). Do not infer `bead_action` from the handoff: a
lander may legitimately `close` its epic and propose a successor in one turn.

### Guards

- At most one staged handoff per turn: the guard refuses when a staged record **or** a
  legacy marker exists. It runs before any side effect (today the gate path checks only
  after publishing).
- Stage first, then declare: staging after an accepted, current final submission is
  refused with a repair message telling the agent to rerun `/sase_final` after staging.
- Finalizer-owned turns (`src/sase/finalizers/owned_turn.py`) still refuse every
  handoff; a recovery turn only completes the declaration and the already-staged handoff
  is adopted after it.
- Staging commands stay root-only; keep them covered by the Claude helper guard
  (`src/sase/llm_provider/_claude_helper_guard.py`) and the helper instructions.
- No waiting on your own staged handoff ("it starts after your turn ends").
- `%auto`-resolved gates are synchronous calls, never staged.

### Failure semantics

| Situation                                                       | Behaviour                                                                                                                | Owner                                              |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------- |
| Staged, no declaration, every obligation clean or carry-allowed | Host synthesizes `carry`, adopts                                                                                         | handoff-wire                                       |
| Staged, no declaration, commit required (dirty plan)            | Existing one-shot declaration-recovery turn, then adopt                                                                  | plan-staging                                       |
| Finalizers fail (after failed-close recovery)                   | Do not adopt; run fails `handoff_not_adopted`; staged record and archived plan kept; notification names the held handoff | plan-staging (re-adopt command: adoption-recovery) |
| Provider crashes or errors after staging                        | Same as missing declaration; never re-run the provider with a stage pending                                              | plan-staging                                       |
| User kill                                                       | Discard the staged record, as legacy markers are today                                                                   | plan-staging                                       |
| One repo commits, another fails                                 | Keep receipts; hold activation; resume only the remaining work                                                           | adoption-recovery                                  |
| Commit succeeded, host crashed before adoption                  | Reconciler adopts the staged record without recommitting                                                                 | adoption-recovery                                  |
| Adoption itself fails (spawn/publish)                           | Run fails with the adoption error; carried work holds its workspace                                                      | adoption-recovery, carry-hardening                 |
| Plan rejected or timed out after a commit                       | Commits stand; never roll back                                                                                           | plan-staging                                       |

### Flag and rollout

- `staged_handoffs`, kind `beta`, created only with `sase flag new` in plan-staging
  (read `sase_flags.md` first). It is epic scaffolding: retire-kill-path deletes the Off
  branch and closes the flag bead before the epic lands.
- The deciding process is the **CLI** (child of the runner, so it reads the runner's
  pinned flag snapshot through the inherited `SASE_FEATURE_FLAGS`). The runner never
  consults the flag: it dispatches on what was written (staged record vs legacy marker),
  so a mid-run `sase flag enable` can never split one turn across protocols. Record the
  protocol used (`handoff_protocol: staged|killed`) in the run's `agent_meta`.
- Until a kind's phase lands, that kind keeps the kill path even with the flag on. Every
  phase tests both flag states.
- One instruction set for both states: each staging command's output teaches the
  protocol, and skills say "if the command reports **Staged**, finish through
  `/sase_final`; otherwise it already ended your turn." User docs keep describing the
  default (flag-off) behaviour until docs-memory.
- Generated skills are edited only in `src/sase/macros/skills/`; preview with
  `sase skill init --diff`. Do not deploy them (`generated_skills.md`); the user deploys
  from the landed tree.
- Dogfooding is the user's call: once plan-staging is landed and installed, they can run
  `sase flag enable staged_handoffs`.

### Non-goals

- PR-mode projects, where `builtin@commit` opens a PR instead of pushing and epic phase
  agents would not see the lander's commit. Record it as a `PROPOSED FOLLOW-UP:` note.
- Carry for tale plans; empty completion acknowledgments; a heuristic warning for dirty
  monitors that wait on master; a lossless worktree snapshot for carried work (holding
  the workspace covers the first cut).

## plan-staging

Build the foundation and convert plans, which alone fixes the observed data loss.

1. **Flag.** `sase flag new staged_handoffs -k beta` with the three authored sentences;
   paste the registry entry into `src/sase/feature_flags/registry.py`; follow the
   printed both-states checklist.
2. **Staged record module and guards** per the shared design. Extend `handoff_guard` /
   `write_pending_handoff_marker` (`src/sase/agent/pending_handoff_write.py`) so both
   protocols refuse a second handoff of either kind.
3. **Adoption epilogue.** In `_run_execution_loop_bound`
   (`src/sase/axe/run_agent_exec.py`), when `execute_workflow` returns without a kill
   (finalizers already ran inside `invoke_agent`, `src/sase/llm_provider/_invoke.py`),
   read the staged record and dispatch through a kind→adopter registry. Register `plan`
   now, reusing `handle_plan_marker` / `_handle_plan_via_gate_turn`
   (`src/sase/axe/run_agent_exec_plan.py`) and preserving their `None` (inline
   continuation, e.g. an auto-approved plan) versus outcome-string semantics. Keep
   `done.json` outcomes, lifecycle phase `handoff`, and completion-notification
   suppression unchanged. Write the receipt. Design each adopter to take the staged
   record plus persisted run metadata so adoption-recovery can call it outside the
   runner; the inline-continuation case is the one runner-only exception.
4. **Real final response.** Save the model's actual final message as the planner chat,
   with the plan preview (`format_plan_as_response`) as an extra section instead of the
   whole synthetic response.
5. **Finalizer interplay.** Confirm (and test) that a staged turn runs finalizers,
   declaration recovery (`src/sase/finalizers/declaration_recovery.py`), and
   failed-close recovery (`src/sase/finalizers/_failed_close_recovery_assess.py`)
   normally, so a failed commit is repaired before any gate exists.
6. **Failure semantics** owned here (table above): fail-closed `handoff_not_adopted`,
   provider failure after staging (check `handle_workflow_error` retries in
   `src/sase/axe/run_agent_exec_retry.py`), user kill discards.
7. **Marker-reader audit.** For each reader of marker presence, decide and test the
   staged-record behaviour: `src/sase/finalizers/controller_run.py`,
   `src/sase/workspace_provider/vcs_release.py`,
   `src/sase/axe/run_agent_runner_signals.py`,
   `src/sase/axe/run_agent_runner_lifecycle.py`,
   `src/sase/axe/run_agent_runner_finalize.py`,
   `src/sase/axe/run_agent_runner_scratch.py`,
   `src/sase/llm_provider/postprocessing.py`, `src/sase/bead/stranded_facts.py`,
   `src/sase/agent/auto_restart/inputs.py`, and the healer helpers.
8. **`sase plan propose`** (`src/sase/main/plan_propose_handler.py`): with the flag on
   and inside an agent, keep validation, formatting, archiving, and link derivation;
   write the staged record (`successor_workspace: none`, `carry_allowed: false`) instead
   of the marker plus `kill_agent_runner_group`; print the Staged message; exit 0. Fix
   the `_derive_links_for_archived_plan` docstring that says no finalizer runs. Confirm
   the archived plan and proposal-time link writes keep their existing host-owned commit
   path and do not become planner obligations. Flag off: byte-for-byte today's path.
9. **Instructions (dual-state).** `src/sase/macros/skills/sase_plan.md` step 6 and
   `src/sase/macros/skills/sase_final.md` (replace the exemption sentences with the
   dual-state rule, add "stage before you declare", keep the bead-closure rule).
   > [!decision] memory_final_exemption Also make the "SASE Final Declaration" text in
   > `src/sase/main/init_memory/templates/memory-sase.template.md` and the matching line
   > in `sase/memory/macros.md` dual-state, then run `sase memory init`.
10. **Tests (both flag states).** Dirty primary plus linked repo → propose → declare →
    approve: commits land before the gate publishes. Same without a declaration →
    recovery turn → commit → gate; fail closed if recovery fails. Clean propose under
    `%final:none` still creates the gate. A later finalizer dirties the tree → no
    adoption until the finalizers reconverge. Legacy markers stay readable and a staged
    turn never takes the marker-skip path. Flip deliberately (flag-on variants):
    `test_handoff_skips_generic_controller`,
    `test_controller_handoff_skip_journals_phase_skipped`,
    `test_handoff_skips_declaration_recovery`, `test_finalizer_owned_turn_handoffs.py`,
    `plan_chain_golden/test_marker_and_loop_golden.py`.

## handoff-wire

The `carry` vocabulary is shared domain behaviour, so it starts in sase-core (open it
with `sase repo open sase-core`; read its `AGENTS.md` and
`docs/contribute/rust-backend.md` in sase).

1. **sase-core** (`crates/sase_core/src/finalizer/`): add `FinalizerHandoffWire`
   (`kind`, `staged_ref`, `successor_workspace: inherit|none`, `carry_allowed`, optional
   `prepared_intent_ref`) and `handoff: Option<…>` on `FinalizerContextWire` with
   `skip_serializing_if` so existing digests are unchanged and the digest covers it when
   present. Validate it in `validate_finalizer_context` (`carry_allowed` requires
   `inherit`; `dirty_repository` with `submission_required = false` is legal only when
   every obligation is carry-allowed or prepared). Add the carry-legality check for
   repository actions in Rust and a predicate in `crates/sase_core/src/continuation/`
   that says a prepared intent bound or reserved for the staged monitor satisfies the
   declaration. Expose bindings by module path (not new `prelude.rs` aliases); add Rust
   and binding tests.
2. **Python mirror and context.** `src/sase/core/finalizer_wire.py`;
   `_build_live_context` and `_context_payload` in `src/sase/finalizers/declaration.py`
   fill the descriptor from the staged record and show the allowed dispositions;
   `declaration_format.py` and `declaration_context_evidence.py` render it;
   `build_context_requirements` (`declaration_store.py`) sets `submission_required`
   false when every dirty obligation may be carried. `manifest_template` keeps `commit`
   as the default action and lists `carry` as allowed.
3. **Manifest validation.** `declaration_manifest_commit.py` accepts `carry` only when
   the Rust check allows it; carry is not a deferral and never goes through
   `declaration_deferrals.py`.
4. **Host-synthesized carry.** When no submission exists and carry is allowed,
   `ensure_current_declaration` (`controller_context.py`) installs a host-authored carry
   declaration, marked as synthesized.
5. **Commit finalizer.** Carried repositories count as satisfied without a commit:
   `commit_execution.py`, `commit_declaration.py`, `pending_instance_ids` in
   `controller_context.py`, every `action == "commit"` dispatch check
   (`commit_dispatch*.py`, `declaration_manifest_io.py`, `commit_revision_pin.py`), and
   `_context_obligations` in `_failed_close_recovery_assess.py`.
6. **Custody.** Record carried repos in the finalizer result and `agent_meta`; adoption
   adds `carried_to: <successor>`. Prove with tests that
   `src/sase/llm_provider/commit_finalizer_inheritance.py` makes the carried dirt the
   same-workspace successor's obligation.
7. Plan staging fills the descriptor (carry illegal), so a dirty plan still requires
   commit or deferral. Move `sase-core-revision.txt` past the sase-core commit in the
   same change (`just ratchet-core-revision`); the context uses `deny_unknown_fields`,
   so an older core must never see a `handoff` field.

## questions-pipe

1. `sase questions` (`src/sase/main/questions_command_handler.py`) and `sase pipe`
   (`src/sase/main/pipe_handler.py`) stage with the flag on (`inherit`, carry allowed)
   instead of writing the marker and killing.
2. Register `questions` and `pipe` adopters wrapping `handle_questions_marker`
   (`src/sase/axe/run_agent_exec_questions.py`) and `handle_pipe_marker`
   (`src/sase/axe/run_agent_exec_pipe.py`), saving the real final response.
3. Skills `sase_questions.md` and `sase_handoff.md`: dual-state wording ("`sase pipe`
   kills the calling agent…" and "no afterward" become conditional on the command's
   output).
4. Tests (both flag states): a dirty questions turn without a declaration carries, the
   answered follow-up inherits and commits; a pipe successor inherits carried work; a
   turn that commits some repos and carries others. Cancelled questions are
   carry-hardening's.

## gate-staging

1. **Split** `create_gate_turn` (`src/sase/gate_turn/transaction.py`). Stage (CLI, agent
   alive): build the `GateSpec` and mint `request_id`, `validate_gate_spec`,
   `effective_auto_state` (`src/sase/notification_gates/service_creation.py`), the
   finalizer-owned and one-handoff guards, launch preview/slot/continuation validation,
   and the sudo request build; persist the normalized spec in the staged record. Publish
   (runner, after finalizers): run the existing creation transaction (member, claim
   move, `create_gate`, notification) with an explicit creator context instead of the
   CLI environment, then `handle_gate_marker` (`src/sase/axe/run_agent_exec_gate.py`).
   Publication is idempotent through the existing `request_id` replay.
2. **`%auto`** stays synchronous at CLI time exactly as today and writes no staged
   record. LaunchApproval rejects an auto state at stage time, not after creation.
3. **Entry points**, through one shared stage helper that replaces each caller's copy of
   "print, should_handoff, will_handoff": `src/sase/main/gate_handler.py`,
   `src/sase/sudo/cli.py`, `src/sase/main/launch_handler.py` +
   `src/sase/agent/launch_request.py`, `src/sase/main/query_handler/_launch.py`
   (`sase run`), and `src/sase/macro/workflow_hitl.py` +
   `src/sase/macro/workflow_hitl_gate.py`.
4. The staged descriptor printed at CLI time carries `request_id` and a staged status
   (no member name or notification id); update its consumers. `sase gate wait` on the
   caller's own staged gate is refused.
5. `carry_allowed` follows the workspace mode: a gate that releases its workspace is
   `none`. On the staged path, CLI-time gate-intent markers
   (`src/sase/agent/gate_intent.py`) are not written, so `raise_if_gate_intent_lost`
   cannot misfire.
6. Skills `sase_gate.md`, `sase_sudo.md`, `sase_run.md`: dual-state wording.
7. Tests (both flag states): `%auto` gate → inline result, no staged record; a pending
   gate publishes only after the commit lands (closing today's answer-before-commit
   race); a second handoff is refused before anything publishes; publish after a
   simulated retry creates one gate.

## monitor-staging

1. **Stage** (CLI, agent alive): everything in `_start_monitor_locked`
   (`src/sase/monitor/start_flow.py`) up to `launch_monitor` — validation, identity,
   replay fingerprint, frozen outcome policy, claim preflight, member creation in a new
   `staged` state, `-J` validation, and a reservation (not a bind) of any `-f` intent.
   Reserve the ToolRun id at stage time so the id the command prints stays valid, and
   cancel it if the stage is discarded or held. Staged members block another start on
   the lane.
2. **Spawn** (runner, after finalizers): `launch_monitor`
   (`src/sase/monitor/start_launch.py`) — ToolRun start or join record,
   `submit_proc_request` (double fork, ack, claim transfer in `after_ack`, `.go`
   barrier), the start-intent record, then the meta writes of `handle_monitor_marker`
   (`src/sase/axe/run_agent_exec_monitor.py`) with the real final response. Verify the
   runner's normal exit respects the moved claim (`run_agent_runner_lifecycle.py`) and
   the follow-up's wait for the starter's terminal marker
   (`src/sase/monitor/followup.py`) still orders correctly.
3. **Prepared (`-f`)**: the descriptor carries `prepared_intent_ref`; the starter's
   commit finalizer leaves those repos for host completion (no starter commit, so the
   sealed observations cannot drift); the intent is bound at spawn; confirm
   `install_prepared_declaration` (`src/sase/monitor/host_completion_state.py`) still
   validates with a handoff descriptor present. Green commits with no model; red commits
   nothing and enters recovery.
4. **`-J`**: the ToolRun is already running; reserve it for adoption and never rerun it.
5. `sase tool wait` on the caller's own staged monitor run is refused.
6. Skills `sase_monitor.md` and the "Prepared Monitor Completion" section of
   `sase_final.md`: dual-state wording, and document "commit, then wait on CI" (a
   monitor started after the commit lands, the `sase-1o0` case).
7. Tests (both flag states): a dirty unprepared monitor synthesizes carry and the
   follow-up inherits and commits; prepared green/red as above; the monitor's check
   starts only after the commit finalizer's `just fix` hook finished; `-J` adoption.

## adoption-recovery

1. Every adopter is idempotent on `staged_id` (gate `request_id` replay, monitor request
   fingerprint, plan gate creation, questions/pipe successor) and checks the receipt
   first.
2. **Reconciler**: a staged record whose run finished finalization successfully, has no
   receipt, and whose runner is gone is adopted from persisted data without
   recommitting. Find the right housekeeping home (existing run reconciliation or
   scheduler housekeeping). An inline continuation that cannot run outside the runner
   launches a fresh successor instead.
3. **Re-adopt command** for runs held as `handoff_not_adopted`: a new subcommand (follow
   `cli_rules.md`), named in the failure notification. It refuses while obligations are
   still unsatisfied and otherwise adopts.
4. **Partial multi-repo failure**: keep the receipts of repos that committed, hold
   activation, and resume only the remaining work.
5. Tests: crash after commit but before adoption, and after spawn but before the
   receipt, each yield exactly one logical gate or monitor and no rerun.

## carry-hardening

1. When carried work reaches a branch with no successor — stopped or lost monitor
   (`src/sase/turns/settlement.py`, `src/sase/monitor/store.py`,
   `src/sase/monitor/reconcile.py`), cancelled question, terminal gate branch with no
   follow-up, failed adoption — hold the workspace the way failed runs are held
   (`run_agent_runner_lifecycle.py`) instead of releasing the claim, take the existing
   worktree diff snapshot, and surface the hold as stranded work with a clear release
   path.
2. This also closes today's loss path for stopped monitors, so apply it whenever the
   released workspace has uncommitted session-owned work, in both flag states.
3. Tests for each branch, plus "successor exists → no hold".

## retire-kill-path

1. Remove the flag: delete every Off branch, make staging unconditional, remove the
   registry entry, and close the flag bead in the same change.
2. Delete the kill machinery that no longer has a caller: `kill_agent_runner_group`,
   legacy marker writes, the handoff branches of `_handle_killed_iteration` and
   `_marker_predates_kill` (keep user-kill and external-kill handling), the in-flight
   handoff marker and late-adoption wait (`src/sase/agent/handoff_inflight.py`), gate
   intents and lost-intent adjudication (`gate_intent.py`,
   `src/sase/llm_provider/gate_intent_guard.py`), monitor-start latency workarounds that
   existed only to print before the kill, SIGTERM claim-release marker exceptions,
   synthetic handoff chat responses, and marker duplicates in
   `src/sase/axe/runner_kill_provenance.py`.
3. Keep historical artifacts readable (old marker files and `handoff_aborted.json` in
   archived runs).
4. Provider adapters: rewrite the handoff text in the Codex single-turn directive
   (`src/sase/llm_provider/_codex_instructions.py`), review the stranded-handoff nudge
   around `_SASE_HANDOFF_COMMAND_PREFIXES`
   (`src/sase/llm_provider/_subprocess_codex.py`) and the Claude single-turn directive
   (`src/sase/llm_provider/claude.py`).
5. Readout classifier (`src/sase/instructions/readout.py`): a handoff with an accepted
   or not-required declaration stays in its handoff class; a dirty handoff without one
   is a miss.
6. Run the full flag-removal checklist from `sase_flags.md` and `symvision.md`.

## docs-memory

1. Skills, final wording: `sase_final`, `sase_plan`, `sase_monitor`, `sase_gate`,
   `sase_questions`, `sase_handoff`, `sase_sudo`, `sase_run` (delete the exemption; one
   protocol). Preview with `sase skill init --diff`; do not deploy.
2. User docs (read `docs_writing.md` first): `docs/reference/automation/monitors.md`,
   `monitor-operations.md`, `notification-gates.md`, `sudo.md`,
   `docs/reference/config/finalizers.md`, and any plan/questions pages that describe the
   kill.
3. Review the `bd/land_epic` prompt in `src/sase/default_config.yml` for guidance that
   assumes a lander's edits cannot commit alongside a proposal, and simplify it.
4. > [!decision] memory_decision_records Write a new decision record (for example
   > `handoffs-end-through-host-completion`): a turn-ending handoff is staged, the turn
   > completes through host completion, and the host then adopts the handoff. Mark
   > `gates-never-block` (the "kills the creating agent's turn immediately" clause),
   > `single-turn-agents` (the "terminates the runner" mechanism), and
   > `host-owned-completion` (the mechanical-exemption sentence) `superseded-in-part`
   > with back-links. The no-blocking and host-ownership principles stay; the reopen
   > condition stays "true suspend/resume".
5. > [!decision] memory_glossary_kill Update `glossary:gate_turn` and
   > `glossary:sase_monitor` so starting one stages a handoff that the host adopts after
   > the turn completes.
6. > [!decision] memory_final_exemption Final wording for the "SASE Final Declaration"
   > template text and `macros.md`: every root turn ends through `/sase_final`,
   > including handoff turns.
7. Run `sase memory init` to regenerate `AGENTS.md`, `CLAUDE.md`, and the memory README.

## Verification

Each phase runs `just check` (read `lint_and_test.md`) and adds the tests listed in its
section, in both flag states until retire-kill-path. Cross-repo phases also run the
sase-core test suite and the pinned-binding check.
