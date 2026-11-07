---
tier: epic
title: Turn every red sase workflow green
goal: 'On sase-org/sase, Master Gate, Full CI, CI Telemetry, and Publish are green
  at the master tip, and CI on the open release-please PR is green at its head. Each
  one is the latest run of its workflow, completed with success on a SHA at or after
  the last fix, with no in-scope job cancelled or skipped. Green evidence is published
  as ci_evidence. Otherwise, after a very large and documented effort, a research-sidecar
  report says exactly what is still needed. Then the land agent closes phase sase-1rq.2
  of epic sase-1rq.

  '
phases:
- id: master-gate
  title: Keep Master Gate green at the master tip
  depends_on: []
  size: medium
  description: 'master-gate: Master Gate is red at tip 0ba9cfc9 (run 42373447720).
    Its only failure is test (3) at tests/ace/tui/test_epic_panel_arrival_frames.py::test_selection_stays_highlighted_and_scrolled_into_view
    (''frame 11: row 14 outside scroll 2+12''). Prove whether it is deterministic
    or flaky on an unchanged tree. Root-cause it if you can; otherwise quarantine
    it with a flake follow-up. Then fix any other Master Gate red at the current tip.

    '
- id: coverage-contexts
  title: Stop the coverage-contexts lane crashing on global disk patches
  depends_on: []
  size: medium
  description: 'coverage-contexts: CI Telemetry''s coverage-contexts job dies in a
    pytest-xdist INTERNALERROR. test_inflight_handler_does_no_synchronous_disk_or_proc_store_work
    patches os.stat and pathlib.Path methods globally with a raising stub. Coverage''s
    ctrace tracer then calls os.path.exists while the patch is active and hits the
    assertion inside the tracer. Scope that guard so it only sees the code under test.
    Audit the other tests that globally patch os, os.path, or pathlib with raising
    stubs for the same hazard, and prove the cov-contexts lane passes on the affected
    modules.

    '
- id: test-cost
  title: Make the test-cost lane finish inside its 180-minute timeout
  depends_on: []
  size: large
  description: 'test-cost: CI Telemetry''s test-cost job is cancelled at its unchanged
    180-minute timeout, about 93% through 60,399 items on 3 workers (run 42347892210).
    Full CI''s test (3.12) runs the same items on the same 3 workers in about 61 minutes,
    so the cost lane''s own plugins (cost recorder plus --sase-detect-global-leaks
    / --sase-fail-on-global-leaks) add roughly 3x overhead. Find and remove that overhead.
    Then make the whole recipe pass: the run, test_cost_report, and check_test_cost_budgets.
    Change no timeouts, budgets, or leak gates.

    '
- id: contention
  title: Make the contention-test lane survive and pass
  depends_on: []
  size: large
  description: 'contention: CI Telemetry''s contention-test job (SASE_CONTENTION_REPEAT=1
    just test-contention, 26 workers on 2 pinned CPUs) prints nothing for about 20
    minutes. Then the runner gets a shutdown signal, on both the 06:54 and 13:48 UTC
    runs on 2026-11-07. The lane has never produced a tally in CI. Find out why the
    runner dies (memory or disk exhaustion is the lead) and fix it without changing
    the width, repeat count, CPU pinning, or timeout. Then make the single repeat
    pass: root-cause each failure under contention, or quarantine proven flakes.

    '
- id: full-ci
  title: Turn Full CI green alongside the sase-1r2.6 repair
  depends_on: []
  size: large
  description: 'full-ci: Full CI on d2aa3ef8 (run 42370668331) failed in visual-test
    (107 first-pass drifts over the 25-node recapture cap) and test (3.12) (test_trailing_reserve_remeasures_when_flag_already_enabled
    wait_for timeout, task bead sase-1qd). Phase sase-1r2.6 of epic sase-1r2 is live
    and recapturing the bulk golden drift. Coordinate with it rather than duplicating
    it. Own whatever it leaves: the CI-only visual drifts it recorded as follow-ups,
    the sase-1qd flake, and any other red. Repeat until a dispatched Full CI is green
    on its first attempt.

    '
- id: release-path
  title: Clear CORE_LAG from Publish and the release PR floor smoke
  depends_on: []
  size: small
  description: 'release-path: Publish''s sync-release-metadata (run 42375016682) and
    the release PR #322 CI''s release-core-floor-smoke (run 42375244107) both failed
    on CORE_LAG with floor 0.43.3. sase-core-rs 0.44.0, which contains sase''s pin
    c8adc878, has been on PyPI since 2026-11-07 14:58 UTC. Confirm that a Publish
    run after that time ratchets the release branch''s floor and goes green, and that
    the floor smoke then passes. Fix the ratchet or smoke tooling only if it still
    fails.

    '
- id: verify
  title: Confirm every in-scope workflow green and publish evidence
  depends_on:
  - master-gate
  - coverage-contexts
  - test-cost
  - contention
  - full-ci
  - release-path
  size: medium
  description: 'verify: with every phase''s fixes on master, dispatch or await Full
    CI, CI Telemetry, and Publish on the tip. Watch Master Gate and the release PR''s
    CI. Repair any residual red across monitor turns. Then publish ci_evidence plus
    the Markdown table, or the research report under the parent contract''s very-large-effort
    rule.'
proposed_by: bbugyi200.athena.sase-1rq.2
proposed_with: claude/opus
parent_bead: sase-1rq.2
create_time: 2026-11-07 10:24:43
status: wip
bead_id: sase-1rq.2.1
---

- **PROMPT:** [prompts/202611/sase_ci_green.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/sase_ci_green.md)
- **PARENT:** [202611/red_ci_repair.md](https://github.com/sase-org/sase--plans/blob/main/202611/red_ci_repair.md)
- **BEAD:** [sase-1rq.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1rq/sase-1rq.2.1.md)

# Plan: Turn every red sase workflow green

This is the child epic of phase `sase-1rq.2` ("Turn every red sase workflow green") of
epic `sase-1rq`, `plan:202611/red_ci_repair.md`. Read that parent plan's **Contract
Shared By Every Phase** before starting. Its scope rule, working rules, landing loop,
evidence format, and research-report fallback all apply here unchanged. The parent
epic's DECISIONS are final:

- `test_quarantine = quarantine_flakes`: a test may be skipped only when it is proven
  flaky on an unchanged tree. Timeouts, job lists, lint levels, budgets, and CI gate
  parameters stay exactly as they are.
- `release_merge = gate`: if a phase needs a release PR merged so CI can go green, it
  waits for that PR's CI to pass, then proposes the exact `gh pr merge …` command
  through `/sase_gate`. Merging sase's own release PR #322 is **not** required here, and
  no phase may dispatch `release.yml`.

Repository: the sase primary repo, which is this project's own workspace checkout. Its
GitHub repo is `sase-org/sase` and its default branch is `master`. Before landing
anything, read the `lint_and_test` and `tui` reference memory (and `tui_screenshot` if
you touch visual snapshots) with `/sase_memory_read`, plus the `ci-two-speed-split` and
`check-full-is-explicit` decision strands.

## What The Planner Verified (2026-11-07, about 15:15 UTC)

Master tip was `0ba9cfc9`. Latest run of each in-scope workflow:

| Workflow                           | Run           | SHA        | Result                                                                            |
| ---------------------------------- | ------------- | ---------- | --------------------------------------------------------------------------------- |
| Master Gate (push)                 | `42373447720` | `0ba9cfc9` | failure: `test (3)` only, 1 failed / 7888 passed                                  |
| Master Gate (push)                 | `42373349936` | `b1329de1` | success                                                                           |
| Full CI (dispatch)                 | `42370668331` | `d2aa3ef8` | failure: `visual-test`, `test (3.12)`; 3.13 and 3.14 green                        |
| CI Telemetry (schedule)            | `42377692789` | `0ba9cfc9` | in progress: `contention-test` already failed                                     |
| CI Telemetry (schedule)            | `42347892210` | `8dce35a8` | failure: coverage-contexts INTERNALERROR, contention killed, test-cost cancelled  |
| Publish (schedule)                 | `42375016682` | `0ba9cfc9` | failure: `sync-release-metadata` exit 3, `CORE_LAG(pin=c8adc878…, newest=0.43.3)` |
| CI on release PR #322 (`2ce544ee`) | `42375244107` | `2ce544ee` | failure: `release-core-floor-smoke`; every other job skipped behind it            |

Facts per workstream:

- **Master Gate.** `0ba9cfc9` only changes axe reviewer-decision passing and VCS error
  refs. The failing test file has not changed since 2026-09-22, and the previous push
  was green. So the failure is probably timing-dependent, but that is unproven. The same
  test family (`epic_panel_arrival_frames`) appears in older flake triage notes on
  `sase-142.5` and `sase-165`.
- **Full CI visual-test.** It has been red on every Full CI run since at least
  2026-11-03, with 107–118 golden updates each time. The last fully green Full CI was
  `40189680772` (2026-10-25). Phase `sase-1r2.6` of epic `sase-1r2` ("One Full CI run on
  master is green in every job") is `in_progress`, and its agent (`sase-1r2.6--1`) was
  live at planning time. Its notes say the bulk drift is the expected `sase-1r2.4`
  conditional AGENT PROMPT heading removal, and that it is recapturing those goldens. It
  recorded three CI-only drifts as follow-ups it is not fixing:
  - `renamed_generic_session_root` scroll drift, a product bug tracked by open task bead
    `sase-1ln`.
  - `stranded_resolve_modal` layout drift (jump badges, doubled border), a focus/timing
    flake with no bead.
  - `artifacts_beads_empty`, a 389 px drift near the inbox label, unidentified.
- **Full CI test (3.12).** The one failure was
  `tests/ace/tui/widgets/decks/test_files_anchor_ownership.py::test_trailing_reserve_remeasures_when_flag_already_enabled`
  (`wait_for` timed out after 5.0 s). It is tracked by READY task bead `sase-1qd`
  ("times out in wait_for under the loaded check lane, passes focused"), which has no
  owner. The parent plan's other test leads from `8dce35a` (isolation guards, proc
  producer inventory) were fixed by `d2aa3ef8`.
- **CI Telemetry** has not been green in its last 100 runs. It runs every 6 hours
  (`47 */6 * * *`) under concurrency group `ci-telemetry`, which does not cancel in
  progress, and accepts `workflow_dispatch`. It has `contents: read` and publishes
  nothing, so dispatching it is safe. Its three jobs fail for three independent reasons:
  - `coverage-contexts` (`just test-contexts`, Python 3.12, `COVERAGE_CORE=ctrace`, 3
    workers, master only). The job dies after about 54 minutes. Traceback: coverage's
    `_should_trace` → `inorout.set_matchers_depending_on_syspath` → `os.path.exists` →
    `fail_disk` in `tests/ace/tui/test_kill_and_edit_last_launch_dispatch.py:281`. That
    test monkeypatches `os.stat`, `os.listdir`, `pathlib.Path.exists`,
    `pathlib.Path.is_dir`, and `pathlib.Path.read_text` process-wide. The
    `AssertionError` is raised inside the tracer, and xdist then aborts with
    INTERNALERROR, so no `.coverage` artifact is uploaded. About 37 test files use
    similar global `os` / `os.path` / `pathlib` / `builtins.open` patches. Most are
    probably harmless, but any that raise can do the same thing.
  - `test-cost` (`just test-cost`, Python 3.13): 3 workers, 60,399 items. It was at 93%
    when the 180-minute timeout cancelled it. The lane adds `-p` for the test-cost
    plugin, `--sase-detect-global-leaks`, and `--sase-fail-on-global-leaks` (see
    `_test_cost_pytest_args` and `_global_state_leak_detector_pytest_args` in
    `tools/run_pytest`). Full CI's `test (3.12)` coverage leg runs 60,402 items on 3
    workers in about 61 minutes. After the run, the recipe also runs
    `tools/test_cost_report` and `tools/check_test_cost_budgets`, and nobody has seen
    those steps' result in CI recently.
  - `contention-test` (`SASE_CONTENTION_REPEAT=1 just test-contention`, Python 3.12
    visual venv). It runs `taskset -c 0,1` with 26 xdist workers
    (`DEFAULT_CONTENTION_WORKERS`, a 13x oversubscription copied from the visual
    contention lane). On both 2026-11-07 runs the log goes silent after worker startup.
    About 20 minutes later it ends with `Received SIGTERM - agent was killed` and "The
    runner has received a shutdown signal", the signature of a hosted runner dying,
    usually from memory exhaustion. The harness returns the first non-zero pytest exit,
    so a green lane also needs zero failures under contention.
- **Publish and the release PR.** The `CORE_LAG` blocker is gone: sase-core phase
  `sase-1rq.1` shipped `sase-core-rs` 0.44.0 (all five files on PyPI by 14:58 UTC), and
  pin `c8adc878` is an ancestor of `v0.44.0`. Core Pin Ratchet PR #319 proposes moving
  the pin to `dec432ca`, which is also inside `v0.44.0`. Publish runs at `17 */3 * * *`
  and accepts `workflow_dispatch`. Release-please only opens or refreshes the release
  PR, or creates a GitHub release for an already-merged release PR, and
  `sync-release-metadata` only pushes floor and lock updates to
  `release-please--branches--master`. Nothing in `publish.yml` uploads to PyPI;
  `release.yml` does that. On the release PR, `release-core-floor-smoke` gates every
  other `ci.yml` job, so once it passes, the PR's CI runs the same lint, test, and
  visual-test lanes as Full CI. The release PR therefore also depends on the `full-ci`
  phase.

## Rules For Every Phase

- **Root-cause fixes first.** A test may be skipped only when it is proven flaky on an
  unchanged tree. "Proven" means failing and then passing with no code change, under the
  same conditions. Use the test area's existing skip style, and put the flake in the
  skip reason. Record
  `sase bead note <your-bead> 'PROPOSED FOLLOW-UP: flake — <nodeid> — <evidence>'` so
  the land agent can file the `flake` bead. If a READY bead already tracks the test (for
  example `sase-1qd`), cite it in the reason and the note instead. Phase workers never
  create beads. Never skip a deterministic failure.
- Never change workflow `timeout-minutes`, job lists, lint levels, test-cost budgets,
  the global-leak gate, the contention width, repeat count, or CPU pinning, the
  visual-test recapture cap or pixel-exact comparison, repository variables or secrets,
  branch protection, or workflow permissions. Changing those is gate relaxation, which
  the `quarantine_flakes` decision forbids. Adding a purely diagnostic step to a
  workflow, such as a memory sampler or `if: always()` artifact upload, is allowed when
  it changes no outcome. Say so explicitly in your final response.
- Shared backend behavior belongs in `sase-core` (see the Rust Core Backend Boundary
  note). If a fix needs core changes, open it with `sase repo open sase-core`, and
  remember sase's `sase-core-revision.txt` pin must not move past a commit that no
  published release contains, or CORE_LAG returns.
- Verify with `sase tool run check` before landing. `just check-full` is explicitly
  authorized for this epic, but run it only through `/sase_monitor` as `lint_and_test`
  describes. Its local screenshot-update stage can rewrite goldens, so never land golden
  changes that belong to `sase-1r2.6`'s live recapture (see the `full-ci` phase). Prefer
  targeted lanes: `tools/run_pytest <mode> <paths>` and
  `just fix-tui-screenshots --check -- <selectors>`.
- **Host capacity.** These phases run in parallel on athena, and the full suite can take
  a quarter to half of the machine. Run any whole-suite local lane (`test-cost`,
  `test-contexts`, `test-contention`, full visual) only under `/sase_monitor` with a
  realistic weight, at most one per phase at a time. Validate on subsets first.
- Land through host completion only, by declaring a commit for the sase repo in
  `/sase_final`. Agents never commit or push. To watch CI on landed work, start a
  zero-weight `/sase_monitor` (`-w 0`, labels `WATCHING CI` → `WATCHED CI`) with a
  `--next` that tells the follow-up turn to read the result and either iterate or
  finish. Never end a turn waiting.
- Other agents land on master constantly. If an unrelated commit turns the tip red for a
  new reason, fix it when the fix is quick. Otherwise record
  `PROPOSED FOLLOW-UP: ci — <job> — <run url> — <first failing test>` and move on.
- If a phase finds more independent failure families than one agent can finish, land the
  strict improvements you have. Then record the remainder as one `PROPOSED FOLLOW-UP:`
  per family, with node ids, run links, and leads, and close the phase. The `verify`
  phase and the land agent pick up from those notes.

## Phase: master-gate

1. Read the newest Master Gate runs:
   `gh run list --workflow master-gate.yml --branch master -L 10`. If a newer push has
   gone green, still investigate `0ba9cfc9`'s failure: a flaky shard will redden a later
   push.
2. Reproduce on an unchanged tree:
   - First focused, with
     `tools/run_pytest fast tests/ace/tui/test_epic_panel_arrival_frames.py` repeated
     about 30 times.
   - Then under load, with the whole module alongside the other `tests/ace/tui` modules
     at Master Gate's shard width, or under `taskset -c 0,1` to mimic the 2-CPU runner.
   - Record how many runs failed.
3. Read the test, `frames.selection_violations`, and the arrival-frame scenario fixture.
   The violation is "selected row 14 outside scroll window 2+12" in frame 11: the
   selection moved, but the scroll-into-view landed a frame late. Decide whether
   production paints a frame with the selection off-screen (a real bug: fix the ordering
   so scroll and selection settle in the same frame, as `8464f4bc` did for removal
   collapses) or the test samples an intermediate frame it shouldn't assert on. Fix the
   real cause. Quarantine only if it is proven flaky and no root cause is reachable.
4. Run `sase tool run check`, land, and then watch the Master Gate push run for your
   commit. Green means that run and every Master Gate push run after it, up to your
   final turn, is `success`. If another agent's push reddens it for a new reason, apply
   the scope rule.

## Phase: coverage-contexts

1. Read the result of the in-progress CI Telemetry run `42377692789`'s
   `coverage-contexts` job, or of any newer run. Note every failure in addition to the
   known crash.
2. Fix `test_inflight_handler_does_no_synchronous_disk_or_proc_store_work`. Keep its
   intent: the in-flight `,X` handler must do no synchronous disk or proc-store work.
   Make the guard see only the code under test, for example:
   - patch the names the handler's modules actually import, not process-wide `os.stat`
     and `pathlib.Path`; or
   - inject a proc-store and filesystem seam; or
   - record calls and assert afterwards, instead of raising from inside arbitrary
     callers. The guard must still fail if the handler starts touching disk, so prove
     that by temporarily adding a disk read to the handler.
3. Audit the other tests that monkeypatch `os.*`, `os.path.*`, `pathlib.Path.*`, or
   `builtins.open` process-wide with a stub that raises. Fix every one that can fire
   inside coverage's tracer, or inside pytest/xdist machinery, the same way. Leave
   harmless ones (pure return-value fakes) alone.
4. Prove it with the real lane on the affected modules:
   `tools/run_pytest cov-contexts <the fixed test files>`, with `COVERAGE_CORE=ctrace`
   as the lane sets it. Show that it finishes without INTERNALERROR and writes
   `.coverage`. Then run `tools/run_pytest cov-contexts tests/ace/tui` under
   `/sase_monitor` to catch neighbours. A whole-suite `just test-contexts` is optional
   and must be monitored.
5. Run `sase tool run check` and land. You need not wait for CI Telemetry here; the
   `verify` phase dispatches it. If you do watch a scheduled run, record the result.

## Phase: test-cost

1. Read the in-progress CI Telemetry run `42377692789`'s `test-cost` job, or a newer
   one, for its progress rate and any failure.
2. Measure locally on a representative subset of a few thousand items (for example
   `tests/ace/tui` plus `tests/history`), at the same width, under four configurations:
   plain `fast`; `cost` with the test-cost plugin only; with the global-leak detector
   only; and with both. Find which hook dominates per test: snapshotting global state,
   walking `sys.modules`, writing per-test records, fsync, and so on. Use `py-spy` or
   cProfile on one worker if it isn't obvious.
3. Remove the overhead without weakening what the lane measures or enforces. Examples:
   snapshot less and cheaper state, compare by identity or version counters before deep
   comparison, batch record writes, and do one-time work once per worker. The leak
   detector must still fail a test that really leaks global state, so keep or extend its
   tests.
4. Target: the whole lane finishes comfortably inside 180 minutes on the 4-vCPU runner.
   Aim for under about 120 minutes, extrapolating from your local measurements against
   the 61-minute Full CI baseline. Then make the recipe's tail pass:
   - `tools/test_cost_report`
   - `tools/check_test_cost_budgets`
   - the `--sase-fail-on-global-leaks` gate

   A budget breach or a real leak is a finding: fix the test or code behind it. Never
   raise a budget or disable the gate. If a breach comes only from the suite's
   legitimate growth, and fixing it is beyond reach, record it as a
   `PROPOSED FOLLOW-UP:` with the numbers and leave the budget alone.

5. Optionally run a whole `just test-cost` locally under `/sase_monitor` (heavy; see
   Rules). Run `sase tool run check` and land.

## Phase: contention

1. Read the in-progress CI Telemetry run `42377692789`'s `contention-test` log and the
   `default-lane-contention-records` artifact, if any.
2. Find why the runner dies. Reproduce locally with `taskset -c 0,1` and 26 workers
   (`SASE_CONTENTION_REPEAT=1 just test-contention`, under `/sase_monitor`). Sample
   total RSS, per-worker RSS, `/tmp` and workspace disk usage, and process counts over
   time. A hosted `ubuntu-latest` runner has 4 vCPUs, about 16 GB RAM, and limited disk.
   Look for:
   - per-worker memory far above the default lane (visual venv imports, Textual apps not
     torn down, leaked subprocesses or threads that outlive tests under starvation);
   - `/tmp` or pytest basetemp growth;
   - runaway child processes when timeouts fire under contention.

   If local evidence is ambiguous, you may land a purely diagnostic step in
   `telemetry.yml` that samples memory and disk to the job log or an `if: always()`
   artifact. Then confirm with one `gh workflow run telemetry.yml --ref master`. The run
   takes hours, so watch it through a zero-weight monitor.

3. Fix the cause in the harness or the tests, keeping the 26-worker, 2-CPU, one-repeat
   shape and the 90-minute timeout. The repeat must finish well inside that timeout.
4. Once the lane produces a tally, the repeat must also pass. For each failing node,
   decide whether it is a real timing assumption, such as a fixed sleep, a short
   `wait_for`, or ordering across threads. Fix those to be timing-independent. A node
   that fails under contention and passes on rerun with no code change is a proven
   flake: quarantine it under the Rules, or fix it if that is cheap. Coordinate with the
   `full-ci` phase on shared nodes such as `sase-1qd`'s
   `test_trailing_reserve_remeasures_when_flag_already_enabled`. Whichever phase lands
   first owns the fix, and the other cites it.
5. This phase may surface many failure families. Follow the Rules' remainder clause
   rather than overrunning one agent.

## Phase: full-ci

1. Coordinate first. Run `sase bead read sase-1r2.6 -r "<why>"` and `sase agent list`,
   and read `gh run list --workflow full.yml -L 5`.
   - While `sase-1r2.6` is open with a live agent, do **not** recapture, update, or land
     any golden in the bulk AGENT-PROMPT-heading family it owns. Never run
     `just fix-tui-screenshots` in update mode over the whole suite.
   - Own exactly what it is not doing: the three CI-only visual drifts listed above,
     `sase-1qd`'s test, and any Full CI red it hasn't claimed.
   - Before starting, append a coordination note to `sase-1r2.6`
     (`sase bead note sase-1r2.6 'COORDINATION: sase-1rq.2 full-ci phase is fixing <items>; it will not touch <family> goldens'`)
     so its agent sees the split.
   - If `sase-1r2.6` is closed, or has no live agent and no progress for over 2 hours,
     take over its remaining Full CI work. Record that takeover in the same kind of
     note.
2. For each CI-only visual drift: compare the CI actual against the golden using the
   failure report artifact (`ace-visual-failure-report`) from the run, and against a
   local capture. A CI-only difference means the snapshot is nondeterministic: focus,
   timing, counts, or environment leaking into the render. Make the capture
   deterministic, which is the root-cause fix. For `renamed_generic_session_root`, read
   task bead `sase-1ln` first. Fixing that product bug is in scope if the fix is
   bounded; otherwise make the snapshot wait for the settled scroll state and cite
   `sase-1ln`. Re-verify with `just fix-tui-screenshots --check -- <selectors>`.
3. For `sase-1qd`: read the bead and reproduce under load (for example `taskset -c 0,1`
   with several workers). `wait_for` timing out at 5 s while the test passes focused
   usually means the test waits on a condition that needs an extra refresh tick, or the
   remeasure is scheduled behind slow work. Fix the trigger, so the test does not just
   wait longer. Quarantine only under the Rules.
4. Land. Once the tip contains your fixes and `sase-1r2.6`'s landed goldens, dispatch
   `gh workflow run full.yml --ref master` (concurrency group `full-ci` never cancels).
   Watch it through a zero-weight monitor, and iterate on any red across turns. Green
   means a Full CI run on a SHA at or after the last fix, with all twelve jobs
   (`prepare`, `build-core`, `lint`, `perf-floors`, `test (3.12/3.13/3.14)`,
   `visual-test`, `ace-page-group-isolation`, `tutorial-tests`, plus `docs-build` and
   `release-core-floor-smoke` skipped as designed on master) `success` on the first
   attempt.
5. If `sase-1r2.6` gets there first, with a green first-attempt Full CI on a SHA at or
   after your last fix, that run counts. Record it and finish.

## Phase: release-path

1. Check that release PR #322 is still open and unmerged: `gh pr view 322`. If the
   release steward (epic `sase-1r2`) has merged it, do not dispatch Publish. Read the
   resulting release and any new release PR, and record the state.
2. Read the newest Publish run: `gh run list --workflow publish.yml -L 3`. A scheduled
   run fires at `:17` every 3 hours. If no run has started after 2026-11-07 14:58 UTC,
   dispatch one with `gh workflow run publish.yml --ref master`. That is safe under
   `release_merge = gate`, because it publishes nothing to PyPI and #322 is unmerged.
   Watch it with a zero-weight monitor.
3. Expect `sync-release-metadata` to ratchet the floor to `sase-core-rs` 0.44.0 and push
   `chore: sync release metadata` to `release-please--branches--master`. That push
   triggers CI on #322. Confirm that `release-core-floor-smoke` passes there, and that
   `Publish` concludes `success`.
4. If CORE_LAG persists, find out why. Possible causes: `ratchet_core_pin` not seeing
   0.44.0 as complete (wheel-tag matching, PyPI simple-index lag), or the pin having
   moved past `v0.44.0` (for example PR #319, or a newer core commit). Fix sase-side
   tooling bugs here. If sase genuinely needs a newer core release, the core release PR
   must be merged under `release_merge = gate`: wait for its CI to pass, then propose
   the exact `gh pr merge …` through `/sase_gate`.
5. The other jobs on #322 (lint, test, visual-test, and so on) mirror Full CI. They are
   the `full-ci` phase's to fix, and the `verify` phase confirms them. Record their
   state.

## Phase: verify

1. Fetch `origin/master` and confirm every phase's fix commit is on it. Read each
   phase's notes, including every `PROPOSED FOLLOW-UP:` and any remainder families.
2. Read the newest run of each in-scope workflow. Then make sure each has a run on a SHA
   at or after the last fix:
   - Dispatch `full.yml` and `telemetry.yml` with `gh workflow run … --ref master`,
     unless a qualifying run is already in progress. Telemetry queues behind any running
     telemetry run and takes hours.
   - Dispatch `publish.yml` under the same check as `release-path` step 1.
   - The next master push runs Master Gate. The release PR's CI reruns when Publish
     refreshes its branch.
   - Watch them through zero-weight monitors with generous timeouts, about 5 h for
     telemetry.
3. Fix residual reds across monitor turns under the Rules, landing and redispatching as
   needed. Apply the scope rule to other agents' regressions.
4. Green means the parent contract's definition holds for Master Gate, Full CI, CI
   Telemetry (all three of `test-cost`, `coverage-contexts`, `contention-test`, plus
   `build-core`), and Publish at the master tip, and for CI on the open release-please
   PR at its head (every job `success`, or skipped only by design). Note first attempt
   versus rerun. A green that needed a rerun needs a `PROPOSED FOLLOW-UP: flake — …` per
   test.
5. In the confirming turn, publish `ci_evidence` through `/sase_var` with the parent
   contract's fields:
   - `repo` `sase-org/sase` and `default_branch` `master`
   - `tip_sha`
   - `runs`: one entry per workflow, with `workflow`, `run_id`, `url`, `head_sha`,
     `event`, `attempt`, `conclusion`, `completed_at`. The release PR's `CI` entry also
     names the PR number and head.
   - `fixes`: landed SHAs with one-line summaries from every phase of this epic, plus
     `sase-1r2.6`'s goldens if they were part of the green
   - `quarantined`: tests and their flake follow-ups or bead ids
   - `notes`

   Repeat it as a Markdown table in the final response, with the reproducing commands
   (`gh run list --workflow <file> --branch master -L 3` per workflow, and
   `gh pr checks <release PR>`).

6. Use the research-report fallback only under the parent contract's very-large-effort
   rule. The report goes in `sase-org/sase--research`, opened with `/sase_repo`, and its
   path and `@research:` reference are published as `ci_report`. Still land every strict
   improvement first.

## Land Agent

The land agent for this epic also finishes parent phase `sase-1rq.2`:

1. Confirm every phase bead in this epic is closed. Re-read the `verify` phase's
   evidence or report.
2. Refresh the evidence (`gh run list` for `master-gate.yml`, `full.yml`,
   `telemetry.yml`, and `publish.yml` on master, plus `gh pr checks` on the open
   release-please PR). If a newer run is red for an in-scope reason, do not close
   `sase-1rq.2`; record what remains as a note on it instead. Otherwise republish
   `ci_evidence` (or `ci_report`) through `/sase_var` in the land turn, so the turn that
   completes `sase-1rq.2` carries it.
3. Triage every `PROPOSED FOLLOW-UP:` note from this epic's phases into task beads
   through `/sase_new_task`, with one `flake` bead per quarantined test. Then make sure
   each quarantine's skip reason names its bead id. If the ids weren't known when the
   skip landed, landing that reason edit is part of the land commit.
4. Run `sase bead epic-symbols sase-1rq.2` and resolve any leftovers. Then close
   `sase-1rq.2` with
   `sase bead close sase-1rq.2 --note "<tip SHA, run ids per workflow, evidence summary or report ref>"`.
5. Never close epic `sase-1rq`, epic `sase-1r2`, or any of their phases other than
   `sase-1rq.2`.
