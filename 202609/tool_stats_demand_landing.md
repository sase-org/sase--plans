---
tier: tale
size: medium
title: Finish and land sase-1dm (tool stats and ToolRun demand)
goal:
  The sase-1dm audit gaps are fixed in sase-core and sase, a live check run records
  demand evidence through the workspace code, and epic sase-1dm is closed with its plan
  marked done.
proposed_by: bbugyi200.athena.sase-1dm.land
bead: sase-1dm
create_time: 2026-09-30 20:34:41
status: wip
---

- **PARENT:**
  [202609/tool_stats_demand.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_stats_demand.md)
- **BEAD:**
  [sase-1dm](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1dm/README.md)

# Finish and land epic `sase-1dm`: `sase tool stats` and ToolRun demand instrumentation

## Context

Epic `sase-1dm` (plan `plan:202609/tool_stats_demand.md`; read it with
`sase bead read sase-1dm -r "<why>"`, which prints the PLAN path) is fully implemented.
All five phases are closed. The sase-core commits `7a9ffad`, `6e23783`, and `b381879`
are on sase-core master, and sase's `sase-core-revision.txt` already pins past them. The
sase commits are `1728715220` (record-demand) and `be6daf95d1` (stats-cli).

The land agent audited everything against the epic plan's **Contract** section. It
recorded the verification, the stats half of the live smoke, and every follow-up outcome
as notes on `sase-1dm`; read them first. The audit found the gaps below, all caused by
the epic. This tale fixes them, finishes the live smoke, and closes the epic.
**Follow-up triage is already done; do not create beads.**

The epic plan's Contract section stays authoritative for every definition referenced
here.

## Part A: sase-core fixes (linked repo)

Open the checkout only with
`sase repo open sase-core -r "Finish sase-1dm remaining core fixes"`, work in the
printed path, and read that repo's `AGENTS.md` first. Never run bare `cargo`. Paths
below are relative to `crates/sase_core/src/tool_run/`.

1. **Skip unparseable sample payloads** (`store/stats.rs`, `load_sample_rows`).
   - Today `serde_json::from_str(&payload).unwrap_or_default()` keeps a bad sample with
     no metrics. That bad sample still adds its run to a bucket, inflating `buckets` and
     `busy_buckets`.
   - Per the contract, skip such rows and count them. When the count is nonzero, add one
     result diagnostic, for example `skipped N unparseable load samples`.
   - Fix the function's doc comment, which currently documents the wrong behavior.
2. **Bare-settled cohort requires `duration_ms`** (`stats/report.rs`: the `bare` filter
   near the top of the per-tool builder, and `bare_points`).
   - The contract says "bare settled" means `succeeded`/`failed`, `duration_ms`
     **present**, and no extra args. Today both filters accept the
     `(settled_ts − running_ts)·1000` fallback.
   - Restrict both filters to rows with `duration_ms`, and use that value.
   - Effective duration (with its fallback) stays as it is for routes, kills, waste, and
     providers.
   - `report.rs` is about 1,358 lines against the repo's 1,500-line cap; keep the change
     small, or move a helper out.
3. **`replayed` means "nothing changed"** (`store/demand.rs`).
   - Today `replayed = demand == before && call_diagnostics.is_empty()`, so replaying a
     request that held an invalid or over-cap grant rewrites identical JSON, touches
     `last_write_ts`, and returns `replayed = false`.
   - Make `replayed = demand == before`, and skip the UPDATE and `touch_write_meta`
     whenever it is true. The result's `diagnostics` still report this call's drops.
4. **Tests.**
   - `stats/tests.rs` (backtest): 19 strictly earlier priors give no prediction and 20
     give one; with more than 60 priors, the window slides, so only the 60 most recent
     priors set `lo`/`hi`. Build the series so that an old outlier changes the band only
     if it is wrongly included.
   - Pressure: one run with several samples in the same 30 s bucket counts once toward
     that bucket's distinct runs and is not busy alone. A bucket with no PSI fields is
     excluded from `buckets_with_psi` and from both share denominators.
   - `store/stats_tests.rs`: a malformed `payload_json` sample is skipped and reported
     by the new diagnostic, and the bucket counts exclude it.
   - Bare-cohort exclusion: a `succeeded` run with NULL `duration_ms` but valid
     running/settled stamps stays out of the duration percentiles and the backtest
     series.
   - `store/demand_tests.rs`: replaying a request that contains an invalid grant returns
     `replayed = true`, still reports the drop diagnostic, and leaves `last_write_ts`
     unchanged.
5. **Performance bench** (the contract's core-stats-detail step 4).
   - Add an `#[ignore]`d test that builds a synthetic store of 10,000 runs and 60,000
     samples, plus some stage rows, through the store API or direct SQL. It times
     `tool_run_stats_report` and asserts under 2 s only when run in release.
   - Use the repo's existing pattern for ignored or bench tests. If the suite forbids
     timing asserts, print the time instead.
   - Run it once through the repo's sanctioned recipe (not bare `cargo`). Record the
     measured time in a `sase bead note sase-1dm "PERF: …"` note, because commits are
     host-owned and you cannot write the commit body.
6. **Verify.** Run `sase tool run check` inside the sase-core checkout.
   - No new binding is added, so sase's `sase-core-revision.txt` pin does **not** move
     in this tale.
   - No sase Python test may depend on the new core behavior.

## Part B: sase fixes (this repo)

1. **Out-of-range grant ints must not lose usage** (`src/sase/tool/demand.py`,
   `_valid_grant`). This is a real bug.
   - Today it checks only `type(...) is int`. The core wire is `u32` for
     `requested_floor`, `requested_ceiling`, `granted`, `budget`, and `selected_files`,
     and `u64` for `wait_ms`. Values like `granted: -1` or `2**40` therefore make the
     binding reject the **whole** request (`invalid value: integer -1, expected u32`).
   - Because the wrapper sends usage and grants together, that one bad line loses the
     run's CPU and RSS figures.
   - Add range checks: `0 <= v <= 2**32-1` for the u32 fields and `0 <= v <= 2**64-1`
     for `wait_ms`. Keep `observed_ts_ms` within i64.
   - Drop an out-of-range grant and add one diagnostic for it. Follow the reader's
     existing diagnostic style, for example `ignored out-of-range demand grant <id>`,
     added once.
   - Add reader tests for a negative value and for a value above u32.
2. **Foreground context capture is fail-open** (`src/sase/tool/executor_entry.py`,
   `foreground_demand = build_demand_context(os.environ) if recorded else None`).
   - Wrap the call like `handoff.py` and `start_launch.py` already do: on any exception,
     use `None`, warn at most once, and bump `TOOL_RUN_RECORDING_ERRORS` with
     `op="demand"`. Reuse `demand.py`'s existing warn-once and metric helper rather than
     copying it.
   - The trigger is `discover_agent_runtime` raising `UnicodeDecodeError` on a non-UTF-8
     `agent_meta.json`.
   - Add a test: with `build_demand_context` patched to raise, the run still spawns and
     succeeds.
3. **Escape Rich markup in the stats tables** (`src/sase/tool/stats_report.py`).
   - Tool names, project names, stage descriptions, provider labels, and route names go
     into Rich unescaped today. `tests [x86]` renders as `tests`, and `oops [/foo] bar`
     raises `MarkupError`, which dies with a traceback instead of exiting 1.
   - Pass them through `rich.markup.escape` (or `rich.text.Text`).
   - Add a test with a bracketed stage description and tool name.
4. **Split the renderer and tidy it.** `stats_report.py` is 485 lines, and the epic plan
   says to split rendering into a sibling module past about 400.
   - Move the Rich rendering (tables, signal lines, formatting helpers) into a new
     `src/sase/tool/stats_report_render.py`, keeping the handler (validation, reconcile,
     request building, JSON, exit codes) in `stats_report.py`. The handler stays lazily
     imported from `handle_tool_command`.
   - Add `sase.tool.stats_report_render` to `tests/tool/test_import_weight.py`.
   - While moving the code:
     - Delete the duplicate `_fmt_hours_value` and use `_fmt_hours`.
     - Render nonzero kill counts red in the ROUTES `KILLS`, PROVIDERS `KILLED`, and
       TREND `KILLED` columns and in the waste line's killed-at-ceiling cell, like the
       TOOL table does.
     - Render the monitor-owned under-2m share with one decimal, like the contract's
       `3 (7.5%)`. `_fmt_share` already does this.
   - Keep every number a formatted core field. Python never re-derives a stats number.
5. **Demand-file read cap** (`src/sase/tool/demand.py`, `read_demand_grants` or whatever
   the reader is named). `Path(path).read_bytes()[:_DEMAND_MAX_BYTES]` reads the whole
   file first. Open the file and `read(_DEMAND_MAX_BYTES)` instead. Add a test with a
   file over 64 KiB whose valid grant lies past the cap; it must not be read.
6. **Token-wait timeouts only for real timeouts** (`tools/run_pytest`,
   `_parallel_worker_grant`).
   - Today every `pytest.UsageError` from `lease.acquire` is recorded as a `lease` grant
     with `granted: 0`. That includes the pool-capacity errors in
     `tests/_suite_gate_lease.py` (missing capacity metadata, and an explicit
     `SASE_TEST_GATE_SLOTS` mismatch), which inflates `token_wait_timeouts`.
   - In `tests/_suite_gate_lease.py`, raise a dedicated subclass for the deadline case,
     for example `class WorkerTokenTimeout(pytest.UsageError)`, at the `now >= deadline`
     raise. Catch only that subclass in `run_pytest` to record the refusal, then
     re-raise.
   - Every existing `except pytest.UsageError` caller keeps working.
   - Extend the timeout test in `tests/test_run_pytest_workers.py`, and add a test that
     a capacity-mismatch error records no grant.
7. **Sampler never raises** (`src/sase/tool/sample.py`). Move the
   `Path(self.proc_root).is_dir()` probe inside the existing `try`, or guard it, so a
   probe error skips the tick.
8. **Docs** (`docs/tool.md`).
   - In the Stats reconsider table's E7 row, list token-wait **runs** as well as
     timeouts.
   - In the kill-rule prose, state that a run with recorded context but no ceiling is
     never counted as a ceiling kill.
   - In "Demand recording", list the reader limits: 64 KiB per file, lines over 4 KiB
     skipped, at most 64 grants, exact-int types, and out-of-range values dropped with a
     diagnostic.
   - Also in "Demand recording", list the typed availability reasons:
     `rusage unavailable` and `tree RSS unavailable on this host`.
9. **Test gaps** (`tests/tool/test_stats_report.py`).
   - Add a parser help test for `sase tool stats -h`: options and the read-only and
     baseline caveats.
   - Tighten the end-to-end test (currently around lines 462–465) to assert **exact**
     outcome counts for both fixture tools, not `>= 1`.

## Part C: verification and live smoke

1. In this repo, run `just install` if the venv is stale, then `just fix`, then
   `sase tool run check`. Read the triage.
   - Known failures **not** caused by this work, each tracked elsewhere:
     - symvision unused-public symbols `HandoffSubmitResult`, `StarterResolution`, and
       `owner_ref` (`sase-1dn`);
     - `get_unread_set_generation`, `has_unread_probe_cache_key`, and
       `note_unread_set_changed` (epic `sase-1d7`);
     - `fit_next_word_ghost` (from `7b3d47c3ea`, next-word work);
     - the `test_agent_header_panel.py` ImportError (`sase-1dh`);
     - the force-reuse launch-seam cluster (`sase-1du`).
   - Anything UNKNOWN in files you touched is yours to fix.
   - Do not run `just check-full`.
2. **Demand smoke.** Agents' default `sase` is an editable install from the primary
   checkout, which may not have pulled the epic's commits yet.
   - So run one `check` through this workspace's venv: `.venv/bin/sase tool run check`.
     Its adopt worker reuses `sys.executable`, so the workspace code records demand.
     This run can double as step 1's check.
   - If it escalates past the sync ceiling, follow the printed escalation block and use
     your `/sase_monitor` skill.
   - Then run `sase tool show <run-id> -j`.
   - `run.demand` must show:
     - `context.provider`, plus `sync_ceiling_seconds` when the agent env carries one;
     - `usage.cpu_user_ms`/`cpu_system_ms`, `max_process_rss_kib`, and
       `peak_tree_rss_kib` with `tree_rss_samples > 0`;
     - at least one `worker_grants` entry with `source: "pytest"`: the scoped stage's
       `serial` or `gear` grant, or a `lease` after escalation.
   - Record the result with `sase bead note sase-1dm "LIVE SMOKE (demand part): …"`,
     including the run id and those numbers.
   - If any field is missing, that is epic work: fix it before closing.

## Part D: close out the epic (final step)

Do these in order, in the same turn as the final code. Never wait for, or order this
after, this tale's own commit, push, or CI.

1. Run `sase bead epic-symbols sase-1dm`. For every listed `--epic-symbol` entry,
   resolve the symbol: wire it up, privatize it, add a non-test pragma, or delete it,
   per the Symvision epic-whitelist policy (read `symvision.md` with `/sase_memory_read`
   first).
   - All `sase-1dm` phases are closed, so there is no open bead to re-key an entry to.
   - At planning time the list was empty.
2. Close the epic:

   ```bash
   sase bead close sase-1dm --note "<verification>"
   ```

   The note summarizes:
   - the five phases' commits, and that the core commits are on sase-core master and
     pinned;
   - the land agent's audit and integration findings (from its epic notes);
   - every fix this tale made, in sase and in sase-core;
   - the `sase tool run check` results, in this repo and in sase-core;
   - both live-smoke notes and the perf time;
   - "follow-up triage recorded in the land agent's epic note".

   Never use `--force` merely to make the close succeed.

3. Run `just symvision`.
   - It still fails today on the unrelated unused-public symbols listed in Part C.
   - Confirm that none belongs to `sase-1dm` and that no `--epic-symbol` line is keyed
     to `sase-1dm` or its phases.
4. Set `status: done` (currently `status: wip`) in the frontmatter of the epic's plan
   file, `plan:202609/tool_stats_demand.md`, at the PLAN path printed by
   `sase bead read sase-1dm -r "Need the plan path to mark it done"`.
5. `sase-1dm` has no `parent_bead`, so the landing ends there.

## Explicitly out of scope

The land agent declined these and recorded why on the epic:

- Changing the stats wire shape (tool `backtest`/`pressure` non-`Option`, and per-stage
  backtests carried in `stage_backtests`).
- The diagnostics-overflow and grant-cap nits in `record_demand`.
- The truncation-diagnostic wording.
- The yellow-for-amber colour.
- Markup escaping in the pre-existing (non-stats) `sase tool` renderers.
- New beads of any kind.
- Memory or skill edits.
