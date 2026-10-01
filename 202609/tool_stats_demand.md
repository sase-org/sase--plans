---
tier: epic
title: sase tool stats and ToolRun demand instrumentation
goal: '`sase tool stats` turns the ToolRun ledger into routine, read-only readouts
  (per tool, stage, route, and provider: p50/p90, outcome and censoring mix, ceiling
  kills and wasted hours, repeats and duplicates, a daily trend, a chronological backtest,
  and host pressure), and every new run records the demand evidence a future admission
  design needs: provider and ceiling context, process-tree CPU and memory, and pytest
  worker grants with token-wait time.

  '
phases:
- id: core-demand
  title: Rust demand record, store column, and binding
  depends_on: []
  size: small
  description: 'core-demand: in sase-core, add the ToolRunDemandWire family, a demand_json
    runs column, a merge-on-write record_demand store function with its tool_run_record_demand
    binding, and expose the record on ToolRunWire.'
- id: record-demand
  title: Record context, resource usage, and pytest worker grants
  depends_on:
  - core-demand
  size: medium
  description: 'record-demand: pin the core, capture provider and ceiling context
    at run start, reap the child with wait4 for CPU and max RSS, sample live tree
    RSS, add the SASE_TOOL_RUN_DEMAND grant channel written by tools/run_pytest, and
    render the record in sase tool show.'
- id: core-stats
  title: Rust stats report over the runs table
  depends_on:
  - core-demand
  size: medium
  description: 'core-stats: in sase-core, add the read-only tool_run_stats_report
    function and binding covering outcomes, durations, routes, ceiling kills, waste,
    reruns, providers, trend, repeats, duplicates, and demand aggregates.'
- id: core-stats-detail
  title: Stage, backtest, and pressure sections in the stats report
  depends_on:
  - core-stats
  size: medium
  description: 'core-stats-detail: in sase-core, extend the stats report with per-stage
    distributions, whole-run and per-stage chronological backtests, and a bucketed
    host-pressure section read from the stages and samples tables.'
- id: stats-cli
  title: sase tool stats command, rendering, and docs
  depends_on:
  - record-demand
  - core-stats-detail
  size: medium
  description: 'stats-cli: pin the core, add the Python facade and the sase tool stats
    subcommand with its JSON envelope and human tables, and document stats plus how
    its fields measure the parked E6, E7, and E8 reconsider conditions.'
proposed_by: bbugyi200.athena.0u4
create_time: 2026-09-30 16:20:02
status: done
bead_id: sase-1dm
---

- **PROMPT:** [prompts/202609/tool_stats_demand.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/tool_stats_demand.md)
- **BEAD:** [sase-1dm](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1dm/README.md)

# Plan: `sase tool stats` and ToolRun demand instrumentation

## Why, and what the research decided

This implements step 4 of the recommendation in
`research:202609/sase_tool_e6_e8_go_no_go/sase_tool_e6_e8_go_no_go.md` (read it with
`sase artifact read`). Step 3's reactive-routing beads, `sase-17e` and `sase-17g`, are
closed. The report's conclusions that shape this epic:

- **`stats` is high value and cheap.** Every finding in the report came from ad-hoc SQL
  over `runs.sqlite`. `stats` makes those readouts routine and measures each "reconsider
  when" condition for the parked rest of E6, E7, and E8.
- **It must report, per tool, stage, machine, and provider:** p50/p90; the outcome and
  censoring mix; kill, timeout, and lost counts with wasted hours; repeat and duplicate
  counts; a trend; and a built-in chronological backtest showing coverage **and** band
  width.
- **Demand instrumentation.** Elapsed time alone can never price admission. Record the
  pytest grant width and any token-wait time, process-tree CPU seconds, and peak RSS on
  every run.
- **Read-only.** `stats` never changes what `run` executes, never routes, never admits,
  and never forecasts for a consumer. The backtest is a measurement of the simplest
  empirical-quantile forecaster, not a forecaster.

It also checks the reactive-routing acceptance that `sase-17e`/`sase-17g` promised: no
ceiling kills, no same-agent kill→rerun pairs, fewer than 5% of monitor-owned runs
finishing under 2 minutes.

Grounding facts from the tree (all verified while planning):

- The ToolRun store is sase-core `crates/sase_core/src/tool_run/store/`. The schema is
  `SCHEMA_SQL` in `store/connection.rs`. New `runs` columns are added through the
  `ensure_child_observation_columns` list there. `load_run` in `store/query.rs` selects
  a `NULL` placeholder for columns an older store lacks, so reads never migrate.
- **Units differ by table.** `runs.created_ts/running_ts/settled_ts` and
  `samples.observed_ts` are epoch **seconds**. `stages.started_ts/finished_ts` are epoch
  **milliseconds** (see `sase-1bo`). Every `*_ms` field is milliseconds.
- Request wires are `deny_unknown_fields`, so a newer sase sending a new field to an
  older installed core fails the whole call. That is why this epic adds **new bindings**
  instead of new fields on `begin`, `finish`, `observe`, or the sample event.
- `tools/run_silent` runs every stage **without** `SASE_TOOL_RUN_EVENTS` (by design; see
  docs/tool.md "Stage recording"), so `tools/run_pytest` cannot reach the events file.
  It keeps `SASE_TOOL_RUN_ID`.
- The inline and adopted paths share `run_recorded_body` in
  `src/sase/tool/executor_run.py`. After `sase-17g`, an agent's plain `sase tool run`
  with a sync budget is reserved as a `handoff` run with a `starter` and executed by the
  adopt worker in a proc. A monitor join sets the run's `join` record.
- `discover_agent_runtime()` in `src/sase/agent/identity.py` resolves the agent's
  provider from env or `agent_meta.json`, and importing it does not pull in
  `sase.llm_provider` (`tests/tool/test_import_weight.py` still holds).
- Rust already owns `sync_wait_budget` (`tool_run/duration.rs`),
  `canonicalize_tool_fingerprint` plus `fingerprint_digest` (`tool_run/fingerprint.rs`),
  and `extra_args_digest` (`tool_run/catalog.rs`). Stats reuses them.

## Contract (applies to every phase)

### Boundary

- **Rust `sase_core::tool_run` owns** the demand wires, their validation and merge, the
  store column, every stats definition, constant, and computation below, and both
  bindings.
- **Python owns** capturing context, rusage, tree RSS, and grant records, plus the CLI,
  rendering, docs, and tests. Python never re-derives a stats number.
- Open the linked sase-core checkout only with `sase repo open sase-core -r '<why>'`,
  work in the printed path, and read that repo's `AGENTS.md` first.

### The demand record

The record is one JSON object per run in a new nullable `runs.demand_json` column,
exposed as `ToolRunWire.demand: Option<ToolRunDemandWire>`. It is absent for runs
recorded before this epic, and serialized only when present, so existing JSON stays
byte-identical.

Put the wires in a new `crates/sase_core/src/tool_run/demand_wire.rs` (`wire.rs` is near
the 1,500-line cap), re-exported from the `tool_run` facade like `handoff_wire.rs`. The
nested wires are **lenient** (no `deny_unknown_fields`) because they are also the stored
shape older readers must keep parsing. Only the top-level request wire is strict.

```rust
pub struct ToolRunDemandContextWire {        // captured by the agent-side starter
    pub provider: Option<String>,            // e.g. "claude", "muse"; None when unknown
    pub sync_ceiling_seconds: Option<u64>,   // SASE_PROVIDER_SYNC_CEILING_SECONDS
    pub sync_soft_ceiling_seconds: Option<u64>, // SASE_PROVIDER_SYNC_SOFT_CEILING_SECONDS
}
pub struct ToolRunResourceUsageWire {        // captured by the executing wrapper
    pub cpu_user_ms: Option<u64>,            // wait4 ru_utime of the reaped child tree
    pub cpu_system_ms: Option<u64>,          // wait4 ru_stime
    pub max_process_rss_kib: Option<u64>,    // wait4 ru_maxrss, normalized to KiB
    pub peak_tree_rss_kib: Option<u64>,      // max over live ~10 s samples of summed tree RSS
    #[serde(default)] pub tree_rss_samples: u32,
    #[serde(default)] pub availability: Vec<String>, // typed "unavailable" reasons, never zeros
}
pub struct ToolRunWorkerGrantWire {          // reported by the child (tools/run_pytest)
    pub grant_id: String,                    // unique per grant; merge dedups on it
    pub source: String,                      // "pytest"
    pub observed_ts_ms: i64,
    pub lane: Option<String>,                // run_pytest mode: "scoped", "fast", "cov", ...
    pub path: String,                        // "serial" | "gear" | "lease" | "bypass"
    pub requested_floor: u32,
    pub requested_ceiling: u32,
    pub granted: u32,                        // 0 = refused or timed out
    pub budget: Option<u32>,                 // host token budget at grant time
    pub wait_ms: u64,                        // time spent waiting for tokens
    pub selected_files: Option<u32>,         // scoped selection size, when known
    pub escalated_from: Option<String>,      // "scoped" when escalation moved it to the full lane
}
pub struct ToolRunDemandWire {               // the stored record
    pub schema_version: u32,
    pub context: Option<ToolRunDemandContextWire>,
    pub usage: Option<ToolRunResourceUsageWire>,
    #[serde(default)] pub worker_grants: Vec<ToolRunWorkerGrantWire>,
    #[serde(default)] pub diagnostics: Vec<String>,
}
```

(Every `Option` field carries
`#[serde(default, skip_serializing_if = "Option::is_none")]`.)

**Write semantics** (`record_demand`, binding `tool_run_record_demand`):

- The request is
  `ToolRunRecordDemandRequestWire { schema_version, run_id, context, usage, worker_grants, diagnostics }`,
  and it is `deny_unknown_fields`.
- Allowed in **any** run state; demand is metadata. An unknown run returns `NotFound`.
- **Merge, never clobber:**
  - `context: Some` replaces the stored context; `None` leaves it.
  - `usage: Some` replaces the stored usage; `None` leaves it.
  - Grants append, deduplicated by `grant_id`.
  - Diagnostics append unique.
- **Validation drops, it does not fail.** A grant with an empty or longer-than-64-char
  `grant_id`, `source`, `path`, or `lane`, or with
  `requested_floor > requested_ceiling`, is dropped. The drop adds a diagnostic naming
  the `grant_id`. The rest of the request still lands.
- **Bounds:** at most `DEMAND_MAX_WORKER_GRANTS = 64` grants and
  `DEMAND_MAX_DIAGNOSTICS = 16` diagnostics are stored. Overflow is dropped with one
  diagnostic.
- The result is
  `ToolRunRecordDemandResultWire { schema_version, run_id, demand, replayed, diagnostics }`.
  `replayed` is true when nothing changed.
- It writes through `with_write_store` in an immediate transaction, like `observe`, and
  touches `last_write_ts`.

**Recording is fail-open** (`decisions:record-before-admit`). Every Python caller wraps
the binding and never changes the child's result. A failure warns at most once per
process (`sase: run demand not recorded (<exc>)`) and bumps `TOOL_RUN_RECORDING_ERRORS`
with `op="demand"`. A stale wheel without the binding (the `AttributeError` from
`require_rust_binding`) is such a failure. Call the binding through
`require_rust_binding`, never `optional_rust_binding`, so the pinned-bindings gate
counts it.

### Who writes what, and when

| Field           | Written by                                                                                            | When                                                                          |
| --------------- | ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `context`       | the agent-side process: `_execute_resolved` (foreground) or `reserve_handoff_run` (hand-off/detached) | foreground: right after spawn; hand-off: right after a successful reservation |
| `usage`         | the executing wrapper (`run_recorded_body`, shared by inline and the adopt worker)                    | after the child is reaped, before `finish_tool_run`                           |
| `worker_grants` | the wrapper, from the child's `SASE_TOOL_RUN_DEMAND` file                                             | same write as `usage`                                                         |

- **Context** comes from a new `demand_context(env)` helper:
  - `provider` is `discover_agent_runtime(env)`, else the new `SASE_TOOL_RUN_PROVIDER`
    overlay (below), else `None`.
  - The ceilings use the positive-integer rule that `routing.py` already applies.
    Promote `_read_ceiling_value` to a public `read_ceiling_seconds` and update its
    callers.
  - Record ceilings only for runs the caller's harness actually bounds: a foreground
    run, or a hand-off reservation that carries a `starter` (a detached run). A
    reservation without a starter records the provider only, even though the reserving
    agent shell has ceilings. This covers `-H` and the verify reservation in
    `src/sase/monitor/tool_handoff.py`, both of which a monitor runs.
  - The adopt worker never writes context. It runs in a proc with agent identity and
    ceilings scrubbed, and writing there would erase the starter's facts.
- **Monitor attribution.** `_tool_run_agent_overlay` in `src/sase/monitor/start.py`
  already passes `SASE_TOOL_RUN_AGENT` into monitor procs. Add `SASE_TOOL_RUN_PROVIDER`
  from `discover_agent_runtime()` beside it, when known. Every `SASE_TOOL_*` variable is
  already scrubbed at agent launch (`scrub_executor_ownership_env`). Ceilings are
  **not** forwarded: a monitor has none.
- **CPU and max RSS** come from reaping the child with `os.wait4`. `wait_child` in
  `src/sase/tool/executor_process.py`:
  - Polls `os.wait4(proc.pid, os.WNOHANG)` on its existing ~0.1 s tick instead of
    `proc.wait(timeout=0.1)`.
  - On reap, sets `proc.returncode = os.waitstatus_to_exitcode(status)` **immediately**.
    Otherwise a later `Popen.poll()` hits `ECHILD` and reports `0`.
  - Returns the code together with the rusage (for example a `ChildExit(code, rusage)`
    NamedTuple); update the one caller.
  - Keeps the escalation branch behaving exactly as today. Its post-SIGKILL wait uses
    the same reaper with the `KILL_WAIT_SECONDS` deadline.
  - On `ChildProcessError`, falls back to the Popen path, and usage is recorded with
    availability `rusage unavailable`.
  - `ru_maxrss` is KiB on Linux and bytes on macOS; normalize to KiB.
  - Semantics to document: the CPU of the child plus every descendant that exited and
    was reaped inside the tree. `max_process_rss_kib` is the largest single process, not
    a sum.
- **Tree RSS** is sampled by `LoadSampler` (`src/sase/tool/sample.py`) on its existing
  ~10 s tick, only while the child is alive.
  - Build a ppid→children map from `/proc/[0-9]*/stat`. Parse after the **last** `)` so
    a `comm` containing spaces or parentheses is safe. Take `ppid` (field 4) and `rss`
    pages (field 24).
  - Walk from the child pid, sum RSS × `os.sysconf("SC_PAGE_SIZE")`, and track the peak
    and the sample count in memory.
  - Keep the `/proc` root injectable for tests. No `/proc` means availability
    `tree RSS unavailable on this host`. Any read error skips that tick; it never
    raises.
  - The per-sample event wire is **unchanged**: only the peak reaches the ledger,
    through `usage`.
- **Worker grants** use a new env contract,
  `SASE_TOOL_RUN_DEMAND=<run log dir>/demand.jsonl` (the directory `prepare_run_paths`
  creates).
  - `child_env` sets it whenever it sets `SASE_TOOL_RUN_EVENTS`, and pops it for
    unrecorded runs.
  - `tools/run_silent` does **not** unset it; that is deliberate and documented.
  - Nested and test processes are protected twice:
    - `tools/run_pytest` pops it (add it to `PYTEST_ENV_UNSET_KEYS`) before pytest
      starts.
    - `tests/_conftest_environment.py` scrubs it at session start (add it to
      `TOOL_RUN_RECORDING_ENV_VARS`).
  - A writer appends one JSON line per grant: one `O_APPEND` write of at most 4 KiB,
    file mode `0o600`, and it never raises. The line is:

  ```json
  {
    "schema_version": 1,
    "kind": "worker_grant",
    "run_id": "<SASE_TOOL_RUN_ID>",
    "grant": {
      "grant_id": "…",
      "source": "pytest",
      "observed_ts_ms": 1759240000000,
      "lane": "fast",
      "path": "lease",
      "requested_floor": 4,
      "requested_ceiling": 14,
      "granted": 12,
      "budget": 24,
      "wait_ms": 182000,
      "selected_files": null,
      "escalated_from": "scoped"
    }
  }
  ```

  The wrapper reads the file once, after the reap:
  - It reads at most 64 KiB, skips lines over 4 KiB, and takes at most 64 grants.
  - It keeps only `schema_version == 1`, `kind == "worker_grant"`, and `run_id` equal to
    this run. A cross-run record adds the diagnostic `ignored cross-run demand record`.
  - It forwards a grant only if its fields have the right primitive types; Rust
    validates the rest.
  - A lost run (wrapper killed) records no usage and no grants. Document that gap; do
    not reconstruct it.

### Stats definitions (Rust owns all of them)

**Constants.** All are `pub` in the stats module and echoed in the result's `thresholds`
object:

| Constant                              | Value                    |
| ------------------------------------- | ------------------------ |
| `STATS_DEFAULT_DAYS`                  | 7                        |
| `STATS_MAX_DAYS`                      | 180, the summary horizon |
| `STATS_MAX_RUNS`                      | 50,000                   |
| `STATS_MAX_STAGES`                    | 200,000                  |
| `STATS_MAX_SAMPLES`                   | 400,000                  |
| `STATS_BACKTEST_LOOKBACK_DAYS`        | 30                       |
| `STATS_BACKTEST_PRIOR_RUNS`           | 60                       |
| `STATS_BACKTEST_MIN_PRIOR_RUNS`       | 20                       |
| `STATS_BACKTEST_WIDTH_FLOOR_MS`       | 1,000                    |
| `STATS_BACKTEST_TARGET_COVERAGE`      | 0.80                     |
| `STATS_BACKTEST_TARGET_MAX_WIDTH`     | 3.0                      |
| `STATS_CEILING_KILL_MIN_PERCENT`      | 85                       |
| `STATS_CEILING_KILL_GRACE_SECONDS`    | 60                       |
| `STATS_LEGACY_CEILING_KILL_MIN_MS`    | 530,000                  |
| `STATS_LEGACY_CEILING_KILL_MAX_MS`    | 550,000                  |
| `STATS_RERUN_WINDOW_SECONDS`          | 1,800                    |
| `STATS_SHORT_MONITOR_RUN_MS`          | 120,000                  |
| `STATS_MEDIUM_MONITOR_RUN_MS`         | 300,000                  |
| `STATS_PRESSURE_BUCKET_SECONDS`       | 30                       |
| `STATS_PRESSURE_MEMORY_PSI_THRESHOLD` | 10.0                     |
| `STATS_PRESSURE_BUSY_MIN_RUNS`        | 2                        |

**Scope and window.**

- Rows are `runs` with `source = 'native'`, optionally filtered by project and by tool.
- Let `since = now − days·86400`. A run is **in window** when `created_ts ≥ since`.
- Rows back to `since − LOOKBACK` are loaded only as backtest priors.
- Ad-hoc runs (`tool_name` NULL) count only in the top-level `adhoc_runs`.
- Group per `(project, tool_name)`, so `check` in two projects never merges.
- Sort groups by in-window runs descending, then by name.
- Hitting a row cap keeps the newest rows, sets a `*_truncated` flag, and adds a
  diagnostic.

**Duration cohort.** "Bare settled" means state `succeeded` or `failed`, `duration_ms`
present, and `extra_args_digest == extra_args_digest(&[])`. This is TYPICAL's cohort
without the definition-digest pin. Report the number of distinct definition digests
seen, because the recipe drifts.

**Percentiles** use nearest rank on ascending values: `index = ceil(q·n) − 1`, clamped,
and `None` when `n = 0`. A missing value is `None`, never `0`.

**Effective duration** is `duration_ms`, else `(settled_ts − running_ts)·1000` when both
are present and non-negative. Otherwise it is unknown, and those runs are counted, never
guessed.

**Outcomes** count `succeeded`, `failed`, `signaled`, `interrupted`, `lost`, and
`unsettled` (`created` or `running`). **Censored** is `signaled + interrupted + lost`.
**Terminal causes** are counted by string, with `unrecorded` for NULL.

**Route.** The first match wins:

1. `escalated`: a `join` record is present. Also escalated: a `handoff` run with a
   `starter` whose recorded context ceilings give a Rust `sync_wait_budget` smaller than
   its effective duration.
2. `detached`: a `handoff` run with a `starter`. This is inline-then-escalate that
   settled inside its budget, or whose budget is unknown.
3. `handoff`: a `handoff` run without a `starter`. This is an explicit `-H` or a
   reserving `sase monitor start`.
4. `owned`: `owner_kind` present. This is a foreground run inside a monitor or proc.
5. `inline`: everything else.

**Monitor-owned** is `handoff + owned + escalated`.

**Ceiling kill.** The run must be route `inline`, state `signaled`, terminal cause
`signal` or NULL, and have a non-empty `agent`. Then:

- With recorded context carrying `sync_ceiling_seconds = C`: it is a kill when
  `C·1000·85/100 ≤ d ≤ (C + 60)·1000`.
- With context recorded but no ceiling: it is never a kill.
- Without any recorded context (legacy runs, the Muse signature): it is a kill when
  `530,000 ≤ d ≤ 550,000` ms.

**Rerun after kill.** For a kill `K`, look for another run of the same group by the same
agent with `created_ts` in `(K.created_ts, K_end + 1800]`. `K_end` is `settled_ts`, else
`created_ts + d`. Count the kills that have at least one such rerun.

**Waste.** Categories are exclusive, and the first match wins:

1. `killed_at_ceiling`
2. `timeout` (terminal cause `timeout`)
3. `stopped` (`stop_requested`)
4. `lost`
5. `interrupted`
6. `other_signal` (any other `signaled`)

Each category carries `runs`, `hours`, and `runs_without_duration`. The group also
carries `total_hours`.

**Provider label.**

- Context absent: `unrecorded`.
- Otherwise the context's `provider`.
- Context present with no provider: `no-agent` when `agent` is NULL, else `unknown`.

**Trend.** One bucket per local day from `since`'s day through `now`'s day, empty days
included. The day index is `floor((ts + utc_offset_seconds)/86400)`. Each bucket carries
`day_start_ts`, `runs`, `succeeded`, `failed`, `censored`, `killed_at_ceiling`,
`monitor_owned`, and a bare-settled `p50_ms`. Python formats the date label.

**Repeats** cover in-window runs only.

- The key is
  `(project, tool, definition_digest, extra_args_digest, fingerprint_digest(canonicalize(fingerprint_before)))`.
- Only runs whose `fingerprint_before` parses and is `completeness.complete` get a key.
  The others count as `unkeyed_runs`.
- Order runs by `(created_ts, run_id)`. A run is a **repeat** when its key appeared
  earlier.
- A repeat is **after censored** when the most recent earlier same-key run was signaled,
  interrupted, or lost. This separates kill→rerun pairs, as the report asked.
- A run is a **concurrent duplicate** when it started (`running_ts`, else `created_ts`)
  before an earlier same-key run ended. An unsettled run ends at `now`.
- Report runs and hours for both.

This is exact-state repetition. The content-equivalent E4b measurement stays
`sase tool receipts`, and the docs say so.

**Demand aggregates.** These come from `demand_json` of in-window runs:

- `runs_with_context` and `runs_with_usage`.
- CPU seconds (user + sys): p50, p90, and total hours.
- `effective_cores`: CPU seconds divided by wall seconds, over runs with at least 1 s of
  duration, at p50 and p90.
- `max_process_rss_kib` and `peak_tree_rss_kib`: p50, p90, and max.
- `runs_with_grants`.
- Per-run grant width (the max `granted`): p50, p90, and max.
- A count per grant `path`.
- `token_wait_runs` (total `wait_ms > 0`), `token_wait_ms_total`, `token_wait_ms_max`,
  and `token_wait_timeouts` (path `lease` with `granted == 0`).
- `escalated_grant_runs`.

**Stages** (the `core-stats-detail` phase) come from `stages` rows of the group's runs.

- A **finished** instance has `incomplete = 0` and `elapsed_ms` present.
- Group by `description`. Report `runs`, `ok` (exit 0), `failed` (nonzero),
  `incomplete`, finished p50/p90, `p90_over_p50`, and `total_hours`.
- Also report `median_offset_ms`, the median of `started_ts − running_ts·1000`. It is
  used only to order stages as the recipe runs them.

**Backtest** is a baseline: unconditioned empirical quantiles.

- **Series.**
  - Whole-run: the group's bare-settled runs, ordered by
    `(running_ts or created_ts, run_id)`.
  - Stage: finished instances, ordered by `(started_ts, stage_id)`.
- **Per element.** Each in-window element with at least 20 strictly earlier elements
  gets a prediction.
  - The priors are the 60 most recent earlier elements; lookback rows count.
  - `lo = p10` and `hi = p90` of the priors.
  - The element is covered when `lo ≤ x ≤ hi`.
  - `width = hi / max(lo, 1000)`.
- **Output.** `predictions`, `covered`, `coverage`, `median_width`, the two targets, and
  `meets_target`. `meets_target` is `None` with no predictions, else
  `coverage ≥ 0.80 && median_width ≤ 3.0`.
- **Not in scope.** Conditioning on selection size or continuation mode belongs to the
  parked forecast work. The docs state that this is the baseline such a forecast must
  beat.

**Pressure** is report-level, over samples of in-window scoped runs.

- `bucket = floor(observed_ts/30)`. Per bucket, take the distinct-run count and the max
  memory, CPU, and I/O `some` PSI, plus max `loadavg_1/logical_cpus`.
- A **busy** bucket has at least 2 distinct runs.
- Report:
  - `buckets`, `busy_buckets`, and `buckets_with_psi`.
  - `memory_over_threshold_share`: buckets with memory PSI > 10 among buckets with PSI.
  - `busy_memory_over_threshold_share`.
  - The p90 of each PSI kind and of load per CPU.
- A share with a zero denominator is `None`.
- OOM kills are not observable from the ledger; the docs say so.

### CLI contract (`sase tool stats`)

```text
sase tool stats [-a/--all] [-d/--days N] [-j/--json] [-t/--tool TOOL]
```

- The default scope is the current catalog project (`tool_project_identity()`) over 7
  days. `-a` includes every project. `-t` filters to one tool and prints its detail
  sections. `-d` must be 1–180, else exit 2.
- It reconciles unsettled runs first, as `list`, `runs`, and `receipts` do. After that,
  the report opens the ledger read-only and never migrates or writes.
- **Exit codes:** 0 when reported (including an empty report, which prints
  `no recorded runs`), 1 on a store or binding failure (message on stderr), 2 on usage
  errors.
- **`-j`** prints the core result plus a top-level `host` (`platform.node()`), sorted
  and indented, with `schema_version: 1`. The ledger is machine-local, so machines are
  compared by running `stats` on each host.
- **Human output** is Rich tables (the house style), with an em dash for every missing
  value and red for kills. Target met is green, not met is amber:

```text
tool stats · sase · athena · last 7d since 09-23 · 1,412 runs (3 ad-hoc)

TOOL        RUNS   OK   FAIL  CENS  P50      P90       KILLED  WASTE   MON<2m
check       1,102  84   920   98    3m 12s   14m 5s    12      21.4h   3/40
test        40     31   9     0     15m 37s  22m       0       0.0h    0/0

run `sase tool stats -t TOOL` for stages, routes, providers, trend, and signals
pressure · 12,040 buckets (3,112 busy) · memory PSI >10: 0.4% (busy 1.2%) · p90 cpu 0.4 · mem 4.9 · io 32.2 · load/cpu 0.6
```

With exactly one group (`-t`, or only one in the window), it also prints `STAGES` (with
each stage's backtest coverage and width), `ROUTES`, `PROVIDERS`, and `TREND` tables,
then signal lines:

```text
backtest   whole run: 75% of 1,114 inside p10–p90 · median width 18.6× · target ≥80% and ≤3×: not met
waste      killed at ceiling 12 (1.8h; 5 rerun within 30m) · timeout 0 · stopped 2 (0.1h) · lost 31 (4.2h; 6 without duration) · interrupted 4 · other signals 49 (14.1h)
routes     monitor-owned 40 · finished under 2m: 3 (7.5%) · under 5m: 9
repeats    32 exact repeats (6.9h; 5 after a censored run) · 3 concurrent duplicates (0.4h) · 211 unkeyed · content-equivalent: sase tool receipts
demand     312 runs with usage · CPU p50 42m (3.9 cores) · max process RSS p90 1.1 GiB · tree RSS peak p90 9.8 GiB · pytest grants in 290 runs, width p50 1 / p90 14 · token waits 4 runs (3m 12s, 0 timeouts)
```

### Rollout, versions, and flags

- No feature flag. No landed phase exposes an unfinished user feature:
  - The core phases change nothing a user sees.
  - `record-demand` lands complete recording plus its `show` rendering.
  - `stats-cli` lands the whole command at once.
- Every wire change is additive. `TOOL_RUN_WIRE_SCHEMA_VERSION` stays 1. Commit subjects
  are `feat(tool-run): …`, not `feat!`.
- A sase phase that calls a new binding moves `sase-core-revision.txt` past the core
  commit with `just ratchet-core-revision`, then runs `just install`, and confirms the
  "Check pinned core bindings" lint (`tools/check_sase_core_rs_bindings`) passes.
  - If the core commit is not on sase-core's remote master yet, stop and record why on
    the phase bead.
  - Never bypass binding validation. See `docs/rust_backend.md` ("The CI source revision
    pin").

## Phase `core-demand` (sase-core)

Work in the linked sase-core checkout, and follow its `AGENTS.md` recipe "Add a core
function and expose it to Python". Never run bare `cargo`.

1. **Wires.** Create `tool_run/demand_wire.rs` with the five wires from the contract
   plus `DEMAND_MAX_WORKER_GRANTS` and `DEMAND_MAX_DIAGNOSTICS`, exported through the
   `tool_run` facade. Add `demand: Option<ToolRunDemandWire>` to `ToolRunWire` with
   `#[serde(default, skip_serializing_if = "Option::is_none")]`, and give every
   `ToolRunWire` literal the new field.
2. **Store.**
   - Add `demand_json TEXT` to `SCHEMA_SQL`.
   - Add `("demand_json", "TEXT")` to `ensure_child_observation_columns`.
   - Add a placeholder projection in `load_run`. Parse it leniently: malformed JSON
     reads as `None` plus a run diagnostic and never fails the load.
   - Retention needs no change; the column lives and dies with its `runs` row.
3. **Write.** Add `record_demand` in a new `store/demand.rs`, per the contract's write
   semantics, exported as `store::record_demand` and from the `tool_run` facade.
4. **Binding.** Add `tool_run_record_demand(store_path, request, busy_timeout_ms=250)`
   in the binding domain that binds `tool_run_observe`
   (`crates/sase_core_py/src/telemetry/`), modeled on it, and register it.
   `telemetry/tests.rs` is already past 1,500 lines, so put the round-trip test in a new
   sibling test module.
5. **Tests.** Put them in a new test module beside `store/demand.rs`; `store/tests.rs`
   is near the cap.
   - Merge: context-only, then usage-only, then grants-only writes combine. Rewriting
     the same context is `replayed`.
   - Grant dedup by `grant_id`, the 64 cap, and invalid-grant drops each produce their
     diagnostics, and valid fields still land.
   - Settled runs accept writes. An unknown run returns `NotFound`.
   - A store created without the column gains it on write. A read-only open of an
     unmigrated store still loads runs.
   - A run without demand serializes byte-identically to today.
6. **Verify.** Run `sase tool run check` inside the sase-core checkout (about 5 minutes,
   inline). Commit with a subject like
   `feat(tool-run): record per-run demand context, usage, and worker grants`.

## Phase `record-demand` (sase)

1. **Pin** per "Rollout, versions, and flags".
2. **Facade and helper.**
   - Add `tool_run_record_demand` to `src/sase/core/tool_run.py` (mirror
     `tool_run_receipts_report`: it adds `schema_version`), and export it.
   - Create `src/sase/tool/demand.py` with:
     - `demand_context(env)`;
     - `record_run_demand(run_id, *, context=None, usage=None, grants=(), diagnostics=())`,
       fail-open per the contract;
     - the demand-file reader;
     - the `/proc` tree-RSS scanner.
   - Add `sase.tool.demand` to the module list in `tests/tool/test_import_weight.py`.
3. **Context capture.**
   - Add `demand_context: dict | None = None` to `RecordedRunContext`
     (`src/sase/tool/executor_models.py`). `_execute_resolved` fills it, and
     `run_recorded_body` records it right after `_observe_spawned_child`, only for a
     recorded run whose `demand_context` is not `None`.
   - `reserve_handoff_run` (`src/sase/tool/handoff.py`) records context after a
     successful reservation, with ceilings only when `starter` is present. A failure
     there never turns a reservation into a refusal.
   - Add the `SASE_TOOL_RUN_PROVIDER` overlay in `src/sase/monitor/start.py`.
4. **Usage.**
   - Make the `wait_child` reaper change from the contract.
   - Pass the child pid to `LoadSampler` for tree RSS, and stop tree sampling once the
     child is reaped.
   - In `run_recorded_body`, after the reap and before `finish_tool_run`, make **one**
     `record_run_demand` call with usage plus the grants read from the demand file.
   - Leave the pre-spawn signal path and the spawn-failure path unchanged: they have no
     child, so no usage.
5. **Grant channel.**
   - Add the `SASE_TOOL_RUN_DEMAND` constant beside `TOOL_RUN_EVENTS_ENV` and wire it
     through `child_env`, both setting and popping it.
   - Add it to `PYTEST_ENV_UNSET_KEYS` and `TOOL_RUN_RECORDING_ENV_VARS`.
   - Add a stdlib-only writer next to the pool code, for example
     `tests/_suite_gate_demand.py` with `record_worker_grant(...)`. It is a no-op unless
     both `SASE_TOOL_RUN_DEMAND` and `SASE_TOOL_RUN_ID` are set.
6. **`tools/run_pytest` records one grant per decision.** Measure `wait_ms` with
   `time.monotonic()` around `lease.acquire`.
   - **Full lane** (`_parallel_worker_grant`): path `lease` with floor, ceiling,
     granted, budget, and wait. On an acquire timeout, record `granted: 0` with the
     elapsed wait, then re-raise. Set `escalated_from: "scoped"` when scoped escalation
     switched the mode.
   - **Bypass** (`SASE_TEST_GATE_DISABLED=1`): path `bypass` with the clamped width and
     wait 0.
   - **Descendant exemption:** record nothing, because the ancestor already did.
   - **Scoped middle gear, when granted:** path `gear` with
     `selected_files = len(gear_candidate)`.
   - **Scoped serial run:** path `serial`, granted 1, with `selected_files`.
   - **Empty selection:** nothing, since pytest never starts.

   Record before `os.execv` and before `_sanitize_pytest_environment()`.

7. **`sase tool show`.** In `print_show` (`src/sase/tool/_query_shared.py`), add lines
   only when `run.demand` is present:
   - `CONTEXT   provider claude · ceiling 4h · soft —`
   - `DEMAND    cpu 41m 12s (3.9 cores) · max process RSS 1.1 GiB · tree RSS peak 9.8 GiB (54 samples)`
   - One `WORKERS` line per grant, for example
     `pytest lease 12 of 4–14 · budget 24 · waited 3m 2s · from scoped`.

   Add a KiB formatter to `src/sase/tool/render.py`. `show -j` already carries
   `run.demand` through the core.

8. **Docs.**
   - `docs/tool.md`: a new "Demand recording" section covering the three writers, the
     `SASE_TOOL_RUN_DEMAND` JSONL contract (so any project's test runner can report
     grants), `SASE_TOOL_RUN_PROVIDER`, the rusage and tree-RSS semantics, and the
     lost-run gap.
   - Amend "Stage recording": the demand variable deliberately passes through
     `tools/run_silent`.
   - Mention `SASE_TOOL_RUN_PROVIDER` wherever `docs/monitors.md` documents
     `SASE_TOOL_RUN_AGENT`.
9. **Tests.**
   - The reaper, in `tests/tool/`:
     - an exit code, a signal (a negative code), and `returncode` set before any `poll`;
     - CPU greater than 0 for a CPU-burning child;
     - the `ChildProcessError` fallback;
     - escalation unchanged.
   - The tree-RSS scanner over a fake `/proc`, including a `comm` with `) (` in it, a
     vanished pid, and no `/proc`.
   - The demand-file reader: valid, malformed, cross-run, oversized, and over-cap input.
   - Context from env, from the overlay, and with none at all. A reservation without a
     starter records the provider but no ceilings.
   - An executor integration test using the fixture-catalog pattern from
     `tests/tool/test_executor.py`. The tool is a Python snippet that burns about 0.2 s
     of CPU and appends one grant line. Assert that `tool_run_show` has context, CPU,
     max RSS, and the grant. A second run with the binding patched to raise still
     succeeds and warns once.
   - The adopt path never writes context.
   - `run_pytest` grant records for the lease, timeout, gear, serial, bypass, and
     exemption paths, in `tests/test_run_pytest_workers.py` and
     `tests/test_run_pytest_scoped.py`, and `_sanitize_pytest_environment` popping the
     variable.
   - The monitor overlay.
   - The `show` lines, present and absent.

## Phase `core-stats` (sase-core)

1. **Layout.**
   - Pure computation goes in a new `crates/sase_core/src/tool_run/stats/` (a `mod.rs`
     facade, `wire.rs`, computation files each under 1,500 lines, and `tests.rs`), with
     no SQL.
   - The SQL loader and the entry point
     `tool_run_stats_report(store_path, request, busy_timeout)` go in a new
     `store/stats.rs`, modeled on `store/receipts_report.rs`: read-only,
     `with_read_store`, `NULL` projections for absent columns, and a missing store
     returning an empty report with the diagnostic `tool run store does not exist`.
2. **Wires.**
   - `ToolRunStatsRequestWire { schema_version, project: Option<String>, tool_name: Option<String>, days: i64, now_ts: Option<i64>, utc_offset_seconds: i32 }`
     is `deny_unknown_fields`, with defaults of 7 days and offset 0. Days outside 1–180
     are `Invalid`.
   - The result carries:
     - `schema_version`, `project`, `tool_name`;
     - `window { days, since_ts, now_ts, utc_offset_seconds }`;
     - `runs_scanned`, `runs_truncated`, `adhoc_runs`;
     - `tools: Vec<ToolRunStatsToolWire>`, `thresholds`, and `diagnostics`.
   - Each tool entry carries:
     - `project`, `tool_name`, `runs`, `definition_digests`, `extra_args_runs`;
     - `outcomes`, `terminal_causes`, `duration` (count, p10, p50, p90, max, plus
       `succeeded` and `failed` sub-summaries);
     - `waste`, `reruns_after_kill`;
     - `routes` (per route: runs, settled, p50, p90, under 2m, under 5m, and kills) and
       `monitor_owned { runs, under_2m, under_5m, under_2m_share }`;
     - `providers` (per label: runs, outcomes, p50, p90, kills, kill hours, and route
       counts);
     - `trend`, `repeats`, and `demand`.
   - Result wires are lenient, and fields added later carry `#[serde(default)]`.
3. **Compute** every section in the contract except stages, backtest, and pressure,
   which belong to the next phase.
   - Call `sync_wait_budget` for the escalated test.
   - Canonicalize and digest fingerprints only for in-window runs.
   - Load in-window and lookback rows in one query ordered newest first.
4. **Binding.** Add `tool_run_stats_report(store_path, request, busy_timeout_ms=250)` in
   the same telemetry domain, with a round-trip test in the new sibling test module.
5. **Tests.** Test the pure functions with hand-built rows:
   - Percentile edges.
   - Every route branch, including the escalation test via the sync budget.
   - Kill rules:
     - recorded-ceiling bounds, including the 85% and +60 s edges;
     - the legacy window edges (530 s and 550 s);
     - no kill when context is present without a ceiling;
     - owned, monitor, and human runs are never kills.
   - Rerun-window edges.
   - Waste exclusivity and unknown durations.
   - Provider labels.
   - Trend bucketing with a nonzero offset and empty days.
   - Repeats, after-censored repeats, duplicates, and unkeyed runs.
   - Demand aggregates, including timeouts.
   - Bare-cohort exclusion of extra args.

   Add one store-level test over a temp SQLite store written through `begin`, `finish`,
   and `record_demand`, and one over an unmigrated store (no `demand_json`).

6. **Verify.** Run `sase tool run check` in the sase-core checkout and commit with
   `feat(tool-run): add read-only ToolRun stats report`.

## Phase `core-stats-detail` (sase-core)

1. **Wires.** Add these with `#[serde(default)]`:
   - `stages: Vec<ToolRunStatsStageWire>` and
     `backtest: Option<ToolRunStatsBacktestWire>` on each tool entry;
   - `stages_truncated`, `samples_truncated`, and
     `pressure: Option<ToolRunStatsPressureWire>` on the result.

   Each stage carries its own `backtest`.

2. **Load.**
   - Stages: join `stages` to the scoped runs (in window plus lookback), capped at
     `STATS_MAX_STAGES`.
   - Samples: join `samples` to scoped in-window runs, capped at `STATS_MAX_SAMPLES`,
     and parse `payload_json` into `ToolLoadSampleWire` fields. Skip unparseable
     payloads with a count diagnostic.
   - Mind the units: stage stamps are epoch ms, run and sample stamps are epoch seconds.
3. **Compute** the stages, both backtests, and pressure exactly as the contract defines.
   Order stages by `median_offset_ms`, with stages lacking an offset last, sorted by
   name.
4. **Tests.**
   - Backtest: fewer than 20 priors yields no prediction; the 60-prior window slides;
     lookback priors count while lookback elements are never predicted; the width floor;
     `meets_target` is true, false, and `None`.
   - Stage grouping, ordering, and the ok/failed/incomplete split, with ms timestamps.
   - Pressure: bucket dedup across runs, busy buckets, shares with zero denominators,
     and missing PSI.
   - A store-level test with stages and samples.
   - Performance: a synthetic 10,000-run, 60,000-sample store builds the report in under
     2 s in a release-profile test, or a `#[ignore]`d bench if the suite forbids timing
     asserts. Record the measured time in the commit body.
5. **Verify.** Run `sase tool run check` in the sase-core checkout and commit with
   `feat(tool-run): add stage, backtest, and pressure stats`.

## Phase `stats-cli` (sase)

1. **Pin** per "Rollout, versions, and flags". This moves the pin past
   `core-stats-detail`.
2. **Facade.** Add `tool_run_stats_report` to `src/sase/core/tool_run.py`, and export
   it.
3. **Parser.** In `src/sase/main/parser_tool.py`, add a `stats` subparser.
   - Order: after `show`, before `stop`.
   - Options, sorted alphabetically, each with a short alias: `-a/--all`, `-d/--days N`
     (default 7), `-j/--json`, `-t/--tool TOOL`.
   - Its description covers the read-only contract, the machine-local scope, and the
     backtest-as-baseline caveat, with an examples epilog like `receipts`.
   - Add `stats` to the group metavar and to the fallback usage string in
     `src/sase/main/tool_handler.py`.
   - Run `just sync-completion-spec` if `tests/completion/test_snapshot.py` drifts.
4. **Handler.** Create a new `src/sase/tool/stats_report.py`, imported lazily from
   `handle_tool_command` so `sase tool run` pays nothing. It:
   - validates `days`;
   - calls `reconcile_unsettled_tool_runs()`;
   - builds the request with `project=None` for `-a`, else `tool_project_identity()`,
     and `utc_offset_seconds` from `datetime.now().astimezone().utcoffset()`;
   - adds `host`;
   - prints JSON, or renders the human form from the contract.

   Keep rendering helpers (hours, percent, ratio, the KiB formatter from `render.py`) in
   small functions. Split rendering into a sibling module if the handler would pass
   about 400 lines.

5. **Docs.**
   - `docs/tool.md`: a "Stats" section covering:
     - the options and exit codes;
     - every definition in plain words: cohorts, routes, the two kill rules, waste,
       repeats versus `sase tool receipts`, trend, the backtest as a baseline, pressure,
       and the OOM gap;
     - the machine-local scope, with `ssh <host> sase tool stats -j` to compare hosts;
     - the `stages` detail-retention horizon;
     - a table mapping each held reconsider condition (from the research artifact's
       recommendation, step 5) to the stats field that measures it:
       - rest of E6 → per-stage backtest coverage and width;
       - E7 → `pressure.busy_memory_over_threshold_share`, concurrent-duplicate hours,
         and token-wait runs and timeouts;
       - E8 → run counts from each host;
       - E4b → `sase tool receipts`.
   - New rows in `docs/cli.md` (the `sase tool` table) and in `docs/configuration.md`
     (the CLI flag table).
6. **Tests.**
   - Parser: options, the `-d` bounds, and help.
   - Handler, with a fake facade returning a fixture envelope:
     - JSON passthrough plus `host`;
     - the human tool table;
     - the single-group detail sections and signal lines;
     - em dashes for `None`;
     - an empty report;
     - a store failure exits 1;
     - a usage error exits 2.
   - One end-to-end test that runs two fixture-catalog tools through `execute_tool_run`,
     then `sase tool stats -j` returns both groups with correct outcome counts.
   - Add `sase.tool.stats_report` to the import-weight list.

## Landing

1. Confirm each phase's evidence. Run `just fix`, then `sase tool run check`.
2. **Live smoke** on this host, recorded in a bead note on the epic:
   - A fresh `sase tool run check`, then `sase tool show <id> -j`. The `run.demand`
     field must show the provider and ceiling, CPU, max-process and tree-RSS numbers,
     and at least one `pytest` grant (the scoped stage's `serial` or `gear` grant, or a
     `lease` after escalation).
   - `sase tool stats -t check -d 7` and `-j`, with wall time noted; it should be well
     under 5 s. Paste the signal lines.
   - Compare the backtest with the research's 75% / 18.6× baseline.
   - State whether the reactive-routing acceptance holds: zero kills since `sase-17g`
     landed, no reruns after a kill, and the monitor-owned under-2m share.
3. Close the epic.
4. File the phases' `PROPOSED FOLLOW-UP:` notes through `/sase_new_task` where they are
   warranted. Likely candidates:
   - recording the research's held reconsider conditions in a bead or decision, if none
     exists yet;
   - an Admin Center or TUI stats surface;
   - demand recovery for lost runs.

## How to tell it worked (post-landing observation, not a landing gate)

- A week after landing, `sase tool stats -t check` on athena shows `runs_with_usage` and
  `runs_with_context` close to the in-window run count, and grants on nearly every
  `check` run.
- The kill and short-monitor numbers let the user judge `sase-17g`'s acceptance with no
  ad-hoc SQL.
- The per-stage backtest shows which stages already meet the ≥80% / ≤3× bar and which
  need conditioning.

## Non-goals

- Forecasts, `run -E`, ETAs, "overdue" and "stalled" states, `timeout: auto`, and any
  routing or admission change. These are the parked E6 remainder and E7.
- Cross-machine aggregation (E8).
- TUI or Admin Center surfaces.
- New fields on the `begin`, `finish`, `observe`, or sample-event wires.
- Backfilling demand for runs recorded before this epic.
- Changes to receipts or triage.
- Fixing `sase-1bo`: it is independent, and it only affects how far back stage rows
  reach.
- **No memory or skill edits.** If a phase finds `lint_and_test.md` or
  `glossary:tool-run` stale about `stats`, it records a `PROPOSED FOLLOW-UP:` note.

## Verification (every phase)

- **sase phases:** run `just fix`, then `sase tool run check`. If the workspace venv is
  stale, run `just install` first.
- **sase-core phases:** run `sase tool run check` inside that checkout.
- Do not run `check-full` (`decisions:check-full-is-explicit`).
- Phase workers never create beads. They append `PROPOSED FOLLOW-UP:` notes to their own
  phase bead.
