---
tier: tale
title: Isolate the Services j/k benchmark sample window
goal:
  Measure only this app's Services j/k actions while preserving the 16 ms p95 gate and
  completing sase-1vy.3.
size: medium
proposed_by: bbugyi200.athena.sase-1vy.3
proposed_with: codex/gpt-6-astra
bead: sase-1vy.3
status: done
---

- **PARENT:**
  [202611/backlog_epic_2.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_2.md)
- **BEAD:**
  [sase-1vy.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vy/sase-1vy.3.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1vy.3](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1vy.3.md)
- **COMMITS:**
  - [a4bbd29](https://github.com/sase-org/sase/commit/a4bbd29cba49713c5cefab84029c771142c9a4ce)
    — fix(bench): isolate Services j/k sample window

# Isolate the Services (Axe) j/k benchmark sample window

Complete phase `sase-1vy.3` of `plan:202611/backlog_epic_2.md`, addressing `sase-199`.
The benchmark must measure cached j/k navigation on Services and retain its existing 16
ms p95 limit, even when other-tab samples or delayed telemetry writes reach the same
log.

## Scope and governing decisions

This is one bounded implementation: repair the benchmark harness and add deterministic
regression coverage. A `medium` tale fits one coding agent. The phase has no prior work
notes or recorded unfinished implementation; its current source still contains the
reported capture pattern. No child epic is needed.

- `decisions:rust-core-required`: changes stay in Python test infrastructure for Textual
  measurements. No shared backend behavior, Rust binding, or pin change is required.
- `decisions:check-full-is-explicit`: run `sase tool run check`; do not run
  `just check-full` or the entire slow benchmark suite.
- `decisions:triage-annotates-does-not-change-exit-codes`: investigate new/unknown
  failures and retain the true verification result. An unrelated failure needs
  independent clean-base evidence.
- `decisions:handoffs-end-through-host-completion` and
  `decisions:host-owned-completion`: stage handoffs and finish through `/sase_final`;
  use `/sase_monitor` before starting verification that cannot fit the turn.

The approved parent has no additional decision choices for this phase and authorizes no
memory edits. Its two-round backlog limit remains binding: do not propose another
backlog epic. This plan changes no memory, production navigation, keymaps, budgets,
screenshots, or unrelated benchmark scenarios.

The user's explicit close scope overrides the phase description's request to also close
`sase-199`: close only `sase-1vy.3` once implemented and verified. Leave the original
task's disposition and all ancestor closure to the epic's land agent.

## Evidence from the current checkout

Read `sase bead read sase-1vy.3 -r "Need phase scope and prior notes"` and
`sase bead read sase-199 -r "Need benchmark defect and all corroborations"` before
implementation, and check whether newer work has already landed. The original task has
four corroborations of Agents rows contaminating the Axe capture under load. The design
was read through
`sase artifact read plan:202611/backlog_epic_2.md "Need parent design"`.

Relevant current code:

- `tests/ace/tui/bench_tui_jk_panes.py::test_bench_axe_jk` constructs an app on its
  default Agents tab, presses Tab twice, warms navigation, takes
  `before = len(_read_samples(...))`, and later summarizes every subsequent log row. It
  asserts the literal tab name `axe`.
- `src/sase/ace/tui/tab_order.py` defines the canonical `SERVICES_TAB = "services"`.
  `axe` is now an input alias. `AceApp(initial_tab=SERVICES_TAB)` is supported and the
  perf timer records the canonical `current_tab` value.
- Tab is a context-dependent action: prompt ownership can suppress it, and
  `BasicNavigationMixin.action_next_tab` can focus an artifact pane without switching
  away from Agents. Two Tab presses are not a reliable benchmark setup contract.
- `src/sase/ace/tui/util/perf.py::JKPerfTimer` records `tab`, `action`, `t_keypress`
  from `time.perf_counter()`, and `app_instance_id`. Its completed in-memory samples are
  independent of disk publication.
- `src/sase/ace/tui/util/perf_writer.py::PerfWriter` publishes through a bounded
  process-wide FIFO thread. A disk row count is a publication position, not an action
  boundary. Even same-tab warmup rows can arrive after that count is taken.
- `tests/ace/tui/_bench_tui_jk_helpers.py` owns `_read_samples`, `_summarize`, the perf
  fixture, warmup, and the 16 ms `_AXE_P95_BUDGET_MS` constant. Preserve its existing
  unscoped reader contract for other consumers.
- `tests/ace/tui/bench_tui_jk.py` re-exports this benchmark for the historical slow
  command. Keep that entry point working.

These are source findings, not a fresh slow-benchmark reproduction. The original
historical failure may involve more than one timing mechanism; the fix must make both
tab selection and the capture boundary explicit.

## Implementation

### 1. Establish the measured tab explicitly

In `test_bench_axe_jk`, construct `AceApp` with `initial_tab=SERVICES_TAB`, retaining
`auto_start_axe=False`, `refresh_interval=0`, and the existing synthetic fixture. Remove
the two positional Tab presses. Wait for startup using the existing helper, install the
fixture, and warm navigation as today.

Assert the canonical active tab after startup/warmup and around measured input so
filtering cannot disguise a benchmark that navigated the wrong surface. Retain the
fixture-size assertion and the existing 20 j then 20 k inputs and pauses. Update the
benchmark's description/table label to identify Services (Axe), while keeping its
historical function name for existing commands.

### 2. Capture by action identity and time, not disk row count

Replace the disk-count boundary for this benchmark with a monotonic measurement
interval: capture its start immediately after warmup and before measured input, and its
end after the measured loop. Use the same `time.perf_counter()` clock as
`JKPerfTimer.t_keypress`.

Add a small test-only selector, in the existing helper module or a focused sibling if
needed for file-size limits. It reads through `_read_samples` and selects rows matching
all of:

- the benchmark app's nonempty `app_instance_id` from its timer;
- `tab == SERVICES_TAB`;
- `action` in `{"next", "prev"}`;
- `t_keypress` within the captured measurement interval.

Use this selector in the actual benchmark. Foreign-app rows, other-tab rows, warmup rows
published late, and later teardown actions must not affect the summary. Do not filter by
latency, discard outliers, or change percentile calculation.

Before the final JSONL read, establish that the benchmark's queued samples have been
published. Prefer a small asynchronous test helper that submits a sentinel callback
through the existing shared `PerfWriter.submit` FIFO and awaits a thread-safe event with
a bounded timeout. Submit it only after the final measured paint has settled; the
callback merely signals completion of earlier writes. Fail clearly if queue admission or
draining fails. Do not call blocking `queue.join()` on the event loop, add arbitrary
settling sleeps, switch production telemetry to synchronous writes, or add a production
writer API for this test.

Use the timer's completed samples to verify that the durable window did not lose
measured rows: compare the eligible keypress identities, and fail on missing or
duplicate rows. The current 48 warmup-plus-measured actions fit the timer's 200-sample
window. Preserve nonempty capture, both `next` and `prev`, the canonical tab-set
assertion, `_AXE_P95_BUDGET_MS == 16.0`, and `_assert_no_stall_rows`. A wrong-tab run or
an empty selected set must fail, never pass vacuously.

### 3. Add deterministic regressions to the normal test lane

Add `tests/ace/tui/test_bench_tui_jk_sample_window.py` (or an equivalently focused
non-slow test module). Cover the real benchmark's use of the selector as well as its
edge cases, using fake app/pilot or controlled telemetry fixtures to avoid requiring
actual host contention or 40 rendered keypresses in a unit regression. Import the slow
benchmark under a non-test alias so it is not accidentally collected again.

Required cases:

1. Mixed log rows: pre-window Services warmup; in-window Agents and Artifacts rows; a
   different app's Services rows; measured Services `next` and `prev`; and post-window
   Services rows. Only the measured pair enters the summary, regardless of write order.
   Give foreign rows extreme `paint_ms` values to prove exclusion happens before
   summarization.
2. Delayed publication: release an old warmup/foreign row and the final measured row
   after the window has started. The read waits for the controlled FIFO boundary,
   excludes the old/foreign rows, and retains the late measured row. Synchronize
   explicitly with events; do not make the test depend on wall-clock load.
3. Invalid capture: wrong active tab, no matching samples, missing direction, or a
   missing measured durable row fails clearly. Exercise timeout/admission failure in the
   drain helper without a real long wait.
4. Latency integrity: a matching Services row over the unchanged p95 threshold still
   fails. Contamination removal must not suppress real regressions.

At least one regression must invoke the actual `test_bench_axe_jk` collection path with
injected non-target rows and fail against the old implementation. Demonstrate that
failure before applying the fix (or by temporarily restoring only the old capture path),
then restore the final code and run it green. A standalone predicate test that passes
while the benchmark still uses the old slice is insufficient.

## Verification and completion

1. Read the current `lint_and_test.md`, `tui.md` / `tui_perf.md`, and `ace.md` through
   `/sase_memory_read` as needed. Use this checkout's virtualenv; if setup is needed,
   follow the existing environment workflow rather than bypassing binding checks.
2. Run the focused regression file and `tests/ace/tui/test_services_tab_id.py` with
   `.venv/bin/pytest -q`. Record the old-path red and fixed-path green outcomes.
3. Run the actual focused benchmark:
   `.venv/bin/pytest -s -m slow tests/ace/tui/bench_tui_jk.py::test_bench_axe_jk`.
   Preserve the output counts, tab attribution, p95 values, and stall check. Run it
   again only if a failure or new change requires more evidence. Use `/sase_monitor`
   before starting if this verification cannot fit the turn; never terminate an
   in-flight run to move it to a monitor.
4. Run formatting (`just fix`, or at least `just fmt`) and `sase tool run check`. Prefer
   a verification monitor for a long final gate, following `/sase_monitor` and
   `/sase_final`. No `just check-full` or screenshot regeneration is required.
5. Investigate new/unknown failures. A failure that reproduces identically on the clean
   base does not keep this phase open: append a
   `PROPOSED FOLLOW-UP: <summary — clean-base evidence and any existing task ID>` note
   to `sase-1vy.3`, retain the true check result, and close after the scoped fix is
   verified. Do not create task beads or weaken a benchmark budget to get green.
6. Run `sase bead epic-symbols sase-1vy.3` immediately before closing. It was empty
   during planning; resolve any new entry or re-key it to a still-open owning bead.
7. Close only `sase-1vy.3` with
   `sase bead close sase-1vy.3 --note "<fix, regression evidence, focused benchmark outcome, check run ID, and any clean-base failures>"`.
   Do not set status by hand, reopen this in-progress bead, close `sase-199`, or close
   `sase-1vy` or another ancestor. Finish through `/sase_final`.
