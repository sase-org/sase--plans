---
tier: epic
title: Bound the runner-slot capacity scan and make the TUI GC policy cost-adaptive
goal: 'The sase TUI and waiting agent runners stop burning CPU and memory on full-archive
  capacity scans: a warm capacity scan costs milliseconds instead of 23 s, the TUI
  holds under 15% of one core and a bounded heap on athena, and full GC collections
  respect a duty-cycle bound.

  '
phases:
- id: core-capacity-scan
  title: 'sase-core: capacity-only scan cost scales with live records, not archive
    size'
  depends_on: []
  size: large
  description: 'core-capacity-scan: in the linked sase-core repo, add a statx-only
    live-marker precheck plus meta-light closure classification (and, per the verdict_cache
    decision, a sticky never-live verdict cache) to the capacity_only scanner path,
    with parity tests and a scan-cost bench guarding identical retained records.

    '
- id: sase-adopt-pin
  title: Ratchet the core pin and re-measure the scan on the real archive
  depends_on:
  - core-capacity-scan
  size: small
  description: 'sase-adopt-pin: move sase-core-revision.txt past the core commit,
    adopt any new scan option in runner_slots/_scan.py, and re-measure cold/warm capacity
    scans and runner-cache behavior on athena against the 22.9 s baseline.

    '
- id: gc-duty-cycle
  title: 'TUI GC policy: full collections respect a duty-cycle bound'
  depends_on: []
  size: medium
  description: 'gc-duty-cycle: record each full collection''s duration in gc_policy.py
    and defer the next full collection by last-pause times a duty factor (clamped),
    applying the floor to idle and backstop triggers alike, with fake-clock unit tests.

    '
- id: athena-soak
  title: Measured verification on athena and residual-leak attribution
  depends_on:
  - sase-adopt-pin
  - gc-duty-cycle
  size: medium
  description: 'athena-soak: soak a TUI instance on athena for at least an hour under
    real agent load, verify CPU, rchar, RSS, GC-pause, and runner-CPU targets against
    the 2026-11-12 baseline, and attribute any residual RSS growth with the GC census.

    '
decisions:
  verdict_cache:
    ask: Should the capacity scan persist a sticky never-live verdict cache keyed
      on artifact-dir mtime?
    choices:
      persist: Warm rescans drop to ~ms for 20k+ dirs; adds one cache file plus invalidation
        rules to sase-core.
      memory_only: No new persisted state, but every process repays a cold full-archive
        meta pass (seconds).
    default: persist
    why: Verdicts are sticky, the archive grows ~4.5k dirs/month, and 26+ runner processes
      pay cold passes.
    answer: persist
  live_tui_restart:
    ask: May the soak phase propose a gated restart of Bryan's live tmux TUI session
      after the soak passes?
    choices:
      separate_instance: Soak a dedicated TUI instance only; Bryan restarts his own
        session when convenient.
      restart_live: After a passing soak, propose a gated quit-and-relaunch of the
        live session.
    default: separate_instance
    why: Bryan may be using the session, and stale_running_code.py already surfaces
      the restart row.
    answer: separate_instance
proposed_by: bbugyi200.athena.1c0
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 16:44:56
status: wip
bead_id: sase-1sl
---

- **PROMPT:** [prompts/202611/capacity_scan_tui_runaway.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/capacity_scan_tui_runaway.md)
- **BEAD:** [sase-1sl](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1sl/README.md)

# Fix the sase TUI CPU/memory runaway: bound the runner-slot capacity scan and make the TUI GC policy cost-adaptive

## Problem

The long-lived `sase tui` process on athena burns more than a full core continuously and
grows to multi-GB RSS. On 2026-11-12 a freshly restarted TUI reached ~119% average CPU,
4.1 GB RSS, and 53 GB cumulative `read()` traffic within 15 minutes of starting
(`/proc/<pid>/io` rchar; ~60 MB/s sustained). The same mechanism makes WAITING/QUEUED
`run_agent_runner.py` processes burn 10–100% CPU each for days. This was reported by the
consolidated host research report `athena_cpu_spikes_swap_and_runaway_sase.md`
(bbugyi200/tailnet--research, 202611) and has been independently re-verified and
root-caused as follows.

## Verified root cause (all numbers measured live on athena, 2026-11-12)

**1. The runner-slot capacity scan walks the entire artifact archive.**

- `collect_runner_slot_records()` (`src/sase/core/runner_slots/_scan.py:32`) calls the
  Rust `scan_agent_artifacts` binding with `RUNNER_SLOT_SCAN_OPTIONS`
  (`capacity_only=True`, `only_workflow_dirs=("ace-run",)`).
- In sase-core (`crates/sase_core/src/agent_scan/scanner.rs`), `capacity_only` skips a
  candidate dir early only when `done.json` exists (scanner.rs:136). But **18,731 of
  athena's 22,662 ace-run dirs (83%) have no `done.json`** — most historical runs never
  wrote one (verified per-month back to 202607) — so every scan fully parses
  `agent_meta.json`, `running.json`, `waiting.json`, and `workflow_state.json` in all
  18,731 dirs (~75k file opens, confirmed by strace), builds full records, and then
  `retain_capacity_records_for_admission` (scanner.rs:437) throws away all but ~175.
- Measured cost: **one capacity-only scan = 22.9 s of CPU** (99% user time, not I/O
  wait) to return 175 records.

**2. The scan runs constantly, host-wide.**

- Every claim/release/waiting-marker mutation bumps `~/.sase/runner_slots.token`
  (`src/sase/core/runner_slots/_signal.py`); with ~26 waiting runners the token churns
  every few seconds.
- A token change invalidates the shared scan cache (`~/.sase/runner_slots.scan.json`)
  for every consumer:
  - Waiting runners poll every 2 s (`src/sase/axe/run_agent_wait_slots.py:105`,
    `max_age=2.0`) → effectively continuous rolling 23 s scans across runner processes
    (the cache file is rewritten every few seconds, verified by mtime sampling).
  - The TUI's 10 s machine-capacity feed
    (`src/sase/ace/tui/actions/event_refresh/_auto_refresh.py:116`,
    `_build_machine_capacity_report` → `machine_capacity_report()`) triggers a display
    rescan at most every 120 s (`src/sase/core/runner_slots/_machine_capacity.py:32`),
    i.e. a ~23 s scan every 2 minutes ≈ 19% of a core, plus the report rebuild on every
    token change.
- py-spy over 60 s of the live TUI: **39% of all samples inside `scan_agent_artifacts`
  under `_build_machine_capacity_report`**, 12% in the agent-list disk load, 7% in
  artifact-index queries, 4.6% in `gc_policy.tick`.

**3. The allocation churn makes the TUI GC policy pathological.**

- Each scan materializes hundreds of thousands of transient objects.
  `~/.sase/logs/tui_stalls.jsonl` `tui_gc_pause` rows show full gen-2 collections every
  ~60 s (`MIN_FULL_INTERVAL_S` in `src/sase/ace/tui/util/gc_policy.py`) each taking
  **1.2–2.3 s and collecting 300k–620k objects**, and even "automatic" gen-0/gen-1
  pauses of 50–96 ms. On the previous, swap-pressured instance (5.3 GB RSS, 940k major
  faults) full collections page through the heap, which matches the report's "half the
  time in `gc_policy.tick`".
- The policy has no cost feedback: it schedules the next full collection purely by
  elapsed time/gen counts, so a heap where `gc.collect()` takes seconds still gets swept
  every minute.

**4. RSS growth.** Heartbeats show 1.5 GB → 3.6 GB → 4.1 GB within 15 minutes; the prior
instance plateaued ~5.3 GB. The dominant inflator is the scan churn (transient Rust
payload + wire-record decode + 1 MB cache JSON re-parses); `malloc_trim` runs only every
10 min. Whether any true retention leak remains must be re-measured after the churn is
fixed (phase `athena-soak`).

## Scope boundary with in-progress epics

- `sase-zu` (agent queries stop escalating to full-archive scans) owns the **Agents-tab
  index/load** path. This epic owns the **runner-slot capacity scan** (`capacity_only`)
  path, which deliberately stays a lock-free filesystem walk and must remain
  index-independent for admission.
- `sase-zn` (typing lag / heap) owns prompt-input responsiveness, notification
  memoization, and /tmp scratch. This epic only touches
  `src/sase/ace/tui/util/gc_policy.py` on the TUI side.
- `sase-132` owns startup time. Do not add work to the startup path.

## Phases

### Phase `core-capacity-scan` — sase-core: capacity-only scan cost scales with live records, not archive size

- **size:** large
- **depends on:** nothing
- **repo:** linked `sase-core` repo (open with `sase repo open sase-core`; read its
  `AGENTS.md` first — guarded checks run as `sase tool run check`, conventional commits,
  no bare `cargo`).

Requirement: on an archive shaped like athena's (22.6k ace-run dirs, 18.7k without
`done.json`, ~175 capacity-relevant records), a warm capacity-only
`scan_agent_artifacts` call must cost well under 1 s (target: tens of milliseconds with
the persisted verdict cache; see the `verdict_cache` decision), and a cold call must be
≥10x cheaper than today's full parse. The retained record set must be byte-identical to
today's for the same filesystem state.

Recommended shape (the phase may refine it, guarded by the parity suite):

1. **Marker precheck (statx-only).** In `capacity_only` mode, after the existing
   `done.json` skip, probe `running.json`, `waiting.json`, and `pending_question.json`
   with `symlink_metadata` (the same uncertainty semantics as `marker_may_exist`,
   scanner.rs:600). Dirs with any live marker take today's full parse path unchanged.
2. **Meta-light classification for the rest.** Dirs with no live marker can only matter
   as (a) not-yet-stopped records whose pid is not provably missing
   (`capacity_record_is_never_live`, scanner.rs:565), or (b) closure targets of live
   records via `parent_timestamp`, `runner_claim_owner_key`, or clan keys
   (`retain_capacity_records_for_admission`, scanner.rs:437). Parse only
   `agent_meta.json` for these dirs to extract
   `stopped_at`/`pid`/`parent_timestamp`/`runner_claim_owner_key`/clan fields; defer
   full record construction until the closure actually retains a dir, then parse just
   those dirs (timestamps map 1:1 to sharded paths).
3. **Sticky verdict cache.**

   > [!decision] verdict_cache = persist

   Persist per-dir `(mtime_ns, never_live_verdict, closure_keys)` so warm scans skip the
   meta parse entirely. Never-live is sticky: `done.json` is never removed and
   `stopped_at` never unsets; a changed dir mtime invalidates the entry. Corrupt/missing
   cache degrades to the meta-light pass (fail-open). Prefer keeping the cache internal
   to the scanner without a wire-schema change; if an option field is unavoidable, it
   must be additive with a serde default and follow the repo's wire-versioning recipe.

   > [!decision] verdict_cache = memory_only

   Keep the classification results only in a per-process memo; every process pays one
   cold meta-light pass per archive state, and the warm-scan budget in step 4 relaxes
   from tens of milliseconds to "well under 1 s".

4. **Tests and proof.**
   - Extend the parity coverage (`crates/sase_core/tests/integration/`
     `agent_scan_parity.rs` area; note open task sase-15u wants these trees split — do
     not grow any file past 1,500 lines) with cases that would catch a wrong skip: a
     no-marker dir with an alive pid; a stopped, no-marker ancestor retained through a
     live child's `parent_timestamp`; owner-key and clan-tribe closure pulling in
     never-live dirs; a `pending_question.json`-only dir; dangling-symlink markers.
   - Add a bench or timed fixture with ~15–20k synthetic no-done dirs asserting the
     cold- and warm-scan budgets, so the next archive doubling cannot silently regress
     this.
   - `scan_agent_artifact_dirs` (scanner.rs:221) shares the capacity retention; apply
     the same precheck there or document why it is exempt.

### Phase `sase-adopt-pin` — sase: ratchet the core pin and re-measure the scan on a real archive

- **size:** small
- **depends on:** `core-capacity-scan`

1. Move `sase-core-revision.txt` past the `core-capacity-scan` commit
   (`just ratchet-core-revision`; see `docs/rust_backend.md`).
2. No Python call-site change is expected (`collect_runner_slot_records` keeps its
   signature); if the core phase added an option field, adopt it in
   `src/sase/core/runner_slots/_scan.py` only.
3. Re-run the measurement that defined the baseline, with the dev install on athena:
   time one `scan_agent_artifacts(sase_projects_dir(), RUNNER_SLOT_SCAN_OPTIONS)` call
   cold and warm against the real `~/.sase/projects` archive. Record before (22.9 s) and
   after numbers in the phase bead. Acceptance: warm < 1 s, cold ≥10x faster than
   baseline.
4. Confirm the shared cache behavior end to end: with a waiting runner present,
   `~/.sase/runner_slots.scan.json` rewrites must stop costing seconds of CPU per token
   bump (observe `pidstat`/`ps` on a `run_agent_runner` process before/after).

### Phase `gc-duty-cycle` — TUI GC policy: full collections respect a duty-cycle bound

- **size:** medium
- **depends on:** nothing (parallel with the core phases)

`src/sase/ace/tui/util/gc_policy.py` currently schedules full collections by elapsed
time and gen-1 counts only (`MIN_FULL_INTERVAL_S = 60`, `FULL_BACKSTOP_S = 300`,
RSS-growth backstop). Make the policy cost-adaptive so a heap where `gc.collect()` takes
seconds cannot be swept every minute:

1. Record the duration of every full collection the policy itself runs (`tick` /
   `_run_idle_census` already wrap collects; time them with the injected `monotonic`).
2. Enforce a duty-cycle floor on _all_ non-startup triggers (`idle`, `backstop`,
   `rss_backstop`): the next full collection runs no sooner than
   `max(MIN_FULL_INTERVAL_S, last_full_pause_s * DUTY_FACTOR)` after the last one, with
   `DUTY_FACTOR` ≈ 30 (≈3% worst-case duty cycle) and the floor clamped to an upper
   bound (suggest 15 min) so collection never stops entirely. The 5-minute and RSS
   backstops relax the _quiet-gate_, not the duty floor.
3. `startup_freeze` behavior is unchanged. The kill switch
   (`SASE_TUI_GC_POLICY_DISABLE`) and telemetry triggers keep their meaning; consider
   surfacing the computed next-due delay via `heartbeat_fields()`.
4. Unit tests drive the existing injectable probes (fake `monotonic`, stub collector)
   with simulated slow collections (e.g. a 2 s pause defers the next full collect ~60 s;
   a 20 s pause defers it to the 15-min clamp; backstop triggers still honor the floor).
   Read `sase/memory/tui.md` → `tui_perf.md` before touching this file (event-loop
   rules).

This phase protects the TUI against _any_ future allocation-churn regression, not just
this one; it is deliberately independent of the scan fix.

### Phase `athena-soak` — measured verification on athena and residual-leak attribution

- **size:** medium
- **depends on:** `sase-adopt-pin`, `gc-duty-cycle`

1. Follow the idle-host CPU diet capture recipe in `docs/contribute/perf-runbook.md`.
   Soak a TUI instance for ≥60 min on athena under normal agent load with
   `SASE_TUI_TRACE=1`.

   > [!decision] live_tui_restart = restart_live

   After the soak passes, propose a gated (user-confirmed) quit-and-relaunch of the live
   `sase` tmux TUI session so athena recovers the leaked CPU/RSS immediately; never
   SIGKILL it from outside the session.

   > [!decision] live_tui_restart = separate_instance

   Leave the live session alone; the stale-running-code detector's restart row and a
   notification tell Bryan the fix is ready.

2. Acceptance targets, against the 2026-11-12 baseline:
   - TUI process CPU sustained **< 15% of one core** (baseline ~119%).
   - `/proc/<pid>/io` rchar rate **< 5 MB/s** (baseline ~60 MB/s).
   - RSS plateaus **< 2 GB** over the soak (baseline 4.1 GB at 15 min and climbing);
     `tui_memory_heartbeat` rows show a plateau, not a slope.
   - `tui_gc_pause` rows: full (gen-2) collections spaced per the duty rule; no
     `tui_pump_stall` rows attributable to GC.
   - Waiting `run_agent_runner` processes near 0% CPU between wakeups;
     `~/.sase/runner_slots.scan.json` no longer rewrites every few seconds when slot
     state is quiet.
3. If RSS still climbs after the churn fix, run a second soak with
   `SASE_TUI_GC_CENSUS=1` (Python ≥3.14 is in use, so the survivor census is available)
   and file a follow-up task bead with the surviving-type attribution — do not widen
   this epic to chase it.
4. Record before/after numbers in the phase bead. The epic's user-visible claim is the
   report's success criterion: "new TUI pid well under 20% CPU" and swap pressure on
   athena no longer driven by sase processes.

## Explicitly out of scope

- Runner lifecycle TTL/auto-reaping of WAITING/QUEUED `run_agent_runner` processes and
  inotify-based `ready.json` waiting (report items 5b/5d) — tracked separately as task
  bead `sase-1sk`; this epic only makes their polling cheap.
- Host-side items from the research report (tmpfiles.d `/tmp` cleanup, backup
  nice/ionice, systemd slice, NVMe/cooling) — they belong to the tailnet repo, not sase.
- Agents-tab load/index work (`sase-zu`) and startup time (`sase-132`).
