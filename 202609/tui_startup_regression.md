---
tier: epic
title: Restore TUI startup time on large-archive hosts
goal: "A sase TUI session on athena is interactive on its visible tab in about 3.5
  seconds at the median again (it is 6.8-8 s today, from 3.25 s in mid-August), every
  stage of the durable startup telemetry returns to its mid-August band, and the
  startup-critical path is regression-guarded so the next creep is caught by measurement
  instead of by feel.

  "
phases:
  - id: baseline
    size: medium
    title: Controlled baselines and startup observability gaps
    depends_on: []
    description:
      "baseline: capture controlled quiet-host and busy-host loader benches plus traced
      live startups at a recorded deployed SHA, and close the instrumentation gaps this
      diagnosis hit - sub-stage spans inside agents.load_from_disk, spans for the axe
      surface first load (none exist in live traces today), a pre-mount split of
      process_start_to_on_mount, and a startup-window marker on spans so startup
      contention is directly queryable."
  - id: startup-sequence
    size: large
    title: Make the visible surface win the startup window
    depends_on:
      - baseline
    description:
      "startup-sequence: stop launching every post-mount background load concurrently
      with the visible tab's first load; sequence or gate non-visible-surface work
      (relations index builds, prompt catalog warm, detail/prompt panel first renders,
      monitor reconcile, bead warmups, dismissed-index sync, proc-shell prune, update
      checks) behind visible-ready with a bounded fallback delay, and deduplicate work
      that runs twice in the window today."
  - id: loader-diet
    size: large
    title: Cut the bounded Tier 1 load's absolute cost
    depends_on:
      - baseline
    description:
      "loader-diet: attribute and reverse the standalone bounded-load creep
      (production_bounded p50 1002 ms on 2026-09-13 to 1632 ms on 2026-09-18 at +3%
      archive growth), including the ~11x decode amplification (about 1,062 records
      decoded to return 96 rows) and index row growth, with Rust-core pushdown evaluated
      under the rust_core_backend_boundary rule."
  - id: premount-diet
    size: medium
    title: Import-graph diet for process start to on_mount
    depends_on: []
    description:
      "premount-diet: bring process_start_to_on_mount back under its mid-August band
      (0.67-0.76 s median; 1.10 s today) by trimming the TUI app import graph (3,285
      modules imported before first paint) and add an import-count/time regression guard
      on the startup-critical path."
  - id: axe-ready
    size: medium
    title: Attribute and fix the doubled axe surface startup cost
    depends_on:
      - baseline
    description:
      "axe-ready: using the axe spans added in baseline, attribute why axe_ready_seconds
      doubled (2.0 s to 3.5 s median, step on 2026-08-27/28) and restore it to about 2
      s, keeping the axe first load off the visible surface's critical path."
  - id: first-paint
    size: small
    title: Trim on_mount to first paint back under 0.3 s
    depends_on:
      - baseline
    description:
      "first-paint: profile the compose/on_mount/first-refresh path
      (on_mount_to_first_paint_seconds doubled from 0.21 s to 0.44 s median) and remove
      or defer the growth so first paint lands in about 0.25 s again."
  - id: verify
    size: medium
    title: Prove the recovery on athena and pin it with regression guards
    depends_on:
      - startup-sequence
      - loader-diet
      - premount-diet
      - axe-ready
      - first-paint
    description:
      "verify: capture after-telemetry over real athena sessions at a recorded deployed
      SHA, compare against this plan's baselines per stage, confirm the regression
      benches and guards fail on reintroduction, and record results on the epic bead
      with explicit non-overlap accounting against the sase-124.8.4 freshness
      acceptance."
proposed_by: bbugyi200.athena.0n7
create_time: 2026-09-18 15:22:30
status: wip
---

- **PROMPT:**
  [prompts/202609/tui_startup_regression.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/tui_startup_regression.md)

# Restore TUI Startup Time on Large-Archive Hosts

## Problem

The user's suspicion is **confirmed by durable telemetry**: TUI startup on athena has
more than doubled since mid-August, and every independently measured stage regressed.
`~/.sase/logs/tui_startup.jsonl` (787 session records, 2026-08-12 through 2026-09-18)
weekly medians:

| week     | pre-mount | first paint | visible_ready | agents_ready | axe_ready | index rows |
| -------- | --------- | ----------- | ------------- | ------------ | --------- | ---------- |
| 2026-W33 | 0.67 s    | 0.21 s      | 4.35 s        | 5.02 s       | 1.72 s    | 551        |
| 2026-W34 | 0.76 s    | 0.25 s      | **3.25 s**    | 4.04 s       | 2.02 s    | 644        |
| 2026-W38 | 1.10 s    | 0.44 s      | **6.75 s**    | 7.84 s       | 3.56 s    | 742        |

(pre-mount = `process_start_to_on_mount_seconds`; first paint =
`on_mount_to_first_paint_seconds`.) Recent days are worse: 2026-09-17 median
`visible_ready` was 8.05 s. Index row growth over the same period is +15% and archive
growth +3.3% (11,182 → 11,554 artifacts, 2026-09-13 → 09-18), so data growth cannot
explain a 2x regression — and pre-mount time does not scale with the archive at all.
Daily medians show distinct steps worth bisecting: pre-mount 0.80→0.96 s on 08-23→24 and
0.89→1.11 s over 09-05→08; axe_ready ~2.3→3.4 s on 08-27→28; visible_ready 6.25→8.05 s
over 09-15→17.

Every phase worker MUST read `sase/memory/tui_perf.md` first via
`sase memory read tui_perf.md -r "<why>"` (rule 9 — "Keep startup off data-scaled work"
— plus the measurement tooling list) and follow `docs/perf_runbook.md` capture recipes.
Startup claims are settled by `tui_startup.jsonl`, never by feel. Note tui_perf rule 15:
a long-running TUI executes the snapshot it imported at start, so before/after telemetry
must be bucketed by deployed SHA.

## Evidence (captured 2026-09-18 on athena)

1. **The startup agents load dominates visible_ready and is the bounded path already.**
   Live-trace records (`~/.sase/perf/tui_trace.jsonl`, 2026-09-17/18) for
   `agents.load_from_disk` with `source=startup` all show `tier=tier1`,
   `artifact_source=artifact_index`, `index_freshness=cached`, `bounded_prefix=true`,
   `requested_limit=96`, `returned_count=96` — and durations of 3805, 4768, 4850, 6582,
   8092, 8099, 8339, 8987 ms. The matching `tui_agent_loads.jsonl` startup records put
   the `disk` stage at p50 4.3-5.1 s (up to 6.5 s), with `prep` ~0.2-0.4 s and `apply`
   ~0.1 s.
2. **The same operation is 3-6x faster standalone and ~2.5x faster mid-session.**
   `bench_agent_load_tiering.py --sase-home ~/.sase`
   (`~/.sase/perf/agent_load_tiering_sase-124.8.3_athena_real_20260918.json`) measures
   `production_bounded` p50 1632 ms on the same day the in-app startup load cost 4.8-9
   s; non-startup tier1 broad loads in the same TUI sessions run p50 1.8-2.1 s. The
   startup window itself inflates the load ~2.5-4x.
3. **The startup window is saturated with non-visible work.** Spans inside one
   representative startup window (2026-09-18 12:53 UTC, visible_ready 8.35 s):
   `agents.load_from_disk` 6582 ms, `relations.index.patches` 874 ms **x2**,
   `widget.agent_detail.update_display` 447 ms, `widget.prompt_panel.update_display` 425
   ms, `widget.prompt_panel.build_detail_header_summary` 278 ms, `agents.worker_prep`
   172 ms, `agents.diff_badge_classification` 120 ms, `agents.update_info_panel` 117 ms
   x7, plus (from `_start_post_mount_background_loads`,
   `src/sase/ace/tui/actions/_startup_loads.py`) the mount-state reads (notifications,
   prompt stash, patches, saved selection), axe init, dismissed proc-shell prune,
   artifact and prompt-source watchers, prompt catalog warm, update-toast check, and
   usage refresh — all scheduled at first paint, concurrently with the visible surface's
   load, in one GIL.
4. **The standalone loader itself crept +63% in five days at +3% archive growth**:
   `production_bounded` p50 1002 ms (09-13) → 1421 ms (09-17) → 1632 ms (09-18). The
   09-13 runbook stage attribution shows the bounded path decodes ~1,062 records to
   return ~100 rows (~11x decode amplification) and that per-row Python work
   (dict-to-wire, `Agent` decode, agents-live filter) dominates the shared cost.
5. **Pre-mount is import cost.** Importing the TUI app module pulls 3,285 modules (~2.5
   s in a warm workspace venv; the `sase.ace.tui.actions.agents._core` subtree alone is
   ~1.06 s cumulative and `sase.ace.tui.widgets.artifacts.patch_entry` ~0.77 s). Durable
   pre-mount telemetry went 0.67 s (W33) → 1.10 s (W38) with clear steps on 08-23→24 and
   09-05→08.
6. **The axe surface doubled with no attribution available.** `axe_ready` went ~2.0 s →
   ~3.5 s median (step on 08-27/28), and the live trace contains **zero** `axe.*` spans
   (30 MB tail scan, 2026-09-17/18), even though `sase/memory/tui_perf.md` rule 14
   documents `axe.collect` spans with `file_opens` counters. Either the instrumentation
   regressed or it never covered the startup path; both are observability gaps.
7. **Trace retention is too short for creep forensics.** The live `tui_trace.jsonl`
   reaches back only ~4 days (514 MB, unrotated), so the Aug/early-Sep steps can no
   longer be attributed from traces — only from `tui_startup.jsonl`, which survives
   because it is one record per session.

## Root causes (ranked by absolute cost)

1. **Startup-window contention (~3-6 s of visible_ready).** Every post-mount background
   load is launched at first paint and competes with the visible surface's bounded Tier
   1 load for the GIL, the thread-pool, and host I/O. The identical load runs 1.6 s
   standalone, 1.8-2.1 s mid-session, 4.8-9 s at startup. tui*perf rule 9 says first
   paint never waits on O(archive) work — that held — but nothing prioritizes the
   _visible surface's own load* over deferrable background work inside the startup
   window.
2. **Bounded Tier 1 loader absolute cost (+0.6 s standalone, more in-app).** Decode
   amplification (~11x), index row growth (551 → 742-883 returned for ~9-16 visible
   agents), and per-row Python work that the 09-13 runbook already measured as the
   dominant shared cost.
3. **Import graph growth (+0.4 s pre-mount).** 3,285 modules before first paint; two
   step-regressions visible in dailies.
4. **Axe surface first load (+1.5 s axe_ready).** Unattributed; instrumentation absent.
5. **Compose/first-paint growth (+0.2 s).** `on_mount_to_first_paint` doubled.

Confounders to control, not assume away: athena's background load has also grown (more
resident agents), and TUI sessions run imported snapshots that can lag landed fixes
(tui_perf rule 15). The in-app-vs-standalone same-day ratio and per-SHA bucketing are
the controls; the baseline phase makes both routine.

## Out of scope / coordination

- **sase-124 / sase-124.8.4 (in progress, "Agents freshness acceptance")** owns
  refresh-tick freshness and its acceptance evidence. This epic owns the _startup_
  window. Both touch `agents.load_from_disk`: record deployed SHAs on every capture and
  state per-signal which epic a gain belongs to, as the sase-124.7 verification note
  already practices. Do not re-fix what sase-124.4 landed (per-load sweep caches, warmup
  coalescing); measure first.
- **Index bloat/vacuum** has its own history (index vacuum tooling exists; hidden-row
  repair cost is documented in `docs/perf_runbook.md`). loader-diet may _consume_
  index-shape findings and file follow-up beads, but a durable index
  compaction/retention redesign is out of scope here.
- **Rust core boundary:** any loader change that moves row filtering/decoding belongs in
  `../sase-core` per the `rust_core_backend_boundary` core memory — Python-side
  reimplementation of core behavior is not acceptable here.
- Memory-file updates (e.g. `sase/memory/tui_perf.md` rule 14 if span names change) are
  not plan steps; if a phase invalidates documented behavior, the worker follows its
  memory-write authorization flow or files a memory task bead.

## Phase: baseline — controlled baselines and startup observability gaps

**Goal:** every later phase can prove its win, and the diagnosis gaps hit while writing
this plan are closed.

1. Capture, at a recorded deployed SHA: (a)
   `bench_agent_load_tiering.py --sase-home ~/.sase` on a quiet host and during a
   representative busy window (label which); (b) at least three live traced startups
   (`SASE_TUI_TRACE=1`, plus one `sase tui --profile` run); (c) one
   `python -X importtime` capture of the TUI entry in the deployed environment. Store
   under `~/.sase/perf/` with the epic prefix and cite in the phase bead.
2. Add sub-stage spans inside `agents.load_from_disk`
   (`src/sase/ace/tui/actions/agents/_loading_helpers.py`): provider `load_agents` (the
   Rust index read), row decode, and `_apply_loaded_agent_disk_projections`, so in-app
   time divides into wait-vs-work without a profiler.
3. Add spans (or restore `axe.collect`) covering the axe surface first load
   (`_run_axe_startup_init` / `_load_axe_status_async`,
   `src/sase/ace/tui/actions/axe_display/_loader_refresh.py`) with a file-opens counter,
   matching what tui_perf rule 14 documents.
4. Split `process_start_to_on_mount_seconds`: record interpreter+CLI import, app-module
   import, `App()` construction, and compose as separate additive fields in the
   `tui_startup.jsonl` record (new fields only — existing fields and their semantics are
   load-bearing for historical comparison and MUST NOT change meaning).
5. Tag spans emitted before the startup stopwatch ends with a `startup_window=true`
   attribute so contention queries stop needing manual window reconstruction.
6. Tests: telemetry record shape (new fields present, old fields unchanged); span
   emission on the startup path under a fake slow loader.

Acceptance: one command sequence (documented in the phase bead and
`docs/perf_runbook.md`'s startup section) reproduces a fully attributed startup capture;
baseline JSONs exist for quiet and busy host states.

## Phase: startup-sequence — make the visible surface win the startup window

**Goal:** the in-app startup `agents.load_from_disk` runs within ~1.5x of the same-day
standalone bench p50 (today: 2.5-4x), and visible_ready drops by several seconds with no
functional regression in what is loaded by the time the user first interacts.

1. Introduce explicit startup-window sequencing in `_start_post_mount_background_loads`
   (`src/sase/ace/tui/actions/_startup_loads.py`): the visible tab's surface load
   (agents on the default tab) starts first; deferrable work waits for visible-ready or
   a bounded fallback (a few seconds, so a wedged loader never starves the rest of
   startup forever). Classify each current startup task explicitly: keep-immediate
   (stall watchdog, artifact watcher — losing early file events would cause missed
   deltas; notifications read that seeds unread state), defer-until-visible-ready
   (relations index build, prompt catalog warm, update-toast check, usage refresh,
   dismissed proc-shell prune, dismissed-index sync, monitor reconcile, bead/preview
   warmups), and axe-init (see axe-ready; it must not contend with the agents load but
   also must not wait indefinitely — axe_ready is itself a measured surface).
2. Deduplicate the double `relations.index.patches` build observed in the startup window
   (874 ms x2), and audit the window for other duplicated work (e.g. repeated
   `agents.update_info_panel` x7).
3. Defer the first `widget.agent_detail.update_display` /
   `widget.prompt_panel.update_display` full renders until the roster apply that makes
   them meaningful, rather than rendering placeholder-then-real twice on the critical
   path (respect tui_perf rule 7: highlight paints immediately, detail panels debounce).
4. Preserve all coalescing/pending guards (tui_perf rule 2) and the established refresh
   paths (rule 5) — this phase re-orders existing work; it does not add new refresh code
   paths.
5. Tests: startup ordering (deferred tasks do not start before visible-ready under a
   fake slow loader; they all run after; fallback fires when visible-ready never
   arrives); dedup (relations index built once per startup); an integration test
   asserting the startup load is not concurrent with the deferred class.

Acceptance: on athena, startup `agents.load_from_disk` p50 within 1.5x of the same-day
standalone `production_bounded` p50, and `tui_startup.jsonl` visible_ready median
improves by ≥2 s at an unchanged loader; deferred surfaces still all become ready
(all_surfaces_ready recorded, no starvation).

## Phase: loader-diet — cut the bounded Tier 1 load's absolute cost

**Goal:** standalone `production_bounded` p50 back to ≤1.1 s on the athena real archive
(mid-September level), which compounds with startup-sequence in-app.

1. First attribute the 09-13 → 09-18 standalone creep (+63% at +3% archive) using the
   baseline phase's sub-stage spans and a controlled quiet-host bench: Rust index read
   vs Python decode vs projections. Bisect against loader-touching commits in that
   window if the shape is a step.
2. Attack decode amplification: ~1,062 records decoded for 96 returned rows. Candidates,
   in order: push the visibility/dismissal/grouping predicates that currently run
   post-decode down into the Rust index query (`../sase-core` change with wire/API +
   bindings + tests per the boundary rule); decode lazily (defer full `Agent`
   materialization for rows outside the returned window); confirm each agents-live
   field's pushdown or `KNOWN_FALLBACK_FIELDS` coverage stays declared
   (`src/sase/ace/tui/models/agent_live_query_pushdown.py`, tui_perf rule 9).
3. Investigate returned-index-row growth (551 → 742-883 for ~9-16 visible agents): what
   proportion is dismissed/hidden/stale rows the query could exclude index-side? File
   findings as beads if the fix is index retention/vacuum (out of scope here).
4. Extend `tests/perf/bench_agent_load_tiering.py` (or a sibling) so the bounded path's
   decoded-record count and p50 are asserted against a floor on the synthetic fixture —
   respecting existing perf-floor flake handling.

Acceptance: standalone `production_bounded` p50 ≤1.1 s on athena's real archive at
≥11.5k artifacts, decoded-records-per-returned-row materially reduced (target ≤3x
amplification on the synthetic fixture), full-history parity checks still pass
(`missing_count=0` semantics per the runbook).

## Phase: premount-diet — import-graph diet for process start to on_mount

**Goal:** `process_start_to_on_mount_seconds` median ≤0.8 s on athena.

1. Measure in the deployed environment with `python -X importtime` (the workspace-venv
   numbers above are indicative only). Attribute the 08-23→24 and 09-05→08 steps by
   running importtime at the SHAs around them if feasible.
2. Trim the app import graph: the `actions.agents._core` (~1.06 s cum) and
   `widgets.artifacts.patch_entry` (~0.77 s cum) subtrees are the measured heavy edges.
   Use the repo's established deferred-import convention (many functions already import
   locally); target imports not needed before first paint.
3. Guard it: add a startup-critical-path import regression test (import the app module
   in a subprocess, assert module count and wall time under a budget with flake-safe
   margins), so the next dependency creep fails a test instead of a feel-check. Mirror
   the "representative job import" budget style from `docs/perf_runbook.md`'s idle-host
   table.
4. Do not lazy-import inside per-keystroke paths (tui_perf rule 11 — keystroke paths
   stay read-only and cheap; first-use import stalls belong at startup or idle, not
   under a key).

Acceptance: pre-mount median ≤0.8 s over ≥10 real athena sessions at the deployed SHA;
the import-budget test fails when a known-heavy module is added back to the eager graph.

## Phase: axe-ready — attribute and fix the doubled axe surface startup cost

**Goal:** `axe_ready_seconds` median ≤2.0 s without pushing cost onto the agents
surface.

1. Using baseline's axe spans, attribute the 2.0 → 3.5 s regression (step on
   2026-08-27/28: chop/routine result-file growth? collector scan shape? new sync work
   in `_load_axe_status_async`?). Check whether the axe collector re-reads files the
   refresh-token path (tui_perf rule 14) would skip.
2. Apply the same shape the agents surface uses (tui_perf rule 5): serve a cheap first
   paint from bounded/cached data, complete in the background, coalesce the follow-up
   refresh — if attribution shows an O(history) scan on the axe startup path.
3. Coordinate with startup-sequence on scheduling: axe init runs after the visible
   surface's load when the agents tab is the initial tab, but axe_ready must still meet
   its own target (it is a recorded stage, not best-effort).
4. Tests: collector file-opens bounded on startup; axe first load does not block or get
   blocked by the agents startup load (ordering test).

Acceptance: axe_ready median ≤2.0 s over ≥10 real athena sessions; file-opens counter on
the axe startup path bounded and asserted.

## Phase: first-paint — trim on_mount to first paint back under 0.3 s

**Goal:** `on_mount_to_first_paint_seconds` median ≤0.3 s (0.44 s today, 0.21 s in W33).

1. Profile compose-to-first-paint with `sase tui --profile` plus the baseline pre-mount
   split (compose is recorded there). Identify what doubled: widget-count growth in
   `compose()`, synchronous work in `on_mount`
   (`src/sase/ace/tui/actions/_startup_mount.py`), or first-refresh work queued before
   the `call_after_refresh` that marks first paint.
2. Move anything not needed for the initial frame behind the first-paint marker; hidden
   tabs' widgets should not render real content before the visible tab paints.
3. Guard with the existing visual-snapshot tooling if layout changes (read
   `sase/memory/tui.md` children per its instructions before touching snapshot-covered
   surfaces).

Acceptance: first-paint median ≤0.3 s over ≥10 real athena sessions at the deployed SHA.

## Phase: verify — prove the recovery on athena and pin it

**Goal:** the headline number is back and the next regression is caught by tooling.

1. After all fix phases are deployed (mind tui_perf rule 15 — restart the resident TUI;
   record the SHA), collect ≥15 real startup sessions on athena across at least two
   days, plus one quiet-host and one busy-window standalone bench run.
2. Acceptance targets (medians from `tui_startup.jsonl` at the after-SHA, compared
   against this plan's Problem table):
   - `visible_ready_seconds` ≤3.5 s (stretch: the W34 3.25 s);
   - `agents_ready_seconds` ≤4.5 s;
   - `axe_ready_seconds` ≤2.0 s;
   - `process_start_to_on_mount_seconds` ≤0.8 s;
   - `on_mount_to_first_paint_seconds` ≤0.3 s;
   - startup `agents.load_from_disk` ≤1.5x same-day standalone `production_bounded` p50.
3. Non-overlap accounting: state explicitly which gains coincide with sase-124.8.4 (or
   other concurrently landed) changes, with SHAs, as the sase-124.7 note format does. If
   a target is missed, attribute the miss with the baseline tooling and record proposed
   follow-ups on the phase bead rather than silently widening targets.
4. Confirm the regression guards bite: the import-budget test, the decoded-records
   floor, and the startup-ordering tests each fail when their guarded regression is
   reintroduced (mutation-style spot check, described in the phase bead note).
5. Update `docs/perf_runbook.md`'s startup section with the new baseline table and the
   attributed-startup capture recipe from the baseline phase.

Acceptance: all six targets met (or misses attributed and follow-ups filed), runbook
updated, guard-bite check recorded on the epic bead.
