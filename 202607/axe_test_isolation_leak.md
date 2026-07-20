---
tier: epic
title: Fix pytest leakage that bricks the real axe daemon and strands WAITING agents
goal: 'The sase test suite can never start, stop, or take over the user''s real axe
  daemon; axe self-heals from a wedged lifecycle lock; and waiting agents unblock
  when their dependencies complete even if the wait_checks chop is down.

  '
phases:
- id: live-paths
  title: Live axe state path resolution
  depends_on: []
  size: medium
  description: '''Live axe state path resolution'' section: replace import-time axe
    state-dir constants with live path functions so the autouse test-home isolation
    actually covers axe lifecycle state, and migrate all src and test call sites.'
- id: test-guard
  title: Pytest lifecycle guard and daemon env hygiene
  depends_on:
  - live-paths
  size: small
  description: '''Pytest lifecycle guard and daemon env hygiene'' section: refuse
    axe daemon start/stop/restart under pytest without an explicit override, and scrub
    pytest variables from the spawned daemon environment.'
- id: self-heal
  title: Wedged lifecycle-lock recovery in axe healing
  depends_on:
  - test-guard
  size: medium
  description: '''Wedged lifecycle-lock recovery in axe healing'' section: detect
    a lifecycle-lock holder that never publishes an orchestrator PID, terminate it
    after a grace period, and surface ensure failures as notifications instead of
    silent no-ops.'
- id: runner-fallback
  title: Waiting-runner fallback dependency resolution
  depends_on: []
  size: medium
  description: '''Waiting-runner fallback dependency resolution'' section: make blocked
    agent runners periodically re-resolve their wait dependencies directly so completed
    dependencies unblock agents even when the wait_checks chop is not running.'
- id: smoke
  title: Cross-layer regression exercises
  depends_on:
  - self-heal
  - runner-fallback
  size: medium
  description: '''Cross-layer regression exercises'' section: end-to-end tests that
    simulate the leaked-orchestrator incident and prove isolation, guard, recovery,
    and fallback resolution each hold.'
create_time: 2026-07-19 21:56:50
status: wip
---

# Plan: Fix pytest leakage that bricks the real axe daemon and strands WAITING agents

## Context and root-cause diagnosis

The reported symptom was an agent (`sase-7z.6`) stuck in WAITING even though its `%w:` wait target (`sase-7z.2`) had
completed: `sase-7z.2` had `done.json` with `outcome: "completed"`, `sase-7z.6/waiting.json` listed exactly that
dependency, yet no `ready.json` was ever written. The `wait_checks` chop resolution logic itself is correct — a manual
walk of `dependency_resolution_status` over the on-disk artifacts resolves the dependency. The chop simply never ran:
the real `waits` lumberjack received a shutdown signal at 21:32:38 (before `sase-7z.2` completed at 21:36:40) and never
came back.

Investigation of the live host found the actual root cause, observed **reproducing live**:

1. `~/.sase/axe/orchestrator.lock` was held by a `sase axe start` orchestrator whose environment contained
   `HOME=/tmp/pytest-of-bryan/pytest-*/popen-gw*/home*`, `PYTEST_CURRENT_TEST=...`, and `PYTEST_XDIST_WORKER=gw*`, with
   its cwd pointing at a deleted pytest tmpdir. Each `just test` run (e.g. from fix-hook agents) produced a fresh
   imposter; `~/.sase/axe/logs/axe.log` accumulated hundreds of `Lumberjack '...' exited (code 1), restarting...`
   crash-loop lines plus `Failed to notify about crash loop ... /tmp/pytest-of-bryan/.../.sase/notifications`.
2. The imposter runs its lumberjacks against the pytest fake home (which gets deleted mid flight) while holding the
   **real** lifecycle lock via the fd handed off by `start_axe_daemon` (`SASE_AXE_LIFECYCLE_LOCK_FD`).
3. Real healing is impossible in this state: waiting runners call `_opportunistic_ensure_axe()` → `ensure_axe()` →
   `start_axe_daemon_result()`, which finds the lock held but no live published orchestrator PID and returns status
   `"blocked"` ("Run `sase axe stop`..."). `ensure_axe` reports `failed`; the runner swallows it and keeps sleeping. The
   system stays bricked until a human intervenes.

Why the test suite can touch the real axe at all:

- `src/sase/axe/state.py` computes `AXE_STATE_DIR = sase_subdir("axe")` (plus `JACK_STATE_DIR`, `SHARED_STATE_DIR`,
  `AXE_OUTPUT_LOG`) at **import time**. In a pytest-xdist worker, imports happen at collection with the real `HOME`, so
  these constants freeze to `/home/<user>/.sase/axe` for the life of the worker.
- The autouse `_isolate_sase_home` fixture in `tests/conftest.py` redirects `~/.sase` expansion per test, but that only
  affects code that calls `sase_home()` at call time. Everything routed through the frozen constants — the lifecycle
  lock path, `desired_state.json`, pid files, lumberjack state — escapes isolation.
- The ACE TUI defaults `auto_start_axe=True`; `_run_axe_startup_init` starts axe when the probed status says it is not
  running, and axe toggle keybindings (`!x`) can stop it. TUI e2e tests that run the full app therefore execute real
  lifecycle transitions: one test stops the user's real lumberjacks, a later test's auto-start spawns a real detached
  `sase axe start` daemon that inherits the pytest environment (`HOME` pointing into a pytest tmpdir,
  `cwd=os.path.expanduser("~")` → the fake home) plus the real lock fd.
- Nothing in `start_axe_daemon_result` / `stop_axe_daemon_result` refuses to run inside a test process, and
  `_compose_axe_daemon_env` scrubs agent/chop identity but not pytest variables.

Four independent weaknesses compounded: frozen import-time paths defeat test isolation; no guard stops tests from
spawning or killing real daemons; a wedged/imposter lock holder permanently blocks healing; and blocked waiting runners
depend exclusively on the `wait_checks` chop writing `ready.json`, with no fallback. Each gets its own phase below — the
first two prevent the leak, the last two make the system survive any similar failure.

## Design overview

Layered defense, ordered so each phase is independently shippable:

- **Prevent** (live-paths, test-guard): tests physically cannot reach the real axe state, and even un-isolated code
  refuses daemon lifecycle transitions under pytest.
- **Recover** (self-heal): if the real axe is ever again wedged by a lock holder that never publishes a PID,
  `ensure_axe` recovers automatically instead of failing silently forever.
- **Degrade gracefully** (runner-fallback): even with zero chops running, a waiting agent whose dependencies complete
  eventually starts.

## Live axe state path resolution

Convert the import-time constants in `src/sase/axe/state.py` into live accessor functions that resolve through
`sase_subdir("axe")` at call time:

- `axe_state_dir()`, `jack_state_dir()`, `shared_state_dir()`, `axe_output_log_path()`.
- Delete the module-level constants entirely (no deprecated aliases) so no call site can freeze a path at import again.
  Update all src call sites: `state.py` itself (`ensure_state_dir`, `atomic_write_json` helpers), `_state_scheduler.py`,
  `_state_lumberjack.py`, `_state_chops.py`, `orchestrator.py`, `ensure.py`, `lock.py` (`_axe_lifecycle_lock_path`),
  `desired_state.py`, `maintenance.py`, `chop_agents.py`, `_process_start.py` (note: this file uses a by-value
  `from .state import AXE_STATE_DIR` today, which is exactly the pattern that defeats the documented
  `patch("sase.axe.state.AXE_STATE_DIR", ...)` seam), and `doctor/checks_deep_axe.py`.
- With live resolution, the existing autouse `_isolate_sase_home` fixture automatically covers all axe state:
  `sase_home()` reads `SASE_HOME`/`HOME` plus the patched `expanduser` at call time. No conftest change should be
  required for isolation itself.
- Migrate the tests that patch `sase.axe.state.AXE_STATE_DIR` (e.g. `tests/test_axe_state.py`,
  `tests/test_axe_process*.py`, `tests/test_axe_restart.py`, `tests/test_axe_orchestrator.py`,
  `tests/_axe_lumberjack_fixtures.py`, `tests/doctor/test_checks_deep.py`, and the other files found by grepping for
  `AXE_STATE_DIR`/`JACK_STATE_DIR` under tests/). Most can drop the patch entirely and rely on the autouse fake home;
  tests that need a specific directory should monkeypatch the new accessor functions. Update the `state.py` docstring,
  which currently instructs tests to patch the constant.
- Add a focused unit test proving the fix: import `sase.axe.state` (and `sase.axe.lock`) first, then redirect the sase
  home, and assert that `axe_state_dir()` and the lifecycle lock path now resolve inside the redirected home.

An alternative — PEP 562 module `__getattr__` keeping the old constant names live — was considered and rejected: it
silently re-freezes for any future `from .state import ...` by-value import, which is the exact regression class this
phase eliminates.

## Pytest lifecycle guard and daemon env hygiene

Belt-and-braces for future regressions (a test that opts out of home isolation, a subprocess that escapes the fixture, a
reintroduced cached path):

- Add a small helper (e.g. in `src/sase/axe/_process_start.py` or a shared module) that detects a pytest context via
  `PYTEST_CURRENT_TEST` / `PYTEST_VERSION` in the environment.
- `start_axe_daemon_result`, `stop_axe_daemon_result`, and the restart flow refuse to run in a pytest context —
  returning an explicit non-exceptional result (a new status such as `"blocked_in_tests"`) **before** any side effect,
  including the initial `write_desired_state` call — unless an explicit override env var (e.g.
  `SASE_AXE_ALLOW_LIFECYCLE_IN_TESTS=1`) is set. Axe's own lifecycle tests that genuinely exercise daemon spawn/stop
  against their fake homes set the override through a fixture.
- `_compose_axe_daemon_env` additionally scrubs `PYTEST_*` variables so a legitimately spawned daemon can never carry
  test context, and daemon spawn fails fast with a clear result message when the resolved home/cwd for the daemon does
  not exist.
- The TUI paths (`auto_start_axe` startup init, `!x` toggle, restart action) need no special casing: they call through
  the guarded process helpers and surface the returned message.
- Tests: pytest-context guard blocks each lifecycle entry point with no side effects (no desired-state write, no lock
  acquisition, no subprocess spawn); the override env restores current behavior; spawned-daemon env contains no
  `PYTEST_*` keys.

## Wedged lifecycle-lock recovery in axe healing

Make the "lock held but no live orchestrator PID" state self-healing instead of terminal:

- Track when a blocked start first observed the wedged state (a small JSON marker under the axe state dir, written by
  `start_axe_daemon_result` when it returns `"blocked"`).
- When the wedged state persists past a grace period (default around 60–120 seconds — long enough for any legitimate
  orchestrator startup to publish its PID file), `start_axe_daemon_result` (or a helper `ensure_axe` invokes on the
  blocked path) identifies the holder via `read_lock_holder_pid()` / `/proc/locks`, verifies it is not the current
  process, terminates it (SIGTERM, escalating to SIGKILL after a timeout, reusing the existing stop machinery where
  possible), clears stale holder state, and retries the normal start path once.
- Clear the wedged-state marker whenever a start succeeds or the lock is observed free.
- Surface outcomes: recovering emits a notification (alongside the existing `notify_axe_healed` flow) that names the
  killed PID; repeated `ensure_axe` failures stop being silent — the blocked/failed path emits a rate-limited
  notification so a bricked axe is visible in the notification inbox instead of only in log files.
- Safety argument for the kill: only orchestrators and in-flight starters ever hold this flock; a healthy starter
  publishes a PID within seconds; the grace period plus a re-probe immediately before signaling bounds the blast radius
  to genuinely wedged processes.
- Tests: a fake lock holder process that never publishes a PID is detected and terminated after the grace period and axe
  starts; a holder that publishes a PID inside the grace period is left alone; the recovery path is exercised through
  `ensure_axe` end to end.

## Waiting-runner fallback dependency resolution

`wait_for_dependencies` in `src/sase/axe/run_agent_wait.py` currently blocks on the `ready.json` marker alone, so a dead
`waits` lumberjack strands agents indefinitely — the exact user-visible symptom. Add a bounded self-serve fallback:

- Inside the poll loop, every N seconds (default around 60s, far coarser than the 2s marker poll to keep scan cost
  negligible), re-run the same resolution the chop performs: `build_wait_dependency_index(project_name)` +
  `dependency_resolution_status(...)` with the waiter's own artifact dir excluded, honoring the `resolved_deps`
  memoization and `wait_for_artifacts` identity deps recorded in `waiting.json` — i.e. reuse the logic of the existing
  `_initial_dependencies_resolved` fast path rather than duplicating it.
- When the fallback resolves, behave exactly as if `ready.json` had appeared: honor the post-dependency `wait_duration`
  / `wait_until` floor, record `wait_completed_at`, clean up both markers, and log that resolution came from the runner
  fallback (so chop outages remain diagnosable from runner output).
- Skip the fallback when `project_name` is unavailable (mirroring the fast path's guard) and keep `ready.json` as the
  primary, cheap signal — the fallback only converts "chop down forever" into "at most one fallback interval of extra
  latency".
- Tests: a waiter whose dependency completes while no chop ever writes `ready.json` unblocks via the fallback; memoized
  identity deps are honored; the duration floor still starts at resolution time; the fallback never fires while
  dependencies remain unresolved.

## Cross-layer regression exercises

A final verification phase that reconstructs the incident shape in miniature:

- An isolation regression test: import the axe modules, then apply the standard test-home redirection, run the
  TUI-equivalent status probe plus a guarded start attempt, and assert zero reads or writes land outside the fake home
  (e.g. by pointing the fake home at a tmp dir and asserting the real-home sentinel paths stay untouched).
- A guard regression test at the process level: with pytest env vars present and no override, every lifecycle entry
  point returns its blocked-in-tests result and spawns nothing (assert on subprocess spawn via monkeypatched
  `subprocess.Popen`).
- An outage-recovery scenario test (extending the existing `tests/test_axe_smoke_outage_recovery.py` coverage):
  dependency completes while the waits chop is down → the runner fallback unblocks the agent; separately, a wedged lock
  holder → `ensure_axe` recovers and restarts axe.
- Run the full `just check` gate and the PNG visual suite untouched-ness (no TUI rendering changes are expected from any
  phase).

## Testing strategy

Each phase lands with its own unit tests as described in its section; the smoke phase adds the cross-layer scenarios.
All phases must pass `just check`. Nothing here crosses the Rust core boundary — axe process lifecycle, pytest fixtures,
and the Python runner wait loop are all Python-side infrastructure — so no `sase-core` changes are required.

## Risks and mitigations

- **Churn risk in live-paths**: many call sites change at once. Mitigation: mechanical rename to accessor functions, no
  behavior change, full test-suite gate; deleting the constants makes any missed site a hard ImportError rather than a
  silent real-home access.
- **Guard false positives**: axe lifecycle tests legitimately spawn daemons. Mitigation: explicit override env var set
  by those tests' fixtures; guard returns structured results rather than raising, so non-test callers are unaffected.
- **Self-heal killing a healthy process**: bounded by the grace period, a re-probe before signaling, and the invariant
  that only orchestrators/starters hold the flock. The recovery emits a notification naming the killed PID for
  auditability.
- **Fallback resolution divergence from the chop**: both paths must agree on resolution semantics. Mitigation: share the
  existing index/resolution helpers rather than reimplementing; tests assert parity on memoized identity deps and
  duration floors.
- **Live incident state**: existing leaked orchestrators on the host are operational state, not code; this plan
  deliberately excludes ad-hoc host cleanup. Once self-heal lands, a regular `ensure_axe` pass (or a manual
  `sase axe stop` + start before then) restores the real daemon, and the prevention phases keep `just test` runs from
  re-breaking it.
