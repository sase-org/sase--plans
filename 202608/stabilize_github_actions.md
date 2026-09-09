---
tier: epic
title: Stabilize GitHub Actions
goal:
  Restore reliable passing CI, documentation, and publication workflows on the sase
  default branch
phases:
  - id: release-floor
    title: Repair core release floor ratcheting
    depends_on: []
    size: medium
    description:
      "release-floor: validate dependency changes semantically and advance the supported
      core binding floor."
  - id: docs-pdf
    title: Repair strict PDF documentation export
    depends_on: []
    size: medium
    description:
      "docs-pdf: identify the browser 404 requests and correct their documentation
      source without weakening strict export."
  - id: unit-liveness
    title: Fix deterministic test failures and the stalled test shard
    depends_on: []
    size: medium
    description:
      "unit-liveness: repair synchronization and lifecycle regressions and isolate the
      Python 3.13 stall."
  - id: visual-baselines
    title: Reconcile ACE visual behavior and snapshots
    depends_on: []
    size: medium
    description:
      "visual-baselines: classify visual diffs, fix nondeterministic state, and accept
      only intentional golden changes."
  - id: performance-floor
    title: Resolve the artifact-scan performance failure
    depends_on: []
    size: small
    description:
      "performance-floor: use repeatable measurements to optimize a regression or
      evidence a narrowly calibrated threshold."
  - id: integration-and-ci
    title: Integrate, exhaustively verify, and observe GitHub Actions
    depends_on:
      - release-floor
      - docs-pdf
      - unit-liveness
      - visual-baselines
      - performance-floor
    size: medium
    description:
      "integration-and-ci: run monitored exhaustive checks and recursively observe and
      repair the exact post-landing Actions run."
proposed_by: bbugyi200.athena.01o
bead_id: sase-m4
create_time: 2026-09-09 19:51:38
status: wip
---

- **PROMPT:**
  [prompts/202608/stabilize_github_actions.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/stabilize_github_actions.md)
- **BEAD:**
  [sase-m4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m4/README.md)

# Plan

Restore GitHub Actions from the observed failures on the default branch, preserving the
meaning of the checks instead of weakening them. The initial `actstat` investigation
found that the last completed `sase` run was not green and that a newer CI run was
blocked behind a long-running Python 3.13 test shard. GitHub run and job logs narrowed
the work into the independent phases below.

## Evidence and boundaries

- Publish fails in `tools/ratchet_core_window`: changing the supported `sase-core-rs`
  floor from 0.26.10 to the already-published 0.27.2 produces legitimate lockfile solver
  churn, but the tool insists on exactly seven textual replacements. The stale 0.26.10
  floor then makes `release-core-floor-smoke` fail because that wheel lacks
  `append_proc`, `prune_procs`, `read_procs_snapshot`, and `update_proc`.
- Docs PDF was last green at `2aff0a03` and first failed at `465d81ec`. The exporter
  version did not change. Strict export now reports browser resource 404s on many pages;
  the exact requested URLs must be captured locally before correcting links, anchors, or
  export configuration.
- CI run `31821769275` reports four repeatable test failures: clipboard tests wait for
  the worker-thread copy event but race the subsequent notification; a commit-finalizer
  test deletes the agent's file even though the newer safety contract correctly rejects
  vanished uncommitted work; monitor help expects argparse punctuation that argparse
  does not render; and one Python 3.12 run refreshes `TabQuickStart` after the parent is
  mounted but before its composed children are queryable. Python 3.13 remains stalled in
  the test step and must be isolated with timeouts and stack dumps rather than guessed
  at.
- The same CI run has widespread ACE PNG differences, including a common global
  notification-indicator region as well as some larger UI changes. Each class of diff
  must be inspected; snapshots may be regenerated only for behavior confirmed to be
  intentional.
- The performance job has one failure: the six-project, 200-process-page
  `scan_agent_artifacts` anchor measured about 239 ms against a roughly 216 ms ceiling.
  One noisy sample is insufficient evidence either to raise the floor or to rewrite the
  scanner.
- Keep shared backend/domain behavior in `sase-core`; these failures currently point to
  Python tooling, tests, docs, and presentation lifecycle code. Do not edit SASE memory
  files. Run `just install` before repository verification in every fresh workspace.

## Phase 1: Repair core release floor ratcheting

Reproduce the ratchet failure and inspect the semantic package changes made by updating
only `sase-core-rs` to 0.27.2. Replace the fixed seven-line lockfile assumption with a
semantic validation that accepts all solver-owned changes required by the requested core
upgrade while still rejecting unrelated direct-dependency or package movement. Add
focused tests covering the expanded safe lockfile diff and a genuinely unrelated diff.
Then ratchet `pyproject.toml` and `uv.lock` to the published 0.27.2 floor and prove that
the oldest supported core exports the complete required binding surface. Run the ratchet
tests, lock consistency checks, and the release-core-floor smoke test used by CI.

Acceptance criteria:

- `tools/ratchet_core_window` can perform and validate the 0.27.2 floor update without
  allowing unrelated dependency changes.
- Project metadata and the lockfile agree on the new compatible core window.
- The release floor smoke test finds every binding expected by current Python callers.

## Phase 2: Repair strict PDF documentation export

Reproduce `just docs-pdf-check` with the same strict exporter path used by Actions.
Capture the actual browser-request URLs behind the 404 messages and compare the first
bad revision with the last good revision. Correct the causal broken links, anchors,
generated paths, or exporter configuration only after the requests are identified; do
not hide failures by disabling strict mode. Add the narrowest practical regression check
for the failure mode, and run both normal documentation validation and the strict PDF
export.

Acceptance criteria:

- The exact 404 source is documented in the phase result and fixed at its origin.
- Internal links and generated resources remain valid in both site and PDF forms.
- `just docs-pdf-check` completes without browser errors under strict mode.

## Phase 3: Fix deterministic test failures and the stalled test shard

Correct the independently identified test and lifecycle problems:

1. Make clipboard tests await completion of the scheduled delivery coroutine (including
   notification), not merely the copy callback running in its worker thread.
2. Make the baseline-exclusion finalizer fixture commit its agent-owned file so it
   exercises exclusion of the pre-existing sibling without violating the vanished-work
   safety invariant. Retain or add a separate assertion for rejecting unexplained
   deletion if coverage needs it.
3. Replace the monitor-help punctuation assertion with the repository's existing
   metavar-aware option check, or an equivalent semantic parser assertion.
4. Make `TabQuickStart` refresh lifecycle-safe when the widget is mounted but its
   composed children are not yet available, and add a regression test for the startup
   ordering that produced `NoMatches("#agent-quickstart-callout")`.

Separately reproduce the Python 3.13 stall with CI-equivalent ordering and dependencies.
Use bounded pytest timeouts, faulthandler/stack dumps, and test bisection to identify
the specific test or leaked task/process. Fix that cause and add a regression that
terminates reliably. Do not solve liveness by globally skipping tests or merely
extending the workflow timeout. Run the focused tests under every supported Python
version involved in the failure.

Acceptance criteria:

- The four known failures pass without loosening production safety contracts.
- Repeated CI-equivalent Python 3.13 runs terminate and leave no leaked child work.
- Focused tests cover both the corrected synchronization and lifecycle boundaries.

## Phase 4: Reconcile ACE visual behavior and snapshots

Run the dedicated visual suite and inspect the generated actual, expected, diff, and
source artifacts by failure cluster. First determine whether the common notification
indicator delta is unintended shared state, nondeterministic fixture state, or an
intentional UI change. Fix rendering or fixture isolation for
unintended/nondeterministic differences. For confirmed intentional UI changes,
regenerate only the affected PNG goldens with `--sase-update-visual-snapshots`, inspect
the updated images, and rerun the suite with exact local pixel comparison. Do not
bulk-accept all snapshots without classifying the larger outliers separately.

Acceptance criteria:

- Visual fixtures deterministically control global notification state.
- Every accepted golden change corresponds to an intentional visible behavior change.
- `just test-visual` passes twice from a clean deterministic test state.

## Phase 5: Resolve the artifact-scan performance failure

Reproduce the failing synthetic anchor with multiple samples on an otherwise idle runner
and compare it with recent known-good measurements. Profile the scanner if the
regression is repeatable and optimize the responsible path without changing observable
results. If measurements instead demonstrate environment variance, recalibrate only the
specific floor using a robust sample set and record the evidence in the benchmark data
or test rationale. Do not raise a threshold from the single 239 ms observation.

Acceptance criteria:

- The decision to optimize or recalibrate is supported by repeatable measurements.
- The six-project, 200-page anchor passes while the floor remains sensitive to a real
  regression.
- The complete performance-floor job passes repeatedly.

## Phase 6: Integrate, exhaustively verify, and observe GitHub Actions

Integrate the phase results and resolve overlaps without dropping regression coverage.
Run `just install`, all focused commands from the phases, `just test-visual`, and
`just check`. Because this epic touches broad test selection and release behavior, run
`just check-full` only through `/sase_monitor`, with a `--next` instruction that makes
the follow-up agent inspect the result, fix any attributable failure, and repeat the
monitored full check until it passes. Resolve any unrelated pre-existing failure via the
project's task-bead workflow rather than masking it.

After the verified changes land and trigger Actions, obtain the exact CI run ID for the
landed commit (do not monitor an older queued run). Use `/sase_monitor` to wait for that
run to reach a terminal state, for example by monitoring
`gh run watch <exact-run-id> --repo sase-org/sase --exit-status` with an appropriate
timeout and a `--next` instruction. The next agent must run `actstat` and confirm that
the `sase` project reports its latest GitHub Actions run as passed; it must also verify
that all workflows triggered for the landed commit, including Docs and Publish where
applicable, are terminal and successful rather than merely queued behind an older run.

If `actstat` does not show a stable passing last run, or any workflow is failed,
cancelled, stalled, or still pending, the follow-up agent must inspect the new job logs
and create a new plan with `/sase_plan` before making further file changes. That plan
must choose the appropriate tier, run `sase plan validate --explain`, revalidate after
every edit until clean, and submit with `sase plan propose`. Its final verification must
again use `/sase_monitor` to wait for the exact new GitHub Actions run and give the next
agent the same `actstat` verification-and-replan instruction. Continue this monitored
plan/fix/verify loop until `actstat` shows the latest `sase` run passing and every
triggered workflow is stable and green.

Final acceptance criteria:

- All focused checks, `just test-visual`, `just check`, and the monitored
  `just check-full` pass.
- A new Actions run for the landed commit completes, and its follow-up agent confirms
  success with `actstat`.
- CI, Docs, and Publish have no failure or stalled job associated with the landed
  commit, and the recursive monitored verification loop remains explicit in every
  follow-up repair plan until that state is reached.
