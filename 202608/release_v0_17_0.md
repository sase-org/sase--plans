---
tier: epic
title: Release sase v0.17.0
goal:
  Release PR 284 is green, submitted by ci_watch, published as sase 0.17.0 on PyPI, and
  independently verified with a durable evidence trail
phases:
  - id: stabilize_ci
    title: Drive release PR 284 to green GitHub Actions
    depends_on: []
    size: medium
    description:
      "stabilize_ci: inspect the generated release PR and its current checks, record
      relevant evidence on the GitHub Actions stabilization epic, and use a dedicated
      SASE monitor plus successor agents to diagnose, plan, fix, and recheck failures
      until the complete applicable check set is green."
  - id: await_ci_watch_submission
    title: Wait for ci_watch to submit the green release PR
    depends_on:
      - stabilize_ci
    size: small
    description:
      "await_ci_watch_submission: confirm ci_watch is healthy and responsible for PR
      284, then use a dedicated SASE monitor and successor agent to wait for the chop to
      submit the PR without manually merging it."
  - id: await_pypi_publication
    title: Wait for sase 0.17.0 to publish to PyPI
    depends_on:
      - await_ci_watch_submission
    size: small
    description:
      "await_pypi_publication: identify the Publish workflow triggered by the release
      merge and use a dedicated SASE monitor plus successor agent to wait until both the
      release workflow and the PyPI project report version 0.17.0."
  - id: verify_and_report
    title: Verify the released artifacts and report completion
    depends_on:
      - await_pypi_publication
    size: small
    description:
      "verify_and_report: independently verify the merged PR, GitHub release and tag,
      PyPI metadata, and an installable package, update the relevant epic evidence, and
      prepare the final detailed success report."
proposed_by: bbugyi200.athena.0a1
bead_id: sase-ry
create_time: 2026-09-09 19:51:22
status: wip
---

- **PROMPT:**
  [prompts/202608/release_v0_17_0.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/release_v0_17_0.md)
- **BEAD:**
  [sase-ry](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ry/README.md)

# Release sase v0.17.0

## Scope and invariants

- Coordinate release-please PR `sase-org/sase#284`, titled
  `chore(master): release 0.17.0`, targeting `master` from
  `release-please--branches--master`.
- Preserve release ownership: GitHub Actions must be green before submission, and the
  configured `ci_watch` chop—not an agent or a direct `gh pr merge` call—must submit the
  PR.
- Use `/sase_monitor` for every material external wait. Each wait gets a distinct phase
  and a monitor successor agent; never poll or sleep inline after starting a monitor.
- Every monitor `--next` prompt must tell the successor to inspect the retained output
  and current remote state. If reaching the phase goal requires any file, configuration,
  or repository change, the successor must use `/sase_plan` before editing. It must
  treat the plan from the immediately preceding agent as its predecessor and let SASE
  write the new plan's canonical `PARENT` link; after archival it must confirm that link
  with the plan-link tooling. SASE owns the plan header block, so agents must not
  hand-author `PARENT` bullets. If there is no predecessor plan artifact, record that
  fact rather than inventing one.
- For any code-changing plan, diagnose first, keep the change narrowly tied to the
  failing release gate, use the normal SASE patch/commit workflow, run `just install`
  before verification, and run at least `just check`; use monitored `just check-full`
  whenever the broadening rules or risk require it.
- Do not alter generated release metadata merely to hide a source or dependency
  incompatibility. Determine whether a failure belongs on `master`, on the generated
  release branch, in the sibling Rust core, or in GitHub configuration, then use the
  appropriate supported workflow.
- Review all in-progress epic beads at each diagnostic handoff. The currently relevant
  epic is `sase-m4` (“Stabilize GitHub Actions”), whose existing notes already cover PR
  284 and `release-core-floor-smoke`; append concise, attributed progress or resolution
  notes there (or to a reopened/new child phase if SASE assigns one). Do not add notes
  to unrelated epics. If newly discovered work needs a task bead, follow
  `/sase_new_task`; epic phase workers record `PROPOSED FOLLOW-UP:` on their phase
  instead of creating a task.

## Phase 1: Drive the release PR to green

1. Re-read the current PR and check state with `gh pr view 284` and `gh pr checks 284`;
   inspect failed job logs with `gh run view ... --log-failed`. Confirm the applicable
   release-please check topology before treating skipped source lanes as errors. At plan
   time, `release-core-floor-smoke` was the sole in-progress release-branch CI job and
   the source lanes were intentionally skipped by `.github/workflows/ci.yml`.
2. Inspect the current Patch state and `sase-m4` history. Add a note to `sase-m4`
   identifying this release attempt, PR 284, the observed run/check, and whether the
   historical core-floor blocker is resolved or still present. Add further notes only
   when the evidence materially changes.
3. Start one `/sase_monitor` wait for the applicable PR checks, using clear
   `CHECKING CI` / `CHECKED CI` statuses, a bounded timeout, and a `--next` prompt
   containing the predecessor-plan rule above. Prefer a GitHub CLI watch command that
   exits nonzero on failure so the successor receives a decisive result.
4. On green, re-query the PR to ensure every applicable required check is successful or
   intentionally skipped and that there is no pending/cancelled required check. On red
   or timeout, the monitor successor diagnoses the exact failure. If a fix is required,
   it first creates and proposes a linked `/sase_plan`, implements only after approval,
   verifies locally, publishes through the SASE workflow, and starts a fresh monitor for
   the replacement CI run. Repeat until green.
5. Record the green run ID, head SHA, required check names, conclusions, and completion
   time for the final report and in a concise `sase-m4` note.

## Phase 2: Let ci_watch submit PR 284

1. Confirm the configured `ci_watch` chop is available and healthy and that the green PR
   remains open and eligible. Inspect the SASE Patch adopted for PR 284 if the external
   mirror has materialized it, including blockers, hooks, comments, mentors, and
   processes.
2. Do not invoke a merge command. Start a dedicated `/sase_monitor` whose batch command
   waits until GitHub reports PR 284 merged/submitted, with `WAITING FOR SUBMIT` /
   `SUBMITTED` statuses and a bounded timeout. Its `--next` prompt must preserve the
   predecessor-plan/link rule and instruct the successor to distinguish an ordinary chop
   interval from a real automation fault.
3. After the monitor finishes, verify `state=MERGED`, capture `mergedAt`, merge commit
   SHA, and the actor when available, and confirm the release tag/workflow has begun. If
   the timeout reveals a real `ci_watch` defect or configuration blocker, use
   `/sase_plan` before any fix, link it to the immediately preceding plan, note
   `sase-m4` if causally relevant, repair through the supported workflow, and wait again
   via a new monitor. Never bypass `ci_watch` to force the merge.

## Phase 3: Wait for PyPI publication

1. Locate the `Publish` workflow run caused by the merge commit and inspect its job
   graph. The expected path is release-please release creation, build, package/install
   smoke checks, and trusted publication to PyPI.
2. Start a dedicated `/sase_monitor` that waits for the relevant Publish run to complete
   successfully and for `https://pypi.org/pypi/sase/0.17.0/json` (or equivalent
   authoritative PyPI API metadata) to become available. Use `WAITING FOR PYPI` /
   `PUBLISHED TO PYPI` statuses, a bounded timeout, and the predecessor-plan/link
   instruction in `--next`.
3. If publishing fails or times out, inspect the exact GitHub job logs and PyPI state.
   Any required fix must begin with a linked `/sase_plan`, be implemented and verified
   through the SASE workflow, and preserve release idempotency. Use another monitor for
   every subsequent workflow or index-propagation wait.
4. Capture the Publish run URL/ID, release/tag, PyPI version, upload time, filenames,
   and hashes needed for independent verification and the final report.

## Phase 4: Verify and report

1. Reconfirm PR 284 is merged, the GitHub release/tag for `v0.17.0` exists, the
   associated Publish workflow is successful, and PyPI's authoritative metadata reports
   `0.17.0`.
2. Perform an isolated install smoke from PyPI with dependency resolution that cannot
   silently reuse the workspace checkout, then verify `sase version` and a lightweight
   public health/help command. Do not mutate the user's global installation.
3. Add a final concise resolution note to `sase-m4` tying the historical
   release-floor/CI issue to the green PR run, chop-driven submission, successful
   Publish run, and PyPI evidence. Do not close that epic; its land agent owns closure.
4. Ensure the workspace has no accidental edits and all planned child work is complete.
   The epic land agent should report the fixes made (or that none were necessary),
   commits/stitches and checks, green CI evidence, `ci_watch` submission evidence,
   publication evidence, package smoke results, and any explicitly deferred follow-up.
5. Before the final user response, use `/sase_final` as the last normal action and obey
   the returned repository obligations. After a successful declaration, make no further
   file or repository changes.

## Acceptance criteria

- Every applicable required check on PR 284 is conclusively green, with the green run
  and head SHA recorded.
- PR 284 is merged by the normal `ci_watch` automation path, not by an agent bypass.
- The GitHub `v0.17.0` release/tag exists and its Publish workflow is successful.
- PyPI serves authoritative metadata and installable artifacts for `sase==0.17.0`, and
  an isolated smoke verification passes.
- Every significant wait used `/sase_monitor` and a distinct successor agent; every
  issue requiring changes was planned before edits and its plan is canonically linked to
  the preceding plan artifact.
- Relevant progress and resolution evidence is recorded on `sase-m4`, with unrelated
  in-progress epics left untouched.
- The final response gives the user a detailed, evidence-backed release report.
