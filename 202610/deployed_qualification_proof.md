---
tier: epic
title: Unblock and retain the deployed qualification proofs
goal: A live release.yml rehearsal at master passes qualification against the published
  core, a wrong-core probe rehearsal at the same SHA is a typed QUALIFY_RED naming
  lint and the missing binding, exact-SHA source-built Full CI passes the blocking
  checks, and the retained evidence lets sase-1n6.5.1.5.2 close.
parent_bead: sase-1n6.5.1.5.2
decisions:
  rerun_once:
    ask: May the live-proof worker rerun failed jobs once when a required rehearsal
      or Full CI job flakes?
    choices:
      once: One `gh run rerun --failed` per run on a test-only first-attempt red;
        keeps both attempts
      never: No reruns; dispatch fresh runs instead, which can move the compared SHA
    default: once
    why: Keeps the exact SHA and exercises attempt-aware carry-forward live, as the
      steward would
    answer: once
phases:
- id: unblock
  title: Clear the three blockers between master and a live rehearsal
  depends_on: []
  size: medium
  description: 'unblock: grant actions:read to Full CI''s reusable call behind a caller/callee
    permission contract test, parse sase-core''s block-form EXPECTED_DIST_SUFFIXES
    through the shared _release_core parser, refresh the stale contract manifest,
    and replay the rehearse build and published-core steps locally against real inputs.'
- id: live-proof
  title: Dispatch and retain the green, probe and exact-SHA source proofs
  depends_on:
  - unblock
  size: medium
  description: 'live-proof: once unblock is on remote master, dispatch the green rehearsal,
    the 0.41.0 probe rehearsal and Full CI at one SHA, monitor them, re-evaluate archived
    inputs through release_candidate evaluate, register durable evidence and record
    the acceptance mapping.'
proposed_by: bbugyi200.athena.12n
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-20 15:25:36
status: wip
bead_id: sase-1n6.5.1.5.2.1
---

- **PROMPT:** [prompts/202610/deployed_qualification_proof.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/deployed_qualification_proof.md)
- **BEAD:** [sase-1n6.5.1.5.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1n6/sase-1n6.5.1.5.2.1.md)

# Unblock and retain the deployed qualification proofs

This child epic finishes phase `sase-1n6.5.1.5.2` ("deployed-proof" in
`plan:202610/qualification_landing_repairs.md`). That phase fixed the reusable-CI
`runner` context error (`e10fa95c0d`, now on master) and then stopped, because no phase
could land a fix and then rehearse the landed SHA. Read the phase, its parent, and both
plans before starting:

```bash
sase bead read sase-1n6.5.1.5.2 -r "Deployed-proof phase this epic finishes"
sase bead read sase-1n6.5.1.5 -r "Parent child epic and its land handoff"
sase bead read sase-1n6.5.1 -r "Original qualification epic and landing-review note"
sase bead read sase-1n6.5 -r "Outer qualify phase notes and publish seam"
sase bead read sase-1n6 -r "Outer epic DISCOVERED ISSUE notes (containment gap, suffix parser)"
sase artifact read plan:202610/qualification_landing_repairs.md "Deployed-proof acceptance steps"
sase artifact read plan:202610/published_core_qualification.md "Original live acceptance and workflow contract"
```

The acceptance bar is that plan's `deployed-proof` section. A green wrapper or a
dispatch acknowledgment is not acceptance, and neither is an old or later SHA. Inherited
constraints stay fixed: `steward_host=workflow`, `release_cadence=daily`, and memory
work stays reserved for the outer exit phase. Nothing here edits memory, publishes to
PyPI, creates or moves tags, changes repository or PyPI settings, or changes admission
policy.

## Evidence at planning time

Observed on 2026-10-20 at origin/master `682e52fc71`:

- **No rehearsal has ever run.** The latest Release (`release.yml`) runs date from June.
  The phase's 05:12Z dispatch returned HTTP 422 (runner context, since fixed).
- **Blocker A: Full CI cannot start.** Every Full CI run since `e10fa95c0d` ends in
  `startup_failure` with "workflow file issue". The runs are `39417102905`,
  `39433321293`, `39447172777` and `39464230362`. `8afb1b7e57` (ci-evidence) added
  workflow-level `permissions: {contents: read, actions: read}` to `ci.yml`, but
  `full.yml` grants only `contents: read`. A reusable callee cannot hold more token
  scope than its caller, so GitHub refuses at startup. Before `e10fa95c0d`, the parse
  error hid this (run `39372188162`, 0s failure). `actionlint` does not catch it.
  `tests/test_github_actions_ci_master_gate.py::test_full_ci_triggers_and_calls_the_reusable_ci_workflow`
  currently pins `{"contents": "read"}` and so enshrines the bug. `release.yml`'s
  `qualify-ci` already grants both scopes. Do not drop `actions: read` from `ci.yml`: a
  callee's own permissions also downgrade the caller's grant, and qualification evidence
  needs that scope.
- **Blocker B: every rehearsal refuses before sealing.**
  `tools/release_candidate parse_expected_suffixes` only matches a bracketed list.
  sase-core's real `release-plz.yml` uses the block form `EXPECTED_DIST_SUFFIXES: |` at
  every tag from `v0.38.0` through `v0.42.1`. The parser is called in two places, so
  both fail for every mode:
  - `release.yml` `build`, in the "Read the core completeness set" step;
  - `build-core.yml` published mode, which `qualify-ci` uses.

  `tools/_release_core.py` already has a strict block-form `parse_expected_suffixes`.
  This is DISCOVERED ISSUE #2 on `sase-1n6`, introduced by `sase-1n6.2`. No task or
  active phase owns it, and the proof cannot run without it.

- **Blocker C: the contract manifest is stale.** It has been stale since `812cdd9e21`
  and `1592f4dbc7`. The marker selects four files the manifest does not list, and the
  manifest still lists `tests/test_release_candidate.py`, now a non-collected facade.
  The missing files are:
  - `tests/test_github_actions_release_steward.py`
  - `tests/test_release_candidate_inventory.py`
  - `tests/test_release_candidate_rerun.py`
  - `tests/test_release_candidate_seal.py`

  Every Master Gate run since fails `test (7)` on this. Qualification's test legs run
  the same suite, so they would fail too.

- **Already resolved:** the `tests/test_release_watch_health.py` toobig from the phase's
  follow-up note was split by `b7f145ef96`. Master Gate lint is green at `682e52fc71`.
- **Core facts:**
  - `pyproject.toml` pins `sase-core-rs==0.42.0`.
  - `sase-core-revision.txt` is `b664b15bf7`, which no published tag contains. The
    latest tag is `v0.42.1`; sase-core PR #333 (`v0.42.2`) is open.
  - PyPI has all five files for 0.34.48, 0.41.0, 0.42.0 and 0.42.1.
  - A source-level scan found every `require_rust_binding` name in sase-core `v0.42.0`.
    `v0.41.0` lacks only `mobile_pairing_issue`, which
    `src/sase/integrations/mobile_gateway.py` requires.
  - So **0.41.0 is the probe core**. The previous note's 0.34.48 is invalid: its
    `release-plz.yml` has no `EXPECTED_DIST_SUFFIXES`, so the published build-core path
    would refuse before lint.
- **Outer gap, not ours:** the core-source-pin/containment gap (DISCOVERED ISSUE #1 on
  `sase-1n6`) is owned by the outer epic. A green qualification must never be credited
  as closing it, and this epic must not duplicate that report.

## unblock

**Owned files:**

- `.github/workflows/full.yml`
- workflow permission contract tests: `tests/test_github_actions_ci_master_gate.py`, or
  a new focused module beside it
- the suffix-parsing path of `tools/release_candidate`, and tests that build
  `EXPECTED_DIST_SUFFIXES` fixtures
- `tests/contract_manifest.txt`

Keep this as release tooling only; it needs no Rust API.

1. Run `git fetch origin master` and check for drift. If origin/master already fixes any
   blocker, skip that item and record the commit that fixed it.
2. **Full CI permissions.**
   - Grant `actions: read` beside `contents: read` in `full.yml`'s workflow-level
     permissions, and update the pinned full.yml contract test.
   - Add a generic contract test over `.github/workflows/*.yml`. For every job whose
     `uses:` names a local `./.github/workflows/<callee>.yml`, the caller's effective
     permissions must explicitly grant every scope the callee declares. Effective means
     job-level if present, else workflow-level. The callee's scopes include its workflow
     level and any of its jobs, and `write` covers `read`.
   - A caller with no explicit permissions fails the test when its callee declares any,
     because the repository's default token scope is not visible in source.
   - Show that the test fails on the old `full.yml` and passes for `release.yml`
     `qualify-ci`, `ci.yml`→`build-core.yml`, and `telemetry.yml`→`build-core.yml`.
3. **Suffix parser.**
   - Make `release_candidate.parse_expected_suffixes` delegate to
     `_release_core.parse_expected_suffixes`, so there is one parser. Load the sibling
     module by path, the way `_release_qualification` is loaded, and keep it stdlib-only
     so `uv run --no-project --with packaging` still works.
   - Map `CoreEnsureError` to `Refusal`, keep the diagnostic, and keep the list return
     type.
   - Replace the bracketed-form test with a real-shaped block fixture: the five
     `v0.42.0` suffixes inside a `jobs`/`env` context. Add refusal tests for a missing
     block, an empty block and a duplicate suffix. Update any other test that writes
     bracketed fixtures; `grep -rn EXPECTED_DIST_SUFFIXES tests` finds them.
   - Check against the real files. Get each tag's file with `sase repo open sase-core`,
     then `git show v<X>:.github/workflows/release-plz.yml` into a temp file.
     `core-suffixes --file` must emit the five suffixes for `v0.42.0` and `v0.41.0`, and
     must refuse `v0.34.48` with a clear diagnostic.
4. **Contract manifest.** Run `just refresh-contract-manifest`, pull in any new
   contract-marked module, and confirm `tests/test_contract_manifest.py` passes.
5. **Local rehearse replay.** Use temp output paths only, with no GitHub mutation and no
   repo `dist/`.
   - Mirror `release.yml` `build`'s argv: `uv build --out-dir`, then
     `core-files --version 0.42.0 --lock uv.lock`, then `core-suffixes --file`
     (`v0.42.0`), then `seal` with placeholder identity values, then `verify`.
   - Then mirror `build-core.yml`'s published path: `published-core` for 0.42.0, and
     again with `--probe-version 0.41.0` (and 0.41.0 core files and suffixes).
   - Then run the lint preflight against the real wheels:
     `uv run --no-project --isolated --with sase-core-rs==0.42.0 python tools/check_sase_core_rs_bindings`
     must pass, and the 0.41.0 run must name `mobile_pairing_issue`.
   - If the replay hits another refusal on the rehearse path in release or qualification
     tooling, fix it here with a focused test. Record anything outside that scope as
     `PROPOSED FOLLOW-UP:`.
6. **Verification.**
   - Read the `tools` and `lint_and_test` memory first.
   - Run the focused suites: CI/master-gate, release, build-core, setup-sase,
     release-candidate seal/published-core/qualify-red, and contract manifest.
   - Run `actionlint` on the changed workflows, format, and run `sase tool run check`.
     Do not run `just check-full`.
7. **Notes.** On this phase, record:
   - the commands and results;
   - the replay outputs, including the probe binding diff;
   - the confirmed probe recommendation;
   - the stable contract the next phase relies on.

   Note on `sase-1n6` that DISCOVERED ISSUE #2 (suffix parser) is fixed by this phase.
   Close only this phase. The host commits at turn end; do not wait for that commit.

## live-proof

**Dependencies:** `unblock` is closed and its commit is on remote master. This phase
mostly records evidence. Read this phase, the `unblock` note, and the beads listed above
again, then:

1. **Drift and readiness.**
   - Run `git fetch origin master`. Confirm origin/master contains the `unblock` commit.
   - Inspect drift since then in `release.yml`, `ci.yml`, `full.yml`, `build-core.yml`,
     `tools/release_candidate`, `tools/_release_qualification*.py`,
     `tools/write_qualification_evidence` and `tools/collect_qualification_jobs`.
   - Confirm no `release.yml` run is queued or running on `refs/heads/master`. Check
     when the next scheduled Full CI starts (cron `17 */2 * * *`, group `full-ci`).
2. **Probe choice.** Re-run the bindings preflight at master against the 0.41.0 wheel,
   and confirm it still lacks at least one required binding. If master moved the pin,
   pick the nearest older complete core whose tag has block-form suffixes (v0.38.0 or
   later) and which lacks a required binding, then record why.
3. **Aligned dispatch.** Dispatch these three back to back:
   - `gh workflow run release.yml -R sase-org/sase --ref master -f mode=rehearse`
   - the same command plus `-f probe_core_version=<probe>`
   - `gh workflow run full.yml -R sase-org/sase --ref master`

   Identify each run by workflow, event `workflow_dispatch`, creation time at or after
   the dispatch, and `headSha`. Retain the URLs. All three `headSha` values must match;
   that value is the compared SHA. Mind the concurrency groups:
   - **Release group:** queues the probe behind the green run. Never dispatch another
     `release.yml` run while one is pending, because it would cancel the pending run.
   - **Full CI group:** a scheduled run at the compared SHA is an acceptable exact-SHA
     run. If a scheduled run displaces the dispatched one, re-dispatch only while master
     still equals the compared SHA. Otherwise use any Full CI run whose `headSha`
     matches.
   - **SHA mismatch:** if Full CI's SHA differs from the probe's, re-dispatch the probe
     rehearsal and Full CI together. Allow at most three tries, and record every
     attempt.

4. **Monitor.** Hand the wait to `/sase_monitor`. Use a bounded poll of
   `gh run view <id> --json status,conclusion` for all runs, under 4 hours total. Write
   a `--next` that carries the run IDs, compared SHA and steps 5–10. A monitor timeout
   is evidence of a blocker, not a proof.
5. **Green proof.**
   - Download `candidate-build`, which holds `candidate-manifest.json`, and the
     `qualification-observation-run-<run>-attempt-<n>` artifact, which holds
     `candidate-gen<G>-qualification-run<R>-attempt<N>.json`.
   - Establish verdict `pass` for the recorded candidate ID, generation and SHA.
     Intended and installed core must both be 0.42.0. The inventory digest must equal
     the sha256 of `.github/release-qualification.json` at that SHA.
   - Retain job IDs, attempts, URLs and conclusions for `qualify-ci / lint` and
     `test (3.12)`, `test (3.13)`, `test (3.14)`. Test legs need positive collected
     counts and zero failures or errors. Also retain advisory results and any `FLAKY:`
     lines.
   - Re-evaluate the archived inputs locally:
     `python3 tools/release_candidate evaluate --manifest … --observations …`.
   - Assess exercise and publication clauses separately; the wrapper's aggregate result
     never replaces the qualification clause.
6. **Probe proof.** The probe observation must be a typed `QUALIFY_RED` that:
   - names lint, and its missing bindings include `mobile_pairing_issue`;
   - keeps intended core 0.42.0 and tested core `<probe>`;
   - reports `rerun_eligible=false`, with its reason.

   Re-evaluate it through `release_candidate evaluate` as well.

7. **Source comparison.** The exact-SHA Full CI run must show `success` for lint and
   `test (3.12)`, `test (3.13)` and `test (3.14)`; advisory jobs may fail. Record that
   run's source-built core revision and the passing bindings check in its lint. A red
   source run does not prove the contrast.
8. **Failure handling.**
   - **Flake:** follow decision `rerun_once`.

     > [!decision] rerun_once = once A rehearsal is eligible for one rerun only when
     > `release_candidate evaluate` reports `rerun_eligible=true`. A Full CI run is
     > eligible only when every failed blocking job is a test failure. Run
     > `gh run rerun <id> -R sase-org/sase --failed` once, then monitor and evaluate the
     > new attempt with the earlier attempt retained. A second red is a blocker.

     > [!decision] rerun_once = never Do not rerun existing runs. Re-dispatch the
     > affected rehearsal (and Full CI, to keep SHAs aligned) as in step 3, and record
     > the red run as evidence.

   - **Rehearse-path defect** in a workflow or in release or qualification tooling:
     - Fix it with focused tests and `sase tool run check`.
     - On this phase and on `sase-1n6.5.1.5.2`, record a `REPROOF REQUIRED:` note with
       the run evidence, the defect, the fix and the exact re-dispatch steps.
     - Finish with the normal commit declaration and close this phase, so the land agent
       schedules the reproof. Never wait for your own commit.
   - **Core lag:** the green qualification is red only because published 0.42.0 lacks
     source behavior that the exact-SHA source run passes.
     - Retain the honest refusal.
     - If a pending core cut plus pin move would change the sealed core, hand the
       original plan's bounded ~36-hour retry to `/sase_monitor`.
     - Otherwise, record the exact dependency on `sase-1n6` and `sase-1n6.5.1.5.2`.
   - **Existing unrelated master failure** that turns `qualify-ci` red:
     - Record the evidence and any existing tracker.
     - Add `PROPOSED FOLLOW-UP:` on this phase.
     - Treat it as a blocker (`REPROOF REQUIRED:`), not a proof.
   - **Never:** bypass the published-core requirement, reseal a probe as normal, move a
     tag, publish, or change settings.

9. **Durable evidence.**
   - Read the `sase_artifacts` memory, then register durable copies through
     `sase artifact create`: the green observation and manifest, the probe observation,
     both evaluate outputs, and a short acceptance mapping.
   - Note the refs, run URLs, compared SHA and acceptance mapping on this phase,
     `sase-1n6.5.1.5.2`, `sase-1n6.5.1` and `sase-1n6.5`.
   - State explicitly that the outer containment gap remains open.
10. **Close.** Close only this phase: after the green proof, the probe proof and the
    source comparison are retained, or after a landed `REPROOF REQUIRED:` fix as in
    step 8. Do not close `sase-1n6.5.1.5.2` yourself; the land agent does that.

## Landing handoff

The child land agent first:

- reviews both phases and all their notes;
- triages any `PROPOSED FOLLOW-UP:` through `/sase_new_task`;
- verifies drift and checks, and runs `sase bead epic-symbols` for this epic.

If any proof is missing, or a `REPROOF REQUIRED:` or core-lag blocker is open, it
proposes the remaining work and does not close anything.

When the green proof, the probe proof and the exact-SHA source comparison are all
retained, it:

- closes this child epic normally;
- then, as the `parent_bead` resumption, runs
  `sase bead close sase-1n6.5.1.5.2 --note "<artifact refs, run URLs, compared SHA, acceptance mapping>"`.

It leaves these alone:

- `sase-1n6.5.1.5`: its waiting land agent owns it and the resumed `sase-1n6.5.1`
  landing;
- `sase-1n6.5.1`, `sase-1n6.5` and `sase-1n6`;
- the outer containment report.
