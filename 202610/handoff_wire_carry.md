---
tier: epic
title: Handoff context and carried repository completion
goal:
  Complete sase-1pj.2 with validated carry dispositions, prepared-monitor protection,
  and auditable successor custody.
parent_bead: sase-1pj.2
phases:
  - id: core-policy
    title: Rust handoff and repository disposition contracts
    size: medium
    depends_on: []
    description:
      "core-policy: add the optional finalizer handoff wire, Rust disposition and
      prepared-intent predicates, bindings, Python wire/facade mirrors, and paired
      revision pin. Keep live context publication unchanged."
  - id: carry-execution
    title: Execute explicit carry decisions and reach a fixed point
    size: medium
    depends_on:
      - core-policy
    description:
      "carry-execution: validate carry through Rust, satisfy carried repository
      obligations without commits or deferrals, and make controller convergence and
      failed-close assessment recognize current carry evidence."
  - id: live-declarations
    title: Publish staged context and synthesize default carry
    size: medium
    depends_on:
      - carry-execution
    description:
      "live-declarations: project staged handoffs into final context and human output,
      retain optional commit templates, and install authenticated host-synthesized carry
      when an unprepared same-workspace handoff has no submission."
  - id: prepared-bridge
    title: Protect prepared monitor work through starter finalization
    size: medium
    depends_on:
      - live-declarations
    description:
      "prepared-bridge: connect staged monitor intent reservation/binding facts to the
      Rust predicate, satisfy only its covered starter obligations, and preserve sealed
      observations and commit-on-green host completion."
  - id: custody-adoption
    title: Adopt carried work with durable successor custody
    size: medium
    depends_on:
      - prepared-bridge
    description:
      "custody-adoption: accept proven carry/prepared coverage in staged adoption,
      persist carried repository and successor evidence, and prove inherited primary and
      linked work becomes the successor's commit obligation."
proposed_by: bbugyi200.athena.sase-1pj.2
proposed_with: codex/gpt-6-astra
create_time: 2026-10-29 05:51:45
status: wip
---

- **PROMPT:**
  [prompts/202610/handoff_wire_carry.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/handoff_wire_carry.md)
- **PARENT:**
  [202610/staged_handoffs.md](https://github.com/sase-org/sase--plans/blob/main/202610/staged_handoffs.md)

# Handoff context and carried repository completion

## Scope and landing ownership

This is a child epic implementing only **handoff-wire**, bead `sase-1pj.2`, from
`plan:202610/staged_handoffs.md`. Read that artifact through `sase artifact read` and
read the assigned bead before implementation. The parent design and its reviewer
DECISIONS are final. This child makes no memory changes and adds no decisions.

The original phase has no earlier-work notes. Its dependency, `sase-1pj.1`, is closed;
the staged plan foundation is present in the inspected sase tree at `894317dccd`. The
optional finalizer handoff field and carry implementation are absent in both checkouts.
Reuse the foundation rather than recreate staging, admission, or adoption.

Five bounded coding phases are appropriate because the change crosses two repositories
and five separate correctness boundaries: wire policy, execution, declarations, prepared
completion, and adoption/inheritance. Dependencies are deliberately serial so each phase
lands against an implemented contract and overlapping finalizer paths have one owner at
a time. Every phase is direct implementation work for one coding agent.

Phase workers close only their assigned child phase. They do not close `sase-1pj.2`, the
child plan bead, or `sase-1pj`. The child epic's land agent verifies this entire scope
and closes `sase-1pj.2` under the originating request's explicit child-epic delegation.
It must leave `sase-1pj` open for its own land agent. This paragraph is not permission
for a phase worker to close an ancestor.

Do not create task beads. Record discovered out-of-scope work with
`sase bead note <assigned-phase> 'PROPOSED FOLLOW-UP: <summary — detail>'` for
land-agent triage. Preserve the outer epic's boundaries: commands that stage questions,
pipe, gates, or monitors belong to its later phases, as do recovery/re-adoption,
workspace holds with no successor, flag retirement, skills, and documentation/memory
changes.

## Existing implementation and required behavior

- `src/sase/agent/staged_handoff.py` already owns the durable staged record, storage
  guards, and handoff-then-declaration lock order. Its Rust contract is in
  `crates/sase_core/src/staged_handoff/`. Read validated records, not a second marker
  format. Runtime dispatch depends on durable protocol artifacts, not the current flag.
- `FinalizerContextWire` in both repos has no `handoff`; Rust uses
  `deny_unknown_fields`. `finalizer_wire_to_json_dict` already drops `None` fields.
- `declaration_store.build_context_requirements` requires a declaration for every dirty
  commit instance. `manifest_template` omits optional payloads entirely.
  `_validate_commit_payload` accepts only `commit`.
- `controller_context.ensure_current_declaration` returns immediately when submission is
  optional. `final_submission_is_current` also returns true before inspecting an
  optional submission. Both need deliberate treatment of explicit declarations.
- `pending_instance_ids` reruns successful commit instances whenever their dirty trigger
  persists. `_finalizer_settlement_refusal` in `axe/run_agent_exec_staged.py` currently
  accepts remaining dirt only through typed deferrals. Neither can simply keep its
  existing behavior for carry.
- Prepared intents are sealed before monitor start. `continuation/completion.rs`
  supports prepared/bound lifecycle states, and the Python store binds intents through
  `_prepare_intents.py`. The descriptor alone must never authorize early commit or waive
  the seal, ownership, command, or observation checks.

The end state is:

| Staged handoff                                    | Dirty repository behavior                                                             |
| ------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Plan, including tale                              | Commit or existing typed deferral; carry is illegal                                   |
| Questions, pipe, unprepared monitor               | Explicit commit or carry; absent submission defaults to host carry                    |
| Gate retaining the same workspace                 | Explicit commit or carry; absent submission defaults to host carry                    |
| Gate releasing the workspace                      | Commit or existing typed deferral; carry is illegal                                   |
| Monitor with valid reserved/bound prepared intent | Starter leaves covered work for prepared host completion; green commits, red does not |
| No staged record or legacy killed handoff         | Existing behavior                                                                     |

Carry is successful custody transfer, not a commit, deferral, empty acknowledgment, or
permission to discard changes. It never fabricates a commit SHA, dispatch marker,
verification verdict, or bead action. Explicit commit decisions retain today's bead
validation; do not infer `keep` or `close` from the handoff kind.

## core-policy

Open `sase-core` with `/sase_repo`, use only its printed checkout path, and read its
`AGENTS.md`. Read `docs/contribute/rust-backend.md` in sase for paired pin behavior.

1. Add `FinalizerHandoffWire` in `crates/sase_core/src/finalizer/` with `kind`,
   `staged_ref`, `successor_workspace` (`inherit|none`), `carry_allowed`, and optional
   `prepared_intent_ref`. Add optional `handoff` to `FinalizerContextWire` with
   `serde(default, skip_serializing_if = "Option::is_none")`. Reuse the existing staged
   kind/workspace vocabulary where practical. Validate required strings and the kind
   table, including plans always rejecting carry, and `carry_allowed` requiring
   workspace inheritance. A prepared intent applies only to a staged monitor.
2. Extend `validate_finalizer_context` so a dirty repository requirement may omit a
   submission only with legal carry or prepared coverage. Keep all repository
   obligations and their digests visible. Handoff fields participate in the context
   digest when present; legacy contexts must serialize and hash identically.
3. Put repository action legality in Rust. Expose a reusable check for `commit` and
   `carry` against the host context; reject unknown actions and carry without a valid
   inheriting handoff. Keep typed deferral adjudication separate. Do not implement a
   second policy table in Python.
4. Add a continuation-domain predicate for whether a sealed intent reserved or bound to
   this staged monitor satisfies the starter's declaration obligation. Its typed request
   must carry enough host-observed facts to check intent/seal validity, creator
   run/agent/turn/workspace, staged identity, monitor identity, exact command and
   request fingerprint, repository coverage, and unchanged observations. A raw
   `prepared_intent_ref` or unrelated prepared intent is insufficient. Return covered
   repository IDs plus a precise refusal, or equivalent typed evidence callers can use
   without reimplementing policy.
5. Specify the reservation facts explicitly in the API and tests. A valid prepared
   intent can remain in its existing `prepared` lifecycle state while a host-owned
   reservation associates it with the staged ID and intended monitor. Do not bind at
   staging time or add a second intent lifecycle just for this predicate. Actual
   reservation creation and spawn-time binding remain the outer monitor-staging phase's
   responsibility. Document the exact adapter inputs for that phase.
6. Register new PyO3 functions in the existing binding domains: finalizer functions are
   currently in `crates/sase_core_py/src/bead_decisions/`, continuation functions in
   `crates/sase_core_py/src/continuation/`. Import by owning module path; add no
   `prelude.rs` aliases. Add Python wire parsing/serialization and thin facade calls in
   `src/sase/core/finalizer_wire.py`, `finalizer_facade.py`, and the continuation
   counterparts. Leave live context construction unchanged until its consumer phase.
7. Add Rust policy/digest tests, PyO3 registration/round-trip tests, and Python facade
   parity tests. Cover every kind/workspace combination, absent optional field, digest
   mutation, malformed prepared evidence, wrong monitor/command/owner, consumed or
   invalidated intent, and incomplete coverage. If any schema version changes, update
   every mirror and schema probe together; prefer the additive optional field while
   preserving legacy digests.

Declare changes in both repos through `/sase_final`. The host's paired-repo finalizer
commits the core sibling first and writes its pushed SHA to `sase-core-revision.txt`
before committing sase. Do not invent a future SHA or manually commit. If consuming an
already-landed core change, use `just ratchet-core-revision` and verify the resulting
pin contains the bindings. Never send the new descriptor to an older core.

## carry-execution

This phase supplies the consumer before live context publication enables carry.

1. Extend `declaration_manifest_commit.py` to validate `{repo_id, action: carry}` using
   the Rust action check. Preserve complete/unique repository coverage and host identity
   checks. Carry requires no commit message or bead action; reject misleading
   commit-only fields. It must not enter `declaration_deferrals.py`, and a repository
   must not simultaneously carry and request a typed deferral.
2. Trace accepted decisions through `commit_declaration.py`, `commit_execution.py`,
   `commit_dispatch*.py`, commit validation/reconciliation helpers,
   `declaration_manifest_io.py`, and `commit_revision_pin.py`. Partition commit work
   from carried work before commit-only hooks and dispatch. Carry must not invoke stitch
   creation, rebase/repair, commit hooks, pin rewriting, or commit bead actions. Retain
   host identity, dirt/ownership, stale declaration, and discarded-work checks. Do not
   change every `action == "commit"` to accept carry blindly: those checks that govern
   actual commits and their pins must remain commit-specific.
3. Return successful carry evidence recording the repository ID, its accepted state
   digest, and the staged identity; retain the ordinary aggregate/result schema where
   its existing evidence fields suffice. Mixed commit/carry results must retain both
   kinds of evidence, including partial commit failure without treating carry as a
   pushed receipt. Prepared satisfaction uses a distinct evidence kind.
4. Make `pending_instance_ids` use proven satisfaction of the current obligations.
   Unchanged carried dirt must reach a fixed point. New repositories or changed digests
   after a later finalizer must be re-observed and covered before settlement; an
   instance's prior success alone cannot suppress them. Use Rust for the shared coverage
   decision if the controller needs a new domain predicate.
5. Adjust `_context_obligations` and associated failed-close assessment so valid carry
   is not mistaken for a failed commit. Exclude only repositories with current proven
   carry/prepared coverage; real uncommitted commit decisions still receive recovery.
6. Add controller/protocol tests for all-carry, mixed primary/linked commit and carry,
   partial commit failure, typed deferral coexistence on different repos, unchanged
   fixed point, and later finalizer dirt. Assert no commit hook, pin write, synthetic
   SHA, or erroneous bead close occurs for carried repositories.

## live-declarations

1. Extend `_build_live_context` and `_context_payload` in `declaration.py` to read the
   validated staged record and populate the descriptor. Use a stable, persisted
   reference to that exact staged record; preserve its staged ID for coverage evidence.
   A stage reservation file is not a completed stage. Invalid staged state must fail
   closed. Observe existing lock ordering: never acquire the handoff lock while already
   holding the declaration lock.
2. Pass the descriptor through `build_context_requirements`. Rust-backed legal carry
   permits dirty commit requirements with `submission_required: false`; it does not
   suppress unrelated providers' required payloads or hide repository obligations. Until
   prepared-bridge is implemented, a prepared descriptor must not be treated as ordinary
   carry or relaxed solely because it names an intent.
3. Render staged identity, kind, successor workspace, and allowed repository actions in
   `declaration_format.py` and `declaration_context_evidence.py`. Keep `commit` the
   default in `manifest_template`, including optional dirty commit payloads so agents
   can elect to commit. Put permitted alternatives in context/help metadata rather than
   adding unrecognized fields to submitted repository decisions. Clean turns still
   require no empty submission.
4. In `ensure_current_declaration`, synthesize and persist a host-authored carry
   declaration only when there is no submission and all relevant repository obligations
   qualify. Use the ordinary context/host identity validation, declaration lock, and
   atomic persistence; add explicit synthesized provenance alongside the envelope, not
   inside Rust wire fields that reject extras. Cover every dirty repo.
5. Preserve a valid explicit declaration, especially explicit commits and mixed
   decisions. Fix the optional-context shortcut so an existing stale/malformed
   submission is not accepted or silently replaced by carry. Required external provider
   payloads and dirty commit-only plans retain the existing one-shot recovery flow.
   Revalidate at execution, not just when the context was printed.
6. Add context, rendering, template, lock/race, staleness, synthesis, and recovery
   tests. Include no submission on an inheriting handoff, explicit commit overriding the
   default, staged field tampering, dirty plan recovery, non-inheriting gate rejection,
   a required external provider, and legacy flag-off behavior. Construct validated
   non-plan records in tests without converting their CLI entry points here.

## prepared-bridge

1. Build the host adapter that loads the referenced intent from `_prepare_intents.py`
   and supplies reservation/binding facts and live repository observations to the Rust
   predicate. Publish the exact reservation/payload contract in a note on `sase-1pj.2`
   for the outer monitor-staging phase. Its future start path must persist those facts
   before publishing the staged record; this child does not launch/reserve ToolRuns or
   change `monitor/start_flow.py`.
2. Accept only a matching bound intent or a valid host reservation for this staged
   monitor. Missing, cancelled, mismatched, consumed, invalidated, or unreadable
   evidence must not fall back to ordinary carry. Reject uncovered new work and stale
   observations. Prepared coverage satisfies only the obligations it actually seals; it
   does not satisfy unrelated finalizers.
3. Resolve the context-digest boundary explicitly: an intent is sealed before the
   handoff descriptor exists. Authenticate the original sealed context and compare live
   identity, plan, decisions, and observations through the Rust predicate, allowing only
   the validated staging transition. Do not rewrite the seal or disable generic
   stale-context checks. Include a handoff-only change positive test and real content,
   plan, owner, and command drift negative tests.
4. Make starter declaration/execution paths recognize prepared satisfaction without
   installing the sealed commit declaration as an immediately executable starter
   decision. Preserve the sealed worktree and avoid commit-mutating hooks. Keep prepared
   evidence distinct from synthesized carry and retain commit-on-green.
5. Verify and adapt `monitor/host_completion_state.py:install_prepared_declaration` so
   the monitor's actual host completion can install/validate and execute its sealed
   commits with the descriptor present, without remaining trapped in starter prepared
   satisfaction. Retain the existing eligibility and verdict checks.
6. Test a reserved starter and an already-bound starter, failed reservation/binding,
   green execution with no model, red execution with no commit, partial/uncovered work,
   and subsequent valid declaration installation. Reuse `test_final_prepare.py`,
   continuation facade fixtures, and monitor host-completion harnesses. Stub launch
   boundaries; do not depend on the outer monitor-staging phase landing first.

## custody-adoption

1. Extend `_finalizer_settlement_refusal` in `axe/run_agent_exec_staged.py` to accept
   current dirty work covered by authenticated carry or prepared evidence, in addition
   to existing typed deferrals. Cross-check descriptor/staged identity, repository
   identity/digest and coverage, accepted explicit/synthesized declaration or prepared
   proof, and the same-turn settled result. Preserve all existing settlement checks.
   Shared coverage policy belongs in Rust; Python supplies persisted and live facts.
2. Persist carried repository records in `agent_meta` using existing metadata helpers
   and the finalizer result evidence. Define one stable record shape containing repo ID,
   accepted digest, staged identity, and origin run. At successful adoption record
   `carried_to` with the actual successor/custodian identity returned by the adopter.
   Never guess it from the command text or mark failed adoption as transferred.
3. Keep one shared custody-recording seam for future questions/pipe/gate/monitor
   adopters. Extend the existing adopter result shape narrowly if necessary, preserving
   the plan adopter's `None` inline-continuation versus outcome-string semantics and
   receipt replay. Do not implement those other adopters here. Record the seam and
   evidence shape in a note on `sase-1pj.2` for its dependent phases.
4. Prove `llm_provider/commit_finalizer_inheritance.py` transfers the original baseline
   and opened-repo discovery into a same-workspace successor. The successor must see all
   carried primary/linked dirt as its own obligation and be able to commit it. Replays
   must not duplicate discovery/audit records; a different workspace must not inherit
   stale checkout paths. Fix only gaps demonstrated by these tests.
5. Extend the staged acceptance/protocol harness with all-carry without a provider
   declaration, mixed carry/commit, inherited linked work, successful successor commit,
   prepared starter satisfaction, malformed/mismatched evidence, dirt added after
   settlement, and failed adoption that leaves origin custody evidence intact. Use a
   controlled test adopter to prove the generic seam. Workspace hold behavior after
   no-successor outcomes remains outer phase `sase-1pj.7`.

## Verification and child land-agent acceptance

Every coding phase reads `lint_and_test.md` and runs sase's `sase tool run check` (the
guarded `just check` recipe), plus focused tests for its changed behavior. Any core
changes also pass `sase tool run check` in the opened core checkout; use its
`just test -p <crate> <filter>` recipes for targeted runs, never bare cargo. Use
`/sase_monitor` for long commands before starting them, and do not end a turn with an
unmanaged live command. Do not run `check-full` without a separate explicit request.

Build the paired extension only into the workspace venv through supported recipes. Run
the existing pinned-binding validation and confirm the recorded core revision contains
each new binding and wire change. If tool schema probes need edits, read `tools.md`
first. No global install or skill deployment is needed.

The child land agent verifies all five phases have landed, checks the real core pin, and
reviews evidence for these outcomes:

1. Legacy contexts retain byte-equivalent digests; staged descriptor changes invalidate
   declarations correctly; illegal carry is rejected in Rust and through Python.
2. Dirty plans still commit or use typed deferral before adoption. Existing staged-plan
   acceptance tests remain green, including dirty `%final:none` refusal.
3. Explicit and synthesized carry settle once without committing, while later dirt is
   re-observed. Mixed repositories commit only those selected and preserve receipts.
4. Prepared starters leave sealed work unchanged, and green/red monitor completion
   retains its existing commit/no-commit behavior.
5. Adoption accepts only matching current proof, records actual custody transfer, and
   the successor inherits and commits the carried primary and opened linked work.
6. Flag-off legacy behavior remains intact; runners do not reread the feature flag.

Before each phase closes, run `sase bead epic-symbols <assigned-phase>` and resolve each
entry or re-key it to the still-open child epic or its actual consumer phase. Before
closing `sase-1pj.2`, the child land agent must run `sase bead epic-symbols sase-1pj.2`
and perform the same cleanup. There were no entries for `sase-1pj.2` during planning;
recheck rather than relying on that observation.

Record verified clean-base check failures as `PROPOSED FOLLOW-UP:` notes with the
reproduction and any existing tracking bead, and close completed work anyway as the
originating request requires. Do not mark unimplemented product behavior complete.
Finally, the child land agent closes `sase-1pj.2` with a note naming the landed
implementation and validation evidence, and leaves the outer epic `sase-1pj` open.
