---
tier: epic
title:
  Diff-scoped selection soundness — measure the blind spot now, stop degrading silently, and stop depending on a CI
  artifact
goal:
  "The reliability of `just check`'s diff-scoped test lane becomes a measured property rather than an assumed one: a
  backtest over real historical commits reports selection recall against per-test coverage ground truth today instead of
  waiting weeks for organic samples, a missing or stale coverage-contexts baseline provokes a named and measured
  compensating action instead of silently narrowing the selection, an agent can see what the scoped lane actually did on
  the success path, and a workspace can obtain a baseline from a local full run instead of depending solely on a 14-day
  CI artifact."
phases:
  - id: backtest
    title: Historical backtest of selection recall against coverage ground truth
    depends_on: []
    size: medium
    description:
      "backtest: add a `tools/selection_backtest` harness (plus a `just selection-backtest` recipe) that replays the
      last N real commits, computes the selection each diff would have produced, and reports recall against per-test
      coverage-context ground truth — so the epic's unmet exit criterion is answerable now rather than after weeks of
      organic sample growth."
  - id: visible
    title: Make the scoped lane's selection and degradation visible on the success path
    depends_on: []
    size: small
    description:
      "visible: `tools/run_silent` discards captured output on success, so a passing `just check` shows only `✓ test
      (scoped)` and hides how many tests ran, whether the run escalated, and whether the contexts baseline was missing;
      surface a one-line scoped summary that survives the success path."
  - id: compensate
    title: A named, measured compensating action for a missing or stale contexts baseline
    depends_on:
      - backtest
    size: medium
    description:
      "compensate: `context-baseline-missing` and `context-baseline-stale` are recorded but are absent from
      `FULL_SUITE_RULES` and widen nothing, so roughly half of scoped runs narrow silently; choose and implement a
      compensating action on the backtest's measured recall evidence, and record it as a named rule."
  - id: baseline
    title: Record a contexts baseline from a local full run
    depends_on: []
    size: medium
    description:
      "baseline: only `tools/fetch_coverage_contexts` installs a baseline into the host-local cache, so an instrumented
      local run's database is invisible to selection; let a local `cov-contexts` run install itself as a `<sha>.sqlite`
      baseline so a workspace is not dependent on the CI artifact alone."
  - id: land
    title: Land the selection-soundness epic
    depends_on:
      - backtest
      - visible
      - compensate
      - baseline
    size: small
    description:
      "land: re-read `just selection-health` and the backtest report on the combined tree, state the honest recall
      reading, file collected follow-ups with /sase_new_task, run `just check-full` and `just symvision`, and close the
      epic."
proposed_by: bbugyi200.athena.tx
create_time: 2026-08-06 08:52:25
status: wip
---

- **PARENT:** [202608/test_suite_tier1.md](202608/test_suite_tier1.md)

# Plan: make diff-scoped selection reliability a measured property

## Why this plan exists

Epic `sase-fp` delivered two-speed verification and it works. That was re-confirmed empirically before writing this
plan, not taken from the bead notes. What `sase-fp` did **not** deliver is evidence that the fast lane is _sound_, and
there is now a measured hole in it.

`sase-fp`'s own landing note is honest about the evidence problem: the epic's exit criterion — zero false negatives
across at least 30 varied changes — is explicitly **not met**, and the `0 false negatives` reading rests on a
correlatable sample of roughly three runs. This plan's contribution is that waiting is not the only option: recall
against per-test coverage ground truth is measurable _today_ over real history, and one of the two safety sources is
absent about half the time in a way nothing currently reports.

## What was measured before writing this plan

All figures measured on the 64-core development host at `6b0976bcb`, in a freshly `just install`ed workspace.

**The fast lane works.** `just check`'s final stage is `just test-scoped` → `tools/run_pytest scoped` →
`tests._test_selection.select_tests`:

| change                                             | selected | share of 2324 test files |
| -------------------------------------------------- | -------- | ------------------------ |
| `README.md` only                                   | 34       | 1.5%                     |
| `src/sase/ace/tui/widgets/_agent_detail_panels.py` | 104      | 4.5%                     |
| `src/sase/agent_lanes.py`                          | 110      | 4.7%                     |

`just test-scoped` over the `agent_lanes.py` edit: **1106 passed, 4 skipped, 72.16s**, serial, no suite-gate lease.
`just selection-health` over the durable store: 22 scoped runs, median 34 files (1.5%), p90 38, median duration 24.6s,
46 full-lane runs, **46,964 worker-seconds avoided**.

**The static closure has a real, silent blind spot.** Comparing the full selection against the same selection with the
coverage-contexts store forced absent isolates what only ground truth found:

| changed file                                       | closure alone | with contexts | tests only contexts found       |
| -------------------------------------------------- | ------------- | ------------- | ------------------------------- |
| `src/sase/ace/tui/_app_layout.py`                  | 123           | 192           | **69 (36% of the correct set)** |
| `src/sase/ace/tui/widgets/_agent_detail_panels.py` | 98            | 104           | **6**                           |
| `src/sase/agent_lanes.py`                          | 110           | 110           | 0                               |
| `src/sase/bead/_project_queries.py`                | 155           | 155           | 0                               |
| `src/sase/agent/_family_attach_candidates.py`      | 59            | 59            | 0                               |

Those 69 test files are not a guess — per-test coverage records them as having executed the changed lines. The depth-2
reverse import walk does not reach them.

**And the mitigation is absent about half the time, with no consequence.** `just selection-health` reports the contexts
baseline present in **11 of 22** recorded runs. `RULE_CONTEXT_BASELINE_MISSING` and `RULE_CONTEXT_BASELINE_STALE`
(`tests/_test_selection_contexts.py:53-54`) are recorded on the manifest but appear nowhere in
`tests/_test_selection_rules.py` and are not members of `FULL_SUITE_RULES` (`tests/_test_selection_rules.py:35-45`), so
a missing baseline neither escalates nor widens anything. For a change like `_app_layout.py` in a workspace with no
baseline, 69 covering test files are skipped with no rule firing and no warning.

**The signal is also invisible.** `tools/run_pytest` prints `summary_line` and `context_line` to stderr specifically so
a failure dump carries them, but `tools/run_silent` captures combined stdout+stderr and **discards it on success**
(`tools/run_silent:6-8`). A passing `just check` therefore shows the agent only `✓ test (scoped)` — not the selected
count, not whether the run escalated, not whether the baseline was missing.

**A local instrumented run cannot supply a baseline.** Only `tools/fetch_coverage_contexts` writes `<sha>.sqlite` into
`contexts_directory(...)` (`tools/fetch_coverage_contexts:150,161`). `just test-contexts` (`cov-contexts` mode) records
per-test contexts but nothing installs the result where `resolve_baseline` looks, so the only supply route is a GitHub
Actions artifact published on master pushes with 14-day retention.

## What is already true — do not redo it

- **The escalation and broadening machinery is correct and functioning.** Seven rules force the full suite, a selection
  above `max_ratio` (0.25) escalates to the governed full lane, a rename or delete buys an extra closure hop, and an
  unresolvable base ref escalates. Verified by reading and by the rule histogram in `just selection-health`.
- **`check` and `check-full` share an identical gate list** and differ only in the final stage, pinned by
  `tests/test_justfile_lint.py`.
- **The scoped lane takes no suite-gate lease** and is serial by construction; `_reject_scoped_worker_overrides` rejects
  attempts to parallelize it.
- **An empty selection cannot silently pass.** The committed contract manifest is non-empty (34 files) and
  `tests/contract_manifest.txt` is itself in `SELECTION_TOOLING_PATHS`, so deleting it escalates rather than empties.
- **`core-identity-changed` escalations are legitimate, not mtime churn.** The environment fingerprint stats
  `st_mtime_ns:st_size` for extension files and `.venv/bin/python`, which looks like a spurious-escalation source; it is
  not. The fingerprint was captured, `just install` was re-run, and the fingerprint was byte-identical
  (`846557a2ceff…`). Do not re-chase this.
- **Coverage contexts require a real working-tree diff against the baseline commit.** `baseline_changed_lines` takes a
  `git diff -U0 <baseline_sha>` and reads _old-side_ line numbers. A synthetic `ChangeSet` with no corresponding file
  modification contributes nothing — this is correct behavior and cost one wrong measurement while preparing this plan.
  Any test or harness that exercises contexts must modify real files.
- **The selection cache lives at `.pytest_cache/sase-selection/`** (`SELECTION_DIRECTORY`), not at a repo-root dotdir.

## Phase `backtest` — historical backtest of selection recall

**Problem.** The false-negative metric can only learn from a later full run **in the same workspace** over a superset
change set. In ephemeral workspaces that effectively only happens at landing, so the correlatable sample grows at
roughly the rate epics land. The exit criterion is unreachable on that schedule.

**Deliverable.** `tools/selection_backtest`, plus a `just selection-backtest` recipe:

- For each of the last N commits (default N configurable, `--limit`), take the commit's own diff against its parent as
  the change set.
- Build the import graph _as of that commit_ and compute the selection the scoped lane would have produced. Use a single
  reusable detached `git worktree` and check out each commit in turn rather than N worktrees; never mutate the invoking
  checkout.
- Compute ground truth for the same change set from the cached coverage-contexts baseline, restricted to commits for
  which the baseline is a usable ancestor. Report — do not silently drop — commits skipped for lack of usable ground
  truth.
- Report per-commit and aggregate **recall**: ground-truth covering test files that the selection contained, over all
  ground-truth covering test files. Report the distribution (median, p90, worst), the count of commits with a non-empty
  blind spot, and the selection-size cost.
- Report recall **twice**: closure-only and closure-plus-contexts. The gap between them is exactly the exposure a
  baseline-less workspace runs with, and it is the number phase `compensate` is tuned against.
- Provide an opt-in `--execute` mode that, for a small sample, actually runs the missed test files at that commit and
  reports whether any fail. Recall is a proxy; an executed failure is a true false negative. `--execute` must never be
  the default and must not be wired into any `check` path.

**Acceptance.** `just selection-backtest --limit 50` produces a recall report over at least 30 commits with usable
ground truth, or states precisely how many commits lacked it and why. The harness is not added to `just check` or
`just check-full` — it is a measurement tool, not a gate.

**Non-goals.** Not CI integration (that is ready task `sase-fv`). Not changing selection behavior — this phase only
measures.

## Phase `visible` — make the scoped lane observable on success

**Problem.** A passing `just check` hides everything the scoped lane decided.

**Deliverable.** The scoped lane's one-line summary reaches the agent on the success path — selected count and share,
escalation status, and contexts-baseline status (present / missing / stale). Implement by having `run_silent` (or the
`test-scoped` recipe) forward a designated single summary line on success rather than by removing output capture from
the other gates; do not make the eleven lint gates chatty.

**Acceptance.** A passing `just check` shows the selected count, whether the run escalated, and whether the baseline was
missing or stale. `tests/test_justfile_lint.py` or a sibling test pins that the summary survives the success path, so a
future change to `run_silent` cannot silently re-hide it.

**Non-goals.** No change to selection behavior or to the manifest schema.

## Phase `compensate` — stop degrading silently without a baseline

**Problem.** `context-baseline-missing` / `context-baseline-stale` are recorded and then ignored. Roughly half of scoped
runs narrow with no compensating action.

**Constraint.** Adding those rules to `FULL_SUITE_RULES` is the wrong fix and must not be the chosen option: it would
escalate about half of all scoped runs to the full suite and destroy the lane's reason to exist.

**Deliverable.** Choose the compensating action **on phase `backtest`'s measured closure-only recall**, not on
intuition, and implement it as a named rule that appears in the manifest, in `just selection-health`, and in phase
`visible`'s summary line. Candidates, in the order they should be evaluated:

1. **Depth + 1 when no usable baseline.** Cheapest and most likely sufficient. Ready task `sase-fu` measured a depth-3
   median selection of 9.9% of the suite — still far below the 0.25 escalation ratio, so the cost is affordable. Must be
   justified by measured recall recovery, not assumed.
2. **Directory-mirror expansion** — include every test under the `tests/` directory mirroring the changed package — for
   changed modules whose closure-only recall the backtest shows to be poor.
3. **Escalate only in the narrow case** where the change touches a module the backtest identifies as
   widely-executed-but-shallowly-imported, which is the `_app_layout.py` shape.

The phase must state the measured recall before and after, and the measured selection-size and duration cost. If the
chosen action does not materially improve recall, say so and record the negative result rather than shipping motion.

**Acceptance.** With the contexts store forced absent, the `_app_layout.py` change recovers a stated and measured share
of those 69 covering test files; a named rule records that compensation happened; the escalation rate and median
selected size after the change are re-measured and reported.

**Non-goals.** Not revisiting the 0.25 `max_ratio` itself (ready task `sase-fw`). Not the depth table re-measurement
(ready task `sase-fu`) beyond consuming its finding. Not breaking the 497-module import cycle that forces depth-bounding
in the first place (ready task `sase-fy`, the actual root cause).

## Phase `baseline` — a baseline a local full run can produce

**Problem.** The only supply route for ground truth is a CI artifact with 14-day retention, published on master pushes
only. A workspace that is idle longer than that, or offline, or on a host that never fetched, falls back to the static
closure alone — which is the exposure phase `compensate` is mitigating rather than removing.

**Deliverable.** A local `cov-contexts` run installs its database into the host-local cache as `<HEAD sha>.sqlite` where
`resolve_baseline` will find it, so any later scoped run on that host — in any workspace — resolves a real ancestor
baseline. Decide and state explicitly whether this is automatic on `just test-contexts` or a separate opt-in step; do
**not** make coverage instrumentation automatic on `just check-full`, whose cost profile agents depend on at landing
time. Respect the existing per-SHA cache layout and pruning.

**Acceptance.** After a local `cov-contexts` run, a scoped run in a different workspace on the same host resolves that
baseline and reports `context-selection` rather than `context-baseline-missing`. Verified end to end, not asserted.

**Non-goals.** Not CI artifact retention, not a scheduled refresh job, not a warning when no baseline is within
retention — all three are ready task `sase-fx`, whose scope this phase must not absorb. This phase adds a _local
producer_; `sase-fx` fixes the _remote supply_. Coordinate, do not duplicate.

## Phase `land` — land the epic

- Re-run `just selection-backtest` and `just selection-health` on the combined tree and record both readings on the epic
  bead, including the correlatable-sample size and the pre-schema-2 exclusion count. State plainly whether the
  ≥30-varied-changes exit criterion is now met by backtest evidence; if it is met by backtest but not by live
  correlation, say exactly that rather than blurring the two.
- File collected follow-ups one at a time via `/sase_new_task`. Corroborate rather than duplicate against ready tasks
  `sase-ft`, `sase-fu`, `sase-fv`, `sase-fw`, `sase-fx`, `sase-fy`.
- Run `just check-full` on the combined tree, then `just symvision` after the close.
- Mark this plan `status: done`.

## Out of scope for this epic

- Anything that changes how much the fast lane costs at the median — the lane's measured median of 34 files and 24.6s is
  the product, not an accident to be traded away.
- CI-side correlation (`sase-fv`), the `max_ratio` threshold (`sase-fw`), CI artifact availability (`sase-fx`), the
  depth table (`sase-fu`), the host-local graph cache (`sase-ft`), and the `src/sase` import cycle (`sase-fy`).
- The load-sensitive flake family already tracked on `sase-e2` and `sase-ct`.
