---
tier: tale
title: Complete and land sase-am
goal:
  The omitted flaky-test follow-ups are tracked independently, current master is integrated, and sase-am is closed
  cleanly with its original plan marked done.
bead: sase-am
create_time: 2026-07-28 19:37:30
status: done
---

- **PROMPT:** [prompts/202607/complete_sase_am.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/complete_sase_am.md)
- **PARENT:** [202607/ci_flakiness_redesign.md](https://github.com/sase-org/sase--plans/blob/main/202607/ci_flakiness_redesign.md)
- **BEAD:** [sase-am](https://github.com/sase-org/sase--beads/blob/main/pages/sase-am/README.md)
- **AGENTS:**
  - bbugyi200.athena.sase-am.land--code
- **COMMITS:**
  - [fe4dc62](https://github.com/sase-org/sase--plans/commit/fe4dc62512a971e2dc0f3a7e810bd606b80152c0) — docs: close CI redesign plan and track flaky fetch test

# Complete and land `sase-am`

## Goal

Finish the one requirement omitted from Phase 1 of `sase-am`, recheck integration against current `master`, and land the
original epic without losing the two flaky test follow-ups.

## Established audit context

- Original epic plan: `${SASE_SDD_PLANS_DIR}/202607/ci_flakiness_redesign.md`
- Epic commits:
  - `4d55dabc17152d033c195fcebdf21df4e16b2170` — restore completed-run signal
  - `61c812a7b7f1e04c44e50330f803868154500e3d` — build and fan out one core wheel
  - `29ca9ac511433323f872213603b1ead19db565c3` — consolidate CI lanes
  - `b5efaf7e7929d41e94c53fc01d1e2e143cc011f9` — derive sidecars from config
- All four child beads are closed and their implementation claims were checked against the current workflow, composite
  action, Justfile, sidecar bootstrap script, and regression tests.
- `actionlint` passes for `ci.yml` and `docs-deploy.yml`; the focused CI, Justfile, and sidecar-bootstrap suite passes
  (48 tests).
- A fresh `sase_core_rs` 0.12.5 abi3 wheel installs through `SASE_CORE_WHEEL` into an isolated Python 3.14 venv and
  passes `sase core health --json`. This verifies integration with the core-minimum bump that landed after the wheel
  commit.
- The interleaved schema-3 fix in `41a01b397c79303acad241f2a44822193b3aeb32` is absorbed by the config-derived bootstrap
  script and its normalization test. The later `0c1e02c3bc14b4c7522bece1e62b0845ff0ee05c` base-branch commit only splits
  unrelated commit-workflow tests.
- Phase `sase-am.1` explicitly recorded that it did not file the two out-of-scope flaky-test beads required by its plan.
  Exact-name and symptom searches found no dedicated beads for either test.

## Phase 1 — File the two independent flaky-test follow-ups

1. Re-run exact test-name and symptom searches first so concurrent work cannot create duplicates:
   - `tests/ace/tui/test_artifacts_plans_filtering.py::test_deep_archive_typing_burst_fetches_once_and_becomes_exact`
   - `tests/test_bead/test_cli_work_epic_lifecycle.py::test_work_missing_bead_json_error_is_one_envelope`
2. If either dedicated bead now exists, retain it and record its ID. Otherwise, create one lightweight `tier: tale` plan
   file for that test in `${SASE_SDD_PLANS_DIR}/202607/`, validate it with `sase plan validate`, and create a top-level
   plan-tier bead with `sase bead create --type "plan(<absolute-plan-path>)" --tier plan`.
3. The first bead must describe the full-xdist typing-burst race that sometimes performs one extra deep-archive fetch
   while passing serially and in isolation. The second must describe the pytest-sandbox isolation-guard failure around
   the missing-bead JSON envelope. Preserve the exact node IDs, observed behavior, and the fact that fixes are
   deliberately outside `sase-am`.
4. Keep both beads top-level and independent: do not make them descendants of `sase-am`, do not close them, and do not
   fix the tests as part of this plan. Confirm both with `sase bead show`.

## Phase 2 — Recheck and integrate current `master`

1. Fetch the current base branch and review every commit since `4d55dabc17152d033c195fcebdf21df4e16b2170`, excluding the
   four epic commits. Pay particular attention to any commit newer than `b5efaf7e7929d41e94c53fc01d1e2e143cc011f9`.
2. Integrate any newly landed workflow, dependency, sidecar, setup, or test change that should consume the single-wheel
   setup, config-derived sidecars, consolidated lanes, or completed-run concurrency policy. Remove any newly introduced
   duplication or conflict. Do not change `publish.yml`; its published-wheel smoke is an intentional non-goal of the
   epic.
3. Inspect live master CI runs. Confirm that a master run which reached job execution was allowed to finish even as
   later pushes arrived; replacement of an older queued/pending run is expected. Genuine lint or test failures remain
   trustworthy terminal signal and are not a reason to restore cancellation.
4. Re-run `actionlint` on the changed workflows and the focused CI/Justfile/ sidecar tests. If integration requires
   source changes, run `just install` and the repository-required `just check`, triaging only reproducible, in-scope
   failures.

## Phase 3 — Close, clean expired symbols, and mark the epic done

This is the final phase and must run only after Phases 1–2 are complete.

1. Re-run `sase bead show sase-am` and all four child shows. Confirm the children remain closed and the two independent
   follow-up bead IDs from Phase 1 exist.
2. Close normally, without `--force`, using: `sase bead close sase-am --note "<verification and integration summary>"`.
   The note must name the four audited epic commits, the two filed follow-up bead IDs, the post-start commits reviewed,
   any integration edits (or the reason none were needed), and the verification results.
3. Only after the close succeeds, run `just symvision`. Remove stale `sase-am` whitelist entries and any unused code it
   exposes, then rerun Symvision until clean. If files in this repository change, run `just install` followed by
   `just check` before finishing.
4. Open the plans sidecar through `sase repo open sase--plans`, then change only the original epic plan's YAML
   frontmatter from `status: wip` to `status: done` at `${SASE_SDD_PLANS_DIR}/202607/ci_flakiness_redesign.md`.
5. Finish with `sase bead show sase-am` and a frontmatter read proving the epic is closed with resolution `done` and its
   original plan is marked done.

Do not force-close the epic. If normal close is rejected, finish or deliberately reopen the named unfinished descendants
and repeat the verification; use a canceled/superseded forced resolution only for a genuine, documented change in
intent.
