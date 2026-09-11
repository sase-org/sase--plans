---
tier: epic
title: Fix ACE TUI typing lag on athena (memory pressure, O(corpus) refresh work,
  unreaped scratch)
goal: 'Typing in the ACE prompt input stays responsive on athena under normal agent
  load: the long-lived `sase ace` process holds a bounded heap instead of growing
  to ~12.5 GB, its periodic refresh work stops re-doing O(corpus) index and notification
  passes, and SASE scratch stops filling a RAM-backed /tmp and a Syncthing-synced
  SASE_TMPDIR.

  '
phases:
- id: host-relief
  title: Reclaim athena now and move SASE_TMPDIR off tmpfs and out of Syncthing
  depends_on: []
  size: small
  description: 'host-relief: reclaim the two 31 GB scratch piles on athena, relocate
    SASE_TMPDIR out of the Syncthing-synced tree, and capture a documented before/after
    responsiveness baseline the later phases measure against.

    '
- id: index-core-sql
  title: Replace the artifact-index N+1 reconcile and full dismissed-table rewrite
  depends_on: []
  size: medium
  description: 'index-core-sql: in the linked sase-core repo, turn the per-candidate
    reconcile queries into set-based SQL and make the dismissed-identity projection
    a diff-based upsert instead of an unconditional 46k-row table replace.

    '
- id: notif-snapshot
  title: Stop re-parsing the whole notification store on every refresh tick
  depends_on: []
  size: medium
  description: 'notif-snapshot: cache the parsed notification snapshot against a cheap
    change token and bound the active notifications.jsonl so a 13.6 MB re-parse stops
    running on the TUI refresh cadence.

    '
- id: stream-bound
  title: Bound retained child-process output in the session proc reporter
  depends_on: []
  size: small
  description: 'stream-bound: cap the unbounded in-memory output list in _stream_subprocess
    and stop mirroring the whole stream into operation-request.json records that reached
    40 MB each.

    '
- id: index-tui-cadence
  title: Narrow the artifact-index lock and stop authoritative syncs bypassing the
    signature check
  depends_on:
  - index-core-sql
  size: medium
  description: 'index-tui-cadence: keep interactive index reads off the long maintenance
    lock and let authoritative dismissed-projection syncs short-circuit on an unchanged
    signature the same way non-authoritative ones already do.

    '
- id: tmp-hygiene
  title: Extend scratch hygiene to agent-created build directories and disk pressure
  depends_on:
  - host-relief
  size: medium
  description: 'tmp-hygiene: give agent-created cargo/build scratch a managed home
    the reaper can actually see, and add size-aware pressure reaping so a multi-gigabyte
    directory is not held for a purely time-based horizon.

    '
- id: heap-attrib
  title: Attribute and fix the residual ACE heap growth
  depends_on:
  - host-relief
  - stream-bound
  size: medium
  description: 'heap-attrib: add an opt-in heap sampler for the long-lived TUI, attribute
    whatever remains of the ~12.5 GB anonymous heap after the bounded-stream fix,
    and fix the retained-object sites it names.

    '
- id: verify
  title: Re-measure on athena against explicit responsiveness targets
  depends_on:
  - index-tui-cadence
  - notif-snapshot
  - tmp-hygiene
  - heap-attrib
  size: small
  description: 'verify: re-run the documented capture recipe on athena under real
    agent load and confirm the keystroke, CPU, and RSS targets hold, recording the
    result in the perf runbook.'
proposed_by: bbugyi200.kellys_mbp.03
create_time: 2026-09-11 12:20:17
status: wip
bead_id: sase-zn
---

- **PROMPT:** [prompts/202609/ace_typing_lag_athena.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/ace_typing_lag_athena.md)
- **BEAD:** [sase-zn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zn/README.md)

# Plan: Fix ACE TUI typing lag on athena

## Verdict on the `/tmp` hypothesis

**Partly confirmed — the observation is right, the mechanism is not, and it is not the
whole cause.**

`/tmp` on athena really is nearly full: 31 GB used of 32 GB, 97%. But the harm is not
"disk full". `/tmp` on athena is a **tmpfs**:

```
$ findmnt /tmp
TARGET SOURCE FSTYPE OPTIONS
/tmp   tmpfs  tmpfs  rw,nosuid,nodev,size=32856300k,nr_inodes=1048576,inode64
```

tmpfs is RAM. Those 31 GB are charged against the machine's 62 GB of memory, and because
`Shmem` in `/proc/meminfo` reads only 2288 kB, essentially all of that tmpfs content has
already been pushed out to swap. Swap is 40.3 GB used of 64 GB. So filling `/tmp` on
this host does not risk failed writes so much as it evicts everything else, including
the TUI, into swap.

That matters, but it is one of three compounding causes, and it is not the largest one.
The single biggest memory consumer on athena is `sase ace` itself.

## What was measured (2026-09-11, athena)

Host state:

- Load average `12.23 / 19.41 / 31.01`; 8+ concurrent `rustc`/`clippy-driver` processes
  from in-flight agent verification runs.
- `/tmp`: tmpfs, 31 G/32 G used, 15,907 entries, of which 864 are `sase*` and 20 are
  cargo target directories (~23 G).
- `/` (ext4): 875 G, 97% used, 31 G free.
- Swap: 40.3 G of 64 G used.

The `sase ace` process (PID 3535215, up 1 day):

- `VmRSS: 12858884 kB` (~12.5 GB), `VmPeak: 18966084 kB`, `VmSwap: 4617100 kB`.
- `smaps_rollup` shows `Anonymous: 12493236 kB` and `Private_Dirty: 12493236 kB` with
  `Pss_File: 3105 kB` — this is Python heap, not mapped files.
- Main-thread CPU: `utime` 1,705,087 ticks at 100 Hz = **4.7 CPU-hours in 24 hours** of
  wall time. Every other thread is negligible (next highest: 2.2 s).

A 25-second `py-spy record` (1933 samples, 1691 of them on worker threads):

| share of active samples | site                                                                                                 |
| ----------------------- | ---------------------------------------------------------------------------------------------------- |
| 41.0%                   | `reconcile_agent_artifact_index_dismissed_family_members` (`src/sase/core/agent_scan_facade.py:279`) |
| 22.3%                   | `read_current_notifications_snapshot` (`src/sase/core/notification_store_facade.py:48`)              |
| 8.0%                    | main-thread event loop                                                                               |
| 4.2%                    | `scan_agent_artifacts` (`src/sase/core/agent_scan_facade.py:119`)                                    |

The always-on stall watchdog (`~/.sase/logs/tui_stalls.jsonl`) recorded 354 events in
the last 7 days — 138 hitches, 14 full stalls, 23 pump hitches — with individual
recoveries of 2.2 s, 2.5 s and 4.4 s. In 189 of them the main thread was parked in
`selectors.poll`, i.e. the loop was not running Python at all: the process was not being
scheduled. That is the signature of swap and CPU starvation, not of a slow handler.

Corpus sizes behind the two hot sites:

- `~/.sase/agent_artifact_index.sqlite` is 205 MB: `agent_artifacts` 10,793 rows,
  **`dismissed_agents` 46,762 rows**, `agent_artifact_model_aliases` 3,397 rows.
- `~/.sase/notifications/notifications.jsonl` is **13.6 MB across 1,505 entries** (~9 KB
  per notification), with a further 31 MB archive alongside.

## Root causes

### 1. The ACE process holds a ~12.5 GB anonymous heap, 4.6 GB of it swapped out

This is the direct cause of the typing lag. When 4.6 GB of the interpreter's own pages
live in swap and the NVMe is saturated by eight concurrent `rustc` processes, a
keystroke that touches a cold page waits on major faults. The watchdog stacks agree: the
loop is stalled while parked in `poll`.

One concrete unbounded accumulator is already identifiable. `_stream_subprocess`
(`src/sase/ace/tui/session_proc_reporter.py:33`) retains every line of a child process's
output:

```python
output_chunks: list[str] = []          # :45
...
        for raw in process.stdout:
            with output_lock:
                output_chunks.append(raw)   # :66
            on_line(raw.rstrip("\r\n"))
```

There is no cap. The stream is also mirrored into the proc log by `_handle_line`, so the
same bytes are retained twice. Because the pipe is opened with `text=True`, Python
applies universal-newline translation, so cargo's `\r` progress repaints are each
delivered as a separate line and each one is appended. A live `py-spy dump` caught
exactly this path running a comprehensive update:

```
_stream_subprocess (sase/ace/tui/session_proc_reporter.py:86)
run_recorded_command (sase/dev_update/command.py:84)
execute_tui_dev_update (sase/ace/tui/modals/plugins_browser_dev_update.py:164)
run_scoped_update (.../plugins_browser_comprehensive_update_execution.py:295)
```

The same bloat is visible on disk: `~/.sase/procs/runtime/*/operation-request.json`
records from today are 21–40 MB each, several per hour.

Whether that single site accounts for all 12.5 GB is **not yet established**, and this
plan does not assume it does. `heap-attrib` measures it rather than guessing, per the
"Measure, don't guess" rule in `sase/memory/tui_perf.md`.

### 2. Periodic refresh work redoes two O(corpus) passes

`sase/memory/tui_perf.md` rule 10 says ticks revalidate and recomputes get a much longer
cadence. Two paths violate that in practice on a fleet this size.

**Artifact-index dismissed-projection maintenance.** Each pass runs `_sync_projection`
(`src/sase/core/agent_artifact_index_lifecycle.py:169`), which calls
`replace_agent_artifact_index_dismissed_agents` — a full replace of all 46,762 rows —
and then `reconcile_agent_artifact_index_dismissed_family_members` (`:204`). In the
linked `sase-core` repo, that reconcile (`crates/sase_core/src/agent_scan/index.rs:892`)
is an N+1 loop: for every candidate row it issues `record_is_dismissed(&conn, ...)` and
`family_root_dismissed_for_candidate(&conn, ...)` as separate queries. Against a 205 MB
index that is tens of thousands of statements per pass, and it is 41% of the TUI's CPU.

The Python side is architecturally correct — `_index_maintenance.py:129` uses
`asyncio.to_thread`, honours the nav gate, and coalesces via `spawn_pump_free_task` —
and the PyO3 binding (`crates/sase_core_py/src/lib.rs:3352`) correctly wraps the call in
`py.allow_threads`, so the GIL _is_ released. The cost is therefore real CPU and disk
contention rather than a blocked event loop. Two things still make it reach the UI:

- The whole operation runs inside `agent_artifact_index_operation_lock()`
  (`src/sase/core/agent_scan_facade.py:270`), a process-wide lock that _every_ index
  facade call takes, including interactive reads. `tui_perf.md` rule 11 explicitly
  forbids keystroke paths taking unbounded shared-store locks.
- When the TUI passes a non-empty `added` set (`_loading_apply.py:317-322`),
  `_sync_projection` sets `authoritative = True` and **skips the
  `_projection_metadata_matches` short-circuit entirely**, so an unchanged projection is
  still replaced and reconciled in full.

**Notification polling.** `_poll_agent_completions_once` is documented as "Called on
every auto-refresh regardless of current tab"
(`src/sase/ace/tui/actions/agents/_notification_polling.py:72`), reached from
`event_refresh/_auto_refresh.py:302`. The default refresh interval is 10 s
(`src/sase/ace/tui/app.py:293`). Each call lands in
`read_current_notifications_snapshot`, which re-reads and re-parses the entire 13.6 MB
`notifications.jsonl`. That is 22% of the TUI's CPU and, at ~9 KB per notification, a
large amount of short-lived object churn every ten seconds.

### 3. SASE scratch accumulates where nothing reaps it

`managed_tmp_reaper` exists and is wired to the hourly `housekeeping` lumberjack
(`src/sase/default_config.yml:1173`, `src/sase/scripts/sase_chop_managed_tmp_reap.py`).
It has two coverage gaps that together produced both 31 GB piles.

**It cannot see `/tmp`, and that is where agents actually write.** The reaper
deliberately refuses broad roots:

```python
_UNSAFE_REAP_ROOTS = frozenset(
    Path(path).resolve() for path in ("/", "/tmp", "/var/tmp")
)                               # src/sase/core/managed_tmp_reaper.py:111
```

That refusal is correct — nothing should blanket-delete `/tmp` children. But nothing
steers agents away from `/tmp` either, so 20 directories named
`/tmp/sase-cargo-target-sase_20-main`, `/tmp/sase10-cargo-target`,
`/tmp/sase-yy-8-6-1-core-target` and similar now hold ~23 GB of RAM-backed build output
that no SASE mechanism will ever reclaim. The repo's own `CARGO_TARGET_DIR` usage is
well-behaved and targets the sase-core checkout (`Justfile:967,976,1025`), so these come
from ad-hoc agent commands, not from committed recipes.

**Its horizons are purely time-based.** `sase-yh4-cargo-target` (23 GB, mtime
2026-09-09) and `sase-yh3-cargo-target` (7.6 GB, mtime 2026-09-08) are stray top-level
entries in the managed root, so they fall under
`DEFAULT_HORIZON_SECONDS = HANDOFF_HORIZON_SECONDS` — **3 days**. Holding 31 GB for
three days is fine on a spacious volume and not fine on one that is 97% full.

**And `SASE_TMPDIR` itself points somewhere actively harmful.** On athena:

```
SASE_TMPDIR=/home/bryan/tmp/sase
/home/bryan/tmp -> Sync/home/tmp          # a Syncthing-synced folder
```

So the managed SASE temp root resolves inside a Syncthing share. 31 GB of cargo build
artifacts are being continuously scanned and replicated; `syncthing` has held ~21% CPU
for the full 5-day uptime. This is host configuration, not repo code — it lives in the
chezmoi-managed environment — but it is a first-order contributor and is fixed in
`host-relief`.

Secondary, lower-priority disk pressure in `~/.sase` (24 GB total): a stale 282 MB
`perf/tui_trace.jsonl` (last written 06:24, tracing is not currently enabled —
`SASE_TUI_TRACE` is absent from the live process environment), several full
`cache/rust-prebuild` target sets with 330 MB `dep-graph.bin` files, a 182 MB telemetry
database, and ~50 MB lumberjack logs.

## Why this presents specifically as prompt-input lag

Nothing in the prompt-input keystroke path is itself slow, and the countdown tick
already gates correctly on typing:

```python
if (
    not self._nav_gate.is_navigating(now_mono=now_mono)
    and not self._prompt_input_active()
):                      # src/sase/ace/tui/actions/_event_countdown.py:49-52
```

The lag is felt most in the prompt input because that is the surface where a human
notices per-character latency at all. j/k navigation repaints immediately from cache and
a 150 ms detail-panel debounce hides small stalls; a 2–4 second gap between pressing a
key and seeing the character is unmissable. The cause is the process not being scheduled
promptly — swap faults plus a machine at load 12–31 — not a slow key handler.

The user's stated expectation ("no lag unless the machine is under serious load") is the
right standard. The machine _is_ under serious load, and ACE is one of the larger
contributors to it.

## Non-goals

- Reducing the number of concurrent agents or their `rustc` builds. Athena is meant to
  run them; the TUI should stay responsive alongside them.
- Blanket-reaping `/tmp`. The existing `_UNSAFE_REAP_ROOTS` refusal stays.
- Resizing the athena tmpfs. Moving SASE's own scratch off it is the fix; re-sizing the
  host mount is the user's call and is not assumed here.
- Any change to the Textual version, the widget tree, or the keymap.

## Discovered defects to file separately as task beads

Both are out of scope for this epic; the `verify` phase agent should file them with
`/sase_new_task` rather than fixing them here.

1. `sase repo open sase-core` fails with
   `Unknown repo 'sase-core' for project 'gh_sase-org__sase'. Valid repos: gh_sase-org__sase`,
   while `sase repo list` in the same workspace lists `sase-core` as a linked, cloned
   repo. The two commands resolve against different inventories.
2. `~/.sase/procs/runtime/*/operation-request.json` records reaching 21–40 MB each.
   `stream-bound` addresses the upstream cause, but these records also have no retention
   policy of their own.

## Phase 1 — `host-relief`: Reclaim athena now and relocate SASE_TMPDIR

Immediate relief, and the baseline everything else is measured against.

1. Capture the "before" numbers so the epic has a reference point: `free -h`,
   `swapon --show`, `df -h /tmp /`, the `sase ace` process's `VmRSS`/`VmSwap` from
   `/proc/<pid>/status`, and a 25 s `py-spy record`. Save them under
   `docs/perf_runbook.md`'s capture conventions.
2. Reclaim the `/tmp` tmpfs. Remove only the cargo/build scratch directories that match
   the observed SASE agent naming (`/tmp/*cargo-target*`, `/tmp/*core-target*`, the
   `sase-*-recovery*` bundles) **after** confirming with the user that no agent is
   mid-build in one. Do not touch `systemd-private-*`, `snap-private-tmp`, or anything
   not clearly SASE scratch. Propose the removal through `/sase_gate` rather than
   running it unprompted.
3. Reclaim the managed root's two stray cargo targets (~31 GB) the same way.
4. Relocate `SASE_TMPDIR` out of the Syncthing tree. The variable is set in the
   chezmoi-managed environment, not in this repo — `grep -rn SASE_TMPDIR` finds no
   producer here, only the consumer at `src/sase/core/paths.py:100`. Open the `chezmoi`
   linked repo with `/sase_repo` and point `SASE_TMPDIR` at a local, non-synced,
   non-tmpfs path (for example `~/.cache/sase/tmp`). Confirm with the user before
   changing their environment.
5. Delete the stale 282 MB `~/.sase/perf/tui_trace.jsonl`.
6. Restart `sase ace` and re-capture. Record the delta.

Expected: swap pressure drops sharply and typing becomes responsive again even before
any code change lands. That is the confirmation that the memory-pressure diagnosis is
correct, and it is why this phase blocks `heap-attrib` — heap measurement is meaningless
while the host is thrashing.

## Phase 2 — `index-core-sql`: Set-based reconcile and diff-based projection

Work in the linked `sase-core` repo. Per `sase/memory/rust_core_backend_boundary`, this
is core backend behavior and belongs there, not in a Python adapter.

1. Rewrite `reconcile_agent_artifact_index_dismissed_family_members`
   (`crates/sase_core/src/agent_scan/index.rs:892`) so the per-candidate
   `record_is_dismissed` and `family_root_dismissed_for_candidate` calls become one
   set-based query (a join or a `WHERE ... IN`/`EXISTS` against the dismissed and family
   tables) instead of two statements per candidate row. The decode and liveness filters
   that need the deserialized `AgentArtifactRecordWire` can stay in Rust, but they must
   run over a single result set.
2. Make `replace_agent_artifact_index_dismissed_agents` diff-based. Replacing 46,762
   rows to change a handful is the dominant write cost. Compute the added/removed
   identity sets and issue only those statements, inside the existing transaction. Keep
   the full-replace path available behind the existing `force` semantics.
3. Confirm the indexes the new queries need exist on `dismissed_agents` and the
   family/parent columns; add them in the schema migration if not, and bump
   `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` if the migration requires it.
4. Preserve the corruption-detection contract: `_CorruptArtifactIndexError` still has to
   be raised for `database disk image is malformed` and not for transient lock errors,
   because `src/sase/core/agent_artifact_index_lifecycle.py` keys its
   quarantine-and-heal behavior on it.
5. Tests: extend the existing reconcile tests
   (`crates/sase_core/src/agent_scan/index.rs:7739,7767,7857`) with a large-fixture case
   — on the order of 40k dismissed rows and 10k artifact rows — asserting both identical
   results to the old implementation and a statement count that does not scale with
   candidate count.

Target: the reconcile stops being the top CPU consumer in a `py-spy record` of an idle
TUI.

## Phase 3 — `notif-snapshot`: Stop re-parsing the notification store per tick

1. Give `read_current_notifications_snapshot`
   (`src/sase/core/notification_store_facade.py:48`) an mtime+size change token and
   memoize the parsed snapshot against it, following the established pattern in
   `tui_perf.md` rule 8 (`current_config_token()` and the memoized model-alias
   resolution). A tick that finds an unchanged token must not re-read the file. Note
   that rule 8 also warns that over-broad cache keys serve stale rows — key on the store
   path, the `include_dismissed` flag, and the change token together.
2. Bound the active store. `notifications-archive.jsonl` already exists (written from
   `src/sase/logs/collectors.py`), so archiving is established; the active file is still
   13.6 MB across 1,505 entries. Add compaction that moves resolved/dismissed
   notifications past a retention horizon into the archive, and run it from the same
   hourly `housekeeping` lumberjack that owns `managed_tmp_reap` — never from an
   interactive path.
3. Check whether the ~9 KB average entry is itself carrying payload that does not belong
   in the active store (the first record carries `notes` and `files` arrays). If large
   payloads can be referenced rather than inlined, note it as a follow-up bead; do not
   expand this phase to restructure the record.
4. Tests: assert an unchanged store produces no re-parse across repeated snapshot reads,
   that a changed store does, and that compaction preserves every unread and actionable
   notification.

## Phase 4 — `stream-bound`: Bound retained child-process output

1. Replace the unbounded `output_chunks` list in `_stream_subprocess`
   (`src/sase/ace/tui/session_proc_reporter.py:45`) with a bounded retention policy — a
   head+tail ring buffer is the right shape, since callers want the command's opening
   context and its failure tail, not its middle. Pick an explicit cap (bytes, not lines)
   and record in the docstring that the returned `CompletedProcess.stdout` is now
   potentially elided, with a marker line where content was dropped.
2. Audit the callers that consume `completed.stdout` for correctness under elision:
   `subprocess_run_fn` (`:203`), `uv_runner` (`:240`), `command_runner` (`:252`), and
   `dev_command_runner` (`:269`). Any caller that parses the full stream — the JSON
   check payloads mentioned in the `run` docstring are the case to watch — must either
   opt out of elision explicitly or be routed to a spill file instead of memory.
3. Stop persisting whole streams into `~/.sase/procs/runtime/*/operation-request.json`.
   Records of 21–40 MB are the on-disk symptom of the same accumulation.
4. Consider whether `text=True` should become byte-mode plus explicit newline handling,
   so `\r` progress repaints stop being counted as distinct lines. If that changes
   observable proc-log output, keep it behind the existing log path rather than altering
   what users see mid-epic.
5. Tests: a child emitting far more than the cap returns an elided, correctly-marked
   result with bounded memory, and a child emitting less than the cap is byte-identical
   to today.

## Phase 5 — `index-tui-cadence`: Narrow the lock, restore the short-circuit

Depends on `index-core-sql`, because shortening the critical section is only safe once
the work inside it is bounded.

1. Let authoritative syncs short-circuit. In `_sync_projection`
   (`src/sase/core/agent_artifact_index_lifecycle.py:169`) the `authoritative` branch
   skips `_projection_metadata_matches` entirely. Allow the signature check to apply to
   authoritative calls too when the incoming `added` set is already reflected in the
   stored projection metadata, so an unchanged projection costs a metadata read instead
   of a full replace plus reconcile. Keep `force=True` as the unconditional escape
   hatch.
2. Keep interactive reads off the maintenance lock. Long-running maintenance should not
   hold `agent_artifact_index_operation_lock()`
   (`src/sase/core/agent_scan_facade.py:270`) against interactive readers.
   `try_agent_artifact_index_operation_lock(timeout_seconds)` already exists in
   `src/sase/core/agent_artifact_index_lock.py:20` — route read-only facade calls on
   UI-reachable paths through a bounded wait that degrades to the cached snapshot rather
   than blocking, per `tui_perf.md` rule 11. Writers keep the blocking lock.
3. Re-check the cadence once 1 and 2 land. Both schedule sites (`_loading_apply.py:317`,
   `_loading_disk_support.py:346`) are already gated on a real dismissed-set change, so
   no new throttle should be needed; confirm that with a trace rather than adding one
   pre-emptively.
4. Tests: an unchanged authoritative sync performs no replace and no reconcile; a
   changed one still does both; a UI read during a held maintenance lock returns cached
   data within the timeout instead of blocking.

## Phase 6 — `tmp-hygiene`: Managed scratch for build output, and pressure reaping

1. Give agent build scratch a managed home. Add a helper alongside
   `get_sase_managed_tmpdir` that returns a per-workspace cargo/build target directory
   under a new managed subdirectory (`build-targets/`), and register it in
   `MANAGED_TMPDIR_HORIZONS` (`src/sase/core/managed_tmp_reaper.py:53`) with its own
   horizon. Build output is regenerable, so it deserves a much shorter horizon than the
   3-day `DEFAULT_HORIZON_SECONDS` it currently falls under as a stray entry.
2. Steer agents to it. The `/tmp/sase-*-cargo-target` directories come from ad-hoc agent
   commands, so the fix is guidance plus an obvious default, not enforcement: document
   the managed path in `docs/rust_backend.md` next to the existing `CARGO_TARGET_DIR`
   examples (`:390-391`), and surface it wherever agents are told how to build
   sase-core. Consult `/sase_memory_write` before changing any `sase/memory/` note as
   part of this.
3. Add size-aware pressure reaping. A purely time-based horizon held 31 GB on a 97%-full
   volume for three days. When free space on the managed root's filesystem falls below a
   threshold, reap the largest regenerable entries ahead of their normal horizon,
   oldest-first within that set, keeping the existing `DEFAULT_MAX_REMOVALS` budget so
   one invocation stays bounded. `build-targets/` is the only bucket that should be
   eligible for early reaping; handoff and run-artifact buckets keep their horizons.
4. Leave `_UNSAFE_REAP_ROOTS` alone. Do not add `/tmp` reaping. The goal is that nothing
   SASE-owned is written there in the first place.
5. Tests: extend `tests/test_managed_tmp_reaper.py` with a `build-targets/` horizon case
   and a low-free-space case asserting early reaping of large regenerable entries and no
   early reaping of handoff or run-artifact buckets.

## Phase 7 — `heap-attrib`: Attribute and fix the residual heap growth

Depends on `host-relief` (so measurement is not drowned in swap noise) and
`stream-bound` (so the one already-identified accumulator is out of the way).

1. Add an opt-in heap sampler for the long-lived TUI, modelled on the existing
   `SASE_TUI_PERF` / `SASE_TUI_TRACE` switches: behind `SASE_TUI_HEAP=1`, take periodic
   `tracemalloc` snapshots on a long interval and append the top allocation sites to
   `~/.sase/perf/tui_heap.jsonl`. It must obey `tui_perf.md` rule 2 — the sampler body
   runs in a pump-free task via `spawn_pump_free_task()`, never directly in a timer
   callback — and must be entirely inert when the flag is unset.
2. Run it on athena against a genuinely long-lived session and attribute the residual
   anonymous heap. The measured starting point is 12.5 GB anonymous / 18.9 GB peak after
   one day; record what fraction `stream-bound` removed and what remains.
3. Fix the retained-object sites the attribution names. Do not pre-commit to a specific
   fix here — the point of this phase is that the remaining leak is identified by
   measurement first. If attribution shows the heap is already bounded after
   `stream-bound`, say so and close the phase; that is a valid outcome, not a failure.
4. Add a regression guard: a slow-marked test that drives a simulated long-running
   session and asserts retained-object growth stays under an explicit bound.

## Phase 8 — `verify`: Re-measure on athena under real load

1. Re-run the `host-relief` capture recipe on athena while agents are actually running,
   and compare against the recorded baseline.
2. Targets, all measured with the machine doing real work:
   - `sase ace` steady-state RSS stays bounded over a multi-day session — no path from
     ~400 MB to 12.5 GB.
   - Main-thread CPU well under the measured 4.7 CPU-hours per 24 hours.
   - No `reconcile_agent_artifact_index_dismissed_family_members` or
     `read_current_notifications_snapshot` frame in the top few entries of a 25 s
     `py-spy record` of an idle TUI.
   - `SASE_TUI_PERF=1` p95 key-to-paint under 16 ms on every tab, per the standing
     target in `sase/memory/tui_perf.md`.
   - New `tui_hitch` / `tui_stall` events in `~/.sase/logs/tui_stalls.jsonl` drop to
     roughly zero during a normal session. The 7-day baseline was 354 events.
   - `/tmp` holds no SASE-created build scratch; the managed root stays bounded under
     agent load.
3. Record the capture-and-compare recipe in `docs/perf_runbook.md` so this is
   reproducible next time, and note the idle-host CPU diet section already there.
4. File the two discovered defects listed above as task beads via `/sase_new_task`.

## Verification (every phase)

- `just check` is the per-agent default; `just check-full` gates landing, per
  `sase/memory/decisions:two-speed-verification`. Read `sase/memory/lint_and_test.md`
  with `/sase_memory_read` before finishing any phase that touched a git-tracked file.
- Phases touching the linked `sase-core` repo must open it with `/sase_repo`, update the
  Rust wire/API and its tests there, then update the Python callers or adapters in this
  repo, per `sase/memory/rust_core_backend_boundary`.
- Any phase whose change affects TUI responsiveness re-reads `sase/memory/tui_perf.md`
  first and re-runs the benches named there:
  `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` and
  `pytest -s -m slow tests/perf/bench_tui_trace.py`.
- Changes to files under `sase/memory/` require `/sase_memory_write` first.
- Host-side remediation on athena that deletes data or edits the user's environment goes
  through `/sase_gate` for confirmation, never straight to a destructive command.

## Risks

- **Reclaiming scratch could delete something an in-flight build needs.** Mitigated by
  confirming with the user and gating the removal, and by never touching non-SASE `/tmp`
  entries.
- **Diff-based projection updates can drift from the source of truth** in a way a full
  replace cannot. Mitigated by keeping `force` as an unconditional full-replace path and
  by asserting equivalence against the old implementation on a large fixture.
- **Bounded stream retention can break a caller that parses full output.** Mitigated by
  the explicit caller audit in `stream-bound` step 2; a caller that genuinely needs the
  whole stream gets a spill file, not a silent truncation.
- **Notification caching can serve stale rows** if the change token is too coarse — the
  failure mode `tui_perf.md` rule 8 calls out by name. Mitigated by keying on path,
  flag, and mtime+size together, plus an explicit staleness test.
- **The heap attribution may find no single culprit.** That is why `heap-attrib` is
  scoped to measure first and is allowed to close with "already bounded" as its result.
