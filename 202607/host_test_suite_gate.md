---
tier: tale
title: Host-global test-suite concurrency gate
goal: 'Concurrent parallel runs of the sase test suite are bounded machine-wide by
  a crash-safe gate, and per-run xdist worker counts adapt to available memory, so
  agent/hook/audit fan-out can no longer drive the host into a fork-and-swap death
  spiral.

  '
create_time: 2026-07-20 09:30:49
status: wip
prompt: 202607/prompts/host_test_suite_gate.md
---

# Plan: Host-global test-suite concurrency gate

## Context and root-cause diagnosis

On 2026-07-20 the host (`athena`, 64 cores / 62 GB RAM / 64 GB swap) suffered a recurring CPU catastrophe: `sar -q`
shows load ramping from ~2 at 07:30 to **1551 at 09:10**, with the process count exploding from a ~1,780 baseline to
**10,856** and 1,164 tasks blocked. `sar -r`/`-S` show committed memory peaking at **157% of RAM** with **94% of the 64
GB swap consumed** — the load in the thousands was mostly swap-thrash: hundreds of processes stuck in uninterruptible
sleep. Process creation ran at 300–650 forks/sec for ~90 minutes. `/tmp` is a 32 GB RAM-backed tmpfs and yesterday's
incident filled it ("No space left on device" in the axe log), adding direct RAM pressure.

The multiplier is the test suite. `just test` (via `tools/run_pytest`) launches pytest-xdist with `cpu_count()//4` =
**16 worker processes**, each a full Python interpreter importing the sase package (Textual, Rich, TUI apps, visual
snapshot rendering). One suite run costs roughly 10–16 GB at peak. Nothing anywhere bounds how many such runs execute
concurrently on the host, and SASE has many independent launchers that each end in `just check`:

- Per-commit audit chops (`recent_bug_audit`, `recent_improvement_audit`) launched agents for each of the ~8 commits
  that landed between 07:36 and 08:35 — the checks lumberjack lists 14+ audit ChangeSpecs from this window.
- A ten-agent user swarm launched 07:32–07:46, plus a ten-agent `toobig` swarm at 08:31.
- ChangeSpec HOOK re-runs (34 active ChangeSpecs carrying `just test` / `just lint` hooks) and their fix-up agents
  (`sase_fix_just_tests_31`, `sase_fix_just_linters_14`, ...).

When enough of these overlap (~6–10 suites, ≈100–160 xdist workers plus the subprocesses their tests spawn), RAM is
exhausted, swap fills, every suite slows to a crawl, hooks and tests fail on timeouts, and the failures spawn _more_
fix/audit work — a positive feedback loop. That is why this "keeps" happening rather than being a one-off.

Adjacent causes that are already fixed and are explicitly **not** re-planned here:

- The `fix_just` chop (auto-spawning test-fix agents) was retired from the external `bugyi-chops` package (v0.2.0)
  earlier today.
- The sase-80 epic's isolation and guard phases landed today (sase-80.1 call-time axe state paths, sase-80.2 pytest
  lifecycle guard, sase-80.4 waiting-runner fallback); sase-80.3/80.5 remain in flight on that epic. Those stop pytest
  from bricking or crash-looping the _real_ axe daemon, but do nothing to bound legitimate concurrent suite runs — the
  gap this plan closes.

## Design overview

Two complementary changes, both confined to test infrastructure in this repo (no Rust-core involvement — this is
Python-side tooling, per the backend boundary rule):

1. **A host-global counting semaphore ("suite gate") acquired by the pytest controller process** of any parallel (xdist)
   run of this suite, limiting the number of simultaneously executing parallel suite runs machine-wide, regardless of
   who launched them (agents, hook runners, audit chops, or a human shell).
2. **Memory-aware worker sizing** in `tools/run_pytest`, so a suite that does start under memory pressure spins up fewer
   workers instead of piling 16 more interpreters onto a starved host.

The gate lives at the choke point where the damage occurs (pytest startup), so it needs no cooperation from any launcher
and covers every entry path, including direct `pytest -n ...` invocations that bypass the Justfile.

## Suite gate module

New test-support module `tests/_suite_gate.py` (naming follows the existing `tests/_axe_lumberjack_fixtures.py`
convention; test-support paths keep Symvision out of the picture), wired from `tests/conftest.py`:

- **Acquisition point**: `pytest_configure` acquires; `pytest_unconfigure` releases. The gate applies only to the xdist
  **controller** of a parallel run: skip when `PYTEST_XDIST_WORKER` is set (we are a worker), and skip when the resolved
  `numprocesses` option is absent/0/1 (serial and targeted iteration runs stay instant). Read `numprocesses` defensively
  so the hook still works if xdist is ever absent.
- **Mechanism**: a fixed slot directory shared by every workspace and clone on the host — default
  `/tmp/sase-suite-gate-<uid>` (override: `SASE_TEST_GATE_DIR`). It contains N slot files; acquisition loops over them
  trying `fcntl.flock(LOCK_EX | LOCK_NB)`, sleeping a few seconds between sweeps. On success the holder writes
  pid/argv/start-time into its slot file for debuggability. Because the lock is an OS flock, the kernel releases it when
  the holder dies for any reason — including the SIGKILLs that agent runners routinely deliver — so the gate can never
  wedge on a dead holder and needs no stale-state cleanup.
- **Slot count**: `SASE_TEST_GATE_SLOTS`, default **2**. Rationale: two 16-worker suites ≈ 32 heavyweight processes and
  ~20–32 GB peak on a 64-core / 62 GB host — busy but healthy; three or more is where today's death spiral started.
- **Waiting UX**: while blocked, print a status line every ~30 s naming the current holders (pid + age from the slot
  metadata) so an agent transcript or human terminal shows _why_ the run has not started.
- **Timeout**: `SASE_TEST_GATE_TIMEOUT` seconds, default 2700 (45 min). On expiry, fail the run cleanly
  (`pytest.UsageError`) with an actionable message listing the holders and the override env vars. Failing is the correct
  pressure valve: proceeding ungated under extreme contention is exactly the scenario that melted the host.
- **Escape hatches**: `SASE_TEST_GATE_DISABLED=1` skips the gate entirely. After the controller acquires a slot it sets
  that variable in its own environment so any _nested_ pytest invocation spawned by tests inherits the exemption and
  cannot deadlock against its ancestor's slot.

## Memory-aware worker sizing

In `tools/run_pytest`, extend `_worker_count()`:

- Explicit `SASE_PYTEST_WORKERS` keeps absolute precedence, unchanged.
- Otherwise compute `min(cpu_count // 4, max(2, mem_available_gib // 2))`, reading `MemAvailable` from `/proc/meminfo`
  (assume ~2 GiB per worker, floor of 2 workers). On platforms or failures where `MemAvailable` cannot be read, fall
  back to the current `cpu_count // 4` behavior.
- Load-average-based scaling was considered and rejected: load is a lagging, flappy signal, and memory exhaustion — not
  CPU contention — is what turned high load into a machine-killing spiral.

## Testing strategy

- `tests/test_suite_gate.py` unit coverage against a tmp gate dir with tiny timeouts: acquire/release cycles; the N+1th
  acquirer blocks and then succeeds after a release; timeout raises with holder metadata in the message;
  `SASE_TEST_GATE_DISABLED`, worker-env (`PYTEST_XDIST_WORKER`), and serial runs all skip; slot metadata is written.
  Crash-safety: a real subprocess holds a slot, is SIGKILLed, and the slot is immediately reacquirable (proves the
  kernel-release property the design leans on).
- Conftest wiring: exercise the `pytest_configure` path with a stubbed config (controller vs worker vs serial) rather
  than nested full pytest runs; one subprocess-level integration test may run a tiny `-n 2` suite against a single-slot
  temp gate dir to prove end-to-end serialization, marked to keep the fast lane quick if it is not cheap.
- `_worker_count()` tests with monkeypatched meminfo readings: pressure shrinks workers to the floor, plentiful memory
  preserves `cpu//4`, override env still wins, unreadable meminfo falls back.
- Gate tests must never touch the real host gate dir: they always inject `SASE_TEST_GATE_DIR` (and the suite's own gated
  controller already exports `SASE_TEST_GATE_DISABLED` to descendants, so these nested exercises are doubly safe).
- Full `just check` gate for the change itself.

## Risks and mitigations

- **Queueing latency for agents**: intended and bounded — visible via the waiting status lines, tunable via
  `SASE_TEST_GATE_SLOTS`, and strictly better than the alternative (every run crawling under a load of 1500).
- **Timeout-induced failures under pathological load**: the 45-minute default only fires when the host is already
  saturated for that long; the error message names holders and knobs so the operator can intervene deliberately.
- **Nested pytest deadlock**: prevented by exporting the disable marker from the slot-holding controller to all
  descendants.
- **CI behavior**: GitHub Actions runners execute one suite at a time, so the gate acquires instantly; no CI
  configuration change is required.
- **Multi-user hosts**: the default gate dir embeds the uid, so distinct users never contend on (or collide over
  permissions of) shared slot files.

## Out of scope / follow-up recommendations

- Tuning the launch _rate_ of the per-commit audit chops lives in the external `bugyi-chops` package, not this repo.
- A global cross-launcher agent-concurrency budget inside axe (beyond the existing per-lumberjack runner caps) is a
  larger scheduler feature and would complement, not replace, this choke-point gate.
- The remaining sase-80 phases (wedged-lock self-heal, cross-layer smoke) stay with that epic.
