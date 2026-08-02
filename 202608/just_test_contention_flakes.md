---
tier: epic
title: Fix just test failures caused by host contention
goal: '`just test` stops failing on a busy host: the suite-gate integration test bounds
  child pytest lifecycles by observable progress instead of idle-host wall clocks,
  and ACE PNG snapshots can no longer capture a stable-but-unfinished frame.

  '
phases:
- id: gate
  title: Load-tolerant suite-gate integration budgets
  depends_on: []
  size: small
  description: 'gate: replace the fixed 60s/20s/10s/15s wall clocks in the suite-gate
    integration test with budgets calibrated from measured child admission latency,
    and fail with child diagnostics instead of a bare TimeoutExpired.'
- id: visual
  title: Close the ACE visual convergence gap
  depends_on: []
  size: medium
  description: 'visual: stop Textual animations from running under PNG snapshot tests
    and teach the convergence helper to treat pending animations as unfinished work,
    so a starved app cannot present five identical mid-animation frames.'
- id: baseline
  title: Revalidate and record the contention baseline
  depends_on:
  - gate
  - visual
  size: small
  description: 'baseline: rerun the visual contention harness and the suite-gate integration
    test under load, confirm both phases hold together, and refresh the recorded harness
    baseline.'
proposed_by: bbugyi200.athena.rw
create_time: 2026-08-02 10:11:34
status: done
bead_id: sase-e9
---

- **PROMPT:** [prompts/202608/just_test_contention_flakes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/just_test_contention_flakes.md)
- **BEAD:** [sase-e9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-e9/README.md)

# Plan: Fix `just test` failures caused by host contention

## Problem

A `just test` run reported two failures:

```
FAILED tests/test_suite_gate_integration.py::test_scaled_suite_runs_share_capacity_and_release_after_sigkill
  - subprocess.TimeoutExpired: Command '[... .venv/bin/python, ... tools/run_pytest, 'fast', '/var/...
FAILED tests/ace/tui/visual/test_ace_png_snapshots_agents_sase_context.py::test_agents_phase_family_bead_and_plan_context_png_snapshot
  - AssertionError: ACE PNG snapshot mismatch: .../snapshots/png/agents_phase_bead_and_plan_context_120x40.png
```

Neither is a product regression. Both are load-sensitivity defects in the test harness, and both were provoked by the
same condition: several full `just test` runs sharing one host.

## Evidence

Host conditions observed while diagnosing (64 CPUs, 62 GiB RAM):

- Three concurrent `just test` runs from three different workspaces shared the host-global worker-token pool
  (`/tmp/sase-pytest-tokens-<uid>/pool.lock` reported `"capacity": 29`).
- I/O pressure was the binding constraint, not CPU: `/proc/pressure/io` showed `full avg300=13.55`, with 9 GiB of swap
  in use. `/proc/pressure/cpu` showed `full avg300=0.00`.
- A full `just test` under that contention took **874 s**, versus **255 s** for the reported run — a 3.4x slowdown of
  the same suite.

### Failure A — suite-gate integration

`tests/test_suite_gate_integration.py` launches four real `tools/run_pytest fast` child sessions, holds three of them on
worker tokens, SIGKILLs one, and asserts the fourth is then admitted. Its liveness budgets are fixed idle-host wall
clocks:

| Budget                                       | Location                                           | Value |
| -------------------------------------------- | -------------------------------------------------- | ----- |
| Released child must exit and close its pipes | `_assert_success` (`communicate(timeout=60)`)      | 60 s  |
| Child must connect to the coordinator        | `coordinator.settimeout(20)`                       | 20 s  |
| SIGKILLed group must reap                    | `_kill_process_group` (`process.wait(timeout=10)`) | 10 s  |
| Child's own gate wait                        | `SASE_TEST_GATE_TIMEOUT` in `_child_environment`   | 15 s  |

The reported run recorded `64.52s call` for this test. The `call` phase measures 4.1–4.3 s under load in this
repository, and a released child exits ~0.6–0.7 s after release, so 64.52 s is ~4.5 s of normal setup plus the 60 s
`communicate` budget expiring exactly. The 60 s budget is what failed.

Two mechanisms were investigated and **ruled out**, so the fix must not chase them:

- _Pipe-buffer deadlock._ A child emits 592 bytes on stdout and 0 on stderr, far below the 64 KiB pipe buffer, so
  `communicate()` cannot deadlock on a full pipe.
- _An orphaned xdist worker holding the inherited stderr pipe open forever._ `execnet.gateway_io` does spawn workers
  with `Popen(args, stdout=PIPE, stdin=PIPE)`, leaving stderr inherited from the controller, but
  `xdist.workermanage.NodeManager.EXIT_TIMEOUT` is 10 s and `execnet.multi.Group.terminate` SIGKILLs any gateway that
  misses that deadline, so the linger is bounded.

What remains is the intended one: under host contention a child pytest session's teardown genuinely exceeded 60 s. There
is also a latent second failure mode in the same table — the child's own `SASE_TEST_GATE_TIMEOUT` (15 s) is _below_ the
test's admission deadline (20 s), so on a slow host the waiting child can abort itself with a `pytest.UsageError` before
the test ever gives up waiting for it.

### Failure B — ACE PNG snapshot

The named test passes in isolation (`5 passed in 14.10s` for its file). Its PNG output is also independent of
scratch-path length, verified by rendering it under both a short `--basetemp` and a long
`/var/tmp/sase-<hash>/pytest-of-<user>/pytest-<n>`-shaped `--basetemp`; both matched the golden. So the mismatch is
neither a stale golden nor renderer drift.

Running the repository's own contention harness (the `just test-visual-contention` shape: 26 workers pinned to two CPUs)
over `tests/ace/tui/visual` reproduced the _class_ of failure:

```
2 failed, 403 passed in 447.28s (0:07:27)
FAILED tests/ace/tui/visual/test_ace_png_snapshots_agents_slow_tools.py::test_agents_slow_tool_calls_fold_levels_png_snapshots
FAILED tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py::test_real_fakey_retry_countdown_png_snapshot
```

That is a regression against the baseline recorded in the `test-visual-contention` recipe comment in `Justfile`, which
documents `363 passed, 1 skipped` with no failures as of 2026-07-27.

The captured artifact for `agents_slow_tool_calls_level_1_120x40` shows this is a content difference, not a rasterizer
difference: `changed_ratio 0.0329`, `material_diff_ratio 0.0329` — 3.3 % of pixels differ materially (alpha-aware colour
distance > 8). The diff image shows the entire detail panel rendered at a **different vertical scroll offset**: the
golden starts at the `Name / ChangeSpec / Workspace / Model / Activity / Timestamps` metadata block, while the actual
capture starts at the `SLOW TOOL CALLS` heading, with every later row shifted by the same amount.

The reported test itself did **not** reproduce in six targeted contention iterations (`19 passed` each time), so it is a
rarer instance of the same class rather than a separate deterministic break. Fixing the class is what closes it.

Root cause of the class: `wait_for_visual_idle()` in `tests/ace/tui/visual/_ace_png_snapshot_waits.py` accepts a capture
once five consecutive `page.pause()` cycles export byte-identical SVG and `_pending_visual_work()` reports nothing
pending. `_pending_visual_work()` inspects three things — the named detail debouncers, running Textual workers, and
short one-shot timers — and **never inspects Textual's animator**. Meanwhile:

- `textual.constants.TEXTUAL_ANIMATIONS` defaults to `"FULL"` (`get_environ("TEXTUAL_ANIMATIONS", "FULL")`).
- `App.__init__` seeds `self.animation_level` from that constant.
- `App.run_test()` disables tooltips and notifications but does **not** disable animations.

So ACE snapshot tests run with smooth scrolling animated. Under CPU starvation the animator's frame timer can be starved
long enough for five consecutive exports to be byte-identical while a scroll animation is still in flight, and the
capture then lands at a non-final scroll offset — exactly the whole-panel vertical shift seen in the artifact.

## Non-goals

- Do not regenerate any PNG golden. The goldens are correct; the captures are premature.
- Do not relax the PNG comparison tolerance. Local runs must keep exact pixel equality.
- Do not weaken any semantic assertion in the suite-gate integration test. Bounded admission, capacity sharing, and
  release-after-SIGKILL must still be asserted exactly; only liveness budgets may move.
- Do not mark either test `slow` or otherwise remove it from `just test`.

## Phases

### Load-tolerant suite-gate integration budgets

Scope: `tests/test_suite_gate_integration.py`.

Replace the four fixed budgets with budgets calibrated from a quantity the test already measures — how long the host
actually took to start and admit the first three children.

1. Time the initial admission: record a monotonic timestamp before launching the three initial children and after
   `_accept_run` has returned all three. Call that `startup_seconds`; it is a direct sample of how expensive one child
   pytest boot is on this host right now.
2. Derive each budget from it, keeping the current values as floors so behaviour on an idle host is unchanged. Suggested
   shape, with the constants named and commented at module level:
   - admission deadline: `max(20.0, 8 * startup_seconds)`
   - released-child exit budget: `max(60.0, 20 * startup_seconds)`
   - SIGKILL reap budget: `max(10.0, 4 * startup_seconds)`
3. Fix the latent inversion: the child's `SASE_TEST_GATE_TIMEOUT` must stay comfortably **above** the test's admission
   deadline, so a slow child never aborts itself before the test stops waiting. Because the child environment is built
   before `startup_seconds` is known, either raise the child gate timeout to a fixed value safely above the largest
   admission deadline the test will use, or rebuild the waiter's environment after the initial admission is timed. State
   which approach was taken in a comment.
4. Replace the bare `subprocess.TimeoutExpired` with a diagnostic failure. On timeout, SIGKILL the child's process
   group, drain whatever output is available, and raise an assertion that includes the child's run id, its stdout and
   stderr, and the token-pool state from `_active_grants`. A future failure should say which child hung and what the
   pool looked like, not just that a command timed out.
5. Keep the existing structure otherwise: `_assert_success` must still assert `returncode == 0`.

Validation: run `tests/test_suite_gate_integration.py` repeatedly (at least 10 iterations) while a second full suite
runs concurrently, and confirm it passes and that the `call` duration stays in the seconds range on an unloaded host.

### Close the ACE visual convergence gap

Scope: `tests/ace/tui/visual/_ace_png_snapshot_waits.py`, `tests/ace/tui/visual/conftest.py`, and the two reproduced
snapshot tests.

1. Disable animations for visual snapshot tests. Snapshots want the resting state, never an animated intermediate. Note
   the ordering constraint: `textual.constants.TEXTUAL_ANIMATIONS` is a module-level `Final` evaluated at import time,
   so setting the environment variable from a fixture is too late once `textual` is imported. Set the level where it
   takes effect — patch `textual.constants.TEXTUAL_ANIMATIONS` and/or assign `app.animation_level = "none"` before the
   app runs — and do it in the autouse visual fixture in `tests/ace/tui/visual/conftest.py` alongside the existing
   `TZ`/`COLORTERM`/app-version pins. Add a test that proves the level is actually `"none"` on a running `AcePage`, so
   this cannot silently regress.
2. Defend in depth in `_pending_visual_work()`: report an in-flight animation as pending work by inspecting the animator
   (`page.app.animator` exposes `_animations` and `_scheduled`; `Animator` also offers `wait_until_complete()`).
   Convergence must not accept a frame while an animation is registered, even if animations are disabled by default — a
   future test that re-enables them must not silently reintroduce this class. Include the pending animation keys in the
   existing convergence-timeout diagnostic message.
3. Confirm the two reproduced tests pass under the contention harness afterwards:
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_slow_tools.py::test_agents_slow_tool_calls_fold_levels_png_snapshots`
   and `tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py::test_real_fakey_retry_countdown_png_snapshot`.
   If either still fails once animations are off, the residue is a second mechanism in that test — most likely the
   `ctrl+j` retry loop in `_focus_slow_tool_section`, which drives the panel until the active section identity matches
   but never pins the resulting scroll offset. In that case add an explicit wait on the settled scroll offset before
   capture rather than adding more pauses.
4. Do not regenerate goldens. If a golden appears to need updating, that is a signal the capture is still premature;
   diagnose instead. Failure artifacts land in `.pytest_cache/sase-visual/` with `expected.png`, `actual.png`,
   `diff.png`, `actual.svg`, and `summary.txt`.

### Revalidate and record the contention baseline

Runs after both fixes land.

1. Run the visual contention harness (`just test-visual-contention`, or the equivalent 26-worker, two-CPU `taskset`
   invocation over `tests/ace/tui/visual`) and confirm zero failures.
2. Run `just test` to completion and confirm it is green.
3. Run `tests/test_suite_gate_integration.py` under a concurrent second suite and confirm it passes.
4. Update the baseline comment above the `test-visual-contention` recipe in `Justfile` with the new measured result and
   date, replacing the stale 2026-07-27 line, so the next agent to touch visual convergence compares against a current
   number.
5. Run `just check`.

## Notes for the implementing agents

- Reproducing either failure requires real contention. A quiet host will show green for both tests and prove nothing.
  The cheapest reliable harness is the 26-worker / two-CPU `taskset` shape used by `just test-visual-contention`; for
  the gate test, run a second full suite concurrently.
- `just install` before running anything, then `just check` before finishing.
- One unrelated observation, deliberately **not** in scope for this plan: on a checkout whose linked `sase-core` is
  older than the `sase` HEAD it is paired with,
  `tests/stats/test_binding_smoke.py::test_statistics_facade_smoke_through_real_bindings` and
  `tests/test_core_agent_runtime.py::test_facade_does_not_use_synthesized_terminal_timestamp` fail deterministically
  (for example `wall_clock_seconds: 144000.0 != 0.0`, and a missing `lanes_counted` field). That is a stale linked-repo
  checkout, not a defect in this repository, and is fixed by updating the linked `sase-core` checkout and re-running
  `just install`. Do not try to "fix" those two tests here.
