---
tier: epic
title: Retire the staged-handoff kill protocol
goal: Make every in-agent handoff stage for host completion, remove obsolete kill
  recovery, and score declarations correctly without losing historical evidence.
parent_bead: sase-1pj.8
phases:
- id: command-staging
  title: Make plan, questions, and pipe commands stage unconditionally
  depends_on: []
  size: medium
  description: 'command-staging: remove legacy branches from plan, questions, and
    pipe entry points and migrate their command tests.'
- id: gate-staging
  title: Make pending agent gates stage unconditionally
  depends_on: []
  size: medium
  description: 'gate-staging: remove legacy agent publication and handoff branches
    from shared gate entry, sudo, launch, run, and HITL while retaining synchronous
    auto and outside-agent behavior.'
- id: monitor-staging
  title: Make agent monitor starts stage unconditionally
  depends_on: []
  size: medium
  description: 'monitor-staging: remove the agent monitor kill branch and print-before-kill
    workarounds while preserving prepared completion, ToolRun joins, replay, and outside-agent
    starts.'
- id: runtime-retirement
  title: Delete kill-only adoption and recovery machinery
  depends_on:
  - command-staging
  - gate-staging
  - monitor-staging
  size: medium
  description: 'runtime-retirement: remove dead marker writers, handoff SIGTERM adoption,
    gate-intent and in-flight recovery, and synthetic chat paths while retaining staged
    custody and archived evidence readers.'
- id: readout-core
  title: Define handoff declaration scoring in Rust core
  depends_on: []
  size: small
  description: 'readout-core: add a pure typed classifier and registered Python binding
    for staged and legacy declaration evidence, with Rust and binding tests.'
- id: readout-adapter
  title: Connect instruction readout to the core policy
  depends_on:
  - readout-core
  size: medium
  description: 'readout-adapter: gather staged, receipt, declaration, and historical
    facts in Python, use the Rust classifier, advance the core revision pin, and cover
    the scoring matrix.'
- id: flag-provider-cleanup
  title: Remove the flag and teach providers the final protocol
  depends_on:
  - runtime-retirement
  - readout-adapter
  size: medium
  description: 'flag-provider-cleanup: remove staged_handoffs registry and schema
    entries and flag test scaffolding, update provider handoff instructions and continuation
    safeguards, and verify the integrated unconditional protocol.'
proposed_by: bbugyi200.athena.sase-1pj.8
proposed_with: codex/gpt-6-astra
create_time: 2026-11-02 20:59:29
status: done
bead_id: sase-1pj.8.1
---

- **PROMPT:** [prompts/202611/retire_staged_handoff_kill_path.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/retire_staged_handoff_kill_path.md)
- **PARENT:** [202610/staged_handoffs.md](https://github.com/sase-org/sase--plans/blob/main/202610/staged_handoffs.md)
- **BEAD:** [sase-1pj.8.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pj/sase-1pj.8.1.md)

# Retire the staged-handoff kill protocol

This child epic implements only the `retire-kill-path` phase of
`plan:202610/staged_handoffs.md`, tracked by `sase-1pj.8`. Read that artifact through
`sase artifact read` and this phase through `sase bead read` when starting work.

## Findings and scope

The planning checkout still contains the complete dual protocol. The assigned bead is in
progress, has no prior implementation/remainder notes, and its dependencies `sase-1pj.6`
and `sase-1pj.7` are closed. Do not redo adoption reconciliation or workspace retention.
The live registry identifies the retirement flag bead as `sase-1pk`, which is open. No
epic-symbol entries currently name `sase-1pj.8`.

The work spans three independent command families, shared runtime cleanup, and a readout
policy change. Separate agents can implement these bounded phases. The Rust policy lands
before the Python caller so the caller can pin an actual core commit. The final
integration phase waits for every command family and runtime cleanup.

The parent epic's reviewer decisions are final. This child epic edits no memory,
generated skill sources, user documentation, or instruction-memory projections.
`sase-1pj.9` owns those changes, including its authorized decision records, glossary
strands, `macros.md`, and final-declaration template. Do not re-ask those decisions or
broaden their authorization. Provider adapter strings and CLI help are in scope here
because they directly explain the changed executable protocol.

## Invariants for every phase

1. In-agent asynchronous handoffs reserve one slot, validate, and persist a staged
   record before any successor, gate publication, or monitor process starts. They print
   `Staged`, exit successfully, and direct the root agent to `/sase_final`. The host
   adopts only after finalizers settle. Never add a new kill-based fallback.
2. Keep one-handoff locking, root/helper restrictions, finalizer-owned-turn refusal,
   stage-before-declare validation, user-kill cancellation, and idempotent adoption.
   Keep the existing Rust-backed ownership and carry rules. Plans cannot carry;
   inheriting questions, pipe, gates, and monitors can. Prepared monitors commit on
   green through their existing prepared intent.
3. `%auto` gates continue synchronously. Calls outside an agent retain their supported
   immediate behavior. Removing the disabled flag branch does not remove those uses.
4. Preserve normal user cancellation, external SIGTERM/OOM evidence, failed adoption
   holds, moved workspace claims, retry suppression after staging, and reconciliation.
   Historical readability does not require retaining executable legacy adoption.
5. During the command migrations, the registry remains as temporary scaffolding; each
   migrated command ignores the flag and has one implementation. The last phase removes
   the registry and all remaining overrides. Tests for commands not yet migrated retain
   their existing behavior until their owning phase lands.
6. Shared domain policy belongs in `sase-core`; Python retains artifact I/O, process
   orchestration, and rendering. Reuse the already-landed staged policy rather than
   inventing replacement Python domain rules during cleanup.
7. When a command migration makes a helper genuinely dead, remove that helper and its
   dead tests in the same phase. Do not defer dead public exports to the runtime phase
   or whitelist them as future work. Runtime-retirement handles the remaining shared
   consumers after the command families are complete.

## command-staging

Change `src/sase/main/plan_propose_handler.py`, `questions_command_handler.py`, and
`pipe_handler.py` to take their existing staged paths unconditionally inside agents.
Remove flag lookups, `use_staged_handoffs` parameters, legacy pending-marker writes,
kill imports, and output that promises an immediate runner termination. Keep validation
and reservation before side effects, archiving and artifact links, plan metadata, and
the existing non-agent contracts.

Update command tests rather than leaving patches for nonexistent kill symbols. Start
with `tests/test_plan_propose_staging.py`, the `test_plan_command_handler*` modules,
`test_plan_propose_derivation.py`, `tests/main/test_pipe_handler.py`, and questions
command coverage. Assertions should prove that commands return with a staged record, no
legacy marker, and no successor publication. Preserve guard failures, archived-plan
associations, JSON/error output, and the data passed to adoption. Delete only tests
specific to the removed Off path.

This phase does not change runner adoption or remove shared writer modules still used by
other command families. Resolve newly dead symbols, or use a narrowly keyed later-phase
epic-symbol only for a real future consumer as `symvision.md` permits.

## gate-staging

Simplify `src/sase/gate_turn/entry.py`: in-agent pending gates always use the existing
staged preparation path; auto gates execute inline; outside-agent calls still publish.
Remove callbacks whose sole purpose is legacy gate intent setup/cleanup, updating
`src/sase/sudo/cli.py` and callers in the gate, launch, and query handlers and
`src/sase/agent/launch_request.py`. Remove the flag branch from
`src/sase/macro/workflow_hitl_gate.py` and follow its callers in `workflow_hitl.py`.

Preserve LaunchApproval's auto refusal, request identity replay, frozen creator and
workspace metadata, signal/lock safety, and the pre-publication one-slot guard. A staged
descriptor has a request ID and status, not a fabricated member or notification. Keep
refusal to wait on one's own staged gate. Do not remove transaction helpers still used
for host publication or outside-agent creation merely because they were named legacy.
Leave wholesale gate-intent deletion to runtime-retirement.

Migrate `tests/gate_turn/test_entry.py`, `test_gate_create_cli_turn.py`,
`test_gate_entry_atomic.py`, `tests/test_launch_request_staging.py`, sudo and run/HITL
tests. Verify pending publication happens after finalization, auto execution is
synchronous, outside-agent publication remains valid, and failed or duplicate
reservation causes no publication. Retain concurrency and request replay coverage.

## monitor-staging

In `src/sase/main/monitor/start.py`, choose staging from agent context alone and remove
the flag-read fail-open route. Keep ordinary outside-agent starts through
`start_monitor`. Update `parser_monitor.py` and output to explain that the command
stages and returns, while execution begins after this turn completes.

Remove in-flight-marker writes and ordering/latency workarounds whose only purpose was
to print before the handoff kill. Keep useful timings and the actual staged transaction:
lane locking, member state, replay fingerprint, ToolRun reservation, `-J` joining
without rerunning, `-f` intent reservation, launch acknowledgment and claim-transfer
ordering. Do not collapse preparation and post-finalizer spawning. Remove unused monitor
handoff exports after their last real caller is gone; the generic marker writer is
retired with the runtime phase.

Migrate `tests/monitor/test_monitor_start_activation.py`,
`tests/main/test_monitor_handler_start_*`, and relevant
`tests/fakey/test_monitor_capacity_e2e.py` coverage. Verify successful staging has no
spawn, adoption launches once, prepared green/red behavior remains intact, joining does
not rerun a command, and outside-agent starts still launch immediately. Old
print-before-kill expectations become staged-output/ordering assertions.

## runtime-retirement

After all command families land, remove obsolete runtime mechanisms by following their
production imports. This phase deletes dead policy; it does not redesign the working
staged adoption, reconciler, or retention service.

- Delete `kill_agent_runner_group` in `src/sase/main/utils.py` and legacy writer and
  handoff exports in `agent/pending_handoff_write.py`, `turns/handoff.py`,
  `gate_turn/agent_handoff.py`, and `monitor/handoff.py` once their live callers are
  gone. Keep any generic guards only when a real staged consumer still needs them.
- Reduce `_handle_killed_iteration` in `axe/run_agent_exec.py` to user/external-kill
  responsibilities. Delete marker timestamp comparisons and legacy marker adoption;
  preserve staged error settlement, user-intent discard, no-provider-retry behavior, and
  external-kill evidence. Remove the duplicated handoff marker list and handoff-origin
  classification in `axe/runner_kill_provenance.py`; retain user and external provenance
  plus OOM evidence.
- Remove marker-only SIGTERM release exceptions in `axe/run_agent_runner_signals.py`.
  Keep staged/held-state custody and actual claim-transfer checks. Audit runner
  lifecycle/finalization/scratch handling and `workspace_provider/vcs_release.py` so old
  marker existence cannot bypass the new completion contract or strand a new run.
- Delete `agent/handoff_inflight.py`'s active write/wait/late-adoption/abort machinery
  and its use in `finalizers/declaration_recovery.py`. Delete `agent/gate_intent.py` and
  `llm_provider/gate_intent_guard.py`, removing active callers in gate transaction,
  `_invoke.py`, runner/retry error handling, and sudo. Before deleting the module, move
  all still-live atomic JSON writes to an existing neutral helper such as
  `core.atomic_json.write_json_marker_atomic`; preserve fsync, atomic replacement, error
  propagation, and temporary-file cleanup semantics. Staged storage, receipts, recovery,
  monitor persistence, and invoke error evidence currently import that writer from the
  gate-intent module and must continue working.
- Remove legacy finalizer skip behavior in `finalizers/controller_context.py` and
  related controller/recovery tests. Audit `llm_provider/postprocessing.py`,
  `bead/stranded_facts.py`, and auto-restart/healer inputs: retain historical facts
  needed for diagnosis, without restoring an executable kill route. Existing staged/held
  guards must continue preventing release or restart of unsettled work.
- Simplify kill-only wrappers and `interrupted`/`persist_chat` branches in the
  `axe/run_agent_exec_{plan,questions,pipe,gate,monitor}*` adopters and helpers.
  Preserve metadata, lifecycle phases, plan preview extras, genuine final replies,
  receipt replay, and inline-continuation `None` semantics. Some staged adopters call
  functions still named `handle_*_marker`; do not delete live host behavior by name.

Keep `agent/pending_handoff.py` or a smaller read-only compatibility module for archived
marker interpretation and collision detection where needed. Keep archived
`handoff_aborted.json`, historical `handoff_protocol: killed`, and old done/metadata
records readable without mutating or adopting those records. If a historical reader uses
a constant or projection helper from a deleted module, move only that reader support to
a neutral location. No migration rewrites of archived runs are required.

Tests include `test_axe_run_agent_exec_killed_iteration.py`,
`test_runner_kill_provenance.py`, runner signal/lifecycle tests, staged acceptance and
reconciliation suites, and finalizer controller/recovery tests. Replace marker-driven
plan-chain and fakey pipe fixtures with actual staged/finalized adoption. Delete tests
for removed recovery functions; keep behavior coverage for user cancellation, external
kill, malformed/held stage custody, real chat preservation, and archived readout.

## readout-core

Use `/sase_repo` to open `sase-core` and read its `AGENTS.md`. Implement a small pure
classifier in the core, with explicit request/response Wire types, and expose it from
the PyO3 binding domain. The existing `instruction_manifest` domain is the adjacent
instruction API; use a dedicated submodule or adjacent domain instead of mixing readout
scoring into manifest normalization. No filesystem access is needed in this policy.
Python supplies observed facts and Rust chooses the class.

The input distinguishes staged evidence from historical killed/unspecified evidence,
recognized handoff kind (including prepared versus plain monitor), aborted status,
accepted declaration, and explicit not-required context. Preserve existing output class
names and readout JSON schema. For staged handoffs, use this precedence:

| Evidence                                                            | Class                       |
| ------------------------------------------------------------------- | --------------------------- |
| Recognized handoff, accepted or explicitly not-required declaration | Corresponding handoff class |
| Recognized handoff, required declaration missing/rejected           | `miss`                      |
| No qualifying handoff, accepted declaration                         | `accepted`                  |
| No qualifying handoff, declaration explicitly not required          | `not_required`              |
| No sufficient completion evidence                                   | `miss`                      |

A staged record by itself is not a declaration exemption. An absent/corrupt final
context is not proof that no declaration was required. Prepared-monitor evidence uses
the existing validated context/intent contract, not merely a tool-call substring.
Preserve legacy archived scoring and aborted-handoff interpretation explicitly; do not
retroactively score old successful kill handoffs as missing new declarations.

Test every handoff kind, accepted/not-required/missing evidence, aborted records, legacy
compatibility, and no-handoff runs. Register the binding and test a real dictionary
round trip. Add no Python fallback. This phase lands core and its binding without
introducing a Python caller that could precede the new binding's CI pin.

## readout-adapter

`src/sase/instructions/readout.py` currently returns `accepted` before looking for a
handoff, and its tail fallback can treat a mere plan validation as a handoff. Keep
artifact gathering and output formatting in Python, but delegate class precedence to the
new Rust API through a thin `src/sase/core/` adapter.

Gather staged record/receipt and `handoff_protocol` evidence, existing host lifecycle
metadata, current accepted-submission evidence, explicit not-required context, and
prepared-intent evidence. Prefer durable host facts over tool-call strings. Preserve
bounded legacy tail scanning for historical runs; command mention/help/validation alone
must not establish a successful staged handoff. A held staged handoff with a missing
required declaration must be a miss, even if no adopted handoff lifecycle entry exists.
Preserve aggregation, provider/delivery grouping, excluded helper/gate/ monitor turns,
memory-read totals, and the existing public JSON schema.

Add the Rust binding to the required-binding validation contract as appropriate (read
`tools.md` before changing files under `tools/`). Read `docs/contribute/rust-backend.md`
and advance `sase-core-revision.txt` past the landed readout-core commit with
`just ratchet-core-revision`. Verify with that binding rather than mocking away the new
policy. Expand `tests/instructions/test_readout.py` with staged and archived artifacts
and add thin-adapter binding tests. Avoid copying the classifier into Python or
migrating unrelated instruction code.

## flag-provider-cleanup

Remove `FeatureFlag.staged_handoffs` and its definition from
`src/sase/feature_flags/registry.py`, the accessor in `agent/_staged_handoff_store.py`,
and its facade export. Regenerate the feature-flags schema with
`just sync-feature-flags-schema`. Remove all remaining test overrides/monkeypatches for
this flag, converting both-state tests to the unconditional behavior. Check
`tests/agent/test_staged_handoff.py`, `test_finalizer_owned_turn_handoffs.py`, and the
staged acceptance matrix as well as feature flag registry/schema fixtures. Do not retain
a flag-shaped helper returning `True`.

Rewrite the handoff wording in `llm_provider/_codex_instructions.py` and
`llm_provider/claude.py`: wait for the staging CLI itself to exit, finish through
`/sase_final`, and do not wait for the staged successor/monitor, which starts after the
turn ends. Keep the general warning about unfinished subprocesses being killed.

Review `_SASE_HANDOFF_COMMAND_PREFIXES` and its parser comment in
`_subprocess_codex.py`, plus `codex.py`'s stranded-command guard. Retain protection
against ending a turn while the staging command is still running, but do not claim that
no handoff took effect without checking durable staged evidence. Once a valid stage
exists, do not ask the provider to blindly rerun the command and collide with the
one-handoff guard; preserve the record for normal finalization and host adoption. Update
the continuation nudge and tests in `tests/llm_provider/test_codex_handoff_guard.py` for
both stage-present and no-stage stranded commands, bounded retries, and normal
successful completion. Use existing staged state readers; do not create another recovery
protocol.

Audit remaining `staged_handoffs`, kill-helper, marker-writer, gate-intent, and
in-flight-helper references with `rg`. Remaining hits must be legitimate historical
readers, test history fixtures, or docs/skills explicitly owned by `sase-1pj.9`. Run the
feature-flag and Symvision gates via the project verification workflow.

## Verification and completion

Each coding phase reads `lint_and_test.md` and runs `sase tool run check` in every
changed repository, plus the focused behavioral tests named above. Rust work uses the
repository's `just` wrappers, never bare cargo. Read `symvision.md` before resolving
unused symbols; test-only references are not production consumers. Do not add broad
whitelists. Use `/sase_monitor` for long verification and wait for the handoff command
itself to exit. Do not run `just check-full` under this plan.

The final integration phase verifies the existing staged acceptance, crash/re-adopt,
workspace-retention, and prepared-monitor suites still exercise the unconditional
protocol. Preserve coverage for dirty plan commit-before-publication, carry inheritance,
commit failure withholding adoption, one logical successor after replay, and explicit
user kill discarding the stage. Keep existing full-suite verification scoped through
`just check`; targeted regressions supplement that gate.

The child epic's land agent reviews the final diff and verification evidence, confirms
the core revision pin, and runs `sase bead epic-symbols sase-1pj.8` before completing
the original phase. Resolve every remaining entry or re-key a genuinely pending consumer
to a still-open bead. Every child phase also checks its own epic-symbols before closing
its assigned bead. Do not close `sase-1pj` or any ancestor plan bead. Only the child
epic's land agent completes `sase-1pj.8`, after all child work is verified; phase
workers provide evidence and close only their assigned child phase.

The registry integrity gate reports an error for an open flag bead without its
definition (`tools/check_feature_flags`, rule 8). `sase-1pk` predates the 24-hour
creation grace period, so registry removal and flag-bead closure must be coordinated.
This is a consequence of the planned change, not a clean-base failure to waive.

The current instruction permits closing only the assigned bead. The final cleanup worker
prepares the complete registry/schema/test change and retirement evidence, then uses
`/sase_gate` to request the exact user-owned action:
`sase bead close sase-1pk --note "staged_handoffs Off branch and kill machinery removed; staging is unconditional"`.
Configure a same-workspace follow-up to run verification after the action settles. Keep
the cleanup phase open until that follow-up verifies and closes it. Do not weaken the
flag check, classify this new orphan error as unrelated, or close the flag from the
worker without authorization. The gate should explain that this confirmation comes from
the user's explicit assigned-bead-only restriction. The gate applies only to this exact
administrative action and is created only after the complete removal diff is ready to
review. Audit the flag bead's epic-symbols before proposing its closure. Do not add a
plan decision for this authorization: auto plan approval cannot override the current
assigned-bead-only restriction.

All other child workers close only their assigned phase. The child land agent closes the
completed original phase `sase-1pj.8`; it never closes original epic `sase-1pj`.

Record newly discovered work only as
`sase bead note sase-1pj.8 'PROPOSED FOLLOW-UP: <summary — evidence>'`; do not create
task beads. If verification fails identically on the clean base, record reproduction
evidence and any existing tracking bead there, and close completed assigned work as the
user instructed. Do not leave a completed phase open on that unrelated failure.
