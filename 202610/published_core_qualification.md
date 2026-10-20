---
tier: epic
title: Qualify release candidates against their exact published core
goal: Complete sase-1n6.5 with candidate-bound qualification evidence, published-core
  CI, and green and deliberately red rehearsal proofs.
parent_bead: sase-1n6.5
phases:
- id: evidence
  title: Define the qualification inventory and fail-closed evaluator
  depends_on: []
  size: medium
  description: 'evidence: add the versioned N1 inventory, typed evidence schema, qualification
    evaluation and bounded-rerun decision, with adversarial fixtures.'
- id: published-core
  title: Supply verified published core artifacts to the CI setup action
  depends_on:
  - evidence
  size: medium
  description: 'published-core: extend build-core and setup-sase to use the manifest''s
    hashed PyPI wheel and tag-built LSP while preserving default source builds.'
- id: ci-evidence
  title: Run reusable CI at an explicit ref and report qualification evidence
  depends_on:
  - published-core
  size: medium
  description: 'ci-evidence: thread qualification inputs and concurrency through ci.yml,
    collect real pytest evidence without changing test commands, and retain job diagnostics.'
- id: release-wiring
  title: Integrate qualification, negative probes and durable observations
  depends_on:
  - ci-evidence
  size: medium
  description: 'release-wiring: wire release.yml to published-core CI and the authoritative
    qualification verdict, preserve red observations, and document the qualification
    contract.'
proposed_by: bbugyi200.athena.sase-1n6.5
proposed_with: codex/gpt-6-astra
create_time: 2026-10-19 20:21:14
status: wip
bead_id: sase-1n6.5.1
---

- **PROMPT:** [prompts/202610/published_core_qualification.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/published_core_qualification.md)
- **PARENT:** [202610/e2_release_train_steward.md](https://github.com/sase-org/sase--plans/blob/main/202610/e2_release_train_steward.md)
- **BEAD:** [sase-1n6.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1n6/sase-1n6.5.1.md)

# Published-core qualification

This child epic implements only the `qualify` phase of
`plan:202610/e2_release_train_steward.md`, attached to `sase-1n6.5`. It is split because
the evidence evaluator, core artifact supplier, CI reporting, and release orchestration
each have separate correctness and verification requirements. Every phase is direct
implementation work for one coding agent. Dependencies serialize workflow and shared
contract-test edits.

The child epic's land agent owns the live proofs and completion of `sase-1n6.5`. Child
phase workers close only their own assigned beads. Neither they nor the land agent may
close `sase-1n6` or another ancestor plan bead.

## Authority and starting evidence

Read the parent phase and design through the audited commands:

```bash
sase bead read sase-1n6.5 -r "Need current phase notes and final epic decisions"
sase artifact read plan:202610/e2_release_train_steward.md "Need the qualify phase and shared release contracts"
sase artifact read research:202610/pypi_release_reliability_design_and_work_split/pypi_release_reliability_design_and_work_split.md "Need the release promise, E2 ownership and acceptance cases"
sase artifact read research:202610/merge_queue_fence_reassessment_and_epic_split/merge_queue_fence_reassessment_and_epic_split.md "Need N1 check classifications"
```

If the research sidecar is absent, prepare it with
`sase repo open research -r "Need the qualification design artifacts"`, then repeat the
artifact reads. Do not read sidecar files directly.

The inherited decisions are final: `steward_host=workflow`, `release_cadence=daily`, and
`memory_release_record=yes`. This child epic edits no memory. The authorized release
decision records belong to the outer epic's exit work, not this phase.

Observed at planning time:

- `sase-1n6.5` is in progress, has no earlier implementation notes, and has no
  `--epic-symbol` entries. Its prerequisites `.2`, `.3`, and `.4` are closed.
- The checkout includes `cc59e6f114` (candidate exercise) and `befa41b575` (exact core
  pin). There are no pending source changes.
- `ci.yml` has inputless `workflow_call`, bare checkouts and `ci-${{ github.ref }}`
  cancellation. Its tests are `just test-cov` on 3.12 and `just test` on 3.13/3.14.
- `build-core.yml` builds the pinned source wheel and LSP. `setup-sase` expects one
  wheel, `sase-core-sha.txt`, and `sase-macro-lsp` in `sase-core-wheel`, and persists
  `SASE_CORE_WHEEL` so later recipes cannot silently replace the wheel.
- `release.yml` already builds and exercises the sealed candidate. It has no qualifier
  and retains the `not-yet-qualified` publication barrier.
- `tools/release_candidate` currently trusts an observation's `verdict=pass` without
  validating qualification details, and always returns `rerun_eligible=false`. This is
  the qualification seam to replace.
- The active epic inventory showed no E3 release-signal epic. Recheck before editing
  `ci.yml`; preserve any intervening E3-owned test-step changes.

The parent epic's explicit qualification list refines the older N1 research: lint and
all three supported Python versions block this candidate's qualification. Do not replace
that list with the older research's narrower wording.

## Shared contract

### Inventory and evidence

Create `.github/release-qualification.json`, with a schema version and one authoritative
classification of checks. Entries have a stable identifier, workflow/job mapping,
admission or release classification, and evidence kind.

- Release qualification blockers: `lint`, `test (3.12)`, `test (3.13)`, `test (3.14)`.
- Release exercise blockers: the four existing Linux/macOS by wheel/sdist legs,
  evaluated by the existing exercise machinery, not mistaken for missing CI jobs.
- Advisory qualification checks: `visual-test`, `perf-floors`, `tutorial-tests`,
  `ace-page-group-isolation`.
- Admission remains the existing Master Gate. Identify it as admission policy; do not
  rerun it or create a second expanded copy of its job list here.
- `release-core-floor-smoke` remains the pre-merge published-core check on the release
  PR. It is not an additional tag-qualification prerequisite.

The inventory is the sole executable list of blocking/advisory job names. The evaluator
loads it; the steward and metrics can import the loader; documentation links to it
rather than maintaining another hand-written list. Unknown schema, duplicate or
conflicting entries, unmapped required evidence, and an empty blocking inventory fail
closed. Hash the exact inventory bytes and retain them with the qualification
observation so later inventory edits cannot reinterpret a historical candidate.

Each job's evidence is bound to repository, candidate ID, generation, checked-out SHA,
expected and installed core version, inventory digest, workflow run ID, job identity,
and execution attempt. The final observation retains the authoritative GitHub job ID,
conclusion, attempt and URL alongside the payload.

Pytest evidence records collected, passed, failed, errors, skipped, whether collection
failed, exit status, first failing node IDs, and verbatim `FLAKY:` lines when present.
Counts must come from pytest's structured reports, not a terminal summary regex or a
constant. Lint has `command` evidence with completed step outcomes and binding
diagnostics; it is not a pytest lane and must not invent a collected count. The
nonzero-collection requirement applies to the three test legs. Exercise keeps its own
evidence kind and existing behavior.

### Evaluation and reruns

`tools/release_candidate evaluate` remains the single public promotion predicate.
Implement its qualification clause in focused `tools/_release_qualification*.py` modules
and call it from the existing CLI. Release promotion is tooling; the outer design
explicitly excludes a move into the Rust runtime.

For every required qualification job, require completed `success`, validated identity
and readable evidence. Test lanes additionally require positive collection, no
collection error, no failed/error outcomes and a successful pytest exit. Reject
contradictory or malformed counts. A job with a timeout, skip, cancellation, missing
upload or failed reporting step cannot become green from a partial pytest report.

Return typed `QUALIFY_RED(...)` failures containing job, failure class (`tests`,
`collection`, `zero`, `harness`, or `missing`), first failing node IDs, and useful
diagnostics. Preserve missing binding names for a lint bindings failure. Unreadable,
stale and identity-mismatched inputs are red with a specific diagnostic, never an
exception that silently discards the requirement.

Expose the existing top-level `rerun_eligible` and `rerun_reason`. Eligibility requires
a red first qualification attempt and every failing blocker classified `tests`.
Collection, harness, missing or zero evidence and any second red are ineligible. This
epic evaluates eligibility only; the steward performs the one `gh run rerun --failed`.

Use execution provenance rather than sorting observations only by generation and
attempt. Never accept another candidate's passing observation or fall back to an older
pass after a newer relevant failure. Reject ambiguous histories. GitHub failed-job
reruns may retain successful jobs from the earlier attempt: accept those only when
authoritative job history proves the same run, candidate, SHA, core, policy and
unchanged successful execution. A rerun job needs its new attempt's evidence. Preserve
both attempts; do not overwrite earlier artifacts. Test this carry-forward case
explicitly, as well as a stale file copied into a new generation.

### Workflow interface

`ci.yml` accepts `ref`, `core-source` (default `source`), and `core-version` (required
at runtime when `published`). Additional internal inputs carry the candidate artifact
locator and the rehearse probe context. Use a validated manifest artifact from the
current release run rather than trusting an unaccompanied version string. The default
`ref` behavior preserves the existing PR merge-ref checkout and Full CI checkout; an
explicit qualification ref is the identity job's immutable SHA.

Qualification uses `qualify-<sha>` with `cancel-in-progress: false`; ordinary CI retains
its current group and cancellation behavior. Keep the outer release group distinct.
Local reusable workflow calls use the caller's workflow revision; thread the requested
source ref into the actual checkouts. See
[GitHub's reusable workflow contract](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows).

Use three clear orchestration roles: a small preparation job exposes validated manifest
fields, `qualify-ci` calls reusable CI, and the public `qualify` summary job determines
promotion readiness. `qualify` runs with `always()` after its dependencies and evaluates
individual jobs and artifacts; it must not equate the reusable workflow's aggregate
conclusion with qualification. Advisory failures stay visible and are included for the
steward, but cannot veto this verdict. Missing or unreadable blocking evidence always
can.

The summary reads completed jobs with paginated Actions API requests and exact
workflow-call scope, not suffix matching against any job in the run. Supply
`actions: read` and preserve needed inherited secrets. Bind attempts using the
[workflow jobs API](https://docs.github.com/en/rest/actions/workflow-jobs), which
supports attempt-specific reads and all-execution history. Keep per-attempt artifact
names unique while payload filenames remain `qualify-evidence-<job>.json`.

## evidence

**Owned files:** the new inventory; `tools/release_candidate` and new qualification
schema/evaluation helpers; focused tests and relevant contract-manifest entries. No
workflows change in this phase.

1. Implement strict inventory and evidence loaders, the inventory digest, job
   normalization, and the pure evaluator described above. Keep GitHub fetching separate
   from evaluation so fixtures drive all refusal cases.
2. Replace the placeholder qualification clause. A bare `pass` observation without
   verifiable qualification details must fail. Preserve the other promotion clauses and
   existing CLI fields. Bound shared observation selection to the manifest identity
   wherever necessary to prevent a qualification bypass.
3. Add a narrow CLI path that produces qualification observation data from manifest,
   inventory, job JSON, and evidence files. Reuse `observe`'s established
   naming/envelope rather than inventing a second manifest or observation format.
4. Test success; every red class; missing matrix entries; wrong SHA/core/policy/
   candidate/generation; malformed and duplicate evidence; old-pass/new-red ordering;
   advisory failure; binding diagnostics; and first/second-attempt rerun eligibility and
   carry-forward.
5. Record the concrete CLI and JSON contract in the phase note for the following workers
   and outer publish/steward phases. Do not add unreferenced public APIs solely for
   later consumers; the CLI must exercise the new entry points.

**Done:** focused evaluator tests and required repository checks pass. Existing
release-candidate tests continue to cover sealing and observation identity.

## published-core

**Owned files:** `.github/workflows/build-core.yml`,
`.github/actions/setup-sase/action.yml`, a focused published-core preparation helper if
needed, and workflow/setup/helper tests. Do not edit `ci.yml` yet.

1. Add explicit-ref plumbing and an optional published-core mode to the reusable build
   workflow. Preserve inputless callers, including `telemetry.yml`, and the
   release-please PR exclusion in source mode.
2. Published mode downloads the sealed manifest and selects exactly one matching
   `sase_core_rs-<version>-...-manylinux_2_28_x86_64.whl`. Fetch it from PyPI and
   validate filename, version, size and SHA-256 against the sealed record before
   advertising it. Missing, yanked, duplicate or mismatched files refuse.
3. Check out sase-core at the selected `v<version>` tag and build only `sase-macro-lsp`;
   never invoke maturin or a source wheel fallback in published mode. Record the
   resolved tag commit in `sase-core-sha.txt`. Reuse existing completeness/containment
   code where needed; do not copy the suffix inventory.
4. Upload the same artifact shape that `setup-sase` consumes. Extend the action with an
   optional expected core version, check the installed distribution immediately after
   installation, and keep its persistent `SASE_CORE_WHEEL` override. Later recipes must
   use those same bytes.
5. Provide an explicit rehearse-only probe path: download an older complete core against
   independently verified probe metadata and record it as a probe. Do not rewrite the
   candidate manifest's core pin. Outside this declared probe path, any difference from
   the manifest version is an error. The probe may reach the bindings check but can
   never qualify the original pair.
6. Test source-default parity, exact-ref resolution, PyPI digest/version refusal,
   absence of a published-mode wheel build, LSP tag provenance, action version checking,
   persistent wheel selection and probe restrictions.

**Done:** workflow and action contracts pass, including existing telemetry and
source-build consumers. Only release tooling changes; no sase-core repo changes or new
Rust binding pin are needed.

## ci-evidence

**Owned files:** `.github/workflows/ci.yml`, a new opt-in pytest reporting helper,
evidence serialization helpers, focused tests and contract-manifest entries.

1. Add the workflow inputs and pass them to `build-core`. Thread explicit `ref` into
   every sase checkout, including conditional jobs. Wire the setup action's version
   assertion for published mode. Reject invalid source/version/manifest combinations
   before any candidate tests.
2. Apply the concurrency contract and preserve ordinary PR, release-please, scheduled
   Full CI and telemetry behavior. All four advisory jobs still run for candidate
   qualification; no admission rerun is added.
3. Add reporting-only configuration around the existing test steps. Keep their commands,
   selection, matrix, worker controls, coverage and exit behavior. A small opt-in pytest
   plugin loaded by the job environment can write the structured session evidence. It
   must count xdist nodes once, distinguish collection errors from test failures, record
   final exit status, and prevent pytest subprocesses inside tests from overwriting the
   outer report. No global reporting activation or changes to `tools/run_pytest`
   selection are needed.
4. Add `if: always()` evidence serialization and uploads for each blocking job. A
   missing/partial pytest session produces explicit invalid evidence, not a fabricated
   green report. Retain command diagnostics for lint, especially the pinned-bindings
   check. Report collection and upload must still work when dependency installation
   failed; use stdlib tooling where possible. Ensure the published-core binding
   diagnostic can run before a full install recipe rejects the deliberately old probe:
   `check_sase_core_rs_bindings` is a static source scan and can run in a minimal
   environment containing the exact wheel. Attribute that preflight to lint and preserve
   its missing-binding names even if subsequent setup cannot proceed. Do not weaken the
   install validators.
5. Retain verbatim `FLAKY:` lines from completed job logs when available; do not
   implement E3's test reruns. Keep advisory conclusions and URLs for aggregation.
6. Test real tiny pytest sessions (pass, assertion failure, collection failure, zero
   collection, setup error and nested subprocess), a small xdist session, and workflow
   contracts proving inputs, default behavior, unchanged test commands and unconditional
   evidence reporting. Use temporary fixture suites, not the repository's full heavy
   suite, for reporter behavior.

**Done:** all required candidate jobs produce identity-bound evidence or an explicit
refusal, while existing default CI contract tests remain meaningful.

## release-wiring

**Owned files:** `.github/workflows/release.yml`, focused Actions fact-reading and
observation helpers, `tests/test_github_actions_release.py` and new focused tests, and
the Qualify section of `docs/contribute/release-train.md`.

1. Add preparation, `qualify-ci`, and final `qualify` roles for release and rehearse
   modes, using the exact build identity and candidate artifact. Resume does not rebuild
   or rerun qualification. Keep the publication barrier until the outer publish phase
   implements publication; do not unlock releases in this phase.
2. Fetch completed scoped jobs and attempt-aware artifacts, then call the shared
   evaluator. Treat API errors, absent jobs, unavailable artifacts or failed reporting
   as typed red outcomes. Avoid requiring the entire workflow to finish before reading
   its completed qualifier jobs.
3. Emit the established qualification observation with inventory bytes/digest, job IDs
   and attempts, counts, diagnostics, advisory failures and `FLAKY:` lines. Persist the
   observation even when evaluation is red: write and upload first, then fail the final
   verdict step. Release mode attaches the observation to the draft with narrowly scoped
   write permission; rehearse uses artifacts. Re-appends must not replace a conflicting
   earlier observation.
4. Expose the final qualification verdict and observation locator for `sase-1n6.6`. Keep
   the public `qualify` job ID stable. Publication must eventually depend on this
   verdict and exercise, not `qualify-ci`'s advisory-sensitive aggregate result. Its
   explicit status condition must allow passing required verdicts despite a failed
   advisory ancestor; default success propagation must not silently skip publication.
   Record this exact integration seam on `sase-1n6.5`.
5. Add `probe_core_version`, refused outside rehearse. Record both intended and tested
   core versions. An older complete core lacking current bindings must reach lint and
   yield `QUALIFY_RED` naming lint and missing bindings, and must remain red even if
   another failure would otherwise mask the mismatch. Normal qualification never
   bypasses the exact installed-version assertion.
6. Add the Qualify documentation: inventory link, source/published distinction,
   evidence/refusal examples, one-rerun policy, probe use and advisory reporting. Keep
   the blocking list in the inventory only.
7. Test the job graph and mode gates, successful blockers with failing advisory jobs,
   skipped/cancelled required jobs, red-observation persistence, immutable attempts,
   probe restrictions, manifest binding, and the unchanged publication barrier. Exercise
   the fact reader with paginated fixture API responses.

**Done:** the workflow and tooling are ready for live qualification rehearsals, and all
implementation checks pass. The land agent performs the deployed proof.

## Verification and land-agent completion

Each coding phase follows the current `tools`, `docs_writing` where relevant, and
`lint_and_test` memory. Run meaningful targeted tests, refresh contract-manifest entries
with `just refresh-contract-manifest` when needed, format, and run
`sase tool run check`. Do not run `just check-full` without explicit authorization. Use
`/sase_monitor` for long checks and waits; do not end with a running command. Agents do
not manually commit, create branches or tags, or publish releases.

The child epic's land agent:

1. Confirms all implementation phases landed and reviews the combined contracts,
   including default source CI, advisory independence, published-core bytes, complete
   matrix evidence, identity binding and rerun provenance. Run the required checks on
   any integration edits.
2. Waits for the host-landed changes to be on `master`, then dispatches
   `release.yml --ref master -f mode=rehearse`. Record dispatched SHA, run URL,
   candidate ID, generation, core version, policy digest and final qualifier
   observation. Green means the actual qualification predicate passed; a green wrapper
   or old unrelated CI run is not proof. No PyPI upload is authorized.
3. Selects an actually retained, complete older core with missing bindings and
   dispatches a second rehearsal using `probe_core_version`. Demonstrate typed
   `QUALIFY_RED` with lint/binding details and ineligible harness rerun status.
   Establish source-built CI at the same source SHA as the comparison, using an existing
   exact-SHA run if available. Do not use a moving-master green as proof.
4. If normal sealing/qualification is blocked by an unpublished required core, retain
   the honest core-containment refusal and hand a bounded retry to `/sase_monitor`
   (about 36 hours, following the outer plan). Monitor the next daily cut and let the
   follow-up rehearse again. A dispatched workflow is not a completed proof. Likewise
   hand long CI waits to the monitor.
5. Record live results and observation locations on `sase-1n6.5`. Any discovered
   unrelated work is a `PROPOSED FOLLOW-UP:` note on that bead, never a new bead created
   by a phase worker. A local check failure reproduced identically on a clean base is
   recorded with evidence and any existing tracker, and does not keep the phase open.
   New regressions in this work must be fixed.
6. Once descendants are complete and the live acceptance evidence is retained, run
   `sase bead epic-symbols sase-1n6.5`, resolve or re-key any remaining entries to a
   still-open outer epic/later phase, then close only the original assigned phase with
   `sase bead close sase-1n6.5 --note "<verified evidence>"` as the authorized
   land-agent completion. Leave `sase-1n6` and all other ancestors open. Root turns
   finish through `/sase_final` unless they have successfully handed off through the
   prescribed monitor or plan mechanism.

This epic does not implement publishing, confirmation, the steward, release metrics,
human repository/PyPI settings, E3's node reruns, or memory supersession.
