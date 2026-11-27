---
tier: tale
title: Make the artifact-link chops honor their work budgets under host load
goal:
  A host load spike makes artifact_link_backfill and agent_publication defer work with a
  logged reason instead of being SIGKILLed at the 300s axe timeout, and any kill that
  still happens names its stage and keeps learned state.
size: medium
decisions:
  raise_backfill_timeout:
    ask:
      Should the housekeeping `artifact_link_backfill` job's axe timeout also be raised
      from 5m to 10m as a backstop?
    choices:
      keep: Timeout stays 5m, so a timeout still flags a budget-enforcement bug
      raise:
        Timeout becomes 10m; absorbs uninterruptible tails but hides future budget
        regressions
    default: keep
    why:
      After this plan the internal budget is enforced; a 5m timeout stays a useful alarm
      for the next gap.
    answer: keep
proposed_by: bbugyi200.athena.1hb
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 12:52:33
status: wip
---

# Plan: Make the artifact-link chops honor their work budgets under host load

## Why (evidence from the apollo digest of 2026-11-27 10:33)

The digest reported three SIGKILLs at the 300s axe timeout:
`housekeeping/artifact_link_backfill` once (09:28–09:33) and
`publication/agent_publication` twice in a row (09:35–09:40 and 09:40–09:45).

- **apollo was overloaded at the time.** In 09:30–09:46 every lumberjack overran its
  tick: `agent_waits` took up to 125s against a 2s interval, `hooks` took 30s+ against
  5s, and `sidecar_sync` took 47s. The next `agent_publication` pass, at 09:45–09:48,
  succeeded. Its bob-cli link drain alone took 145s, logged as
  `publish=100.48s admission=39.29s beads=31.12s aggregate=27.96s`.
- **The jobs have almost no headroom even on a normal day.** The last 10
  `artifact_link_backfill` runs took 150–270s of the 300s timeout (271s, 264s, 260s,
  304s killed, 259s, ...). On 11-22 the perf runbook recorded a median of 101s. Almost
  all of the time goes to `gh_bobs-org__bob-cli`:
  - The sweep took 90–195s while scanning only 1–7 documents, against a "45s" sweep
    budget.
  - The full reconcile took 88–102s.
  - The drain's publish phase spends about 40s in `admission` alone.
- **Both chops have internal 240s budgets, but the expensive steps don't check them:**
  1. `src/sase/scripts/_artifact_link_backfill_jobs.py` `run_project` always reconciles
     the first project of a tick (the `first_project` bypass of the estimate gate). In
     the killed run, bob-cli's sweep had used about 150s, leaving about 65s of budget,
     yet `reconcile/aggregate` still started with a ~100s estimate. It was killed while
     still in that phase: `preview_reconciled_aggregate` only checks the deadline
     _between_ stores, and reducing the first store can't be interrupted.
  2. `publish_artifact_link_events` (`src/sase/sdd/_artifact_link_event_publish.py`)
     takes the per-project `artifact-link-events.lock` with a _blocking_
     `locked_file(...)` and no timeout, even when the caller passed a deadline. It also
     starts admission, which loads the full event snapshot (about 11.4k event files for
     bob-cli), whenever the deadline hasn't passed yet. So the publication job's bob-cli
     link drain can outrun its 240s budget, either waiting on the lock or doing slow
     work under load.
  3. A SIGKILL throws away the backfill's learned state.
     `sase_chop_artifact_link_backfill.py` writes the schedule (reconcile
     memo/estimates) and the project cursor only at the end of the tick. So the next
     tick repeats the same first project with the same stale estimate.
- **The publication timeouts left no output (`[output absent]`).** In
  `src/sase/agents_sync/_publication_job_run.py`, `_maybe_progress` only prints stage
  lines once 60s have elapsed (`PUBLICATION_PROGRESS_THRESHOLD_SECONDS`). A stage that
  starts before 60s and then hangs never logs its name. The successful run's first
  progress line came at 145s.

**Conclusion:** yes, fix it. Load spikes will keep happening, so the goal is that a
spike leads to a logged deferral rather than a SIGKILL, and that a SIGKILL, if one still
happens, names the stage it was in and keeps the learned state.

No decision record governs these budgets. The change stays inside the Python chop and
drain orchestration that already owns them, so there's no sase-core change.

## Changes

### 1. Publication pass: report a stalled stage (`src/sase/agents_sync/_publication_job_run.py`)

- In `_run_publication_job`, record every `_maybe_progress(step, project_key)` call as
  the _current stage_: step, project, and elapsed time at start. Record it before the
  60s gate, so stages that start early are tracked too. The gated line emission itself
  doesn't change.
- When `progress` is not `None`, start a daemon watchdog thread. It first wakes at
  `PUBLICATION_PROGRESS_THRESHOLD_SECONDS`, then every new
  `PUBLICATION_STALL_REPORT_INTERVAL_SECONDS = 60.0`. On each wake, if the current stage
  hasn't changed since the last line, it emits:
  `agent_publication progress: still in <step> (started at <s>s) at <elapsed>s project=<key>`.
  Protect the shared stage state with a `threading.Lock`. Stop the thread in a `finally`
  (`Event.set()` plus a short `join`). `ChopLogger.info` prints with `flush=True`, so
  calling it from the thread is fine.
- Fast passes (under 60s) must stay completely quiet, as they are today.

### 2. Deadline-aware event publication (`src/sase/sdd/_artifact_link_event_publish.py`)

- Add `_PUBLISH_MIN_HEADROOM_SECONDS = 60.0`. Before admission, a deadline-bearing
  publish must have at least this much left. Admission, states, roots, and
  local_receipts can't be interrupted (about 45s for bob-cli normally). So the
  `_publish_event_objects_locked` entry check becomes
  `time.monotonic() + _PUBLISH_MIN_HEADROOM_SECONDS >= deadline` → deferred report. Keep
  the same `BEAD_PROJECTION_DEFERRED_DIAGNOSTIC` deferred shape.
- When `deadline is not None and not already_locked`, take the lock with
  `locked_file(lock_path, fcntl.LOCK_EX, timeout=max(0.0, deadline - time.monotonic() - _PUBLISH_MIN_HEADROOM_SECONDS))`.
  On `sase.memory.locks.LockTimeoutError`, return a deferred report
  (`attempted=len(objects)`, `deferred=True`). Its diagnostic should name the busy
  `artifact-link-events.lock`.
- Callers without a deadline (`artifact_cli/link_ops.py`, `artifact_link_inlet.py`,
  `_artifact_link_import_apply.py`) keep today's blocking behavior.
- Check that both deadline callers treat the deferred report as "retain and retry":
  - `drain_artifact_link_outbox`: no settled keys, so rows stay queued and
    `report.deferred` is set. The publication job then keeps its link hints.
  - `persist_derived_link_candidates`, via `_persist_derived_link_candidates_as_events`:
    `outcome.deferred` stops the sweep without marking the chunk's documents as swept.

### 3. Backfill reconcile gate without the unconditional first-project bypass

These changes are in `src/sase/scripts/_artifact_link_backfill_jobs.py` and
`_artifact_link_backfill_schedule.py`.

- Add `_RECONCILE_ESTIMATE_MARGIN = 1.5`. A full reconcile, meaning one where the memo
  can't vouch for a skip, starts only when `remaining >= estimate * margin`. That way a
  2x slowdown still finishes inside the 60s gap between the 240s budget and the 300s
  timeout.
- Replace the `first_project` bypass with a persisted _reconcile priority_:
  - `BackfillSchedule` gains a per-project `reconcile_priority` flag, plus a serialized
    field with a backward-compatible default.
  - The flag is set whenever reconcile is deferred for budget, cut short by the
    deadline, or found interrupted (see change 4). A full reconcile that completes
    clears it.
  - A project with priority still goes first on the next tick through the existing
    `first_deferred` cursor. On that tick it **skips its sweep**, leaving the sweep
    checkpoint untouched, so drain and reconcile get nearly the full budget.
  - Only in that priority-first case may reconcile bypass the estimate gate. This keeps
    a project whose estimate exceeds the budget from starving, but it only runs with the
    full budget, not after a 150s sweep.
- In every other case, the first project defers reconcile exactly like later projects
  do.

### 4. Kill-safe backfill state (`src/sase/scripts/sase_chop_artifact_link_backfill.py`, `_artifact_link_backfill_jobs.py`)

- After each project, persist the schedule and the project cursor, as is already done
  for the sweep checkpoint. A SIGKILL mid-tick then keeps the deferrals, priorities, and
  estimates learned so far. Keep the end-of-tick writes as well.
- Before starting a full reconcile, write an in-flight marker
  (`reconcile_started_at[project]`) and persist the schedule. Clear the marker when
  reconcile returns, whatever the outcome.
- At the start of a tick, a leftover marker means the previous run died mid-reconcile.
  Set that project's reconcile priority, clear the marker, and log a warning:
  `<project>: previous reconcile did not finish; prioritizing it`.

### 5. Sweep phase timings (`src/sase/sdd/artifact_link_backfill.py`, `_artifact_link_backfill_jobs.py`)

- `run_artifact_link_backfill_batch` gains `phase_seconds` on its report, timing:
  - `documents` (`sweepable_artifact_link_documents`)
  - `inputs` (`artifact_link_derivation_inputs`)
  - `derive`
  - `persist`
  - `persist.<name>`, carried from the publish report when the derivation outcome can
    expose it. Add that pass-through if it's missing.
- `run_project` logs it like the drain line:
  `[artifact_link_backfill] <project>: sweep phases: documents=… inputs=… derive=… persist=… …`,
  reusing `render_drain_phase_seconds` if it fits. This is the data needed to explain
  the 90–195s sweeps of 1–7 documents.

### 6. Config descriptions and docs

- `src/sase/default_config.yml`:
  - Update the `artifact_link_backfill` description, which currently says "reconcile
    starts only when its learned duration fits the remaining budget...". It should say
    the reconcile gate has a safety margin, and that priority ticks skip the sweep in
    place of the first-project bypass.
  - Add one clause to the `agent_publication` description: a stalled stage is reported
    every minute after the first, and a busy event lock defers instead of blocking.
- `docs/reference/automation/scheduler-housekeeping.md` (around lines 290–310): replace
  "the first project of a tick always reconciles" with the priority rule and mention the
  sweep-phase timing line. In the publication section (around line 51), add the
  stalled-stage line and the bounded lock wait.
- Read `docs_writing.md` with `/sase_memory_read` before editing docs.

> [!decision] raise_backfill_timeout = raise In `src/sase/default_config.yml`, change
> the `artifact_link_backfill` job's `timeout` from `"5m"` to `"10m"` and mention the
> backstop in its description. Leave `publication/agent_publication` at 5m: it runs
> every 15s, so a longer timeout only holds the publisher lock longer.

> [!decision] raise_backfill_timeout = keep Leave every axe timeout at its current
> value. The enforced budgets above are the fix.

### 7. Tests

- `tests/test_axe_chop_artifact_link_backfill_cadence.py`, around line 194: the test
  currently asserts the first project reconciles despite a 500s estimate. Rewrite it for
  the new rules:
  - Without priority, a first project whose estimate doesn't fit defers and gains
    priority.
  - With priority, the project skips its sweep and reconciles.
- New backfill tests, in the existing `test_axe_chop_artifact_link_backfill_*` files:
  - The margin gate.
  - The priority flag is set on a cut-short or deferred reconcile and cleared on
    completion.
  - The schedule and cursor are persisted after each project. Simulate an exception or
    kill after the first project and assert the state is on disk.
  - A leftover in-flight marker sets priority and logs the warning.
  - The `sweep phases:` line is logged.
- `tests/agents_sync/test_publication_slow_pass_progress.py`:
  - With monkeypatched small threshold and interval constants, a drain stub blocks on a
    `threading.Event` past the threshold. Assert exactly the `still in <stage>` line(s)
    for a stage that began before the threshold.
  - Fast passes still emit nothing.
  - The watchdog thread is stopped when the pass returns.
- Publish tests, next to the existing `_artifact_link_event_publish` tests:
  - With the events lock held by another fd and a near deadline, publish returns a
    deferred report without raising.
  - Inside the headroom window, it defers before admission (stub admission and assert it
    isn't called).
  - `drain_artifact_link_outbox` keeps the rows queued in both cases.
- Follow `lint_and_test.md`, read with `/sase_memory_read`, before finishing.

## Out of scope (record as follow-up task beads with `/sase_new_task` if not already tracked)

These drive the base cost that this plan only bounds:

- bob-cli's `artifact-link-outbox.jsonl` holds 8,341 retained rows from 1,543 agents
  (5.4 MB). Every drain spends about 9s just reading it.
- Admission re-loads the full event snapshot on every publish, about 40s.
- A full reconcile takes about 100s, and its first-store reduction can't be interrupted.
- The sweep costs 90–195s even for a handful of documents. Change 5 provides the data
  for that one.
