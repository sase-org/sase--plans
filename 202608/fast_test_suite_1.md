---
tier: epic
title: Make `just test` fast under agent contention
goal: '`just test` runs the same 27,978 tests in roughly half the CPU-seconds and
  a fraction of the wall clock it costs today, on an idle host and — especially —
  when several SASE agents run it concurrently, with no test deleted, skipped, re-marked
  `slow`, or weakened, and with no increase in host CPU or memory pressure.

  '
phases:
- id: baseline
  title: Suite cost harness and committed baseline
  depends_on: []
  size: medium
  description: 'baseline: build the per-cause suite cost harness (idle vs CPU, app
    boots, subprocess, parser, collection, worker RSS) on top of the existing timings
    store, and commit the measured starting numbers every later phase is scored against.

    '
- id: idle
  title: Eliminate idle waiting in ACE TUI tests
  depends_on:
  - baseline
  size: large
  description: 'idle: replace the 20ms-granularity CPU-idle heuristic behind every
    `pilot.pause()` and every bounded waiter with event-driven barriers, so TUI tests
    stop spending over half their wall clock asleep.

    '
- id: boot
  title: Amortize ACE app startup across tests
  depends_on:
  - baseline
  - idle
  size: large
  description: 'boot: cut the cost of one ACE app boot and add a supported way for
    a group of tests to share one booted app, then migrate the heaviest TUI files
    onto it without weakening isolation.

    '
- id: overhead
  title: Cut cross-cutting per-test overhead outside the TUI
  depends_on:
  - baseline
  size: medium
  description: 'overhead: remove the repeated full-argparse parser builds, gettext
    lookups, YAML/config reparses, and avoidable CLI subprocess round-trips that the
    harness attributes across the non-TUI suite.

    '
- id: footprint
  title: Shrink worker memory and collection cost
  depends_on:
  - baseline
  size: medium
  description: 'footprint: reduce the per-worker collection time and the resident
    memory each xdist worker holds and grows, which is what currently caps how many
    workers the host can afford.

    '
- id: gate
  title: Fair worker allocation when agents run in parallel
  depends_on:
  - baseline
  - footprint
  size: medium
  description: 'gate: make the host-global worker-token pool split fairly between
    concurrent runs instead of granting the first arrival 28 tokens and every later
    arrival the floor of 4.

    '
- id: guard
  title: Lock in the win with a cost regression gate
  depends_on:
  - idle
  - boot
  - overhead
  - footprint
  - gate
  size: small
  description: 'guard: turn the harness into a standing regression gate with committed
    budgets, and document the new cost model for future contributors.'
proposed_by: bbugyi200.athena.wk
create_time: 2026-08-09 10:29:39
status: wip
bead_id: sase-ib
---

- **PROMPT:** [prompts/202608/fast_test_suite_1.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/fast_test_suite_1.md)
- **BEAD:** [sase-ib](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ib/README.md)

# Plan: Make `just test` fast under agent contention

## Why

`just test` is the command every agent in this repo runs before replying, and it is
currently slow in a way that compounds: it is slow on an idle host, and it degrades
catastrophically when a second agent runs it at the same time.

Measured on `athena` (64 cores, 64 GiB) at `957219ef2`:

| Measurement                                              | Value                                                                  |
| -------------------------------------------------------- | ---------------------------------------------------------------------- |
| `just test` wall, 4 worker tokens granted (host busy)    | **1007s (16:47)**                                                      |
| `just test` wall, 28 worker tokens granted (host idle)   | **220s (3:35)**                                                        |
| Tests run                                                | 27,978 passed, 10 skipped                                              |
| Total per-test seconds (host timing table, `mode: fast`) | **3,719s** across 2,470 test files                                     |
| Collection, single process                               | **27.6s** — paid again by _every_ xdist worker                         |
| Worker RSS                                               | **~0.5 GiB** after collection, growing to **0.7–1.1 GiB** by minute 13 |

The 1007s run above was not a pathological case. It is what happens whenever a second
agent is already testing: the suite gate hands the first arrival 28 of the ~32 host
tokens and the next arrival gets exactly the floor of 4, so the second agent's identical
run takes **4.6x longer** than the first's.

The 28-worker run also reproduced the known load-sensitive flake tracked as task
**sase-hk** (corroborated with this run's evidence while writing this plan):
`tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_offers_project_local_xprompts_by_canonical_name`
and its `..._directory_key_spelling_also_resolves` sibling failed as a pair on an empty
`_extra_prompts`, while the same file passes 17/17 in 2.33s in isolation. That is
precisely the failure mode this epic's wait changes must not multiply.

## Where the time actually goes

Every number below was measured in this workspace, not estimated.

**1. Over half of ACE TUI test wall clock is sleep, not work.** A serial run of
`tests/ace/tui/modals` used 42.0s of CPU across 91.8s of wall — **47% CPU utilization in
a single-process run**. The cause is a shared helper, not scattered test bugs:
`pilot.pause()` with no delay calls Textual's `wait_for_idle()`, which loops on
`await asyncio.sleep(1/50)` and only exits once a 20ms slice passes with under 1ms of
process CPU time. Every bare `pause()` therefore costs a **20ms floor of pure sleep**,
and every bounded waiter in `src/sase/ace/testing/wait.py` and `AcePage.expect_*` polls
by calling `pause()` in a loop. There are 2,373 static `.pause()` call sites and 1,904
bounded-waiter call sites in `tests/`. Under xdist the heuristic is also measuring the
wrong thing — a timesliced worker looks "idle" for reasons that have nothing to do with
the app being settled.

**2. Roughly a seventh of the whole suite is ACE app startup.** Instrumenting
`App.run_test` across `tests/ace/tui` (non-visual): **2,148 app boots consuming 422s of
1,171s of test call time — 36%**. `tests/ace/tui` is 1,381s of the 3,719s suite (37%),
so app boot alone is ~500s of the suite, and `AcePage.__aenter__` measured 390s over 506
entries (0.77s each) on top of the raw `run_test` cost. The archetype is
`tests/ace/tui/widgets/test_vim_normal_key_containment.py`: 19 parametrized cases,
31.7s, 1.3–1.8s each, each booting the entire app to press one key.

**3. Cross-cutting per-test overhead outside the TUI.** Profiling `tests/main` (1,223
tests, 75.8s): `create_parser()` is called 410 times at ~41ms each — **~22% of that
directory** — and 348,472 of the `gettext.find` filesystem probes it triggers are pure
waste (memoizing `gettext.find` alone halves `create_parser` to 21ms). A probe over a
1-in-8 stratified sample of the whole suite (3,623 tests) attributed 32.3s to 5,586
`subprocess.run` calls, 8.1s to YAML loads, and 7.5s to 113 parser builds; the
suite-wide extrapolations are ~260s, ~65s, and ~60s respectively. `tests/` has 512
`create_parser(` call sites and 234 files using `subprocess`.

**4. Fixed per-worker cost, paid 28 times.** Collection is 27.6s in a single process and
every xdist worker repeats it, so at 28 workers roughly 45s of wall clock elapses before
the first test runs. `import sase.ace.tui.app` alone is **1.53s**.

**5. The gate turns contention into a 5x penalty.** `tests/_suite_gate.py` computes a
host budget of `min(cpus - cpus/8, memory headroom / 950 MiB, 32)`, then
`automatic_worker_range` returns floor 4 and ceiling `min(28, budget - 4)`. The first
run greedily takes the ceiling. With a 32-token pool that leaves 4, which is exactly the
next run's floor. Observed live during this investigation: `sase_11` running at `-n 28`
while `sase_14` ran the identical command at `-n 4`. Worker RSS growing from 0.5 GiB to
over 1 GiB during a run is what keeps the memory-derived budget small in the first
place.

## Strategy

Two independent axes, both required:

- **Cost** — cut the 3,719 total per-test seconds (phases `idle`, `boot`, `overhead`).
- **Concurrency** — cut the fixed and per-worker cost that decides how many workers the
  host can afford, and split that capacity fairly (phases `footprint`, `gate`).

`idle` comes before `boot` deliberately: both touch the ACE test harness, the idle fix
is lower-risk and its win is larger, and doing it first means `boot` is measured against
an already-de-idled baseline rather than taking credit for the same seconds twice.

## Coordination with in-flight work

This epic overlaps two things that are already moving, and must not stomp on either.
Each phase below that touches them re-checks their state before it starts, because they
may well have landed by then.

- **Epic sase-h8** ("Retire the parallel-suite flake class") and its in-progress child
  **sase-h8.10** own the ACE wait idioms for _correctness_. Phase `sase-h8.2` created
  the single bounded-wait primitive in `src/sase/ace/testing/wait.py`, `sase-h8.8`
  committed a flake baseline gate, and `sase-h8.10` is still closing "wait-idiom gate
  gaps". The `idle` phase proposed here rewrites exactly that primitive's polling
  strategy. It must make the helper _faster without making it weaker_: coordinate with
  sase-h8's work rather than reverting it, keep the flake baseline gate passing, and
  treat any wait that sase-h8 deliberately strengthened as load-bearing until proven
  otherwise.
- **Task sase-hk** is the open diagnosis of the two `test_vcs_tag_*` nodes above. It is
  not this epic's job to fix, but the `idle` phase will be touching the same
  load-sensitivity surface and should report whether its changes made those nodes
  better, worse, or unchanged.
- **Epic sase-h8's own history is the cautionary tale for `gate`.** Its contention
  harness once ran `-n 64` with the gate disabled, holding zero tokens while consuming
  64 workers of memory against a 32-token pool; athena reached a load average of 97.60
  with 25 GiB in swap. The harness's demand was invisible to the pool, and the pool's
  arithmetic was correct the whole time. Any change to allocation in this epic must keep
  every run's demand visible to the pool.

## Constraints — what a phase may not do

These are hard requirements. A phase that hits its number by violating one of them has
not done the work.

- **No coverage loss.** No test is deleted, `skip`ped, `xfail`ed, or moved behind the
  `slow` or `visual` markers to get it out of `just test`. The collected-and-run node
  count for `just test` must not fall (27,978 passed + 10 skipped at `957219ef2`); if a
  refactor legitimately merges parametrized cases into one test, every original
  assertion and input must still execute, and the phase must say so explicitly in its
  report.
- **No assertion weakening.** Shortening a wait is only legitimate when the new barrier
  is at least as strong as the old one. Replacing a bounded waiter with a bare
  `pause()`, or lowering a timeout so a slow machine fails, is a regression disguised as
  a speedup.
- **No buying speed with host resources.** Raising `_DEFAULT_HARD_TOKEN_LIMIT`,
  `_DEFAULT_AUTOMATIC_CEILING`, or the automatic floor to make one run finish faster is
  explicitly out of bounds. The `gate` phase changes how the existing pool is _shared_,
  not how big it is; if it lowers the per-token memory reservation it must justify that
  from the `footprint` phase's measured RSS curve.
- **No new flakiness.** Every phase that touches waits or app lifetime must come back
  clean from `just test-contention` (the deliberate-starvation soak) and from the
  committed flake baseline gate sase-h8.8 landed, in addition to `just check-full`,
  because a wait that only works on an idle host is a flake the next contended run will
  find.
- **Respect the core boundary.** If a hot path turns out to be shared backend or domain
  behavior rather than test scaffolding, it belongs in `../sase-core/crates/sase_core`,
  not in a Python-side workaround here.
- **Measure, then change.** Every phase reports before/after numbers from the `baseline`
  phase's harness. "It felt faster" is not a result.

## baseline: Suite cost harness and committed baseline

Nothing else in this epic can be scored without this, so it lands first.

Build a cost-attribution harness on top of the machinery that already exists —
`tests/_test_selection_timings.py` already records per-test-file wall seconds into
`~/.sase/test-selection/<project>/timings/`, and `tools/run_pytest` already arms plugins
for the full lanes. Extend that pattern rather than inventing a parallel store.

Deliver:

- A pytest plugin (armed by an opt-in `tools/run_pytest` mode, e.g. `just test-cost`)
  that records, per run and per test file:
  - **CPU seconds vs wall seconds** per worker, so the idle share is a first class
    number rather than something a human infers from a profile.
  - Attributed counts and seconds for the causes this investigation found: Textual
    `App.run_test` enter/exit, `AcePage.__aenter__`/`__aexit__`,
    `textual.pilot.wait_for_idle`, `Pilot.pause` (split by `delay is None`),
    `subprocess.run`/`Popen`, `sase.main.parser.create_parser`, `gettext.find`, YAML
    loads, and `sase.config.core.load_merged_config`.
  - Per-worker **peak RSS** and **collection seconds**.
- A reporter (`tools/test_cost_report` or a `just` recipe) that prints the attribution
  table and the top-N files by each cause, and can diff two recordings so a later phase
  can show its delta directly.
- The committed baseline itself, checked into the repo as data (not prose) so `guard`
  has something to compare against: total per-test seconds, per-cause attribution,
  collection seconds, peak worker RSS, and `just test` wall clock at both a granted
  width of 28 and of 4.

Keep it cheap and off by default: the plugin must not be armed for the ordinary `fast`,
`cov`, or `scoped` lanes, because a probe that taxes the lane it measures is its own
regression.

Acceptance: `just test-cost` produces a stable attribution table on two consecutive
runs, the numbers reconcile with the totals in the timings store to within a few
percent, and `just check` is unaffected when the mode is not requested.

## idle: Eliminate idle waiting in ACE TUI tests

The largest and cleanest win, and it is concentrated in shared helpers rather than
spread across 400 test files.

Root cause, in order:

1. `Pilot.pause(None)` → `textual._wait.wait_for_idle(0)` → a loop of
   `await asyncio.sleep(1/50)` that exits only after a 20ms slice with under 1ms of
   process CPU. Minimum cost per call: 20ms of sleep. Maximum: 1s.
2. `src/sase/ace/testing/wait.py::wait_for`, `AcePage.expect_state`,
   `expect_screen_contains`, `expect_screen_not_contains`, and their siblings poll by
   awaiting `pilot.pause()` — so each polling iteration pays that same 20ms floor.

Work:

- Give the ACE harness its own settle barrier that is event-driven rather than
  CPU-clock-driven: drain the Textual message pump and any pending screen refresh, then
  return, without the fixed sleep. `Pilot.pause(0)` plus an explicit pump/screen barrier
  is the shape the codebase already endorses — `AcePage.pause`'s own docstring documents
  passing `0` when the caller has a semantic or frame-convergence barrier. Make that the
  default path, and keep the CPU-idle heuristic reachable for the handful of tests that
  genuinely need it.
- Rewrite the bounded waiters to poll on the new barrier, with a real sleep only as a
  backstop after several unproductive iterations, preserving the existing 5s timeouts
  and the existing failure messages exactly.
- Audit the 78 `time.sleep` and 45 `asyncio.sleep` call sites in `tests/` and replace
  fixed delays with the observable end state they are standing in for.
- Extend `tools/check_test_wait_helpers` — which already rejects hand-rolled
  `_wait_until` loops — to also reject new fixed-delay sleeps and new bare polling loops
  in `tests/`, so the win does not erode. Check first whether sase-h8.10 has already
  closed part of this gap.

Sized `large` because the settle barrier is a correctness contract for 8,425 TUI tests
and is jointly owned with an in-flight epic: getting it subtly wrong produces flakes
across the whole lane, so it deserves a planning handoff rather than a direct edit.

Acceptance: CPU utilization of a **serial** `tests/ace/tui/modals` run rises from the
measured 47% to at least 80%; the harness reports at least a 40% reduction in
`tests/ace/tui` wall clock; `just check-full`, `just test-contention`, and the sase-h8.8
flake baseline gate are all clean. Any test whose timeout had to change is called out
individually with the reason, and the phase reports the observed effect on the sase-hk
nodes.

## boot: Amortize ACE app startup across tests

2,148 app boots costing 422s of the 1,171s of call time in `tests/ace/tui`. Attack it
from both ends.

**Make one boot cheaper.** The profile of `tests/ace/tui/modals` +
`tests/ace/tui/actions` shows where a boot's CPU goes: `stylesheet.apply` 23.1s
cumulative with 1.4M CSS tokenizer calls (the app's CSS is re-tokenized per app
instance), compositor `render_update` 21.4s, and `Strip.__init__` 4.1s tottime.
Candidates, each to be justified by measurement rather than assumed:

- Cache the parsed/compiled stylesheet across app instances within a worker process,
  since it is derived from source that does not change during a run.
- Confirm animations are actually disabled under the test harness, and disable them at
  the harness level if not.
- Stop paying full compositor cost for tests that never assert on rendered output —
  `AcePage`'s default pilot size is `(120, 40)`; a smaller default for non-rendering
  tests, with the current size retained wherever screen content is asserted, cuts render
  work proportionally.
- Audit what `AcePage.__aenter__` does beyond `run_test` — it measured 0.77s against a
  ~0.2s raw `run_test` enter, so roughly three quarters of an `AcePage` boot is the
  harness's own patching and startup settling, not Textual.

**Boot less often.** Add a supported way for a group of related tests to share one
booted app — a session/module-scoped `AcePage` with an explicit, documented reset
between tests — and migrate the heaviest files the `baseline` harness identifies onto
it, starting with the parametrized key-containment families
(`test_vim_normal_key_containment.py` and its neighbours) where 19 boots exist only to
press 19 different keys.

Sharing an app trades isolation for speed, so it must be paid for:

- The reset between shared tests must be explicit and auditable (focus, modal stack,
  prompt state, selection, notification state), not "it seemed to work".
- Every assertion and every input from the original tests must still run.
- Add a lane that runs the migrated files one-app-per-test — reusing the existing opt-in
  lane pattern, and wired into CI rather than left to a human to remember — so
  cross-test leakage is caught rather than assumed absent.
- Sharing is opt-in per file. Do not convert the whole TUI suite.

This phase is sized `large` because the isolation model is a design decision worth
planning before implementing, not because the diff is big.

Acceptance: at least a 50% reduction in the harness's measured app-boot seconds for
`tests/ace/tui`; node count unchanged; the isolated lane passes; three consecutive
`just test-contention` runs show no new per-node failures.

## overhead: Cut cross-cutting per-test overhead outside the TUI

Smaller individually, but they are spread across the 2,300 non-TUI test files and they
are cheap to fix.

- **argparse.** `create_parser()` costs ~41ms per full build and is called from 512
  sites in `tests/`. `create_parser(only=...)` already exists and costs 7ms for a single
  command tree — production uses it via `parser_only_hint`. Give tests a shared helper
  that either reuses a cached parser or passes the `only=` hint the test already knows,
  and convert the call sites the harness ranks highest. Verify that a reused parser is
  genuinely reusable (argparse parsers carry mutable defaults) before caching one.
- **gettext.** Every argparse help string triggers a `gettext.find` filesystem probe;
  `tests/main` alone made 348,472 of them. Memoizing the lookup halved `create_parser`.
  Prefer a fix that is correct in production too rather than a test-only monkeypatch,
  and state which was chosen and why.
- **Subprocess round-trips.** 5,586 `subprocess.run` calls in a 1-in-8 sample (~260s
  suite-wide). Where a test spawns the `sase` CLI only to assert on stdout and an exit
  code, call the entry point in-process instead. Keep a deliberate, named set of
  real-subprocess tests for the process boundary itself — argv handling, exit codes,
  signal behavior — and say which tests those are.
- **Config and YAML.** `load_merged_config` was called 4,924 times in `tests/main`
  because the autouse `_clear_config_caches` fixture drops the cache before every test.
  Keep the isolation, but stop re-parsing identical bytes: a content-keyed parse cache
  below the config cache preserves the isolation semantics at a fraction of the cost.
  `markdown_print_width` / `sase_content_layout` shows the same shape and deserves the
  same look.

Acceptance: at least a 40% reduction in the harness's attributed seconds for these
causes combined, with no change to node count or to what any test asserts.

## footprint: Shrink worker memory and collection cost

This is the phase that decides how much parallelism the host can afford, which is what
makes the difference for concurrent agents.

- **Collection.** 27.6s per worker, repeated by all 28. Find what dominates it
  (`--collect-only` with import profiling; `import sase.ace.tui.app` alone is 1.53s) and
  reduce it — deferring heavy module-scope imports in the `sase` package and in test
  modules is the obvious lever. Target: under 15s.
- **Worker RSS.** Workers start at ~0.5 GiB after collection and reach 0.7–1.1 GiB after
  13 minutes. Find what accumulates — module-level caches, retained Textual apps,
  per-test fixtures that never release — and fix the growth, not the symptom. This is
  the number the gate's memory budget divides by, so every 100 MiB saved is real
  concurrency.
- Report the measured RSS curve (start, median, peak) so `gate` can set the per-token
  reservation from evidence instead of the current fixed 950 MiB.

Acceptance: collection under 15s per worker, peak worker RSS at or below 700 MiB, and
the harness's recorded numbers to prove both.

## gate: Fair worker allocation when agents run in parallel

Today's allocation is first-come-take-almost-everything: ceiling `min(28, budget - 4)`
against a 32-token pool leaves exactly the 4-token floor for the next arrival, which is
how an identical run became 1007s instead of ~200s.

Work:

- Make the automatic ceiling responsive to demand rather than fixed. A run should ask
  for a share of the pool that accounts for other holders and waiters — two concurrent
  full runs should land near 14 tokens each, not 28 and 4.
- Let an oversized lease give tokens back when another run is waiting, rather than
  making the newcomer wait out a 16-minute run at the floor. The existing
  `WorkerTokenLease` already grows greedily from a floor; shrinking is the mirror image
  and must be equally crash-safe.
- Re-derive `_MEMORY_KIB_PER_WORKER` from the RSS curve `footprint` measured. Only
  reduce it if the measurement supports it, and record the measurement in the comment
  the way the existing constants already do.
- Preserve every existing escape hatch (`SASE_TEST_GATE_SLOTS`,
  `SASE_PYTEST_WORKER_FLOOR`/`CEILING`, `SASE_TEST_GATE_DISABLED`) and the 4-vCPU CI
  behavior the current divisor exists to protect.

Explicitly **not** in scope: raising the hard token limit or the ceiling to make a
single run faster. Total host demand must not increase.

Acceptance: with two concurrent `just test` runs on an idle host, both complete within
~1.5x of a single uncontended run's wall clock, neither is starved to the floor, and
total host CPU and peak memory during the pair do not exceed today's
single-run-plus-floor-run peak. Include the two-agent experiment's numbers.

## guard: Lock in the win with a cost regression gate

A speedup that nothing defends decays. Close the loop:

- Record the suite cost summary from the `baseline` harness on the full lanes and commit
  budgets for the numbers that matter: total per-test seconds, attributed app-boot
  seconds, idle share, collection seconds, and peak worker RSS.
- Fail `just check-full` (and the CI leg) when a budget regresses beyond a tolerance
  wide enough to absorb host noise but narrow enough to catch a real regression — follow
  the tolerance style the visual snapshot lane already uses for local-vs-CI drift.
- Document the cost model and the new harness in `docs/development.md` and
  `docs/perf_runbook.md`: how to run `just test-cost`, how to read the attribution
  table, and which patterns (bare `pause()`, per-test app boots, full `create_parser()`
  builds, CLI subprocess round-trips) are now considered defects in new tests.
- Update the repo's own instructions if the recommended workflow changed. Memory files
  under `sase/memory/` may only be edited with the user's explicit permission in that
  agent's own conversation; if this phase concludes a memory update is warranted, it
  proposes the change rather than making it.

Acceptance: an artificial regression (e.g. reinstating a bare `pause()` in a hot helper)
is caught by the gate, and the docs let a contributor who has never seen this epic
diagnose a slow test on their own.

## Verification for the epic as a whole

The land agent confirms, on `athena`:

1. `just check-full` is clean.
2. `just test` node count is unchanged: 27,978 passed, 10 skipped (allowing for tests
   legitimately added in the meantime, which must be named).
3. `just test` wall clock on an idle host is at least 2x better than the committed 220s
   baseline.
4. Two concurrent `just test` runs each finish in well under the 1007s the contended
   baseline took, with neither run starved.
5. `just test-contention` shows no new per-node failures across its repeats.
6. Host peak memory and CPU during a concurrent pair are no higher than today.
