---
tier: tale
title: Ship configurable ci_watch release gates
goal:
  ci_watch can safely release repositories using explicit fast and fresh heavy workflow
  evidence with the configured merge strategy.
size: medium
proposed_by: bbugyi200.athena.sase-um.2
bead: sase-um.2
create_time: 2026-08-26 19:17:37
status: wip
---

- **PARENT:** [202608/release_gate_liveness.md](release_gate_liveness.md)
- **BEAD:**
  [sase-um.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-um/sase-um.2.md)

# ci_watch release-gate allowlists, freshness, and merge strategy

## Objective

Complete phase `sase-um.2` in `bbugyi200/bugyi-chops` by making release eligibility
independent of the actstat notification classification when explicitly configured,
requiring a recent green heavy workflow, supporting the repository's allowed merge
strategy, and preparing a tagged plugin release without rolling out host configuration.

## Implementation

1. Extend `src/bugyi_chops/ci_watch.py` configuration with:
   - `merge_method`, accepting only `merge`, `squash`, or `rebase` and defaulting to
     `merge`;
   - `gating_workflows`, a list of workflow names defaulting to empty;
   - `heavy_workflows`, a list of workflow names defaulting to empty; and
   - `heavy_max_age_hours`, a positive numeric freshness window with a six-hour default.
     Preserve the empty-allowlist behavior for existing installations and reject invalid
     shapes or values at invocation parsing.

2. Add a fail-closed release-gate evaluation over the existing bounded
   `GitHubReader.head_ci_evidence(repo, head.sha)` response. Restrict the evaluation to
   configured workflow names and distinguish `gating_workflow_missing`,
   `gating_workflow_in_flight`, and red workflow evidence. Use this evaluation only for
   release planning; keep actstat-based `classify_repo` / `decide_repo`, incident
   notifications, counters, and their query cap unchanged. Ensure both the initial
   decision and the final pre-merge reread evaluate the exact current HEAD.

3. Add a heavy-lane evaluator over the existing default-branch `_workflow_runs` data.
   For every configured heavy workflow, select the newest completed run, require a green
   conclusion, parse its completion timestamp, and require it to fall inside
   `heavy_max_age_hours`. Return stable release reasons for missing/non-green evidence
   (`heavy_lane_not_green`) and expired evidence (`heavy_lane_stale`). Reuse one bounded
   branch-run query per release decision for both generator-busy and heavy-lane checks,
   and repeat the guards during live merge revalidation.

4. Thread the configured merge method into `MergePlan` / `GitHubReader.merge()` and emit
   the matching `gh pr merge --merge|--squash|--rebase` flag while retaining
   `--match-head-commit <evaluated-head-oid>`. Update durable release outcome wording,
   the module docstring, and README descriptions/configuration so no documentation
   claims squash-only behavior.

5. Extend `tests/test_ci_watch.py` at the public/fake GitHub boundaries. Cover empty
   allowlists preserving current behavior; gating workflow green, red, in-flight, and
   missing cases; heavy workflow green, red, stale, and missing cases; bounded and
   malformed evidence; final reread fail-closed behavior; all three merge flags with
   head matching retained; and configuration validation for the new variables.

6. Bump the package version in `pyproject.toml` for the feature release expected by the
   tag-driven publish workflow. Do not change live SASE/chezmoi chop configuration; that
   belongs to the later `sase-um.7` rollout phase. Repository commit/tag/publish actions
   remain subject to SASE's host-owned finalization workflow.

## Verification and completion

1. Run focused `tests/test_ci_watch.py` coverage while iterating, then run the external
   repository's full `just check` equivalent (`just check` when available, otherwise its
   documented format, lint, mypy, pytest/coverage, build, and Twine checks).
2. Inspect the final external-repository diff and verify the primary SASE workspace has
   no unintended implementation edits.
3. Run `sase bead epic-symbols sase-um.2` and resolve or re-key every remaining symbol
   to an open bead before closure.
4. Close only `sase-um.2` with a note naming the tests and package/release metadata
   verified; leave `sase-um` and every other phase open.
