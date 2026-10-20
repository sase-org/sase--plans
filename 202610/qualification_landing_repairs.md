---
tier: epic
title: Repair qualification evidence transport and prove the deployed workflow
goal:
  Qualification produces verifiable candidate-bound evidence from real Actions jobs,
  persists every attempt, and has retained green and wrong-core rehearsal proofs.
parent_bead: sase-1n6.5.1
phases:
  - id: repair
    title: Repair qualification producer and consumer integration
    size: medium
    description:
      "repair: fix authoritative job identity, paginated artifact transport, observation
      persistence, and refusal diagnostics with executable integration tests."
    depends_on: []
  - id: deployed-proof
    title: Retain green and wrong-core rehearsal proofs
    size: medium
    description:
      "deployed-proof: exercise the host-landed repair through release rehearsals and
      retain the exact-SHA source-CI comparison and candidate observations."
    depends_on:
      - repair
proposed_by: bbugyi200.athena.sase-1n6.5.1.land
proposed_with: codex/gpt-6-astra
create_time: 2026-10-20 00:28:13
status: wip
---

- **PROMPT:**
  [prompts/202610/qualification_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/qualification_landing_repairs.md)
- **PARENT:**
  [202610/published_core_qualification.md](https://github.com/sase-org/sase--plans/blob/main/202610/published_core_qualification.md)

# Qualification landing repairs

Finish only the remaining work discovered while landing `sase-1n6.5.1`. Its approved
plan is `plan:202610/published_core_qualification.md`; its parent is phase `sase-1n6.5`,
in `plan:202610/e2_release_train_steward.md`. Read both through `sase artifact read`,
and read the landing-review note on `sase-1n6.5.1` before implementation.

This is a child epic because the live proof must exercise the repaired workflow after
the host commits it to master. The code repair is one bounded implementation phase. The
second phase depends on that landed code and owns the external CI waits through
`/sase_monitor`. A tale coder cannot wait for its own end-of-turn commit and then close
the interrupted landing in the same turn. No phase below closes `sase-1n6.5.1`, runs its
post-close Symvision check, or marks its plan done; the `parent_bead` link delegates
that work to this child epic's land agent after the child lands.

## Evidence and constraints

Review baseline: `54a9c14d2393ed168504eb64b2335b2e8fd1825e`, confirmed as remote master
on 2026-10-20. All four original phase beads are closed, but the required live
qualification proofs are absent. All eight child notes were reviewed; none contains
`PROPOSED FOLLOW-UP:`.

The relevant commits are `443fba33d9` (evaluator), `9edbacd4ef` (published core),
`8afb1b7e57` (CI evidence), and `54a9c14d23` (release wiring). Intervening commits
`5003ef69bd`, `23cccbc969`, and `112a5f4757` add the core pin and steward
adapter/fact/health readers. The reader already delegates to
`release_candidate evaluate`; it discovers observations by their candidate-generation,
kind, run, and attempt filenames. Preserve that integration. Intervening deck change
`de1d960d11` has no release integration requirement.

Confirmed defects:

1. `ci.yml` exports no `GITHUB_TOKEN` in either evidence-writing step.
   `tools/write_qualification_evidence._github_jobs` therefore returns no rows. Even
   with rows, `resolve_job_id(job_name="lint")` returns `None` for the actual
   reusable-call name `qualify-ci / lint`. The tests inject numeric IDs instead of
   exercising this producer/consumer path.
2. `release.yml` fetches `/actions/runs/R/attempts/N/artifacts`, which does not exist. A
   read-only request for run `39360520824` returned 404; the run-scoped artifact
   endpoint returned 200. It also uses `gh api --paginate` without a page-array
   envelope, while the inline downloader and collector parse one JSON document. Two
   valid adjacent page objects reproduce `Extra data`.
3. Draft attachment checks for the canonical `candidate-gen...json` asset but uploads
   the local filename `qualification-observation.json`. The steward's asset filter
   ignores it, and another observation collides with that same filename.
4. Failed probe preflight prevents dependency setup, yet installed-version reporting
   looks only in `.venv`. Command evidence validation exits on core-binding mismatch
   before adding the actual missing-binding names to the failure diagnostics.
5. A fact-fetch error produces an observation with a valid outer run/attempt but null
   run/attempt in its qualification result. `_latest_qualification_report` rejects that
   observation as a provenance disagreement, losing the intended typed refusal.
6. The Qualify documentation says a failed blocker may be rerun once. Actual policy
   allows this only for a first attempt whose failing blockers are all test failures.

API contracts: GitHub lists artifacts at the
[workflow-run endpoint](https://docs.github.com/en/rest/actions/artifacts#list-workflow-run-artifacts).
[gh api](https://cli.github.com/manual/gh_api) supports `--paginate --slurp` for a page
array. [gh release upload](https://cli.github.com/manual/gh_release_upload) uses the
uploaded file's name; `#label` changes a display label, not asset identity.

Inherited decisions stay fixed: `steward_host=workflow`, `release_cadence=daily`, and
the authorized memory work remains in the outer epic's exit phase. This repair edits no
memory, changes no admission policy, and implements no publishing or E3 test reruns.
Preserve default source CI and telemetry callers, current test commands and
worker/coverage behavior, published-core wheel verification, the inventory as the sole
blocking list, advisory independence, and the publication barrier unless the outer
publish phase has legitimately replaced it.

Follow-up triage is already complete for the original four children: no proposals. A
separately discovered pre-existing `core_source_pin`/containment gap was processed
through `/sase_new_task` and attached as `DISCOVERED ISSUE` to causal active epic
`sase-1n6`; no duplicate task or new task was warranted. The current build records the
package requirement rather than `sase-core-revision.txt`, and the dependency predicate
does not prove tag containment. Do not silently credit qualification with satisfying
that outer requirement or duplicate that report. Recheck the outer epic's response
before interpreting a live core-lag refusal.

## repair

**Owned files:** `.github/workflows/ci.yml`, the qualification portion of
`.github/workflows/release.yml`, `tools/write_qualification_evidence`,
`tools/collect_qualification_jobs`, `tools/_release_qualification*.py`, the narrow
`tools/release_candidate` qualification path if needed, their tests, and the Qualify
section of `docs/contribute/release-train.md`. Preserve changes from the outer
publishing/steward agents. Keep release tooling here; this is not shared runtime backend
behavior and needs no Rust API.

1. Give evidence production explicit Actions read credentials and exact workflow-call
   identity. Thread an internal caller scope through reusable CI where necessary; keep
   inputless source callers working. Resolve exactly one numeric job ID from the correct
   run and execution attempt, including the full caller prefix and matrix name. Preserve
   the evaluator's normalized local job name. Do not use unscoped suffix matching,
   accept the first ambiguous match, or invent an ID. Missing credentials, partial
   pagination, mismatched provenance and ambiguity produce explicit invalid evidence.
   Verify the checkout SHA from the actual checkout where identity is written.
2. Fetch run-scoped artifacts and attempt-specific jobs with complete pagination. Either
   use `--paginate --slurp` consistently or explicitly decode the streaming pages; the
   fetcher and consumer must agree. Associate artifacts to attempts by their exact
   existing `qualify-evidence-<check>-attempt-<N>` names and bound payloads. Preserve
   failed-only rerun carry-forward of unchanged successful executions. Reject
   duplicate/ambiguous artifact identities and conflicting payload filenames instead of
   silently overwriting or selecting one. Keep API/download failures fail-closed and
   retain their useful diagnostics. Extract inline transport into a small helper only if
   it materially improves executable testing.
3. Write/upload draft observations under the canonical asset filename from the
   observation envelope. Verify that the uploaded name and bytes agree with that
   envelope. Re-appending identical bytes is a no-op; conflicting bytes refuse without
   replacement. Rehearse artifacts must also preserve distinct attempts and the earlier
   red observation so a later passing rerun retains a complete history.
4. Record the actually installed preflight wheel version before the bindings scan can
   fail. Carry it into lint evidence when `.venv` was never created. Retain intended and
   tested versions, missing-binding names, and scan outcomes in the durable observation
   and typed lint failure. Preserve useful validated command diagnostics even when
   identity also mismatches; never relax mismatch refusal to make a probe pass. Normal
   published setup must still assert exact installed version.
5. Preserve authoritative expected run/attempt identity in generated red observations
   when fact collection or parsing fails. `release_candidate evaluate` must be able to
   revalidate these as typed `QUALIFY_RED` with `rerun_eligible=false`; contradictory or
   genuinely forged histories must still refuse. Keep the steward's shared evaluator
   integration intact.
6. Correct only the existing rerun guidance in the Qualify section: one failed-job rerun
   is eligible only after a first attempt with all failing blockers classified `tests`;
   collection, missing, zero, harness and second-attempt failures are ineligible. Read
   `docs_writing` and `docs/contribute/writing-docs.md` first.

**Verification:** add an executable producer-to-consumer integration test using the real
workflow command wiring or its extracted helper and a fake Actions transport. Do not
merely add more substring assertions. A realistic `qualify-ci / lint` plus three scoped
matrix jobs must produce numeric identities and a passing final qualification with one
failed advisory. Exercise two pages, API failure, missing token, wrong caller/attempt,
duplicate jobs/artifacts, a failed-only retry, and a probe with failed setup and named
missing bindings. Verify canonical draft upload argv/bytes and identical/conflicting
re-appends with a fake `gh`, without live release mutation. Verify fetch-error
observations remain typed red through the real CLI and the existing steward evaluator
adapter.

Run the relevant focused suites, including release-candidate, evidence writer,
collector, Actions CI/release/setup/build-core, and existing fact-reader contracts. Read
`tools` and `lint_and_test`; format and run `sase tool run check` (the governed
`just check`). Do not run `just check-full`. Record commands and results plus the stable
contract for the next phase. Close only this phase after it is complete; the host
commits its code at turn end. Do not wait for this phase's own commit.

## deployed-proof

**Dependencies:** `repair` must be complete and its host-created changes available on
remote master. This phase primarily records evidence, not a second implementation. Read
`sase-1n6.5.1` and `sase-1n6.5` again, the repair phase note, the original child plan's
live acceptance section, and any newer outer-epic integration notes. Inspect post-repair
drift before dispatching.

1. Record the exact deployed SHA containing the repair. Dispatch
   `gh workflow run release.yml -R sase-org/sase --ref master -f mode=rehearse`.
   Identify the new run by dispatch timing and `headSha`, and retain its URL. A
   previously green wrapper or a dispatch acknowledgment is not acceptance.
2. Use `/sase_monitor` for CI waits, with a bounded timeout and explicit continuation to
   inspect artifacts and finish this phase. Read the completed qualification observation
   and candidate manifest. Establish actual passing qualification for the recorded
   candidate ID, generation, SHA, intended/installed core version and exact inventory
   digest. Retain all required job IDs/attempts, counts, conclusions, URLs, advisories
   and diagnostic evidence. Re-evaluate the archived inputs through the shared
   predicate. Exercise/publication clauses can be assessed separately; a wrapper's
   aggregate result cannot replace the qualification clause.
3. Select a retained complete older core that actually lacks a binding required by the
   chosen source SHA. Open any other repo with `/sase_repo` and use audited artifact
   reads where applicable. Dispatch the explicit `probe_core_version` rehearsal and
   retain its observation. Prove `QUALIFY_RED` names lint and the missing bindings,
   preserves intended and tested versions, and is not rerun-eligible.
4. Establish source-built CI at the probe's exact source SHA, using an existing
   exact-SHA run when available. If dispatching Full CI at master, verify its `headSha`
   equals the compared rehearsal SHA; do not substitute a later green tip. Require the
   source-built blocking checks to pass for that comparison; source CI that is also red
   does not prove the required contrast. Record the actual results and binding evidence.
   Preserve default source CI behavior.
5. If no complete published core contains the required source pin, retain the honest
   refusal and use the original plan's bounded roughly 36-hour monitored retry of the
   next daily cut. Do not bypass a published-core requirement, reseal a probe as normal,
   move a tag, publish to PyPI, or modify repository/PyPI settings. The pre-existing
   source-pin gap is already on `sase-1n6`; if it prevents trustworthy acceptance,
   record the exact dependency there and on this phase and keep the phase open. A
   timeout or unrelated existing failure is evidence of a blocker, not a successful
   proof.
6. Fix any remaining regression caused by qualification within this scope, with focused
   tests and `sase tool run check`. If another code change needs a deployed proof, use
   the normal host finalization/monitor continuation; never wait for an uncommitted
   turn's SHA. Phase workers record unrelated discoveries as `PROPOSED FOLLOW-UP:` on
   their own bead for the land agent.
7. Register durable copies of the evidence through `sase artifact create` as needed, and
   note their canonical refs, run URLs and acceptance mapping on this phase,
   `sase-1n6.5.1`, and `sase-1n6.5`. Close only this phase after both live proofs and
   the exact-SHA source comparison are complete.

## Landing handoff

The child land agent must review both phases and all notes, triage any new proposals,
and verify drift, checks, deployed proofs and symbol exemptions before closing this
child epic normally. Its `parent_bead: sase-1n6.5.1` link then resumes the interrupted
landing: recheck that plan's descendants, linked plan and landing-review notes before
any parent close. No force-close may advance a successful nested landing.

When `sase-1n6.5.1` is actually complete, the resumed land agent retires its symbol
exemptions, closes it normally, checks Symvision, and marks
`plan:202610/published_core_qualification.md` done under the standard landing
instructions. Its direct parent is phase `sase-1n6.5`: verify the recorded live proofs
satisfy that phase, close only that phase normally, and leave outer epic `sase-1n6` to
its waiting land agent. The outer containment report and unrelated outer phase
follow-ups remain that land agent's responsibility.
