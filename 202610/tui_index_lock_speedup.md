---
tier: epic
title: Restore TUI speed by ending artifact-index lock starvation
goal: 'The TUI stops falling into 24–30 s O(archive) fallback scans. The stranded-work
  collector holds the artifact-index lock for seconds instead of minutes. A busy lock
  is answered from the last good snapshot, never a filesystem scan. The smaller UI-thread
  hitch sources the research found are trimmed. A restarted TUI can then be re-measured
  mechanically before any GC-policy change.

  '
decisions:
  core_lookup:
    ask: How should sase-core make the stranded session lookup proportional to its
      matches?
    choices:
      pushdown: Push session/name equality filters into SQL for every caller; no new
        binding
      binding: Also add a dedicated stranded-records binding; more API surface and
        pin gating
    default: pushdown
    why: Speeds every session/name filter caller and leaves the binding surface unchanged
    answer: pushdown
  rollback_flag:
    ask: Keep the old lock-busy source-scan branch reachable behind a sunset feature
      flag?
    default: false
    why: Each phase is one revertible commit; a flag adds both-state tests and a removal
      bead
    answer: false
  tui_perf_rule:
    ask: Add a tui_perf rule that a busy index lock is answered from cache, never
      a source scan?
    memory:
    - tui_perf.md
    default: false
    answer: false
phases:
- id: stranded-batch
  title: Batch the stranded-work session query per project
  depends_on: []
  size: small
  description: 'stranded-batch: replace the per-assignee index query loop with one
    batched, chunked query per project, keeping the exact recheck and dedupe, with
    parity tests and a before/after timing.'
- id: stranded-cadence
  title: Throttle and memoize stranded reports
  depends_on: []
  size: medium
  description: 'stranded-cadence: raise the stranded report TTL to 60 s, floor forced
    recomputes, drop the dead beads_watcher force source, and memoize per-tick stranded
    project keys, verdict indexes, and counts.'
- id: loader-lock-busy
  title: Never answer a busy index lock with a source scan
  depends_on: []
  size: medium
  description: 'loader-lock-busy: serve the last good tier-1 snapshot or keep the
    current roster when the index lock is busy, retry through existing flags, log
    artifact_source and repair_reason on slow loads, and document the lock''s invariants.'
- id: core-session-lookup
  title: Indexed session lookup in sase-core
  depends_on:
  - stranded-batch
  size: medium
  description: 'core-session-lookup: push agent_session/agent_name equality filters
    into SQL with NOCASE indexes, fix the workflow-scoped reconcile deletion bug,
    keep the Rust post-filter, and move the sase-core pin.'
- id: agents-tick-trims
  title: Trim Agents-tab tick and capacity patch work
  depends_on:
  - stranded-cadence
  - loader-lock-busy
  size: medium
  description: 'agents-tick-trims: stop capacity patches from rebuilding panel titles,
    skip unchanged capacity applies, and check a cheap revision key before fingerprinting
    clan members for runtime aggregation.'
- id: pump-probe-trims
  title: Move small synchronous work off the UI pump
  depends_on: []
  size: small
  description: 'pump-probe-trims: use one persistent malloc_trim worker, move the
    tool-runs token probe off the countdown tick, and debounce the prompt subtitle
    restyle while keeping typing and the caret immediate.'
- id: capacity-feed
  title: Calm the machine-capacity feed
  depends_on:
  - pump-probe-trims
  size: small
  description: 'capacity-feed: lengthen the display rescan interval, reuse the parsed
    slot-scan cache while unchanged, and move the capacity token probe off the UI
    timer.'
- id: remeasure-kit
  title: Mechanical re-measurement and follow-up handoff
  depends_on:
  - stranded-batch
  - stranded-cadence
  - loader-lock-busy
  - core-session-lookup
  - agents-tick-trims
  - pump-probe-trims
  - capacity-feed
  size: small
  description: 'remeasure-kit: teach tui_freeze_report to summarize slow agent loads
    by index-stage bucket and fallback reason, document the post-fix remeasure and
    GC decision recipe, and record the evidence-gated follow-ups.'
proposed_by: bbugyi200.athena.research.5k.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 18:38:06
status: done
bead_id: sase-1q5
---

- **PROMPT:** [prompts/202610/tui_index_lock_speedup.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/tui_index_lock_speedup.md)
- **BEAD:** [sase-1q5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1q5/README.md)

# Plan: Restore TUI speed by ending artifact-index lock starvation

## Context

The TUI has regressed from about 3.6% of wall time in loop hitches and 5.7% in GC (23
Oct baseline) to about 12–13% hitches and 10–11% GC. RSS swings between 4 and 16 GB,
with up to 7.9 GB swapped out. The consolidated research report found the cause and a
recommended fix. The user agreed with all of its requirements. Read it before starting a
phase:

```bash
sase artifact read file:explicit:a012614c6eedb39b1e7193ff "<why you need it>"
```

The label is
`research:202610/tui_slowdown_index_lock_fallback_scans/tui_slowdown_index_lock_fallback_scans.md`.
The chain it established, re-verified in code while planning:

1. **Lock monopoly.** `src/sase/bead/stranded_facts.py:_session_record_cache` loops over
   every candidate assignee: 218 keys for `gh_sase-org__sase`. Each key gets its own
   `query_agent_artifact_index` call with `include_full_history=True`,
   `include_hidden=True` and the default `freshness="revalidate"`. Every call holds the
   process-wide `_ARTIFACT_INDEX_OPERATION_LOCK`
   (`src/sase/core/agent_artifact_index_lock.py`), at about 0.5 s per call, so one
   report takes about 113 s. Planning found why each call is expensive: a revalidated
   full-history query runs `reconcile_source_directories` in sase-core
   (`crates/sase_core/src/agent_scan/index/query.rs`, `index/refresh.rs`). That is a
   filesystem walk of every project's `ace-run` directories plus a dirty-check of every
   indexed row, and it ran 218 times per report. One batched query over all keys
   measured 1.51 s with zero result mismatches.
2. **Cadence.** The report cache TTL is 15 s
   (`src/sase/ace/tui/models/agent_stranded.py`, `_CACHE_TTL_SECONDS`). Every Agents
   apply schedules the bead warmup, which recomputes any expired report. A 113 s report
   and a 15 s TTL therefore run back-to-back. The warmup's force set names
   `beads_watcher`, but no production caller sends that source. Only the test
   `tests/ace/tui/test_agents_bead_warmup.py` and the user-initiated `stranded_resolve`
   path use the force set.
3. **Amplifier.** `query_artifact_index_for_loader`
   (`src/sase/ace/tui/models/_agent_loader_artifacts.py`) waits only 50 ms for the lock
   (`query_agent_artifact_index_bounded`). On timeout it runs `scan_artifacts`: 24 s and
   1.5 GB for the "bounded" tier-1 scan, 30 s and 2.5 GB for full history. The tier-1
   index read it replaces costs 0.35 s and 140 MB. `artifact_snapshot_for_tui_load` then
   source-scans again whenever a full-history index result is not complete. The tier-1
   fallback also reports `complete_visible_inbox=False`, which arms a tier-2 reconcile,
   which hits the busy lock again.
4. **Freezes.** Those scans' transient heaps make young collections take 50–500 ms and
   idle full collections 3–7 s. `GCPolicy.tick` runs the full collections on the UI's 1
   s timer. GC policy is **not** changed in this epic: the research rejects changing it
   before the load fix is measured (see [Out of scope](#out-of-scope-and-follow-ups)).

## Guardrails (every phase)

- Read `tui_perf.md` with `/sase_memory_read` before touching TUI code. Follow its
  rules. Reuse the existing coalescing flags, `patch_row`, pump-free tasks, the
  quiet-time reconcile and `_schedule_agents_async_refresh`. Add no new refresh paths or
  schedulers. Re-capture tab and selection after every `await`.
- Completeness stays honest. Never mark a stale or partial snapshot complete. History
  stays incomplete until an authoritative reconcile lands.
- Behavior other than speed stays the same. Each behavior-preserving change gets a test
  that pins the old observable result (parity), not just the new code path.
- Verify with `just check` plus the focused tests named in your section. Do not run
  `just check-full`. Do not run global installs (`just install`/`install-dev`).
- A running TUI keeps executing the code it imported at start (tui_perf rule 15). Do not
  claim a live speedup; report standalone measurements.
- Epic phase workers never create beads. Record discovered work as
  `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own phase bead.

## Phase: stranded-batch

Goal: one stranded report costs about one index query per project instead of one per
assignee. Results must be identical.

- In `stranded_facts._session_record_cache`, replace the per-assignee loop with batched
  queries for the project:
  - Use the same `AgentArtifactIndexQueryWire` and `AgentArtifactScanOptionsWire` as
    today (full history, hidden included, `ace-run` only, `only_projects=(project,)`).
  - Set
    `candidate_filter = {"kind": "any", "filters": [equals agent_session=k, equals agent_name=k for each key k in the chunk]}`.
  - Chunk the sorted keys at 100 per query.
- Group the returned records per requested key with the existing exact
  `_record_matches_assignee` recheck. A record can match more than one key (its session
  is one key and its name another), so give it to every key it matches. Keep the per-key
  `artifact_dir` dedupe and the order in which the index returned records.
- Failure semantics stay per key, as today:
  - A missing index file yields `([], False)` for every key.
  - A chunk whose query raises one of today's caught exceptions yields `([], False)` for
    exactly that chunk's keys.
  - Every other key keeps `True`.
- Keep `collect_stranded_work_input`'s output schema unchanged. The wait-checks chop
  (`src/sase/scripts/_chop_wait_checks_stranded.py`) and `sase doctor`
  (`src/sase/doctor/checks_beads.py`) share this collector and get faster for free.
- Tests:
  - Add a parity test: the batched grouping versus the old per-assignee function. Keep
    the old function as a test-only reference, or compute the old answer inline.
  - Cover session roots (`agent_session` null, matched by name), successors that share a
    session, one record matching two keys, duplicate `artifact_dir`s, a failing chunk,
    and more than 100 keys (multi-chunk).
  - Put the tests beside `tests/test_stranded_single_turn.py` and the
    `tests/_stranded_collector_helpers.py` fixtures.
  - `tests/test_stranded_collector_recovery.py` wraps `_query_session_records` to assert
    which keys were queried. Update those tests to assert the requested key set of the
    batched call instead of dropping the assertion.
- Measure in a separate process; the TUI's in-process lock is untouched:
  - Time `collect_stranded_work_input("gh_sase-org__sase")` with the workspace `.venv`.
  - Record the before and after wall times, plus the query count, in a bead note.

## Phase: stranded-cadence

Goal: stranded reports refresh at a bounded cadence, and per-tick stranded lookups stop
allocating.

- Raise `_CACHE_TTL_SECONDS` in `src/sase/ace/tui/models/agent_stranded.py` from 15 s to
  60 s. Stranded flags may now show up to about 60 s late, which the research accepted.
- Keep the bead-store fingerprint comparison in `warm_stranded_work_reports`, so a store
  change still invalidates on the next warmup that includes the project. Add no new
  UI-thread I/O to detect it sooner.
- Floor forced recomputes:
  - A forced warmup recomputes a project only if its last computation is at least 30 s
    old.
  - The explicit user `stranded_resolve` source is exempt and stays immediate, because
    the user is waiting on their resolve action.
  - Store the last-computed time in the cache entry.
- Source cleanup:
  - Grep for senders of the `beads_watcher` source first. If none exists in production
    code, remove it from both force-set checks in
    `src/sase/ace/tui/actions/agents/_loading_bead_warmup.py` and update
    `tests/ace/tui/test_agents_bead_warmup.py`.
  - If a sender does exist, keep the source and subject it to the 30 s floor.
- Memoize the per-tick helpers. Explorers found these run on every 1 s countdown tick
  and per rendered row:
  - **`stranded_project_key_for_agent`:** memoize the `Path(project_file).parent.name`
    derivation by the `project_file` string, with a bounded cache.
  - **`_verdict_index`:** build the `bead_id → verdict` dict once per report, when the
    report is stored in `_StrandedReportCache.set`, instead of on every call.
  - **`stranded_count_for_agents`:** called from
    `src/sase/ace/tui/actions/agents/_display_detail_info.py` outside the metrics cache.
    Recompute it only when the stored reports or the represented project-key set change.
    Key it by report identity plus roster identity; do not use wall-clock time.
- Renders and counts must be identical. Add tests for:
  - TTL expiry at 60 s;
  - the forced-refresh floor, including the `stranded_resolve` exemption;
  - fingerprint invalidation still recomputing;
  - memo invalidation when a new report is stored.

## Phase: loader-lock-busy

Goal: "artifact-index lock busy" never triggers an O(archive) scan in the TUI loader. A
busy lock degrades to the last good data plus a retry.

Requirements, in `src/sase/ace/tui/models/_agent_loader_artifacts.py`,
`src/sase/ace/tui/models/agent_loader.py`, and the Agents loading actions:

1. **Tier-1 request, lock busy** (`query_index` returns `None`): return the newest
   snapshot cached under the _same_ snapshot-cache key, even when its index signature is
   stale.
   - Mark it `repair_reason="artifact_index_lock_busy_served_cached"` and
     `index_error="artifact index operation lock busy"`, with `complete_history=False`.
     Carry no claim that it is current.
   - `_ArtifactSnapshotCache.get` deletes an entry on signature mismatch today. Retain
     the last good entry per key for this path, for example a stale-read accessor that
     ignores the signature. The normal path must still treat a signature mismatch as a
     miss.
   - Never serve a snapshot cached under a different key: a different filter or window
     would show the wrong rows.
2. **Tier-1 request, lock busy, no cached snapshot for that key:**
   - Before the Agents tab's first load (startup with nothing displayed), keep the
     existing bounded first-paint fallback (tui_perf rule 9). Pass this condition in
     from the caller; the model-layer function cannot see app state.
   - Otherwise, defer: keep the currently displayed roster unchanged (apply nothing) and
     retry.
3. **Full-history (tier-2) request, lock busy:** never call `scan_artifacts`. This
   covers both `query_artifact_index_for_loader` and the outer
   `artifact_snapshot_for_tui_load`, which today source-scans whenever the index result
   is not complete.
   - Keep the current roster and leave history incomplete.
   - Re-arm the existing quiet-time reconcile (`_agents_history_reconcile_pending` in
     `src/sase/ace/tui/actions/agents/_loading_refresh_polling.py`).
4. **Retries reuse the existing machinery.** `_loading_refresh_polling.py` already
   special-cases `artifact_index_lock_busy_bounded_fallback` by retrying a plain refresh
   (`ROSTER_COMPLETION_SOURCE`). Extend that branch to the new lock-busy reasons. Keep
   the loading and pending flags, so the last request wins. Prefer an existing
   discard/no-apply path for "keep the current roster" over inventing one.
5. **Source scans stay only where legitimate:**
   - the index file is missing;
   - the index query raises;
   - startup has no snapshot at all;
   - the `full_history and not complete_history` index-incomplete branch, which this
     phase only makes visible.
6. **Make fallbacks visible.** Add `artifact_source`, `repair_reason`, `tier` and
   `complete_history` to every `tui_agent_load_slow` row where the load state is known.
   They pass through `_record_slow_loader_stages(**fields)` in
   `src/sase/ace/tui/actions/agents/_loading_disk_support.py`. The fields let the
   remeasure tooling count fallbacks directly instead of inferring them from durations.
7. **Document the lock.** In `src/sase/core/agent_artifact_index_lock.py`, write a short
   module docstring that states:
   - what the lock serializes (in-process index readers, writers and repair work);
   - that it is process-local (SQLite WAL already allows concurrent readers across
     processes);
   - that bounded readers must degrade to cached data, never to a source scan.

   Do not change lock behavior.

- Out of scope: `src/sase/agent/listing_snapshot.py` has the same fallback for CLI
  listing processes, where the TUI's lock is not shared. Leave it unless you find a
  TUI-process caller. If you do, apply the same rule there and say so in a bead note.
- Tests (extend `tests/ace/tui/actions/test_agent_loader_phase5_fallback_wiring.py` and
  its neighbors):
  - Tier-1 lock busy, with and without a same-key cached entry. Include a stale
    signature and a different-key entry that must not be served.
  - Tier-2 lock busy never calls the scanner, keeps the roster, and re-arms the
    reconcile.
  - Index missing and query error still scan.
  - Startup with no snapshot still uses the bounded fallback.
  - The retry is scheduled through the existing flags.
  - Slow-load rows carry the new fields.

> [!decision] rollback_flag Create a `sunset` flag with `sase flag new` (read
> `sase_flags.md` first). The `On` branch is the new behavior. The `Off` branch keeps
> today's lock-busy source scan reachable. Add both-state tests, and keep the `Off`
> branch isolated so its later removal is a deletion.

> [!decision] tui_perf_rule Through `/sase_memory_write`, add one concise rule to
> `tui_perf.md`: a busy artifact-index lock is answered from the last good snapshot plus
> a retry through the existing flags, never with an O(archive) source scan. Scans remain
> only for a missing index, a query error, or startup with no snapshot. Name
> `query_artifact_index_for_loader`. Run `sase memory init` afterwards.

## Phase: core-session-lookup

Goal: in the linked `sase-core` repo, session/name lookups cost O(matches), and the
stranded query's reconcile cannot delete rows outside its scope. Open the repo with
`sase repo open sase-core` and read its `AGENTS.md`. Paths below are under
`crates/sase_core/src/agent_scan/` unless noted.

- **Pushdown.** Today `candidate_filter` is evaluated only in Rust
  (`index/candidates.rs:candidate_filter_matches`), after `selection.rs` selects every
  row of the tier.
  - Translate an `equals` on `agent_session` or `agent_name`, an `any` of such `equals`,
    and such terms inside a top-level `all`, into a SQL predicate. Apply it in both the
    full selection and the windowed candidate query.
  - Follow the `add_project_filter_to_where` / `select_dirs_in_column` patterns, with
    bound parameters.
  - The SQL predicate must select a superset of what Rust accepts. Rust equality
    lowercases both sides, while SQLite `NOCASE` folds ASCII only. So push down only
    ASCII values, after confirming (and citing) that agent names and sessions are
    validated ASCII slugs at creation. If they are not, skip pushdown for that filter.
  - Keep the Rust post-filter as the exact backstop. Leave every other field unpushed.
- **Indexes.** Add `CREATE INDEX IF NOT EXISTS` entries for
  `agent_artifacts(agent_session COLLATE NOCASE)` and
  `agent_artifacts(agent_name COLLATE NOCASE)` in the post-migration index block of
  `index/storage.rs`. The existing `agent_session` index is BINARY and cannot serve a
  `NOCASE` comparison, and there is no `agent_name` index on `agent_artifacts`.
  - Do not bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION`. A bump can trigger the TUI's
    stale-schema rebuild path.
  - Read-only opens will not create the indexes. Queries must stay correct without them.
  - Assert index use with `EXPLAIN QUERY PLAN` in a test.
- **Latent reconcile bug (fix it here; it sits on the stranded query path).**
  `reconcile_source_directories` (`index/refresh.rs`) walks with
  `discovery_scan_options`, which keeps `only_workflow_dirs`, but it diffs against
  _every_ indexed row under the projects root. The stranded query passes
  `only_workflow_dirs=("ace-run",)`, so any non-`ace-run` row would be deleted on every
  stranded report and rediscovered by the TUI's reconcile.
  - All 22,268 rows are `ace-run` today, so nothing has been lost yet.
  - Restrict the diff and deletions to rows inside the walked workflow scope.
  - Stamp the source-reconcile watermark only when the walk covered the scope the
    watermark claims.
  - Replace the `missing.iter().any(..)` scan with a set lookup.
  - Add a regression test: a non-`ace-run` row survives a revalidated full-history query
    with `only_workflow_dirs=["ace-run"]`.
- **Tests:**
  - Extend `index/tests/agent_session_filter.rs` and `agent_name_filter.rs` with
    pushdown-versus-no-pushdown parity, covering case variants, the `any`/`all` shapes,
    a non-ASCII value that is not pushed down, and windowed queries.
  - Run sase-core's own checks per its `AGENTS.md`.
  - Use Conventional Commits. Never edit versions or changelogs by hand (release-plz
    owns them).
- **Pin.** No new binding is needed. Make sure sase's `sase-core-revision.txt` moves
  past the new core commit:
  - The host does this automatically when your declaration commits both repos. See "The
    CI source revision pin" in `docs/contribute/rust-backend.md`.
  - From the sase side, run the stranded collector timing from `stranded-batch` against
    the locally built core, and record the before and after in a bead note.

> [!decision] core_lookup = binding Also add a
> `stranded_session_records(index_path, projects_root, project, keys)` binding. It runs
> the source reconcile at most once, selects matches through the new indexes, and
> returns records grouped by key with the exact recheck.
>
> - Register it in `sase_core_py`, with a round-trip test in
>   `sase_core_py/src/agent_scan/tests.rs`.
> - In sase, add it to `REQUIRED_BINDINGS` in `tools/validate_sase_core_rs` and wrap it
>   in `src/sase/core/agent_scan_index_queries.py`.
> - Switch `stranded_facts` to it, keeping the batched-query parity tests. There is no
>   Python fallback; the Rust core is required.

## Phase: agents-tick-trims

Goal: steady-state Agents ticks and capacity applies stop doing work whose inputs did
not change. Displayed output stays identical.

- **Capacity patches:**
  - `src/sase/ace/tui/actions/agents/_loading_filter.py` (around line 220) calls
    `try_patch(row, refresh_info=False)`, but `refresh_title` defaults to `True`
    (`_display_panel_patches.py`, around line 401). Pass `refresh_title=False` there.
  - Refresh affected panel titles once per apply, and only when their title inputs
    changed.
- **Skip unchanged capacity applies:**
  - Today each compute makes up to two `copy_agent_rows_relinked(roster)` copies (around
    lines 91 and 100) and always clears the metrics cache and refreshes the info panel.
  - Skip the copies, the patch pass and the panel refresh when the occupancy inputs
    (report token or records, plus roster identity) equal the last applied ones.
  - Keep the generation checks.
- **Runtime aggregation:**
  - In `src/sase/ace/tui/models/_agent_time_aggregate.py` (`_aggregate_runtime`, around
    lines 260–287), the recursive member walk and per-member `_member_fingerprint` run
    before the cache lookup. That happens every second for every ticking clan row.
  - Check a cheap key first: clan identity, a runtime-input revision, and the display
    bucket. Bump the revision whenever agent rows are applied or patched.
  - Fall back to the full fingerprint path on a revision change.
  - Live intervals must still advance every second.
- Tests:
  - Runtime parity: displayed runtime strings are identical across a simulated tick
    sequence that includes member start, finish and reparent events.
  - A capacity apply with unchanged inputs performs no copy or patch.
  - A capacity apply with changed occupancy patches exactly the changed rows and
    refreshes titles once.
- Run the j/k bench, `pytest -s -m slow tests/ace/tui/bench_tui_jk.py`, before and
  after. Record p50/p95 in a bead note.

## Phase: pump-probe-trims

Goal: remove three small synchronous costs from UI timers and keystroke paths.

- **GC trim worker:**
  - `src/sase/ace/tui/util/gc_policy.py:_spawn_trim_thread` starts a new `Thread` per
    dispatch (at most once per 10 min, from the UI tick). Under GIL contention,
    `Thread.start()` has blocked the pump.
  - Replace it with one persistent daemon worker that waits on a `threading.Event`.
    Start it at install and stop it at uninstall.
  - Change nothing else in the GC policy: thresholds, quiet windows and collection
    placement stay as they are.
- **Tool-runs token probe:**
  - `_maybe_probe_tool_runs_drift` (`src/sase/ace/tui/tool_runs/loader.py`, about every
    2 s from the countdown) stats `runs.sqlite` and its WAL on the UI thread.
  - Consume the token that the auto-refresh worker's surface probe already collects
    (`_surface_tokens.py`), or run the probe inside a pump-free task. Keep the drift
    detection semantics.
- **Prompt subtitle:**
  - `on_text_area_changed` (`_prompt_input_bar_stack_lifecycle.py`) calls
    `refresh_cursor_readouts()`, which rebuilds the Rich `border_subtitle` on every
    keystroke and selection change.
  - Skip the rebuild when the rendered inputs are unchanged.
  - Otherwise coalesce it through `DetailPanelDebouncer`
    (`src/sase/ace/tui/util/debounce.py`), so typed text and the caret paint immediately
    and the readout settles within about 150 ms.
  - Update the "never debounced" docstring and any tests that assert a synchronous
    subtitle. Tests must still prove the final readout is correct.
- Tests cover all three:
  - the trim worker starts once and stops cleanly at uninstall;
  - tool-runs drift is still detected;
  - the prompt subtitle converges and is skipped when nothing changed.

## Phase: capacity-feed

Goal: the machine-capacity display feed stops repeating a 17 s, 0.45 GB artifact walk
every 30 s.

- In `src/sase/core/runner_slots/_machine_capacity.py`:
  - Raise `DISPLAY_RESCAN_INTERVAL_SECONDS` from 30 s to 120 s.
  - The stale-cache path already serves axe's `runner_slots.scan.json` with
    `scan_current=False` while rate-limited. Keep that, and keep forced rescans working.
- In `_scan_cache.py`, reuse the parsed cache while the file's `(mtime_ns, size)` and
  token are unchanged instead of re-running `json.loads` on every 10 s feed tick.
- Move `probe_runner_capacity_token` (three stats) out of the synchronous
  `_on_machine_capacity_feed_tick` UI timer
  (`src/sase/ace/tui/actions/_auto_refresh.py`). Run it in the feed's existing pump-free
  task, or consume the surface-probe token.
- An index-backed capacity query is out of scope. If the walk still shows up, record a
  `PROPOSED FOLLOW-UP`.
- Tests:
  - rate limiting at the new interval;
  - parsed-cache reuse and invalidation on file change;
  - forced rescans still rescanning;
  - the feed tick no longer stats on the UI thread.

## Phase: remeasure-kit

Goal: the user can restart the TUI and verify the fix mechanically, and Step 3 and Step
4 of the research are handed off with their gates intact. Read `tools.md` and
`docs_writing.md` with `/sase_memory_read` first.

- **Freeze report.** Extend `tools/tui_freeze_report` (via its `_tui_freeze_report_*.py`
  helpers) with a slow-agent-load section that reads
  `~/.sase/logs/tui_agent_loads.jsonl` and its `.1` rotation, filtered by instance and
  window like the existing sections. For each instance it reports:
  - slow full loads per hour;
  - the `index` sub-stage in buckets: <1 s, 1–5 s, 5–15 s, 15–30 s and ≥30 s;
  - counts by `artifact_source` and `repair_reason`. Rows without the new fields count
    as `unrecorded`.
  - the top `source` values.

  Emit it in both the text and the JSON output, with fixture-based tests beside
  `tests/test_tui_freeze_report_*.py`.

- **Runbook.** Add a "Re-measure after the index-lock fix" recipe to
  `docs/contribute/perf-runbook.md`, covering:
  - restarting the TUI first, because a running TUI keeps its imported code;
  - `SASE_TUI_PERF=1` and the capture preflight;
  - the leading indicators: the ≥15 s index-stage mode falls from about 84% of slow full
    loads to about 0%, RSS stays under about 5 GB, and swap stays near zero;
  - the acceptance targets: j/k p95 < 16 ms, idle full-GC p95 ≤ 0.5 s, GC < 3% and
    hitches < 0.5% of wall time, RSS growth < 0.5 GB over 4 h, zero swap;
  - the GC decision rule. Only if idle gen-2 p95 still exceeds 0.5 s, adopt the
    long-quiet, cost-gated gen-2 policy on the GC track. Never lower `threshold2`,
    disable GC, or "move" collection to a worker;
  - the one-time `SASE_TUI_GC_CENSUS=1` census, run before any retention work.
- **Follow-ups.** Record `PROPOSED FOLLOW-UP` notes on this phase bead for the land
  agent:
  1. Re-run the GC track `sase-1nc`'s soak on a restarted instance now that this epic
     has landed. Its earlier soak failures measured this load regression.
  2. The GC census, to attribute the residual 8 M → 19 M block growth.
  3. Documenting-then-splitting the index lock into reader and writer locks.
  4. An immutable, versioned Agents read model.
  5. The JSON name-registry retention gap.
  6. If earlier phases' notes show it still matters, the per-report full
     source-reconcile walk on the stranded query.

## Out of scope and follow-ups

- **GC policy changes** (research Step 3) wait for a measured, restarted instance. The
  existing READY track `sase-1nc` owns them, and `remeasure-kit` hands that sequencing
  off.
- **Structural work** (research Step 4) stays evidence-gated: a reader/writer lock
  split, an immutable read model, and registry retention.
- **Rejected by the research; do not implement:**
  - lowering `threshold2`;
  - `gc.disable`;
  - worker-thread `gc.collect()`;
  - a new single-flight load coordinator;
  - clearing Textual's private `Strip` caches.

## Phase ordering

`stranded-batch`, `stranded-cadence`, `loader-lock-busy` and `pump-probe-trims` can
start together. The remaining edges exist for two reasons:

- **Measurement:** `core-session-lookup` measures against the batched query.
- **Shared files:** the other edges serialize phases that would edit the same files.
  - `agents-tick-trims` waits for `stranded-cadence` (both touch the info-panel path)
    and for `loader-lock-busy` (both touch the Agents apply path).
  - `capacity-feed` waits for `pump-probe-trims` (both may touch the surface-token
    probe).

`remeasure-kit` runs last, so its runbook describes the final state.

## Verification summary

Each phase runs `just check` plus its focused tests and records its measurements in a
bead note. Live acceptance is a human step after restart, using the `remeasure-kit`
recipe.
