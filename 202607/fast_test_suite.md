---
tier: epic
title: Fast test suite under multi-agent load
goal: 'A solo `just test` on athena completes at least 2x faster than today''s ~4-minute
  wall time with an identical test selection and no weakened assertions, and any number
  of concurrent agent suite runs share one host-wide worker budget so every run starts
  promptly and total resource use stays bounded below the levels that caused the 2026-07-20
  host meltdown.

  '
phases:
- id: worker-tokens
  title: Host worker-token budget
  depends_on: []
  size: medium
  description: '''Host worker-token budget'' section: replace whole-suite gate slots
    with a crash-safe host-global pool of xdist worker tokens so a solo run gets ~2x
    more workers and concurrent runs share capacity fairly without head-of-line queueing.'
- id: tui-harness
  title: ACE pilot harness cost reduction
  depends_on: []
  size: medium
  description: '''ACE pilot harness cost reduction'' section: profile and eliminate
    the systematic per-test startup and teardown overhead of the AcePage/pilot TUI
    harness, including the 3-4.5s config-pane teardowns, without weakening any assertion.'
- id: hot-tests
  title: Top-offender test optimizations
  depends_on:
  - tui-harness
  size: medium
  description: '''Top-offender test optimizations'' section: fix the slowest individual
    tests surfaced by the durations baseline - repo-scanning audit tests, zoom-panel
    and keymaps e2e files, and visual PNG snapshot hot spots - keeping every test
    and assertion intact.'
- id: scheduling
  title: Distribution scheduling and stragglers
  depends_on:
  - worker-tokens
  size: medium
  description: '''Distribution scheduling and stragglers'' section: evaluate xdist
    worksteal distribution against the current loadfile mode, prove within-file order
    independence or fall back to splitting straggler files, and eliminate the end-of-run
    tail.'
- id: fixed-overhead
  title: Fixed recipe overhead trim
  depends_on: []
  size: small
  description: '''Fixed recipe overhead trim'' section: measure and cut the ~30-50s
    of non-pytest overhead in every `just test` invocation (dependency-validation
    scripts, nested just recursion, collection) without touching tools/run_pytest.'
- id: verify-under-load
  title: Verification under concurrent load
  depends_on:
  - worker-tokens
  - tui-harness
  - hot-tests
  - scheduling
  - fixed-overhead
  size: small
  description: '''Verification under concurrent load'' section: record final before/after
    solo timings, prove bounded-and-fair behavior with concurrent scaled suite runs
    against a tiny token pool, confirm the coverage gate is unchanged, and update
    the docs with real numbers.'
create_time: 2026-07-20 10:59:51
status: wip
---

# Plan: Fast test suite under multi-agent load

## Context and baseline measurements

`just test` (via `tools/run_pytest fast`) runs 19,744 tests. A full run measured on athena (64 cores / 62 GiB RAM) on
2026-07-20, while one other agent's suite ran concurrently:

- **Total recipe wall time: 4:04.** The pytest segment is **194 s**; the remaining ~50 s is Justfile
  `_setup-visual`/`_setup` validation, nested `just` calls, gate wait, and pytest/collection startup.
- **Total test CPU: ~1,911 s** (1,724 user + 187 sys) at ~780% CPU — i.e. under 10 of 64 cores busy on average.
  `tools/run_pytest` sized the run at 14 workers: `min(cpu_count // 4 = 16, MemAvailable/2 GiB = 14)`.
- **Top durations concentrate in four buckets:**
  1. **ACE TUI pilot tests** using the `AcePage`/pilot harness (`src/sase/ace/testing/__init__.py`): 3–20 s per test.
     `tests/ace/tui/test_agents_zoom_panel_search.py` has 17–20 s tests; `tests/ace/tui/test_agents_zoom_panel_files.py`
     has a dozen 4–6 s tests; `tests/ace/tui/test_config_pane_widget.py` shows **teardown alone costing 3.2–4.5 s on ~12
     tests** (~45 s of pure teardown in one file).
  2. **Visual PNG snapshots** (276 tests, in the default lane): 3–10 s each.
  3. **Source-scanning audit tests** (`tests/test_agent_artifact_*_audit.py`,
     `tests/test_check_sase_core_rs_bindings_tool.py`): 4–7.6 s each, several per file, each re-scanning the repository
     from scratch.
  4. **`tests/test_keymaps_e2e.py`**: 4–10 s per test.
- **Concurrency behavior:** the host-global suite gate (`tests/_suite_gate.py`, landed 2026-07-20 in response to that
  morning's load-1551 / swap-exhaustion meltdown; see `sase/repos/plans/202607/host_test_suite_gate.md`) allows **2
  whole-suite slots**. A third concurrent agent waits up to 45 minutes doing nothing, while the 2 admitted suites use at
  most ~32 of 64 cores.

Two structural conclusions follow:

- With ~1,911 CPU-seconds of work, the theoretical floor at 32 workers is ~60 s + startup. The current 14-worker cap,
  the end-of-run straggler tail, and per-test harness overhead are each worth roughly a minute or more of wall time.
- The whole-suite gate solves the meltdown but at the cost of head-of-line blocking. Governing **workers** instead of
  **suites** preserves the same host-wide bound while letting every run start immediately and letting a solo run use far
  more of the idle machine.

## Hard constraints

These apply to every phase and are the acceptance bar for the epic:

- **No coverage reduction.** No test is deleted, skipped, or moved out of the fast lane (no re-marking as
  `slow`/`visual` to dodge the default selection). No assertion is weakened or removed. Harness stubs must not silently
  disable code paths that existing tests assert on: any test that covers a stubbed loader must opt out of the stub. The
  CI `just test-cov` 50% coverage gate and its measured percentage must not drop.
- **No resource-crash regression.** The guarantees of the 2026-07-20 suite-gate fix are preserved: host-wide concurrent
  pytest worker count stays bounded by one budget regardless of how many agents launch suites; locks are kernel-released
  on holder death (SIGKILL-safe); a waiting run times out cleanly with actionable holder metadata rather than piling
  onto a saturated host; escape-hatch env vars keep working. Worst-case admitted load must not exceed today's ceiling
  (two 16-worker suites, ~20–32 GiB peak).
- **No flakiness.** Waits stay event-driven (no sleep-based synchronization). Any scheduling-mode change must be
  validated with repeated full runs before it becomes the default and must keep a one-env-var fallback to the previous
  behavior.
- **Rust core boundary:** all of this is Python-side test infrastructure and stays in this repo.

## Design overview

Six phases in three independent tracks, merged by a final verification phase:

- **Capacity track:** `worker-tokens` (host-global worker budget) then `scheduling` (per-run distribution efficiency).
  Both touch `tools/run_pytest`, so they are strictly ordered.
- **Test-cost track:** `tui-harness` (systematic per-test overhead) then `hot-tests` (remaining individual offenders,
  re-measured after the harness fix so effort is not wasted on tests the harness already fixed). These touch
  `src/sase/ace/testing/`, ACE teardown paths, and test files only.
- **Overhead track:** `fixed-overhead` (Justfile recipe fixed costs; explicitly barred from `tools/run_pytest` to avoid
  conflicting with the capacity track).

`verify-under-load` lands last and proves the combined result.

## Host worker-token budget

Evolve the suite gate from "N concurrent suites" to "N concurrent **xdist workers**, host-wide":

- **Token pool.** A shared directory (default `/tmp/sase-pytest-tokens-<uid>`, override env var) holds one slot file per
  worker token, each claimed via `fcntl.flock(LOCK_EX | LOCK_NB)` exactly like the existing gate, so tokens auto-release
  when any holder dies. Budget size is computed at acquisition from host facts: roughly
  `min(cpu_count - reserve, MemAvailable // gib_per_worker)` with a conservative per-worker memory figure calibrated by
  actually measuring worker RSS during a real run (the prior plan's figure: a 16-worker suite peaks at 10–16 GiB, ≈1
  GiB/worker). The default budget must keep worst-case admitted workers at or below today's two-suite ceiling (~32–40) —
  far under the ~100–160 that melted the host.
- **Acquisition in `tools/run_pytest`.** Before choosing `-n`, the runner acquires between a floor (~4) and a per-run
  ceiling (~24–32) of tokens: wait (with the existing status-line and timeout UX) until at least the floor is free, then
  greedily take whatever is free up to the ceiling and set `-n` to the acquired count. Because `run_pytest` calls
  `os.execv`, the flock'd file descriptors (with CLOEXEC cleared) survive into the pytest process and are held for
  exactly its lifetime.
- **Backstop for direct `pytest -n` runs.** `tests/_suite_gate.py` keeps its `pytest_configure` hook: a parallel run not
  already token-governed (marker env var set by `run_pytest`) acquires tokens equal to its worker count from the same
  pool, so raw pytest invocations and `run_pytest` runs share one budget. Serial runs and xdist workers remain exempt,
  as today.
- **Env contract.** `SASE_PYTEST_WORKERS` keeps absolute precedence (bypasses sizing but still acquires that many
  tokens). `SASE_TEST_GATE_DISABLED=1` keeps meaning "ungoverned" and is still exported to descendants of a governed run
  so nested pytest invocations spawned by tests can never deadlock against their ancestor. Existing `SASE_TEST_GATE_*`
  vars are honored or explicitly superseded with documented replacements.
- **Outcome.** Solo run: ~24–32 workers instead of 14 (bounded by the memory budget). Three or more concurrent agents:
  all start immediately with at least the floor, sharing the same total budget that two suites consume today, instead of
  one agent idling 45 minutes.
- Unit tests mirror the existing `tests/test_suite_gate.py` patterns (tmp token dirs, tiny timeouts, SIGKILL
  crash-release proof, floor/ceiling/greedy-acquisition behavior, budget arithmetic with monkeypatched meminfo). Update
  `docs/development.md` and `docs/configuration.md`.

## ACE pilot harness cost reduction

The `AcePage` harness and sibling pilot harnesses boot the full ACE TUI app per test; teardown can block for seconds.
This phase attacks the systematic per-test constant:

- **Profile first.** Instrument one representative file (e.g. `tests/ace/tui/test_config_pane_widget.py`) to split cost
  into app construction, mount/`_mount_state_loads_done` wait, test body, and `run_test().__aexit__` teardown. Identify
  exactly what mount-time loads run under the harness and what teardown waits on (background workers, thread pools,
  timers).
- **Stub non-essential mount work under test.** The visual conftest already stubs the projects loader and plugin
  incoming-commits fetch; generalize that approach for the non-visual pilot harness so app boot does not perform real
  filesystem scans, agent scans, or update checks that the test does not exercise. Stubs must be opt-out per
  test/fixture so tests that assert on those loaders keep their real paths (coverage constraint).
- **Fix teardown.** Make app shutdown under `run_test` cancel or promptly join background workers and timers instead of
  waiting them out — the 3.2–4.5 s per-test teardowns in the config-pane file are the proof case. If a production-code
  change is needed (e.g. a worker missing cancellation handling), that is in scope and is itself a UX win for real
  quits.
- **Success bar:** an empty `AcePage` test (enter + exit) costs well under 1 s; config-pane teardowns drop below ~0.5 s;
  a re-run of the durations baseline shows the pilot bucket collapsing. Full-suite green with zero new flakes across
  repeated runs.

## Top-offender test optimizations

Re-measure `--durations=80` after `tui-harness` lands, then fix what remains of today's list:

- **Audit tests** (`tests/test_agent_artifact_*_audit.py`, `tests/test_check_sase_core_rs_bindings_tool.py`): each test
  re-scans the source tree (4–7.6 s). Share one scan per file or session via module/session fixtures and assert against
  the shared snapshot. Same assertions, one scan instead of N.
- **Zoom-panel and keymaps files** (`test_agents_zoom_panel_search.py` 17–20 s tests, `test_agents_zoom_panel_files.py`,
  `test_keymaps_e2e.py`): profile what dominates after the harness fix (e.g. large static file content re-renders,
  per-key pilot presses, redundant modal boots) and restructure the tests' setup — not their assertions — to remove it.
- **Visual PNG snapshots:** keep every snapshot asserted, but look for shared-cost wins: reuse one app boot for
  multi-snapshot tests already structured that way, ensure the rasterizer is invoked efficiently, and confirm no debug
  artifacts are written on the passing path.
- Each fix must show a measured before/after for its file, and the phase ends with a fresh durations table recorded for
  the final phase.

## Distribution scheduling and stragglers

The suite currently uses `--dist=loadfile` (pytest-xdist 3.8.0), which strands whole heavy files on single workers and
leaves a low-utilization tail (~780% CPU average on a 14-worker run):

- **Evaluate `--dist=worksteal`.** It preserves initial file grouping but lets idle workers steal work, which directly
  attacks the tail without any file surgery. Safety validation: repeated (at least 3) full green runs, plus a check that
  no test in the suite depends on same-worker, same-file execution (shared module-level mutable state, module/session
  fixtures with worker-local side effects). Ship behind an env override (e.g. `SASE_PYTEST_DIST`, default the new mode)
  with `loadfile` as the documented fallback.
- **If worksteal proves unsafe,** keep `loadfile` and instead split the measured straggler files (largest wall-time
  files from the fresh durations data) into smaller files along existing helper boundaries, so no single file dominates
  the tail. Splitting moves tests verbatim — no selection or assertion changes.
- **Measure the outcome** as end-to-end wall time and CPU utilization of the pytest segment on otherwise-idle athena,
  before and after, at the token-governed worker count.

## Fixed recipe overhead trim

`just test` spends ~30–50 s outside pytest. Attribute and trim it:

- Time each `_setup-visual`/`_setup` validation step (`validate_dependency_group`, `validate_editable_metadata`,
  `validate_sase_core_rs`, nested `just` invocations such as `_core-overrides-arg`) and the pytest collection phase.
- Cache validation verdicts keyed on the content hashes/mtimes of their actual inputs (`pyproject.toml`, `uv.lock`, venv
  metadata) so an unchanged environment skips straight to pytest; keep a documented env var to force revalidation, and
  keep the slow path automatic whenever inputs change so stale-venv protection (the reason `_setup` exists in ephemeral
  workspaces) is not weakened.
- Consolidate redundant interpreter/`just` launches on the hot path where cheap to do.
- Constraint: this phase does not modify `tools/run_pytest` (owned by the capacity track); its surface is the Justfile
  and the validation tools it calls.

## Verification under concurrent load

Final phase, after all tracks merge:

- **Solo timing:** record 3 consecutive `just test` runs on otherwise-idle athena; report recipe wall time, pytest
  segment, worker count, and CPU utilization against the 2026-07-20 baseline (4:04 wall / 194 s pytest / 14 workers /
  ~780% CPU). Acceptance: ≥2x total wall-time improvement.
- **Concurrent behavior:** an integration exercise (marked appropriately if not cheap) launches several scaled-down
  parallel suite runs against a tiny token pool in a temp directory and asserts: every run starts within the floor
  guarantee rather than queueing whole-suite; the sum of granted workers never exceeds the pool; killing a holder
  mid-run frees its tokens for a waiter. On the real host, launch 3 concurrent `just test` runs and record that all
  three progress simultaneously with bounded total worker processes and no swap growth.
- **Coverage parity:** run `just test-cov` and confirm the selected-test count and coverage percentage match the
  pre-epic values (no lane shrinkage).
- **Docs:** update `docs/development.md` and `docs/configuration.md` with the final worker-budget model, env vars, and
  measured numbers; note the supersession in the suite-gate story where relevant.

## Testing strategy

- Every infrastructure change (token pool, dist mode, caching) carries unit tests in the established
  `tests/test_suite_gate.py` style: temp dirs, tiny timeouts, subprocess crash-safety proofs, and no contact with the
  real host pool (always inject the override dir env var).
- Every performance claim is backed by a measured before/after on the same host, recorded in the ChangeSpec/commit
  description of the phase that makes it.
- Repeated-run flake checks gate the risky changes (harness stubs, teardown changes, dist mode) before they become
  defaults.
- `just check` gates every phase, per repo policy.

## Risks and mitigations

- **Token pool admits more total workers than two suites do today** (e.g. many 4-worker floors): the budget arithmetic
  caps the pool itself, not just per-run counts — floors are grants from the same fixed pool, so the worst case is the
  pool size, which is set at or below today's ceiling.
- **Harness stubs mask real regressions:** stubs are opt-out and narrowly scoped to loaders with dedicated coverage
  elsewhere; the coverage-parity check in the final phase catches silent lane shrinkage.
- **worksteal exposes hidden test-order dependencies:** repeated full runs plus the env-var fallback make the rollout
  reversible in one line; the straggler-file-split alternative achieves most of the tail win with zero ordering change.
- **Teardown changes race real shutdown paths:** teardown fixes prefer explicit cancellation APIs and are soak-tested
  with repeated runs (the repo already has a residual-freeze soak precedent).
- **Validation caching goes stale in ephemeral workspaces:** cache keys are input hashes, so any dependency change
  invalidates automatically; a force-revalidate env var and the unchanged `just install` path remain.
- **CI differences:** GitHub Actions runners are small and single-tenant; the token budget floor and memory math must
  degrade to at least today's worker counts there, and CI runs are part of each phase's gate anyway.

## Out of scope

- Axe/launcher-side scheduling of when agents run `just check` (a broader scheduler feature; the token pool makes it
  unnecessary for test safety).
- Changing which tests exist in which lane, coverage thresholds, or the visual-goldens policy.
- Rust-core (`../sase-core`) changes — everything here is Python test infrastructure.
