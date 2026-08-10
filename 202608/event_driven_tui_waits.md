---
tier: tale
title: Replace ACE test idle sleeps with event-driven settling
goal:
  ACE TUI tests drain Textual work and rendered frames without paying the CPU-idle
  heuristic's fixed 20ms sleep, while bounded waits retain their timeouts and
  diagnostics, intentional real-time tests remain explicit, and contention coverage
  shows no new flakes.
size: medium
proposed_by: bbugyi200.athena.sase-ib.2
bead: sase-ib.2
create_time: 2026-08-09 11:32:26
status: done
---

- **PROMPT:**
  [prompts/202608/event_driven_tui_waits.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/event_driven_tui_waits.md)
- **PARENT:**
  [202608/fast_test_suite_1.md](https://github.com/sase-org/sase--plans/blob/main/202608/fast_test_suite_1.md)
- **BEAD:**
  [sase-ib.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ib/sase-ib.2.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ib.2](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ib.2.md)
- **COMMITS:**
  - [cfe18d7](https://github.com/sase-org/sase/commit/cfe18d7f0de46080e1a5b9e509845261e543b946)
    — perf(test): make ACE TUI waits event-driven

# Plan: Replace ACE test idle sleeps with event-driven settling

## Outcome and baseline

Textual 8.0.1 implements `Pilot.pause(None)` as `_wait_for_screen()` followed by
`textual._wait.wait_for_idle(0)`. The latter sleeps in 20ms slices and judges idleness
from process-wide CPU time. That gives every bare pause a 20ms floor and becomes a
particularly poor signal under xdist, where unrelated work in the same worker changes
the process CPU reading.

The committed suite-cost baseline records the resulting shape:

- serial `tests/ace/tui/modals`: 91.8s wall / 42.0s CPU, or 47% utilization;
- 2,148 Textual app boots and thousands of bare pause calls in the ACE lane; and
- 3,719 aggregate per-test seconds in the fast suite.

This phase replaces that timing heuristic only in the test harness. It does not alter
production event-loop behavior, shorten the existing 5s bounded-wait defaults, weaken
assertions, move tests out of the fast lane, or change any test's coverage. The old CPU
idle path remains an explicit escape hatch for the few tests that truly need it.

## Barrier contract

1. Add a focused helper module under `src/sase/ace/testing/` that exposes:
   - an event-driven `settle_pilot(pilot)` barrier which first runs Textual's zero-delay
     pilot path to enqueue and drain all app/screen/widget message pumps, then awaits an
     explicit `App.wait_for_refresh()` callback so layout, repaint, and screen callbacks
     are complete before returning; and
   - an explicitly named CPU-idle escape hatch which delegates to Textual's original
     `Pilot.pause(None)` behavior.

   Keep the Textual-private compatibility boundary in this one module, document why the
   zero-delay pump drain and after-refresh callback are both required, and add focused
   tests proving queued widget work and a requested frame have completed when the
   barrier returns. A test should patch `textual._wait.wait_for_idle` (or the captured
   original pause boundary) to fail if the normal settle path accidentally invokes the
   CPU-idle heuristic.

2. Make event-driven settling the ACE test default without rewriting thousands of call
   sites:
   - install a scoped autouse fixture for ACE TUI tests which makes bare `Pilot.pause()`
     use `settle_pilot`;
   - preserve every explicitly numeric `Pilot.pause(delay)` as a real requested delay;
   - route `AcePage.pause()`, `AcePage` startup settling, its `expect_*` methods, and
     the lightweight editor harnesses through the shared barrier so callers outside the
     `tests/ace/tui` fixture receive the same default; and
   - expose a clearly named `AcePage`/helper escape hatch for the CPU-idle behavior,
     rather than overloading the new bare-pause default.

   Test restoration/isolation of the fixture so one test cannot leak a monkeypatch into
   another. Keep action helpers such as `Pilot.press()` intact; they already use
   Textual's pump-drain primitive and must not acquire a fixed sleep.

## Bounded waiters

3. Consolidate raw-pilot `sase.ace.testing.wait.wait_for` and the four `AcePage` polling
   families (`expect_state`, screen contains/not-contains, and `wait_for`) on a small
   internal polling engine:
   - evaluate the observable predicate immediately;
   - after a miss, await the event-driven settle barrier;
   - after several consecutive misses, yield a short real sleep as a backstop for
     thread, worker, timer, and pump-free-task completions so a hot predicate cannot
     spin or starve its producer; and
   - check the monotonic deadline on every iteration, never sleeping past it.

   Preserve the public 5s defaults and every existing timeout message byte-for-byte,
   including the last observed value in `expect_state`. Add deterministic unit tests
   with fake pilots/clocks for immediate success, success after queued pump work,
   off-pump success after backoff, timeout, non-spinning behavior, and exact error text.
   Do not replace semantic waiters with bare pauses.

4. Extend the opt-in cost recorder and labels so the report distinguishes the new ACE
   settle barrier and the explicit CPU-idle escape hatch. The cost plugin must remain
   off for ordinary fast/cov/scoped runs. Retain the old `Pilot.pause(None)` and
   `textual.wait_for_idle` causes so before/after recordings remain interpretable and
   can prove the normal ACE path no longer reaches them.

## Fixed-delay audit and regression lint

5. Inventory every actual `time.sleep(...)` and `asyncio.sleep(...)` AST call under
   `tests/` (ignoring strings used as child-process programs, mocks of production sleep,
   and zero-delay cooperative yields). For each positive fixed delay:
   - replace it with an observable `Event`, queue item, future, thread join, fake clock,
     or existing ACE bounded waiter when the delay merely guesses that work finished;
   - retain real elapsed time only when passage of wall time or event-loop starvation is
     the behavior under test (for example watchdog, debounce/TTL, filesystem watcher,
     process-timeout, and deliberate-contention tests); and
   - mark every retained call with a narrow inline pragma carrying a non-empty reason.

   Extend `tools/check_test_wait_helpers` and its unit tests to reject new positive
   literal fixed-delay calls without that explicit rationale, along with the existing
   private `_wait_until` and inline pilot-polling rules. The checker should allow
   `sleep(0)` cooperative yields and deadline-clamped backoff expressions, reject empty
   or detached pragmas, report a stable finding kind and source line, and leave
   production code outside `tests/` out of scope. Run the checker after the audit and
   ensure the repository has zero findings.

## Performance and correctness verification

6. Run `just install` before verification. Iterate with focused tests for the barrier,
   `AcePage`, raw bounded waiter, cost plugin, wait-helper checker, and every test file
   whose fixed delay changed. Confirm the new assertions fail against the old behavior
   (especially the “no CPU-idle call” assertion) and pass after the change.

7. Record a one-worker `just test-cost` run for `tests/ace/tui/modals` and compare it to
   `tests/perf/baselines/test_cost_baseline.json`. Report wall seconds, CPU seconds, CPU
   utilization, settle counts/seconds, and any remaining CPU-idle calls. Acceptance is
   at least 80% serial CPU utilization and at least a 40% wall-clock reduction for the
   measured ACE TUI target. If the target misses, use the cost report's top-file tables
   to find and remove remaining heuristic/fixed waits rather than relaxing the target.

8. Verify correctness in increasing scope:
   - all focused helper and audited-sleep files;
   - the complete non-visual `tests/ace/tui` lane;
   - `just check-full` (not only `just check`, because the wait default affects the
     broad ACE surface);
   - `just selection-health --fail-on-new-flake` for the committed sase-h8.8 baseline;
   - `just test-contention` for the ACE TUI selection, using the established 26-worker /
     two-CPU harness and its repeat tally; and
   - a targeted contention repeat of
     `tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py` which reports the two
     `test_vcs_tag_*` nodes as better, worse, or unchanged for active task `sase-hk`.

   Keep all current timeout values unless a test's actual semantic contract requires a
   change. If any timeout changes, name the test and justification in the bead close
   note. Do not update a flake baseline to hide a new failure. Do not accept PNG golden
   changes; this phase changes scheduling cost, not intended rendering.

9. Review the final diff and cost report for coverage neutrality: no deleted/renamed
   nodes used to reduce counts, no new skip/xfail/slow/visual markers, no weakened
   assertions, and no production TUI behavior changes. Close only `sase-ib.2` with
   `sase bead close sase-ib.2 --note "<measurements and verification>"`; do not close
   parent epic `sase-ib`. Record genuinely out-of-scope discoveries only as
   `PROPOSED FOLLOW-UP:` notes on `sase-ib.2`, never as new beads.

## Non-goals

- Sharing or reducing ACE app boots; that belongs to dependent phase `sase-ib.3`.
- Changing production timers, debounce intervals, worker scheduling, or Textual itself.
- Raising worker-token limits or spending more host CPU/memory to improve wall time.
- Fixing the root cause of the `test_vcs_tag_*` flake tracked by `sase-hk`; this phase
  only measures whether the new settling strategy changes its observed behavior.
- Editing SASE memory files, generated instruction shims, or the parent epic.
