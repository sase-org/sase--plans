---
tier: epic
status: done
title: Get monitor reconciliation off the ACE startup critical path
goal: "`sase ace` startup returns to at or below its 2026-08-12 baseline (median
  `visible_ready` <= 2.8 s, `agents_ready` <= 3.4 s), monitor reconciliation no longer
  runs synchronously inside the agents disk load, and an operation-count regression gate
  keeps data-scaled work off the startup path.

  "
phases:
  - id: guards
    title: Reorder the reconcile guards
    depends_on: []
    size: small
    description: "guards: reorder `should_reconcile_dead_supervisor` so the cheap
      `monitor_state`/`pid` rejects run before `proc_shell_owns()`, which currently does
      a full proc-store read for all 147 records when 0 survive the next two lines; add
      a test asserting the proc lookup is skipped for terminal and pid-less records.
      Measured saving 1.47 s.

      "
  - id: snapshot
    title: Kill the N+1 proc-store reads
    depends_on: []
    size: medium
    description: "snapshot: add a snapshot-scoped proc lookup so `get_proc` stops
      re-reading and re-parsing the whole store per id, thread one snapshot through the
      reconcile pass and `_with_proc_projection`, and test that proc-store reads stay
      bounded regardless of record count.

      "
  - id: bounded-query
    title: Stop the O(archive) index query
    depends_on: []
    size: medium
    description: "bounded-query: give reconciliation its own bounded artifact-index
      query instead of the unbounded full-history `include_hidden` scan of the 115 MB
      index, leave the `list_monitors` listing path unchanged, and pin the new bounds
      with a test. Escalate to the Rust core if the predicate is not expressible in the
      existing wire query.

      "
  - id: off-read-path
    title: Take reconciliation off the synchronous load
    depends_on:
      - guards
      - snapshot
      - bounded-query
    size: medium
    description: "off-read-path: remove the synchronous
      `_reconcile_dead_monitor_supervisors_for_tui()` call from
      `_load_agents_from_disk_impl` and run it in the background with a coalesced
      follow-up refresh, reusing the existing loader-cleanup shape, while preserving
      settlement semantics and preventing overlapping passes.

      "
  - id: gate
    title: Pin the win with a regression gate
    depends_on:
      - off-read-path
    size: small
    description:
      "gate: add a `tests/perf/` bench and baseline asserting bounded proc-store reads
      and index queries per disk load and no synchronous reconciliation, preferring
      deterministic operation counts over wall-clock seconds, and ask the user before
      recording the incident in the tui_perf memory note."
proposed_by: bbugyi200.athena.03q
bead_id: sase-n7
create_time: 2026-09-09 19:51:59
---

- **PROMPT:**
  [prompts/202608/tui_startup_monitor_reconcile.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/tui_startup_monitor_reconcile.md)
- **BEAD:**
  [sase-n7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-n7/README.md)

# Get monitor reconciliation off the ACE startup critical path

## Problem

`sase ace` startup has regressed ~2.2x in four days. From
`~/.sase/logs/tui_startup.jsonl` (n=101, medians per day):

| day        | `visible_ready` | `agents_ready` | `process_start_to_on_mount` | `on_mount_to_first_paint` |
| ---------- | --------------- | -------------- | --------------------------- | ------------------------- |
| 2026-08-12 | 2.81 s          | 3.44 s         | 0.62 s                      | 0.21 s                    |
| 2026-08-13 | 3.48 s          | 4.19 s         | 0.67 s                      | 0.22 s                    |
| 2026-08-14 | 4.20 s          | 4.86 s         | 0.65 s                      | 0.21 s                    |
| 2026-08-15 | 4.81 s          | 5.45 s         | 0.68 s                      | 0.21 s                    |
| 2026-08-16 | 6.31 s          | 7.01 s         | 0.72 s                      | 0.22 s                    |

Import time and first paint are flat. The entire regression is in the agents load.
`agent_row_count` (~31) and `index_row_count` (~530 to ~596) are roughly flat over the
same window, so this is not simple row growth.

`~/.sase/logs/tui_agent_loads.jsonl` (n=11,101 slow-load records) isolates it to the
`disk` stage: median 2.38 s (08-14) to 4.57 s (08-16), p90 3.38 s to 10.15 s. `prep`
(~0.07 s) and `apply` (~0.04 s) are negligible.

## Root cause

`_load_agents_from_disk_impl`
(`src/sase/ace/tui/actions/agents/_loading_helpers.py:431`) calls
`_reconcile_dead_monitor_supervisors_for_tui()` synchronously at the top of every agents
load, including the startup load. A cProfile run against real `~/.sase` state measured
that single call at **3.12 s of a 3.84 s disk load (82%)**.

Reconciliation is a settlement/write concern running inside the TUI's read path. It
carries three compounding defects.

### Defect 1 — guard ordering (~1.47 s)

`should_reconcile_dead_supervisor` (`src/sase/monitor/reconcile.py:47`) calls the
expensive `proc_shell_owns(record.monitor_id)` **first**, before the cheap
`monitor_state != "running"` and `pid is None` rejects.

Measured on real state: `proc_shell_owns()` runs for **147 of 147** monitor records;
**0** survive the cheap guards. Every one of those full proc-store reads is discarded
one line later. A simulated reorder (cheap rejects first) took
`reconcile_dead_supervisors` from 3.12 s to 1.65 s — **1.47 s saved, 47%**.

### Defect 2 — N+1 full proc-store reads

`get_proc()` (`src/sase/procs/store.py:70`) calls `read_procs()`, which reads and parses
the _entire_ proc store, then linear-scans for one id. `read_procs` has no caching. So
each `proc_shell_owns()` re-reads all ~101 procs: 147 monitors x 101 procs = **14,847
`Proc.from_dict` calls** (0.766 s in the Rust `read_procs_snapshot` binding + 0.69 s
parsing) for what is one snapshot's worth of data.

`_with_proc_projection` (`src/sase/monitor/store.py:331`) has the same N+1 and is worse
— it calls `proc_shell_owns()` _and_ `get_proc()`, two full reads per record. It sits on
`list_monitors` (`src/sase/monitor/store.py:301`), a CLI path, not a TUI path.

Fixing defect 1 hides most of defect 2 today, but leaves the N+1 live: it reappears the
moment monitors are actually running, which is exactly when the user has agents in
flight.

### Defect 3 — unbounded full-history index query (~1.5 s, and growing)

`_project_records` (`src/sase/monitor/store.py:417`) queries the 115 MB
`~/.sase/agent_artifact_index.sqlite` with `include_full_history=True`,
`active_limit=None`, `recent_completed_limit=None`, `include_hidden=True`. Measured at
**1.51 s per call**, and it is the residual after defect 1 is fixed.

This is O(archive) work, so it is the growth curve behind the day-over-day trend.
Reconciliation also roughly doubles the per-load index query count — the profile shows
`query_agent_artifact_index` called twice at 0.786 s each, once for reconcile and once
for the actual agent load.

### Timeline corroboration

- **2026-08-13** `29cb7924a fix(monitor): reconcile dead monitor supervisors` wired
  `_reconcile_dead_monitor_supervisors_for_tui` into the loader. `visible_ready` 2.81 s
  to 3.48 s.
- **2026-08-15** `8b4635ad1 feat(monitor): run monitors through the shared proc service`
  put `proc_shell_owns()` into the guard. 4.20 s to 4.81 s.
- **2026-08-16** 6.31 s.

### Memory-note violations

This breaks `sase/memory/tui_perf.md` rule 9 ("Keep startup off data-scaled work...
First paint never waits on O(archive) work") and rule 3 ("Run slow user-initiated
operations as tracked procs").

## Goal

Return startup to at or below the 2026-08-12 baseline and make the data-scaled-work
regression structurally hard to reintroduce.

Target: median `visible_ready_seconds` <= 2.8 s and median `agents_ready_seconds` <= 3.4
s on the author's real `~/.sase` state, with `disk`-stage medians in
`tui_agent_loads.jsonl` back under 2.0 s (the slow-load logging threshold, so routine
loads stop being logged as slow at all).

## Non-goals

- Rewriting the artifact index schema or its Rust query implementation.
- Changing monitor settlement _semantics_. A monitor whose supervisor died must still be
  settled, and phantom-running rows must still disappear from the Agents tab. Only
  _when_ and _how often_ reconciliation runs changes.
- Touching `process_start_to_on_mount` (import time). It grew only 0.62 s to 0.72 s and
  is not the regression.

## Verification baseline

Capture before starting phase `guards`, and re-capture after each phase, using the
author's real `~/.sase` state:

```bash
# Disk-load profile (the harness used to produce every number above)
.venv/bin/python - <<'EOF'
import time
from sase.ace.tui.actions.agents._loading_helpers import load_agents_from_disk_with_state
from sase.ace.patch import find_all_patches_cached
from sase.ace.dismissed_agents import load_dismissed_agents
dis = load_dismissed_agents()
patches = find_all_patches_cached(include_states="all")
for i in range(3):
    t = time.perf_counter()
    r = load_agents_from_disk_with_state(set(dis), patch_snapshot=patches, source="bench")
    print(f"disk_load={time.perf_counter()-t:.3f}s agents={len(r.all_agents)}")
EOF
```

Then confirm end to end against the real TUI: launch `sase ace`, quit, and read the
newest record in `~/.sase/logs/tui_startup.jsonl`.

`SASE_TUI_LOADER_LOG_THRESHOLD_SECONDS` lowers the slow-load log threshold for short
verification runs.

## Phases

### Phase `guards` — reorder the reconcile guards

- **size**: small
- **depends on**: nothing

Reorder `should_reconcile_dead_supervisor` (`src/sase/monitor/reconcile.py:47`) so the
cheap, pure-in-memory rejects run before any I/O:

1. `record.monitor_state != "running"` -> `False`
2. `record.pid is None` -> `False`
3. `proc_shell_owns(record.monitor_id)` -> `False`
4. `_is_pre_reboot_monitor(record)` -> `True`
5. `not supervisor_is_alive(...)`

This is behavior-preserving: all three of the reordered checks are pure predicates
returning `False`, so their relative order cannot change the result — only how much work
is done before returning it.

Add a regression test in `tests/monitor/test_monitor_store_reconcile.py` asserting that
`proc_shell_owns` is **not** consulted for a record that is already terminal or has no
pid (spy/monkeypatch a counter onto the proc lookup and assert zero calls). That test is
what stops the ordering from silently regressing.

**Expected**: `reconcile_dead_supervisors` 3.12 s -> ~1.65 s. Disk load ~3.84 s -> ~2.4
s.

**Done when**: the reordering test passes, existing monitor reconcile tests still pass,
and the profile harness shows the drop.

### Phase `snapshot` — kill the N+1 proc-store reads

- **size**: medium
- **depends on**: nothing (independent of `guards`; both edit different files)

Give the proc store a way to answer many lookups from one snapshot instead of re-reading
per id.

- In `src/sase/procs/store.py`, add a snapshot-scoped lookup so a caller can read the
  store once and resolve many proc ids against it. Keep the existing `get_proc(proc_id)`
  signature working for single-shot callers — it is used widely — but implement it on
  top of the new primitive.
- In `src/sase/monitor/proc_adapter.py:49`, let `proc_shell_owns` accept an optional
  caller-supplied snapshot, falling back to a fresh read when absent.
- In `reconcile_dead_supervisors_for_records` (`src/sase/monitor/reconcile.py:80`), read
  the proc snapshot **once** for the whole pass and thread it through
  `should_reconcile_dead_supervisor`.
- Fix the sibling N+1 in `_with_proc_projection` (`src/sase/monitor/store.py:331`) the
  same way, so `list_monitors` stops doing two full store reads per record. This is a
  CLI path, so it is a correctness-of-design fix rather than a startup win — do it here
  because it shares the root cause and the new primitive.

Prefer a snapshot passed explicitly down the call chain over a module-level cache. A
cache would need invalidation against a store that other processes write to, and
`sase/memory/tui_perf.md` rule 8 warns that over-broad cache keys serve stale rows.

Add a test asserting that one `reconcile_dead_supervisors` pass over N monitor records
performs exactly one proc-store read (count binding invocations), and a test that
`list_monitors` does not scale its proc reads with record count.

**Expected**: with `guards` also landed, the residual per-record proc cost goes to zero,
and stays zero when monitors _are_ running.

**Done when**: the read-count tests pass and monitor/proc test suites are green.

### Phase `bounded-query` — stop the O(archive) index query

- **size**: medium
- **depends on**: nothing (independent of `guards` and `snapshot`)

`_project_records` (`src/sase/monitor/store.py:417`) is the residual ~1.5 s and the
growth curve. Reconciliation only ever acts on monitors that are
`monitor_state == "running"` with a live pid — a tiny, bounded set — yet it pulls the
entire hidden, full history.

Narrow the query that reconciliation uses so it retrieves only the candidate monitors it
can act on, rather than the whole archive. The `AgentArtifactIndexQueryWire` already
exposes `include_active`, `include_recent_completed`, `include_full_history`,
`active_limit`, `recent_completed_limit`, `include_hidden`, and `only_monitors`.

Do **not** narrow `_project_records` itself — `list_monitors` is a separate caller that
legitimately wants full history for `sase monitor` listings. Give reconciliation its own
bounded query path and leave the listing path alone.

Check whether the needed predicate is expressible in the existing wire query. If the
index cannot filter to running monitors directly, that filter belongs in the Rust core
per `CLAUDE.md`'s Rust core backend boundary — in which case, stop and file the
core-side change rather than reimplementing the predicate in Python. Record which route
was taken.

Add a test pinning the reconciliation query's bounds so a later edit cannot quietly
restore `include_full_history=True` / `active_limit=None`.

**Expected**: reconcile index cost ~1.5 s -> bounded and archive-independent, and the
per-load `query_agent_artifact_index` count drops from 2 to 1.

**Done when**: the query-bounds test passes, `sase monitor` listing behavior is
unchanged, and the profile shows the drop.

### Phase `off-read-path` — take reconciliation off the synchronous load

- **size**: medium
- **depends on**: `guards`, `snapshot`, `bounded-query`

Even fully optimized, reconciliation is data-scaled settlement work sitting inside the
TUI's read path on _every_ refresh and at startup. That is what
`sase/memory/tui_perf.md` rule 9 forbids, and it is why this regressed incrementally
rather than in one jump. Removing the call site is the durable fix; the earlier phases
make the deferred work cheap enough that the follow-up refresh is not itself a stall.

Remove the synchronous `_reconcile_dead_monitor_supervisors_for_tui()` call from
`_load_agents_from_disk_impl`
(`src/sase/ace/tui/actions/agents/_loading_helpers.py:431`) and run reconciliation in
the background instead, following the pattern rule 9 already prescribes: serve the load
immediately, reconcile in the background, then coalesce a follow-up refresh so settled
monitors stop showing as running.

Follow the established mechanisms rather than inventing a path — rule 1 of
`tui_perf.md`. `_schedule_loader_cleanup` / `_run_loader_cleanup`
(`src/sase/ace/tui/actions/agents/_loading_disk_support.py:192`) is the closest existing
shape: latest-wins coalescing, `spawn_pump_free_task`, a `NavigationGate` check, and
cancellation at teardown. Reuse it or mirror it exactly, including releasing the
running/pending guards if spawning fails.

Preserve the semantics the sync call provided:

- Phantom-running monitors must still disappear from the Agents tab. If the first paint
  can briefly show a monitor as running before reconciliation settles it, that is
  acceptable _only_ if the follow-up refresh reliably corrects it. Verify this specific
  transition by hand.
- Reconciliation must still run often enough that a monitor whose supervisor died is
  settled promptly, not just on the next manual refresh.
- Reconciliation writes (it settles records and can launch follow-ups). Two concurrent
  passes must not race. Keep the existing lane lock (`monitor_lane_lock_path`) honored
  and make sure the coalescing guard prevents overlapping passes from one TUI.

Add tests covering: startup load does not call reconciliation synchronously;
reconciliation still runs and settles a dead-supervisor monitor; a burst of refreshes
coalesces to a bounded number of passes.

**Expected**: reconciliation leaves the startup critical path entirely. `visible_ready`
back to at or below the 2.81 s baseline.

**Done when**: the tests pass, a real `sase ace` launch shows target `visible_ready` in
`tui_startup.jsonl`, and a monitor with a dead supervisor is confirmed by hand to settle
without a manual refresh.

### Phase `gate` — pin the win with a regression gate

- **size**: small
- **depends on**: `off-read-path`

Every prior phase is one careless call site away from returning. Add a gate.

`tests/perf/` already has the infrastructure: `bench_*.py` benches, `baselines/`, and
`check_*_regression.py` / `test_*_regression.py` checkers (see `tests/perf/README.md`).
Add a bench plus baseline for the agents disk load that asserts:

1. The disk load performs a bounded number of proc-store reads and index queries
   regardless of monitor-record count.
2. Reconciliation is not invoked on the synchronous load path.

Prefer asserting on **operation counts** rather than wall-clock seconds. Counts are
deterministic in CI; seconds are not, and a flaky perf gate gets muted, which is how the
original regression survived four days of daily use.

Also update `sase/memory/tui_perf.md` to record this incident under rule 9 — the
existing rule text cites the artifact-index example; this adds monitor reconciliation as
a second instance of the same failure mode, with the "settlement work on a read path"
framing that made it easy to miss.

**Note**: `CLAUDE.md` requires explicit user permission for memory-file edits, and that
permission has **not** been granted for this plan. The implementing agent must ask the
user before touching `sase/memory/tui_perf.md`, and must run `sase memory init`
afterward if permission is given. If permission is withheld, land the rest of the phase
and skip the memory edit.

**Done when**: the gate fails against a revert of any earlier phase, and passes on the
fixed tree.

## Risks

- **Deferring reconciliation changes observable state timing.** The Agents tab may
  briefly show a phantom-running monitor before the background pass settles it. Phase
  `off-read-path` must verify the follow-up refresh corrects it; if it cannot be made
  reliable, keep reconciliation synchronous and rely on phases
  `guards`/`snapshot`/`bounded-query` alone, which already recover most of the
  regression. Say so explicitly rather than shipping a flicker.
- **Reconciliation writes.** It settles records and can launch follow-up agents. Moving
  it off the read path must not allow two overlapping passes. The lane lock plus the
  coalescing guard are the defense; test the burst case.
- **`bounded-query` may need a Rust core change.** If the running-monitor predicate is
  not expressible in the existing wire query, the filter belongs in `../sase-core` per
  `CLAUDE.md`. That is a repo boundary crossing — file it rather than working around it
  in Python.
- **Measurements are from one machine's real state** (147 monitor records, ~101 procs,
  115 MB index, 173 agents). Ratios should hold; absolute numbers will not reproduce
  elsewhere. Re-baseline before trusting a delta.

## Verification

Per phase, as described above. Before landing the combined tree, run `just check-full`
through `/sase_monitor` (it routinely outruns a single agent turn) per `CLAUDE.md`, with
a `--next` action so the follow-up agent acts on the result.
