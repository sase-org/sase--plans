---
tier: epic
title: Fix red CI - bead-store test isolation and visual snapshot convergence flakes
goal: 'Every job in the sase CI workflow passes on master again, and stays passing
  across repeated runs: the bead-backend and test-matrix jobs stop tripping the pytest
  state write guard, and the visual-test job stops failing on timing-dependent PNG
  snapshot captures. Both fixes address the underlying causes; neither is worked around
  by regenerating goldens or by loosening the exact-equality PNG comparison.

  '
phases:
- id: bead-fix
  title: Restore bead-store isolation for the epic-work CLI tests
  depends_on: []
  size: small
  description: 'bead-fix: give the unisolated epic-work CLI test the project_dir fixture
    its siblings use, and close the same latent gap in the neighbouring preview tests.

    '
- id: visual-converge
  title: Contention repro harness and load-robust visual convergence
  depends_on: []
  size: medium
  description: 'visual-converge: build a repeatable contention harness that reproduces
    the visual snapshot flakes locally, then make render-convergence detection robust
    to CPU starvation instead of relying on a wall-clock quiet period.

    '
- id: visual-capture
  title: Guarantee the compared PNG frame is the converged frame
  depends_on:
  - visual-converge
  size: medium
  description: 'visual-capture: eliminate the gap between the frame that convergence
    proved stable and the frame that is actually rasterized and compared, then fix
    whatever residual per-test races the hardened harness still exposes.

    '
- id: ci-verify
  title: Confirm CI is green and durable
  depends_on:
  - bead-fix
  - visual-capture
  size: small
  description: 'ci-verify: prove on real CI that every job passes and that the visual
    job is no longer flaky across repeated runs, within the existing job timeout budgets.

    '
create_time: 2026-07-27 06:57:27
status: done
bead_id: sase-9y
---

- **PROMPT:** [202607/prompts/fix_ci_bead_isolation_and_visual_flakes.md](prompts/fix_ci_bead_isolation_and_visual_flakes.md)
- **BEAD:** [sase-9y](https://github.com/sase-org/sase--beads/blob/main/pages/sase-9y/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-9y.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9y.land/README.md)
  - [bbugyi200.athena.sase-9y.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-9y.land.md#member-code)
- **COMMITS:**
  - [a947469](https://github.com/sase-org/sase/commit/a947469eece2988bdfff48bd6ee40b5a9701172f) — docs: record final visual contention baseline (sase-9y)

# Plan: Fix red CI - bead-store test isolation and visual snapshot convergence flakes

## Context

CI on `sase-org/sase` master is red. `actstat --repo sase-org/sase -n 3` shows the `CI` workflow failing on every recent
settled commit, and the failures come from exactly two independent causes across five jobs. Everything below was
confirmed against real CI logs, downloaded CI failure artifacts, and local reproduction; the important conclusions are
recorded here so no phase agent has to re-derive them.

### Cause A - `bead-backend` and `test (3.12 / 3.13 / 3.14)`

All four of these jobs fail on the same single test:

```
FAILED tests/test_bead/test_cli_work_epic_lifecycle.py::test_work_missing_bead_json_error_is_one_envelope
  sase.core.state_write_guard._PytestStateIsolationError: Refusing pytest bead-store init_store write to
  /home/runner/work/sase/sase/sdd/beads: target is outside pytest sandbox root ...
```

`test_work_missing_bead_json_error_is_one_envelope` (`tests/test_bead/test_cli_work_epic_lifecycle.py:439`) is the only
test in that file that does **not** request the `project_dir` fixture. `project_dir` (`tests/test_bead/conftest.py:27`)
initializes a `BeadProject` in `tmp_path` and calls `isolate_bead_store_resolution()`, which plants a `CheckoutMarker`,
sets the SDD policy to `in_tree`, and — critically — `monkeypatch.chdir(checkout)`. Without it, `handle_bead_work`
resolves the bead store relative to the real checkout, lands on `<repo>/sdd/beads`, and calls `init_store`, which the
pytest state write guard correctly refuses.

The omission arrived with commit `1f1c4064c` ("fix(bead): restore epic work CLI contracts (sase-9v.7)", 2026-07-26),
which added this test and, in the very same diff, added its sibling `test_work_non_plan_bead_json_error_is_one_envelope`
_with_ `project_dir: Path`. This is a straightforward oversight, not a design decision.

This test passes on a developer machine and fails in CI because the two environments resolve the bead store differently:
a numbered `sase_<N>` workspace is a managed SASE checkout whose marker routes resolution away from `<repo>/sdd/beads`,
and `sdd/beads/` does not even exist in the checkout (`git ls-files sdd/beads` is empty). A bare CI checkout has no
marker, so resolution falls back to the repo-relative path and tries to create the store. Do not "fix" this by making
the guard more permissive — the guard caught a real isolation bug.

The failure is deterministic: `bead-backend` has failed on every recent run including the newest one (`f2c53c2`). The
`test (3.13)` leg burned 1h13m before failing on this one test, so fixing it also stops a large amount of wasted CI
time.

An AST audit of `tests/test_bead/` for tests that touch `handle_bead_work` / `BeadProject` / `bead_cli` without any
isolation fixture found two more, both in `tests/test_bead/test_cli_work_from_plan_preview.py`:
`test_bead_id_mode_rejects_parent_override` (line 165) and
`test_bead_id_mode_rejects_plan_file_only_linking_options_as_json` (line 188). They pass today only because argument
validation rejects before store resolution is reached — they are latent copies of the same bug.

### Cause B - `visual-test`

The `visual-test` job fails on ACE PNG snapshot comparisons, but **the goldens are correct and must not be
regenerated**. The evidence that these are timing flakes rather than stale goldens:

- The failing set is different every run: 15 snapshots on `dd114a6`, 8 on `9e63c5e`, 3 on `9b3074c`. Only
  `agent_neighbor_modal_descendants_dismissed_60x30` recurs across all three.
- The full visual suite passes locally, unloaded: `just test-visual` -> `362 passed, 1 skipped in 64.84s`.
- Constraining the same suite to two cores reproduces the failure class overwhelmingly:
  `taskset -c 0,1 just test-visual` -> `116 failed, 246 passed, 1 skipped in 567.51s`, including one hard
  `wait_for_visual_idle` timeout (`Timed out waiting for ACE visual render convergence after 8.00s; stable_frames=1/3`)
  and 54 large-area diffs consistent with panels that had not finished painting.

The diffs are semantic, not renderer noise. Decoding the CI artifacts from run `30221775115` gives:

| snapshot                                           | changed px    | what actually differs                                     |
| -------------------------------------------------- | ------------- | --------------------------------------------------------- |
| `query_edit_modal_120x40`                          | 378 / 1520532 | focused cursor block present in golden, absent in capture |
| `placeholder_raw_only_highlight_120x40`            | 288 / 1520532 | focused cursor block present in golden, absent in capture |
| `agent_neighbor_modal_descendants_dismissed_60x30` | 83 / 586500   | a count renders `1` where the golden has `2`              |

Because these are real content differences that happen to be small, **relaxing the pixel comparison is not an acceptable
fix**. A wrong digit is roughly 80 pixels; any ratio tolerance loose enough to pass these would blind the suite to
genuine regressions. Note for whoever is tempted: `SASE_VISUAL_PNG_MAX_DIFF_RATIO`,
`SASE_VISUAL_PNG_MATERIAL_DIFF_THRESHOLD`, and `SASE_VISUAL_PNG_MAX_MATERIAL_DIFF_PIXELS` do exist and are unit-tested
in `tests/ace/tui/visual/test_png_diff.py`, but they are set nowhere in `.github/workflows/ci.yml`, and every CI
`summary.txt` records `tolerance_source: default` with `max_diff_ratio: 0.0`. CI therefore compares at exact equality
today. That contradicts the repository memory in `CLAUDE.md`, which claims "CI allows a small ratio-only renderer drift
tolerance" — see the note at the end of this plan.

The underlying defect is in `tests/ace/tui/visual/_ace_png_snapshot_waits.py`. `wait_for_visual_idle` decides the UI has
settled when three consecutively exported SVGs are byte-identical and at least `_VISUAL_STABLE_MIN_SECONDS = 0.1` of
wall-clock has elapsed, polling with `page.pause(0)` (a queue drain) plus 10 ms sleeps, under an 8 s timeout. Under CPU
starvation this heuristic inverts: an unchanging frame stops meaning "the app finished" and starts meaning "the app has
not been scheduled yet". `_pending_visual_work` covers debouncers, running workers, and short one-shot timers, but not
pending focus delivery, `call_after_refresh` callbacks, or in-flight layout/paint — which is exactly what the observed
diffs (missing focus cursor, half-painted panels, a stale count) look like.

One structural detail matters for the fix. CI is **not** oversubscribed: `tools/run_pytest` sizes its xdist pool from
`tests/_suite_gate.py`, where `_RESERVED_CPUS = 4` drives the token budget to 1 on a 4-vCPU GitHub runner, and the CI
log confirms `created: 1/1 worker`. The visual job runs single-worker at about 3.3 s per test, which is not slower per
test than local. So CI flakes come from ordinary shared-vCPU scheduling jitter, and reducing parallelism is not
available as a fix. The wait logic itself has to stop depending on wall-clock quiet.

`tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py::test_agent_neighbor_modal_dismissed_descendant_png_snapshot`
is worth reading before starting: it already carries two rounds of hand-applied anti-race patches with explanatory
comments, and it still fails. It also illustrates a second, distinct gap — it performs further `await`s
(`wait_for_svg_contains`) _after_ `wait_for_visual_idle` returns, so the frame that convergence proved stable is not the
frame that `assert_page_png` later re-exports and rasterizes.

### Constraints that apply to the whole epic

- Do not run `--sase-update-visual-snapshots`. The goldens are right; accepting current output would bake in a missing
  cursor and a wrong count.
- Do not add PNG diff tolerance in CI, and do not weaken `sase.core.state_write_guard`.
- Runtime budgets are real and some are tight. `visual-test` has `timeout-minutes: 45` and currently takes ~20 min, so
  there is roughly 2x headroom. The `test` matrix job has `timeout-minutes: 90`, and the 3.13 leg already took 1h13m
  _without_ visual tests (`SASE_PYTEST_EXCLUDE_VISUAL` is true for 3.13/3.14; the 3.12 leg does include them). A
  convergence fix that makes every wait unconditionally slower could push `test (3.12)` over its ceiling — prefer waits
  that cost nothing when the app is already idle and only pay when it is not.
- Per `CLAUDE.md`, run `just install` before `just check` in a workspace that may be stale, and run `just check` before
  reporting completion.

## Phase `bead-fix`: Restore bead-store isolation for the epic-work CLI tests

Add the `project_dir: Path` fixture parameter to `test_work_missing_bead_json_error_is_one_envelope` in
`tests/test_bead/test_cli_work_epic_lifecycle.py`, matching every sibling test in the file. The fixture yields a fresh
empty bead project, so `sase-missing` is still absent and the asserted envelope
(`{"ok": false, "mode": "bead_id", "epic_id": "sase-missing", "error": "issue not found: sase-missing"}`) is unchanged —
confirm that rather than assuming it, since the point of the test is the exact error envelope.

Then close the same latent gap in `tests/test_bead/test_cli_work_from_plan_preview.py` for
`test_bead_id_mode_rejects_parent_override` and `test_bead_id_mode_rejects_plan_file_only_linking_options_as_json`.
These currently pass by luck of validation ordering; isolating them makes that independent of the order in which
`handle_bead_work` validates.

Because the bug is invisible in a managed `sase_<N>` workspace, verify in a way that actually reproduces CI. Running the
test in a bare checkout, or otherwise forcing resolution down the repo-relative path, is what proves the fix; a plain
local pass does not, since the test already passes locally today.

Consider whether a cheap guard is warranted so this class cannot silently return — for example a check that tests in
`tests/test_bead/` exercising the bead CLI declare an isolation fixture. Add it only if it can be made precise enough
not to become noise; the audit above found the full current population, so this is optional hardening rather than a
requirement.

This phase alone should turn `bead-backend`, `test (3.13)`, and `test (3.14)` green, and remove this failure from
`test (3.12)`.

## Phase `visual-converge`: Contention repro harness and load-robust visual convergence

Start by making the flake reproducible on demand, because the default local suite passes and gives no signal. Two useful
operating points, both already validated:

- Fast and violent, for iteration: `taskset -c 0,1 just test-visual`. Note that `os.cpu_count()` ignores CPU affinity,
  so the suite gate still granted 26 workers against 2 usable cores — that 13x oversubscription is why it reproduces 116
  failures in under 10 minutes. Useful precisely because it is extreme.
- Faithful to CI, for confirmation: a single worker (`SASE_PYTEST_WORKERS=1`) on constrained CPU with competing load,
  matching the `created: 1/1 worker` that CI actually uses.

Land the harness in a form a future agent can rerun (a `just` recipe or documented env knob), and record the baseline
failure count before changing any behavior, so the fix can be measured rather than asserted.

Then make convergence detection robust. The goal is that `wait_for_visual_idle` never reports "settled" merely because
the process was starved of CPU. Directions worth evaluating, in rough order of expected value:

- Use Textual's full `Pilot.pause()` CPU-idle heuristic inside the convergence loop rather than only `page.pause(0)`.
  The current comment says this was avoided for speed; correctness comes first, and the cost should be measured against
  the runtime budgets above rather than assumed prohibitive.
- Stop treating a fixed wall-clock quiet period as proof. Either scale the stability window by observed event-loop
  responsiveness (schedule a no-op and measure its latency), or require stability across scheduler progress rather than
  elapsed time.
- Widen `_pending_visual_work` to cover the work classes the observed failures implicate: pending focus delivery,
  `call_after_refresh` callbacks, and in-flight layout/paint.
- Raise the 8 s timeout, which the two-core run demonstrably exceeded. A convergence timeout should be a loud failure,
  not a routine occurrence.

Success is measured on the harness: the two-core baseline must drop from 116 failures to zero, or to a small residue
that is then handed to `visual-capture`. Keep `just test-visual` on an unloaded machine comfortably fast.

## Phase `visual-capture`: Guarantee the compared PNG frame is the converged frame

Even with convergence hardened, there is a structural hole: `wait_for_visual_idle` proves a frame stable, and then
`assert_page_png` separately re-exports the page and rasterizes _that_ export. Any `await` a test performs in between —
`wait_for_svg_contains`, further `expect_*` calls, and similar — reopens the window, which is exactly the shape of the
`agent_neighbor_modal_descendants_dismissed` failure.

Close that hole so the bytes that are compared are provably the bytes that convergence validated. Pick the design with
the smallest defensible blast radius; the constraint is what matters, not a particular mechanism. Options include
carrying the converged SVG forward into the comparison, or having the assertion itself re-establish convergence. Note
that roughly 360 call sites use the synchronous `assert_page_png`, so a design requiring every call site to become
`await`-ed needs to be worth its cost — weigh it honestly rather than defaulting to the largest change.

Then rerun the harness from `visual-converge` and fix whatever residual per-test races remain. Prefer fixing the shared
helpers over sprinkling per-test waits: the neighbors test shows where that pattern leads, having accumulated two layers
of bespoke waits and remained flaky. Where a test genuinely does need bespoke sequencing, order it so the convergence
wait is the last thing before capture.

## Phase `ci-verify`: Confirm CI is green and durable

Verify on real CI, not only locally, since the whole point is that these two bugs are environment-dependent.

- Confirm every job in the `CI` workflow passes: `install-smoke`, `lint`, `test (3.12)`, `test (3.13)`, `test (3.14)`,
  `bead-backend`, `visual-test`, `docs-build`, `fmt-md-check`, `phase7-perf-floor`, `published-core-minimum-smoke`,
  `launch-perf-floor`, `build`. Use `actstat --repo sase-org/sase` alongside `gh run view`.
- Because the visual failures were probabilistic (3 to 15 snapshots per run), a single green run does not prove the fix.
  Re-run the `visual-test` job several times and require green every time before declaring the flake closed.
- Confirm the runtime budgets still hold, with attention to `visual-test` against its 45-minute ceiling and
  `test (3.12)` against its 90-minute ceiling — the 3.13 leg's prior 1h13m shows how little slack the matrix job has.

Finally, report a documentation discrepancy to Bryan rather than fixing it unilaterally. `CLAUDE.md` states that "CI
allows a small ratio-only renderer drift tolerance", which is not true of the current workflow: no tolerance environment
variable is set in `.github/workflows/ci.yml`, and every CI failure summary records `tolerance_source: default`. That
memory text is under `sase/memory/`, and SASE rules require Bryan's explicit permission in-conversation before editing
memory files — so surface the discrepancy and let him decide whether the documentation or the workflow should change. Do
not edit it as part of this epic.
