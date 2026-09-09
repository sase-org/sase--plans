---
tier: epic
title: Make the commit declaration an authoring step, not a consent vote
goal: "The finalizer declaration asks an agent only to author commit messages. Leaving a
  tree dirty becomes a rare, typed, host-adjudicated deferral that is corrected while
  the agent is still alive and never destroys a run.

  "
phases:
  - id: core
    title: Typed deferral and a non-failing refusal policy in Rust core
    depends_on: []
    size: medium
    description: "core: add the typed deferral-reason enum, the `defer` refusal policy,
      and the non-failing Deferred statuses to the Rust finalizer wire, then release the
      crate.

      "
  - id: adopt
    title: Adopt the released core floor and the deferral config schema
    depends_on:
      - core
    size: small
    description: "adopt: raise the sase_core_rs floor, mirror the new wire records in
      sase.core.finalizer_wire, and accept `refusal: fail | defer` in finalizer config.

      "
  - id: adjudicate
    title: Adjudicate deferrals at submit time instead of after the turn
    depends_on:
      - adopt
    size: medium
    description: "adjudicate: replace the free-text `refuse` action with typed `defer`
      decisions and make `sase final submit` reject unfounded deferrals as repairable
      validation errors while the agent still holds its context.

      "
  - id: escape
    title: A deliberate deferral escape hatch that does not fail the run
    depends_on:
      - adjudicate
    size: medium
    description: "escape: add `sase final defer`, honor `refusal: defer` in the
      controller, and report a deferred turn as a completed run with a dirty tree that
      the user is told about, instead of a FAILED run with a stranded workspace.

      "
  - id: consent
    title: Publish the commit consent model where agents actually read it
    depends_on:
      - adjudicate
    size: medium
    description: "consent: carry the commit-by-default rule and per-repository
      provenance evidence in `sase final context` output, rewrite the `/sase_final`
      skill around authoring, and remove the self-contradiction in the
      declaration-recovery prompt.

      "
  - id: acceptance
    title: Historical regression corpus, live acceptance, telemetry, and docs
    depends_on:
      - escape
      - consent
    size: medium
    description:
      "acceptance: replay every historical refusal as a regression fixture, prove the
      protocol live end to end, add refusal and deferral telemetry, and synchronize the
      finalizer documentation."
proposed_by: bbugyi200.athena.0ca
status: done
bead_id: sase-sp
create_time: 2026-09-09 19:50:20
---

- **PROMPT:**
  [prompts/202608/finalizer_commit_authoring.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finalizer_commit_authoring.md)
- **BEAD:**
  [sase-sp](https://github.com/sase-org/sase--beads/blob/main/pages/sase-sp/README.md)

# Plan: Make the commit declaration an authoring step, not a consent vote

## The incident

`research.10.cdx` (run `20260824083442`, artifacts under
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/24/20260824083442/`) was
one of three researchers in clan `research.10`. It did its job well: it opened the
research sidecar through the audited path, read six prior reports, wrote a 29 KB report,
and verified its own formatting. Then it died with:

```
BuiltinCommitFinalizerError: commit finalizer refused dirty repository research:
The user asked for the research report to be written for review but did not
explicitly request a commit; leave the new report and its audited-read provenance
metadata uncommitted.
```

The run is `FAILED`, its workspace is held, and the report never reached the shared
`research` sidecar. The clan lead `research.10.final` exists precisely to merge the two
researchers' reports; it could not read a report that lives only in a sibling clone's
working tree. The user had to rescue the file by hand
(`ef390b5 feat: Add 202608/decision_web_seed_adrs.md`).

Nothing was wrong with the tree. `sase stitch create` would have succeeded.

## What the agent actually saw

The transcript (`tool_calls.jsonl`, items 115-116) is short and damning:

1. It read `/sase_final`, including the line _"A `refuse` decision needs a substantive
   reason about the changes. Missing conversational context is not a valid reason."_
2. It ran `sase final context -f json`. The context named one obligation
   (`sdd:research`, seven paths) and a `manifest_template` offering
   `{"action": "commit", "message": "feat(scope): describe the completed work"}`.
   Nothing in that payload said what happens if the agent chooses otherwise.
3. It submitted a `refuse` whose reason is verbatim the forbidden kind — missing
   conversational context.
4. **`sase final submit` printed `Accepted final declaration for: commit` and
   exited 0.**
5. The agent wrote a normal final answer explaining that the report "remains uncommitted
   for review".
6. Only after the turn ended did the host convert that accepted declaration into a
   terminal error.

Its prompt genuinely never asked for a commit. The `#research` xprompt
(`src/sase_research_artifacts/xprompts/research.md`) says only _"Write this research to
a new markdown file"_. Every general-purpose coding agent carries a strong learned prior
that it must not commit unless asked. SASE inverts that prior — because a numbered
workspace is ephemeral, uncommitted work is lost work — but never tells the agent so at
the moment it decides.

## This is the last remaining finalizer failure mode

Across all 257 finalizer runs on this machine since the protocol became the only
completion path (`2f9c4ae29`, 2026-08-21), 27 did not succeed:

| Day        | Runs | Failed | Classes                                                                                                                                |
| ---------- | ---- | ------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-08-21 | 106  | 12     | `missing_commit_declaration` 4, `stitch_failed` 4, **`commit_refused` 2**, `missing_commit_result` 1, `dirty_after_commit_decisions` 1 |
| 2026-08-22 | 67   | 11     | `dirty_work_discarded` 8, **`commit_refused` 3**                                                                                       |
| 2026-08-23 | 59   | 3      | **`commit_refused` 3**                                                                                                                 |
| 2026-08-24 | 25   | 1      | **`commit_refused` 1**                                                                                                                 |

Every mechanical defect has been fixed one at a time — `c2f46e84e` serialized accept,
`cf72b00d1` proved auto-commits with a pre-reconciliation ledger, `d8c408552` rejected
stale declarations, `65367452b` gave declaration recovery an evidence brief. The rate
fell from 11% to 4%. `commit_refused` is the only class that never fell, because it is
not a bug in the mechanism. It is the mechanism working as designed.

### The nine refusals

| Run              | Agent                       | Reason (abridged)                                                       |
| ---------------- | --------------------------- | ----------------------------------------------------------------------- |
| `20260821091141` | `098--code`                 | "no commit was requested for this turn"                                 |
| `20260821112329` | `09l--code`                 | untracked link sidecars are "not part of this implementation"           |
| `20260822055049` | `toobig-3e…declaration.0`   | "did not request a git commit"                                          |
| `20260822155532` | `research.0w.cld`           | "Declaration-recovery turn: do not mutate repositories"                 |
| `20260822173908` | `toobig-3h…controller.0--1` | "did not ask for a git commit"                                          |
| `20260823115128` | `0bg--2`                    | "The user did not ask to commit"                                        |
| `20260823120149` | `sase-s9.2`                 | "I lack context to … authorize a commit"                                |
| `20260823154709` | `0by--1`                    | committing the sibling repo "would violate the requested single … turn" |
| `20260824083442` | `research.10.cdx`           | "did not explicitly request a commit"                                   |

Five are the same mistake: the agent applied its default consent prior to a system that
does not use it. Two are recovery-turn confusion. Two (`09l--code`, `0by--1`) are real
scope judgments that deserved a real answer, not a dead run.

**Seven of the nine reasons argue about the conversation rather than about the changes,
and every one of the nine was accepted and then destroyed a run.** Not one of them
describes an unsafe tree.

## Root cause: two different jobs share one field

`payload.repositories[].action` conflates a question only the agent can answer with a
question only the host can answer.

- **Authoring** — _"what does this change say?"_ That is genuine agent work. Only the
  agent knows the change well enough to write `feat(ace): …`.
- **Consent** — _"may this tree be committed at all?"_ That is host policy. It follows
  from facts the agent cannot see from inside its prompt: that its workspace is an
  ephemeral clone, that a clan sibling in a different clone must read the file, that the
  user's standing configuration says agent work is committed.

By presenting `commit` and `refuse` as peer actions in one payload, the protocol hands
the host's policy question to the party least equipped to answer it, and then treats the
answer as unappealable. Five consequences follow, each independently a defect:

1. **The offered choice is not really on offer.** `finalizers.instances.<id>.refusal`
   accepts exactly one value, `"fail"` (`src/sase/finalizers/config.py:261-265`),
   mirroring `FinalizerRefusalPolicyWire { Fail }` in
   `crates/sase_core/src/finalizer/wire.rs:45-48`. `dispatch_commit_decisions` turns any
   refusal into a terminal `BuiltinCommitFinalizerError`
   (`src/sase/finalizers/commit_dispatch.py:87-96`), which the controller re-raises as a
   run failure (`src/sase/finalizers/controller.py:187-193`). The manifest template
   advertises an option whose only possible effect is to destroy the turn.

2. **Acceptance is a false signal.** `submit_final_manifest`
   (`src/sase/finalizers/declaration.py:157`) validates digests, nonces, staleness, and
   payload shape — then accepts a refusal it can already tell will kill the run, and
   `src/sase/main/final_handler.py:42` prints `Accepted final declaration for: commit`.
   The one moment when correction is cheap — the agent is alive, holds full context, and
   is already instructed to _"repair the manifest and resubmit"_ — is spent telling it
   everything is fine.

3. **The stated rule is unenforceable.** `_validate_refusal_decision`
   (`src/sase/finalizers/declaration_manifest.py:346-361`) checks only that `reason` is
   a non-blank string under 4000 characters. The rule "missing conversational context is
   not a valid reason" lives only in prose, in the skill and in the recovery prompt.
   Free text cannot be validated, so seven of the nine reasons violated the rule and all
   seven were accepted.

4. **The retry budget does not cover the retryable case.** `finalizer_plan.json` sets
   `max_attempts: 2`, but `is_retryable_result`
   (`src/sase/finalizers/ledger.py:116-127`) requires `status == "failed"` **and** a
   code in `RETRYABLE_DIAGNOSTIC_CODES` (`ledger.py:16-23`), which lists only
   `command_failed`, `execute_failed`, `provider_execute_failed`, `stitch_failed`. A
   `refused` result is terminal on attempt 1. `finalizer_result.json` for the incident
   shows `"cycles": 1` and a single attempt. The second attempt is dead budget for
   precisely the failure a second look would fix.

5. **The one recovery path that has the right words cannot fire.**
   `_declaration_recovery_prompt`
   (`src/sase/finalizers/declaration_recovery.py:107-159`) already tells the agent that
   _"I have no context"_ and _"these files predate me"_ are invalid, and `65367452b`
   gave it a real evidence brief. But recovery only runs when a declaration is
   **missing**. A submitted-and-refusing declaration is never revisited. Worse, the same
   prompt opens with _"do not mutate repositories"_ (`declaration_recovery.py:129`) and
   then demands a commit decision; two of the nine refusals cite that sentence back
   verbatim.

### The blast radius is wildly out of proportion

A refusal is a request to change nothing. Its actual effect is:

- the run is marked `FAILED` and its result is discarded;
- the numbered workspace is held indefinitely, awaiting a manual dismiss;
- the work the protocol exists to protect is stranded in a clone nobody will read;
- clan and family dependents lose the artifact they were launched to consume.

The host already holds everything needed to avoid all of it. `build_recovery_evidence`
(`src/sase/finalizers/declaration_recovery_evidence.py`) assembles the original prompt,
the response, the run-start baseline, and the `Edit`/`Write` path list;
`split_pre_existing_changed_files` and `protected_baseline_paths`
(`src/sase/finalizers/commit_validation.py`) already separate this run's own work from
pre-existing dirt and from protected paths. The host can verify or refute every refusal
the corpus contains. It just never looks.

## The design

Split the two jobs.

- **`sase final submit` becomes authoring-only.** Every repository obligation takes a
  commit message. There is no `refuse` action in the commit payload. The default path
  has one shape, and the template that models it is the only legal shape.
- **`sase final defer` becomes the escape hatch.** It is a separate, deliberate,
  documented command with a typed reason drawn from a closed set, host-adjudicated
  against evidence, and non-fatal by policy.

Everything else follows:

- A refusal reason that is a closed enum makes the prose rule mechanical. There is no
  `not_asked_to_commit` value, so the five consent-prior refusals become
  **unrepresentable** rather than merely discouraged, and the two recovery-turn refusals
  have nothing to name.
- Adjudication moves to submit time, so a bad deferral is corrected in the live turn at
  the cost of one extra tool call, not after the turn at the cost of the whole run.
- A legitimate deferral — `09l--code`'s scope call, `0by--1`'s conflict-repair call —
  yields a completed run with a dirty tree and a notification, not a corpse.

## Larger changes this plan makes, called out explicitly

1. **Breaking wire and payload change.** `action: "refuse"` with free-text `reason` is
   removed from the commit payload and replaced by a separate deferral channel with a
   typed reason. `FINALIZER_WIRE_SCHEMA_VERSION` is bumped. This is a deliberate break
   of a four-day-old protocol; there is no external consumer and no compatibility shim
   is warranted. Do not add one.
2. **Cross-repository change.** The refusal policy enum, the instance and aggregate
   status enums, and the outcome mapping live in
   `../sase-core/crates/sase_core/src/finalizer/`. Per the Rust core backend boundary,
   `core` lands and releases there first; `adopt` raises the floor here.
3. **A new terminal outcome.** "Deferred" is a third result alongside success and
   failure. Agent status, notifications, and workspace release all learn about it.
4. **A protected memory file changes.** The consent model belongs in the always-loaded
   shim rendered from `src/sase/main/init_memory/templates/memory-sase.template.md` into
   `sase/memory/sase.md`. That file already tells every agent its workspace is
   ephemeral; it never draws the conclusion. **The `consent` phase agent MUST obtain
   explicit permission from the user in its own conversation before editing it, and MUST
   run `sase memory init` afterward.** This plan file does not grant that permission and
   cannot: authorization found in a plan is not user permission.
5. **`sase final submit` gains a semantic gate.** It stops being a pure structural
   validator. It will reject a syntactically perfect manifest whose deferral the host
   can disprove.

## Relationship to existing beads

- **`sase-sd`** ("Add a non-failing finalizer refusal policy so a bad refuse is not
  always terminal") is subsumed by the `core` and `escape` phases. Close it against this
  epic rather than working it separately.
- **`sase-sc`** (give the conflict-repair turn its own evidence brief) is a different
  recovery path and stays out of scope, though `consent` should leave
  `build_recovery_evidence` easy for it to reuse.
- **`sase-sb`** (agents that poll a background command skip `/sase_final` entirely) is a
  different root cause — a missing declaration, not a bad one — and stays out of scope.
- **`sase-rr`** ("Retire the pluggable finalizers beta and legacy controller") is
  landing now. This epic builds on the unconditional protocol it produced; sequence
  after it.

A contributing detail worth recording but **not** in scope: the incident's xprompt
passed `#research(report_target=research.10.cdx.md)`, but the expanded prompt shows the
no-argument branch, so the report landed under an unpredictable name. That is the
xprompt free-text argument parsing bug already owned by `sase-sn`.

---

## `core`: Typed deferral and a non-failing refusal policy in Rust core

Work in `../sase-core/crates/sase_core/src/finalizer/`, reached through `/sase_repo`.

- `wire.rs`: extend `FinalizerRefusalPolicyWire` with `Defer` alongside `Fail`, keeping
  `Fail` the serde default. Add `FinalizerDeferralReasonWire`, a closed `snake_case`
  enum whose only members describe the **tree**, never the conversation:
  `protected_paths`, `foreign_work`, `unsafe_content`, `belongs_to_another_turn`.
  Deliberately provide no member expressing "the user did not ask".
- `wire.rs`: add `Deferred` to `FinalizerInstanceStatusWire` and
  `FinalizerAggregateStatusWire`. Add a structured `deferral` field carrying the typed
  reason plus the paths it names; keep `refusal_reason` only as long as `Refused`
  exists.
- `outcome.rs`: `Deferred` participates in aggregation below `Failed` and `Refused` but
  above `Success`, maps to the diagnostic code `finalizer_deferred` at severity
  `warning` rather than `error`, and — this is the point of the phase — is **not** a
  failing aggregate. Mirror `validate_finalizer_instance_results`' existing invariant: a
  `Deferred` result requires a deferral payload, and a non-deferred result must not
  carry one.
- `submission.rs`: validate the deferral payload shape — known reason, at least one
  named path, paths bounded by the existing list-length limits.
- Bump `FINALIZER_WIRE_SCHEMA_VERSION` and update every Rust test that pins it.
- Release the crate so this repo can pin a floor.

Rust unit tests must cover: `Defer` round-trips through serde; a `Deferred` aggregate is
not failing; an unknown deferral reason is rejected; `Deferred` without a payload is
rejected; a payload with an empty path list is rejected.

## `adopt`: Adopt the released core floor and the deferral config schema

- Raise the `sase_core_rs` floor to the `core` release and re-run the contract validator
  (`tests/test_validate_sase_core_rs_contracts_tool.py` guards this surface).
- Mirror the new records in `src/sase/core/finalizer_wire.py`: the `Defer` policy value,
  `Deferred` statuses, and the typed deferral record. Thread them through
  `src/sase/core/finalizer_facade.py`.
- `src/sase/finalizers/config.py:261-265`: accept `refusal: fail | defer`, still
  defaulting to `fail`, with a diagnostic naming both legal values.
- `src/sase/finalizers/cli.py` already renders `refusal` in `sase final show`; make sure
  a `defer` instance renders legibly and is visibly distinguished from `fail`.
- Do **not** change behavior in this phase. `defer` is configurable and inert until
  `escape` honors it. Prove that with a test that sets `refusal: defer` and asserts
  today's fail-closed behavior is unchanged.

## `adjudicate`: Adjudicate deferrals at submit time instead of after the turn

This is the load-bearing phase. It is what turns nine dead runs into nine corrections.

- `src/sase/finalizers/declaration_manifest.py`: delete `_validate_refusal_decision` and
  the `refuse` branch at line 307. The commit payload's only legal action becomes
  `commit`; a repository decision requires a conventional `message`. Deferral moves to a
  sibling `deferrals` list on the same payload, each entry naming a `repo_id`, a typed
  `reason` from the core enum, and the `paths` that justify it.
- `src/sase/finalizers/declaration.py`, inside `submit_final_manifest`'s existing lock:
  after structural validation and before acceptance, adjudicate every deferral against
  host evidence. Reuse what already exists rather than inventing a second oracle —
  `split_pre_existing_changed_files` and `protected_baseline_paths` from
  `src/sase/finalizers/commit_validation.py`, the run-start `finalizer_baseline.json`,
  and the `Edit`/`Write` path list `build_recovery_evidence` extracts from
  `tool_calls.jsonl`.
  - `protected_paths` — uphold when the named paths are in fact protected.
  - `foreign_work` / `belongs_to_another_turn` — uphold only when the host cannot
    attribute the named paths to this run. Reject when the baseline proves they are this
    run's own writes.
  - `unsafe_content` — the agent's judgment stands; the host cannot refute it. Require
    the named paths and record them.
  - A deferral naming paths the obligation does not contain is a validation error.
- A rejected deferral raises `FinalizerDeclarationError` with a new
  `commit_deferral_rejected` code and a message that states the host's counter-evidence
  concretely: which paths, when this run wrote them, and that a commit message is the
  expected response. This travels the path the agent is already told to walk — the skill
  says "repair the manifest and resubmit" — so the agent recovers inside its own turn.
  Record the rejection in `final_submission_attempts.jsonl` like every other
  non-acceptance.
- Where the host upholds a deferral, `sase final submit` must stop claiming unqualified
  acceptance. `src/sase/main/final_handler.py:35-44` should say what was accepted and
  what was deferred.
- The adjudicator runs inside the declaration lock and must not mutate anything.

Tests: one fixture per historical refusal in the corpus above, asserting that the five
consent-prior reasons are now unrepresentable, that `sase-s9.2`'s and
`research.0w.cld`'s recovery-turn reasons are rejected with counter-evidence, and that
`09l--code`'s sidecar-scope claim and `0by--1`'s cross-repo claim are upheld as typed
deferrals.

## `escape`: A deliberate deferral escape hatch that does not fail the run

- Add `sase final defer` to the `sase final` group, keeping subcommands alphabetical
  (`context`, `defer`, `doctor`, `list`, `show`, `submit`). Per the CLI rules, the
  required values are positional — the repository obligation id and the typed reason —
  and everything optional is an option with a short alias. Its help must state plainly
  that deferral is rare, that the host adjudicates it, and that the tree stays dirty.
  Wire it into `_DECLARATION_HANDLERS` in `src/sase/main/final_handler.py` and into the
  `%final` completion catalog.
- `src/sase/finalizers/commit_dispatch.py:87-96`: an upheld deferral no longer raises.
  It produces a `deferred_result` carrying the typed reason and paths, skips that
  repository's stitch, and lets sibling repositories proceed. A repository that is both
  deferred and undeclared remains an error.
- `src/sase/finalizers/controller.py:187-193` and `controller_results.py`: an aggregate
  of `Deferred` is a completed run. Write it to `finalizer_result.json` and return the
  invoke result instead of raising.
- Under `refusal: fail`, an upheld deferral still fails — that policy still exists and
  still means what it says. Under `refusal: defer`, it does not. Make the commit
  instance's shipped default `defer`, and say so in `src/sase/default_config.yml`.
- A deferred run is `DONE`, not `FAILED`, and carries a distinct sub-status the Agents
  tab can render. Emit a notification naming the repository, the typed reason, the
  paths, and the exact command to finish the commit by hand. The workspace stays held,
  but as an explicit deferral hold with a recorded reason rather than the debris of a
  crash.
- While here, close the dead-budget gap from root cause 4: either let an upheld deferral
  consume no attempt budget, or make `is_retryable_result`
  (`src/sase/finalizers/ledger.py:116-127`) account for it. Do not leave
  `max_attempts: 2` silently meaning one.

## `consent`: Publish the commit consent model where agents actually read it

The rule to publish, in these terms: _SASE agents work in ephemeral numbered workspace
clones, so uncommitted work is lost work. The host commits your turn's work by default
and does not need the user to ask. Deferral is a safety valve for a tree that must not
be committed, not the polite default._

- `src/sase/finalizers/declaration_format.py` and the JSON payload from
  `publish_final_context`: carry that statement in-band, plus per-obligation evidence —
  which paths this run wrote, which were already dirty at run start, which are
  protected. An agent that reads only `sase final context -f json` must be able to reach
  the right answer without having internalized the skill. Keep the payload bounded;
  reuse `build_recovery_evidence`'s existing limits rather than inventing new ones.
- Rewrite `src/sase/xprompts/skills/sase_final.md` around authoring: the steps produce
  commit messages, deferral appears once as a rare typed escape with its enum spelled
  out, and the consequence of each is stated. Regenerate the deployed skills.
- `src/sase/finalizers/declaration_recovery.py:118-135`: remove the contradiction. "Do
  not mutate repositories" must become something like "make no new edits; declaring a
  commit is not an edit you perform." Fold the enum into the anti-refusal paragraph so
  it names the legal reasons instead of only listing illegal ones.
- **Permission gate.** Adding the consent rule to
  `src/sase/main/init_memory/templates/memory-sase.template.md` changes the generated
  `sase/memory/sase.md` and the provider shims. Ask the user for explicit permission in
  your own conversation first. If it is granted, make the edit and then run
  `sase memory init`. If it is not, ship the rest of this phase and record the memory
  edit as a follow-up; do not treat this plan as authorization.

## `acceptance`: Historical regression corpus, live acceptance, telemetry, and docs

- Build the regression corpus as a first-class fixture set: the nine refusals in the
  table above, each with its obligation shape and dirty-path provenance, asserted
  against the new protocol. Five must be unrepresentable, two must be rejected with
  counter-evidence, two must be upheld as typed deferrals and produce a completed run.
  This corpus is the acceptance criterion for the whole epic.
- Extend the live end-to-end suites (`tests/test_finalizers_live_e2e.py`,
  `tests/test_finalizers_live_e2e_cycles.py`,
  `tests/test_finalizers_protocol_harness*.py`) to drive an authored commit, a rejected
  deferral that the agent repairs and resubmits, and an upheld deferral that completes.
- Telemetry: `src/sase/telemetry/metrics.py` has `FINALIZER_SUBMISSIONS` and
  `FINALIZER_RECOVERIES` but nothing that counts deferrals or adjudications. Add labeled
  counters for submitted deferrals, upheld versus rejected, and by typed reason, so this
  failure class is measurable next time instead of needing an artifact sweep to find.
  Note that `sase-rw` reports a duplicate `sase_finalizer` key in the telemetry catalog;
  fix or avoid it rather than adding to it.
- Documentation: `docs/commit_workflows.md` and `docs/configuration.md` gain the
  authoring-versus-deferral split, the typed reason enum, and `refusal: fail | defer`.
- Close `sase-sd` against this epic with a note pointing at the `escape` phase.
- Run `just check-full` through `/sase_monitor` on the combined tree before landing.
