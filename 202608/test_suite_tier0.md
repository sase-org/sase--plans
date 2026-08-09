---
tier: tale
title: Test suite Tier 0 — fix the CI worker collapse, drop visual from the default lane, and right-size the gate
goal:
  The default `just test` lane stops running the visual PNG suite and the three dismissed-bundle scale tests, the
  suite-gate stops collapsing to a single worker on small-CPU hosts, and the per-worker memory reserve matches measured
  worker RSS — with every relocated test still covered by a CI job.
size: medium
proposed_by: bbugyi200.athena.tm
create_time: 2026-08-05 19:11:25
status: done
---

- **PROMPT:**
  [prompts/202608/test_suite_tier0.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/test_suite_tier0.md)
- **AGENTS:**
  - [bbugyi200.athena.tm](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.tm.md)
- **COMMITS:**
  - [9672c56](https://github.com/sase-org/sase/commit/9672c5602816c39f3d6f2e4af2b50a2e032f0d5e) — fix(tests): stop CI
    worker collapse and drop visual from default lane

# Plan: Test suite Tier 0

## Background

The research report `202608/test_suite_verification_architecture/test_suite_verification_architecture.md` in the
`sase--research` sidecar concludes that the suite is slow because of **admitted demand**, not slow tests: roughly
200–400 full-suite runs per day at ~61 worker-minutes each, against a host that supplies ~46,000 worker-minutes per day.

This plan implements only that report's **Tier 0** section — four low-risk changes it describes as "this week, hours of
work, no architectural risk". Tier 1 (diff-scoped selection, `just check-full`, the no-lease scoped path) and Tier 2 are
explicitly **not** in scope.

The canceled epic `sase-fd` ("Make the default parallel test suite reliable under host contention") was closed with the
reason "See research.y agents (I plan on going with their plan instead)". This plan is that replacement for the Tier 0
slice. Change 2 in particular retires the lane with the known contention-flake profile that `sase-fd` was chasing.

### Measurements taken while writing this plan

All figures below were measured directly, at commit `4330fd0d5`, on the 64-CPU / 62-GiB development host. They supersede
the report's figures where they differ, and the reasons for the differences are stated.

| Fact                                                 | Value                                   | How it was measured                                                                                           |
| ---------------------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Gate budget on a 4-vCPU runner **today**             | **1** worker                            | `_calculate_default_token_budget(cpu_count=4, ...)`; `cpu_limit = max(4 - 4, 1) = 1`                          |
| Gate budget on a 4-vCPU runner **after Change 1**    | **3** workers                           | same function with the proportional reserve                                                                   |
| `slow` lane runtime today                            | **23.5 s**, 7 passed / 2 skipped        | `SASE_PYTEST_WORKERS=4 tools/run_pytest slow`                                                                 |
| The three scale tests, warm and serial               | **25.17 s + 14.99 s + 4.25 s = 44.4 s** | `pytest tests/test_dismissed_bundle_persistence.py --durations` with `TMPDIR` on the disk-backed scratch root |
| The same three tests with `TMPDIR` on `/tmp` (tmpfs) | 0.71 s + 0.64 s + 0.14 s                | same command, default `TMPDIR`                                                                                |

Two consequences of that last pair matter to the implementer:

1. **The scale tests are I/O-bound, not CPU-bound.** `tools/run_pytest` redirects `TMPDIR` to a disk-backed scratch root
   under `/var/tmp`, which is why the report saw 160 s under 12-worker contention while a naive `pytest` invocation on
   tmpfs shows 1.5 s. Do not "verify" these tests by running bare `pytest` without the redirect — you will measure the
   wrong thing and conclude the change is pointless.
2. **The report's unnamed "third scale test" is `test_bundle_save_does_not_leak_index_file_descriptors`** (4.25 s,
   matching the report's "~13 s" under contention at the same ~3.4× inflation ratio the other two show). The other two
   are named in the report.

## Scope

Four changes, plus the guard rails each one needs. All four are small and land as **one commit** — they overlap in
`tests/_suite_gate.py`, `tests/test_suite_gate.py`, `tests/test_github_actions_ci.py`, and `docs/development.md`, and
splitting them would cost more full-suite runs than the changes themselves save.

---

## Change 1 — Make the gate's CPU reserve proportional

**Problem.** `tests/_suite_gate.py` reserves a flat 4 CPUs:

```python
_RESERVED_CPUS = 4
...
cpu_limit = max(cpu_count - _RESERVED_CPUS, 1)
```

On a 4-vCPU GitHub runner that yields `cpu_limit = 1`, so `budget = 1`, so `automatic_worker_range(1)` returns `(1, 1)`,
so the entire CI test matrix runs with `pytest -n 1`. `.github/workflows/ci.yml` sets no `SASE_TEST_GATE_SLOTS` to
override it.

**Edit `tests/_suite_gate.py`.** Replace `_RESERVED_CPUS` with a proportional reserve:

```python
# Reserve a proportion of the host rather than a flat count. A flat 4 is noise
# on the 64-core development host but is the entire machine on a 4-vCPU CI
# runner, where it collapsed the budget to one token and made the whole CI test
# matrix run serially.
_RESERVED_CPU_DIVISOR = 8
_MINIMUM_RESERVED_CPUS = 1
```

and in `_calculate_default_token_budget`:

```python
reserved_cpus = max(_MINIMUM_RESERVED_CPUS, cpu_count // _RESERVED_CPU_DIVISOR)
cpu_limit = max(cpu_count - reserved_cpus, 1)
```

**Measured effect** (memory column assumes Change 4 has also landed):

| Host                                 | Budget today | After Changes 1+4 |
| ------------------------------------ | -----------: | ----------------: |
| CI runner, 4 vCPU / 14 GiB available |            1 |             **3** |
| dev, 64 CPU / 48 GiB available       |           32 |     32 (hard cap) |
| dev, 64 CPU / 30 GiB available       |           18 |            **23** |
| dev, 64 CPU / 20 GiB available       |            9 |            **12** |
| dev, 64 CPU / 14 GiB available       |            4 |             **6** |

The CPU change alone is a no-op on the development host (memory and the 32-token cap bind there, not CPU). That is
intended: Change 1 is the CI fix, Change 4 is the dev-host fix.

**Do not** set `SASE_TEST_GATE_SLOTS` in `ci.yml` as the alternative the report offers. Fixing the formula fixes every
small host — CI runners, contributor laptops, future runner sizes — instead of one workflow file, and an explicit slot
count also flips `capacity_is_explicit`, which changes the gate's error messages.

## Change 4 — Right-size the per-worker memory reserve

(Presented next because it shares a file and a test with Change 1.)

**Edit `tests/_suite_gate.py`.** `_MEMORY_KIB_PER_WORKER` is `1229 * 1024` (1.2 GiB), calibrated from an old
eight-worker run that peaked at 0.65 GiB/worker. The report sampled live worker RSS across concurrent sibling workspaces
at **0.74–0.85 GiB**. Set it to 950 MiB, which still leaves ~12% headroom over the top of that measured range:

```python
# Live worker RSS sampled across concurrent sibling workspaces ranges from
# 0.74 to 0.85 GiB. Reserving 950 MiB per token keeps headroom over the top of
# that range while leaving memory as a real rather than an inflated constraint;
# the previous 1.2 GiB over-reserved by ~45% and shrank the host pool by ~30%
# exactly when contention made memory scarce.
_MEMORY_KIB_PER_WORKER = 950 * 1024
```

### Tests for Changes 1 and 4

`tests/test_suite_gate.py::test_default_budget_arithmetic` is parametrized over
`(cpu_count, mem_available_gib, expected)`. Exactly two existing cases change; the other four are unaffected:

| Case            | Today | After | Why                                               |
| --------------- | ----: | ----: | ------------------------------------------------- |
| `(64, 64, 32)`  |    32 |    32 | unchanged — 32-token hard cap binds               |
| `(8, 64, 4)`    |     4 | **7** | Change 1: reserve on an 8-core host drops 4 → 1   |
| `(64, 10, 1)`   |     1 | **2** | Change 4: `(10 - 8) GiB / 950 MiB = 2`            |
| `(2, 64, 1)`    |     1 |     1 | unchanged — `max(1, 2 // 8) = 1`, `cpu_limit = 1` |
| `(None, 64, 1)` |     1 |     1 | unchanged — unknown CPU count                     |
| `(64, None, 4)` |     4 |     4 | unchanged — missing-memory fallback               |

**Add a case that pins the bug that motivated Change 1**, so a future edit cannot silently reintroduce it:

```python
# A 4-vCPU CI runner must get real parallelism; a flat CPU reserve used to
# collapse this shape to a single worker and serialize the whole CI matrix.
(4, 14, 3),
```

## Change 2 — Exclude the visual lane from the default `just test`

**Problem.** `pyproject.toml`'s own `addopts` already carries `-m "not slow and not visual"`, and CI runs visual in a
dedicated `visual-test` job — but `tools/run_pytest` overrides the marker with `FAST_MARKER_EXPRESSION = "not slow"`, so
the local default lane is _looser_ than CI. That lane is 902.9 s (27% of runtime) for 420 tests (1.6%), and it is the
lane with the documented contention-flake profile (`just test-visual-contention` records a pre-fix baseline of 116
failed / 246 passed under 13× oversubscription).

**Edit `tools/run_pytest`:**

- Set `FAST_MARKER_EXPRESSION = "not slow and not visual"`.
- Delete `FAST_NON_VISUAL_MARKER_EXPRESSION`, `EXCLUDE_VISUAL_ENV`, and the `_fast_marker_expression()` helper. Once the
  default excludes visual, the `SASE_PYTEST_EXCLUDE_VISUAL` opt-out can only ever be a no-op, and leaving a no-op
  environment variable wired through CI is actively misleading.
- In `_pytest_command`, use `FAST_MARKER_EXPRESSION` directly for both the `fast` and `cov` branches.

**Edit `.github/workflows/ci.yml`:** remove both `SASE_PYTEST_EXCLUDE_VISUAL: "true"` `env:` blocks (lines ~180–188, the
coverage leg and the plain-test leg). Their comment "The dedicated visual-test job is the sole visual execution" is now
enforced by the default, not by the workflow.

**Escape hatch.** No new flag is needed. `pytest`'s `-m` is last-one-wins and `_pytest_command` appends caller args
after its own `-m`, so `just test -- -m "not slow"` still runs visual alongside everything else.

### Do NOT change the `_setup-visual` dependency

It is tempting to switch `just test` and `just test-cov` from `_setup-visual` to `_setup` now that visual is deselected.
**This breaks the suite.** Marker deselection happens _after_ collection, and `tests/ace/tui/visual/conftest.py` imports
`tests.ace.tui.visual.png_diff` → `_png_diff_artifacts` → `_png_diff_comparison` → `from PIL import Image, ImageChops`
at module scope. Verified by blocking the `PIL` import and collecting with `-m "not slow and not visual"`:

```
ImportError while loading conftest '.../tests/ace/tui/visual/conftest.py'
E   ImportError: blocked: PIL
```

Leave both recipes on `_setup-visual`. Making the visual conftest import lazily is worth doing but is separate work (see
Follow-ups).

### Tests for Change 2

- `tests/test_run_pytest_tool.py:~98` — `assert result[-1] == "not slow"` becomes `"not slow and not visual"`.
- `tests/test_run_pytest_tool.py:~315` `test_cov_mode_selects_non_slow_marker_to_include_visual_tests` and `~333`
  `test_cov_mode_can_exclude_visual_tests_for_noncanonical_ci_legs` — these two exist only to test the two sides of the
  deleted environment variable. Replace them with a single test asserting the `cov` mode emits
  `["-m", FAST_MARKER_EXPRESSION]` along with the `--cov` flags, and rename it to describe the new invariant (the
  coverage leg excludes visual, matching the dedicated `visual-test` job).
- Grep for `EXCLUDE_VISUAL_ENV` and `FAST_NON_VISUAL_MARKER_EXPRESSION` afterwards; both must be gone.
- `tests/test_github_actions_ci.py:172` `test_visual_suite_runs_only_in_dedicated_job` asserts
  `run_tests["env"]["SASE_PYTEST_EXCLUDE_VISUAL"] == "true"`. **Rewrite it, do not delete it** — it guards a real
  invariant. It should now assert that no step in the `test` job sets any visual opt-in env var, and keep its existing
  assertions that the `test` job uploads no `sase-visual` artifacts and that `visual-test` is the job running
  `just test-visual`.
- `docs/development.md:270` states "The 3.13 and 3.14 legs set `SASE_PYTEST_EXCLUDE_VISUAL=true`". Update it to say the
  default lane excludes visual and the dedicated `visual-test` job is the sole visual execution.

## Change 3 — Move the three dismissed-bundle scale tests behind `slow`, and keep them in CI

**Edit `tests/test_dismissed_bundle_persistence.py`.** Add `@pytest.mark.slow` to exactly these three:

| Test                                                            | Warm serial call time | What makes it expensive                                         |
| --------------------------------------------------------------- | --------------------: | --------------------------------------------------------------- |
| `test_save_dismissed_bundle_is_fast_with_many_existing_bundles` |               25.17 s | 1,000 `save_dismissed_bundle` calls through the production path |
| `test_bundle_no_limit`                                          |               14.99 s | 600 `save_dismissed_bundle` calls                               |
| `test_bundle_save_does_not_leak_index_file_descriptors`         |                4.25 s | repeated saves to count SQLite index file descriptors           |

The other 16 tests in the file must stay in the fast lane — together they are under 1 s.

### The `slow` lane currently runs in no CI job — fix that in the same commit

This is the part the report does not mention and it is the one way this change can go wrong. There is **no CI job that
runs `just test-slow`**: `ci.yml` has `build-core`, `lint`, `docs-build`, `test`, `visual-test`, `perf-floors`, and
`published-core-minimum-smoke`, and none of them invoke it. Marking a test `slow` today therefore does not relocate it —
it retires it from all automated verification.

That matters most for `test_save_dismissed_bundle_is_fast_with_many_existing_bundles`, whose entire assertion is
`elapsed < 1.0`: it is a performance regression guard, and silently retiring it is exactly the failure mode it exists to
catch.

**Edit `.github/workflows/ci.yml`.** Add a slow-lane step to the existing `perf-floors` job — the natural home, since
these are scale/performance assertions and that job already collects performance floors. Place it alongside the other
floor steps, following their `if: always()` convention:

```yaml
- name: Run slow tests
  if: always()
  run: just test-slow
```

Cost is negligible: the `slow` lane measures **23.5 s** today (9 collected tests; the `tests/perf/bench_*.py` modules
carry `pytest.mark.slow` but pytest never collects them, because they do not match the default `test_*.py` pattern and
are instead run as scripts by dedicated `just` recipes). Adding the three tests above puts the lane around 70 s against
`perf-floors`' 45-minute budget.

**Add a regression test** in `tests/test_github_actions_ci.py` asserting some job runs `just test-slow`, so the lane
cannot silently disappear again the way it silently never existed.

**One thing to watch.** The slow lane currently prints a "system temp leakage" report — two `uv-setuptools-*.lock` files
left in the scratch root by `tests/uv_tool/test_real_uv_harness.py`. In the measured run this was informational and the
run exited 0 (`7 passed, 2 skipped`). Confirm it is still non-fatal once the lane runs in CI; if it does fail the job,
file a task bead for the `uv_tool` harness rather than deleting the new CI step or disabling the guard.

## Documentation

`docs/development.md` (line numbers as of `4330fd0d5`):

- **36** — `just test  # Fast parallel test run, including PNG visual snapshots` → state that it excludes visual and
  that `just test-visual` is the PNG lane.
- **40** — `just test-cov  # ... including visual snapshots` → same correction.
- **67** — "a solo run can receive 28 workers from the 32-token pool" — still correct; leave it.
- **71–73** — "reserves four CPUs and 8 GiB of available memory, allows 1.2 GiB per worker" and the "0.65 GiB RSS per
  worker ... nearly 2x per-worker headroom" calibration sentence → describe the proportional CPU reserve
  (`max(1, cpu_count // 8)`), the 950 MiB per-worker figure, and the 0.74–0.85 GiB measured-RSS provenance.
- **270** — the `SASE_PYTEST_EXCLUDE_VISUAL` sentence, per Change 2.

## Explicitly out of scope

- **Do not edit `sase/memory/build_and_run.md`.** Its lines 12–15 (`just test ... includes PNG visual snapshots`) and
  line 15 (`just test-cov ... also runs the visual snapshot suite`) become stale with Change 2, but SASE memory files
  may only be edited with the user's explicit permission given in a live conversation, and **a plan file does not grant
  that permission**. Leave the file untouched and surface the needed correction to the user instead.
- **Do not rewrite the three scale tests.** Both source reports note their setup could use direct fixture construction
  instead of thousands of production save calls, preserving the assertions at a fraction of the cost. That is the better
  long-term fix and belongs in its own task.
- **Do not switch `just test` to `_setup`** — see the Pillow collection trap above.
- **No Tier 1 or Tier 2 work**: no diff-scoped selection, no `just check-full`, no no-lease path, no gate fair-share, no
  per-PR runtime budget, no result cache.

## Verification

1. `just install` first — workspaces are ephemeral and may be stale.
2. `just check` must pass. Note that `just _lint-symvision` is known-red on master for an unrelated reason
   (`progress_fingerprint` in `src/sase/llm_provider/commit_finalizer_git.py`, tracked as `sase-fj`). If that is the
   only symvision failure, it is pre-existing — do not attempt to fix it here, and do not let it mask a real one.
3. Targeted re-runs of every test file this plan touches:
   `just test -- tests/test_suite_gate.py tests/test_run_pytest_tool.py tests/test_github_actions_ci.py tests/test_dismissed_bundle_persistence.py`
4. Confirm the visual lane is actually deselected: the fast lane's collected count should drop by ~420 relative to
   master. Compare `just test -- --collect-only -q | tail -1` before and after.
5. Confirm the relocation worked in both directions:
   - `just test-slow -- --collect-only -q` lists all three dismissed-bundle tests (lane goes from 9 to 12 collected).
   - `just test -- --collect-only -q tests/test_dismissed_bundle_persistence.py` lists 16, not 19.
6. Confirm the CI fix arithmetically — `test_default_budget_arithmetic`'s new `(4, 14, 3)` case is the assertion that
   the 4-vCPU runner now gets 3 workers instead of 1.

## Follow-ups to propose (do not implement here)

Use `/sase_new_task` for each; do not create beads without it.

- Rewrite the three dismissed-bundle scale tests to build their archive fixtures directly instead of calling
  `save_dismissed_bundle` 1,000 times, then return them to the fast lane.
- Make `tests/ace/tui/visual/conftest.py` import Pillow lazily so `just test` can drop its `_setup-visual` dependency
  and skip the visual dependency install in every fresh workspace.
- Correct `sase/memory/build_and_run.md` to match the new default lane (needs the user's explicit permission).
