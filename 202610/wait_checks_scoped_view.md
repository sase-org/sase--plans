---
tier: epic
title: Scope wait resolution to live waiters so wait_checks stops timing out
goal: 'A wait_checks tick with live named-agent waiters loads only the artifact rows
  those waiters can depend on instead of the whole artifact history. Ticks stay far
  below the 2-minute job timeout under host load, release latency moves toward sase-1hn''s
  p50 <= 5 s / p95 <= 15 s target, and parked runners'' 60 s fallback stops rebuilding
  the full project history. Release decisions stay exactly what the full view would
  decide.

  '
phases:
- id: tick-diagnostics
  title: Make wait_checks timeouts diagnosable
  depends_on: []
  size: small
  description: 'tick-diagnostics: flush chop log lines so a SIGKILLed job keeps its
    output, and add per-stage progress lines plus integer timing and view-size counters
    to the wait_checks summary.'
- id: batch-index-build
  title: Remove superlinear and redundant work from WaitDependencyIndex.add_many
  depends_on: []
  size: medium
  description: 'batch-index-build: refresh effective clan tribes once per touched
    clan generation, invalidate query caches once per batch, and skip plan_path.json
    reads that cannot matter, with a parity test proving batch and sequential adds
    build identical indexes.'
- id: scope-keys
  title: Shared scope keys and scoped row selection for named waits
  depends_on:
  - batch-index-build
  size: medium
  description: 'scope-keys: factor the identity keys _add_prepared files a row under
    into one helper, add marker scope keys (None for unscopable shapes) and row selection,
    and prove scoped and full views give identical decisions for named waits.'
- id: scoped-wait-checks
  title: Resolve scopable waiters from a scoped view in wait_checks
  depends_on:
  - tick-diagnostics
  - scope-keys
  size: medium
  description: 'scoped-wait-checks: build the resolving and confirmation views from
    scoped rows for scopable live waiters, build the full view lazily only for unscopable
    ones, and require a read-only host parity sweep over every waiting marker.'
- id: scoped-runner-fallback
  title: Scope the runner's startup and 60 s fallback resolution
  depends_on:
  - scope-keys
  size: small
  description: 'scoped-runner-fallback: make _resolve_marker_release build scoped
    project indexes for scopable markers, keeping the full project build for every
    other shape.'
- id: scope-key-cache
  title: Persistent scope-key cache so live ticks stat instead of read
  depends_on:
  - scoped-wait-checks
  - scoped-runner-fallback
  size: medium
  description: 'scope-key-cache: persist per-artifact scope keys fingerprinted by
    agent_meta.json stat data, written only by wait_checks and read by runners, so
    selection costs one stat per artifact instead of one JSON read.'
proposed_by: bbugyi200.athena.0zo
create_time: 2026-10-11 11:18:14
status: wip
bead_id: sase-1jy
---

- **PROMPT:** [prompts/202610/wait_checks_scoped_view.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/wait_checks_scoped_view.md)
- **BEAD:** [sase-1jy](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jy/README.md)

# Plan: Scope wait resolution to live waiters so wait_checks stops timing out

## Context

### The incident

`~/.sase/axe/error_digests/digest_20261011_104924.txt` reports
`agent_waits / wait_checks: timed out after 120s` (exit -9, `[output absent]`). This is
not a one-off. The `agent_waits` routine log
(`~/.sase/axe/lumberjacks/agent_waits/logs/output.log`) shows:

- **5 timeouts in ~3 days:** 2026-10-10 13:28 and 13:32, 2026-10-11 07:52, 07:56, and
  10:37.
- **Tick duration:** p50 8.6 s, p90 12.3 s, p99 36 s. During busy hours the job runs
  back-to-back at 70–108 s, because the `.ace_refresh_pulse` trigger changes every tick.
  Between 10:33 and 10:49 on 2026-10-11, every run took 70–120 s.
- **Duration grows with history.** At 06:06 on 2026-10-08 the corpus was 16,963
  artifacts; it is now 17,891, growing about 290 per day. Of these, 16,060 are in
  `gh_sase-org__sase`.

The timeout has real cost. The most recent `ready_json` / `runner_fallback` release,
`research.49.linker.w1`, was released by the **runner fallback** at 10:37:32 EDT with
`wait_release_latency_s = 152`. That was two seconds after `wait_checks` was killed.

Release telemetry over the last 4 days, from `wait_release_latency_s` in
`agent_meta.json` (see "Wait release telemetry" in `docs/axe.md`):

- **Latency:** p50 14.6 s, p95 about 1,022 s (n = 23).
- **Fallback share:** 15 of 72 dependency releases were `runner_fallback`.

That misses the gate on task **sase-1hn** ("Scoped or incremental wait resolution ... if
release latency misses p50 <= 5 s / p95 <= 15 s"). This epic implements sase-1hn's
item 1.

### Root cause (read-only measurements on athena, 2026-10-11, load ~13)

After sase-1hf, a tick with no live waiter costs about 0.6 s. Since then, though, epics
keep about 18 live named-agent and bead waiters parked for days; those are legitimate
`*.land` and phase waiters on in-progress epics. A tick with at least one live waiter
loads the **entire** artifact history:

| Stage                                                               | Time      |
| ------------------------------------------------------------------- | --------- |
| imports                                                             | 0.55 s    |
| `scan_waiting_markers` walk (37 projects, 17,891 dirs, 913 markers) | 0.36 s    |
| liveness classification (18 live, 803 dead)                         | 0.03 s    |
| `_filesystem_dependency_rows` (reads all 17,870 `agent_meta.json`)  | 1.8 s     |
| `WaitDependencyIndex.add_many` over 17,870 rows                     | 2.9 s     |
| first bead-store touch (Rust read-model cold open, then ~50 ms)     | 1.1–2.7 s |
| per-waiter resolution (all 18)                                      | ~0.6 s    |
| **total** (peak RSS 643 MB, ~7.2 CPU-s)                             | **~8 s**  |

cProfile of the view build shows about 62k JSON opens and 32k stats per tick. Inside
`add_many`, the costs are:

- `done.json` reads for every row.
- `_refresh_effective_clan_tribe`: 9,757 calls, ~1.0 s. It re-resolves an entire clan
  generation on every row add, which is O(members²).
- `plan_path.json` reads for every row: 17,870 opens, ~0.43 s.
- Agent-session-turn parsing.

When a waiter becomes releasable, the confirmation pass rebuilds a fresh full view of
the confirmation projects (another ~3–4 s). So the slowest ticks are exactly the ones
that would release someone. Under host contention (load avg 22–30, ~14 GiB swapped),
this allocation-heavy O(history) work stretches 8–13x to 70–120 s and hits the SIGKILL.
`ChopLogger` prints without flushing, so a killed run leaves an empty log (hence
`[output absent]`).

The runner side does the same thing.
`src/sase/axe/run_agent_wait_deps.py::_resolve_marker_release`, used by the startup fast
path and by the 60 s fallback that every parked runner polls, calls
`build_wait_dependency_index(project_name)`. For `gh_sase-org__sase` that costs 4.2 s
and ~470 MB peak. With 18 parked runners, that is roughly 75 CPU-s per minute of
background load. That load makes `wait_checks` itself slower.

### Why scoping is safe

The dependency shape of ~98% of all `waiting.json` markers on athena is plain
`waiting_for` names, with or without `wait_for_beads`:

| Shape                               | Markers |
| ----------------------------------- | ------- |
| `waiting_for` only                  | 620     |
| `waiting_for` + `wait_for_beads`    | 137     |
| `wait_for_beads` only               | 3       |
| `wait_for_epics_of` + `waiting_for` | 7       |
| `wait_for_fork_sources`             | 7       |
| `@tribe` names                      | 5       |
| `wait_for_artifacts`                | 1       |

A plain named wait `X` resolves through `WaitDependencyIndex.is_resolved(X)` (in
`src/sase/core/wait_dependency_resolution/_index_queries.py`). That consults only
entities keyed by `X`:

- `clan_candidate(X)`: rows whose clan name is X.
- `agent_session_candidate(X)`: rows whose session base is X. Its handoff-successor
  presence checks look only at names inside that session's own members, plus the
  waiter's own row via `_excluded_present_names`.
- `workflow_candidate(X)`: rows whose `workflow_name` is X.
- `named[X]`: rows whose `name` is X.
- `_planner_row_candidate(X)`: a submitted planner row whose `<base>--plan` name is X or
  its legacy alias.

`_add_prepared` (in `_index.py`) files each row under exactly these keys. So for named
waits, an index built only from:

- rows whose identity keys intersect the waiter's dependency names (plus planner-row
  aliases), and
- the waiter's own row,

answers every query identically to the full index. Extra rows belong to other entities
and cannot change those answers. The exceptions are `@tribe` names (global tribe
universe), hoods, fork sources, identity deps, and armed epic follows. Those stay on the
full view.

### Boundaries and invariants

- **Same decisions, less data.** Scoping changes only which rows a view is built from,
  never resolution semantics. Every release still goes through `resolve_wait_release`
  with the fresh-membership confirmation. The fail-closed release invariant from
  sase-1hf is unchanged. Unscopable markers keep today's full view.
- **Single source of truth for keys.** Selection must use the same helper
  `_add_prepared` uses to derive name, workflow, session, clan, and planner-row keys, so
  the two cannot drift.
- **Rust boundary.** The wait-dependency resolver already lives in Python
  (`src/sase/core/wait_dependency_resolution/`), and sase-1hf recorded "porting
  resolution to Rust is out of scope". This epic only changes how much history the
  existing Python resolver loads, so it needs no sase-core change. Porting stays out of
  scope.
- **No timeout change.** `job_timeout: "2m"` on `agent_waits` stays. The fix is to make
  ticks cheap; the timeout is the alarm that caught this.
- **Polling backstops stay.** `max_quiet: "120s"` stays, and so does the runner's 60 s
  fallback interval.
- **Host safety.** Never run the real `wait_checks` job against the live host; it would
  publish real `ready.json` markers. Measure with read-only snippets that never call
  `publish_ready_marker` or `apply_wait_epic_follow_patch`, and pass a no-op `sync_hint`
  to bead lookups. Do not restart the scheduler or run `sase update`.
- Every phase finishes with `sase tool run check` (the lint-and-test memory) and updates
  `docs/axe.md` when behaviour or counters change.

### Out of scope (record, do not do)

- **sase-1hn item 2:** scheduler event-wake. It stays gated on post-deploy telemetry.
- **sase-1hk:** `bead_claim_checks` full-history index query in the `waits` routine.
- **sase-1hm:** archiving dead `waiting.json` markers. Dead markers already cost ~0.03
  s.
- **Bead read-model cold open** (~1.1 s on the first call per process). It is
  per-process, not O(history).

## tick-diagnostics

Goal: the next slow or killed `wait_checks` run says where its time went.

1. **Flush chop logs.** `ChopLogger` (`src/sase/chops/sdk.py`): flush the stream after
   every `info` / `debug` / `warning` / `error` line (`print(..., flush=True)`). Lines
   already logged then survive the lumberjack's SIGKILL in the run log and in the error
   digest's "Subprocess Output". Add a test with a non-tty stream that records `flush()`
   calls.
2. **Progress lines.** In `src/sase/scripts/_chop_wait_checks_run.py`, emit one progress
   line after the waiter walk and liveness classification (counts plus elapsed seconds).
   Emit another after building the resolving view (row count plus elapsed seconds).
   Verify `parse_summary` does not treat these lines as summaries.
3. **Summary fields.** Add integer summary fields: `walk_ms`, `view_ms`, `resolve_ms`
   (the per-waiter loop), and `view_rows` (rows in views built this tick, 0 when none).
   Use monotonic timers. Update tests that assert exact summary dicts.
4. **Docs.** List the new counters in the `wait_checks` part of `docs/axe.md`.

## batch-index-build

Goal: `WaitDependencyIndex.add_many`
(`src/sase/core/wait_dependency_resolution/_index.py`) does linear, non-redundant work
and yields an index state identical to adding the same rows one at a time with `add()`.
This helps every caller: `wait_checks`, the runner fallback, and anything using
`build_wait_dependency_index`.

1. **Defer clan-tribe refresh.** In batch adds, skip the per-row
   `_refresh_effective_clan_tribe(clan, generation)` call. Collect the touched
   `(clan, generation)` keys and refresh each once after all rows are added. `add()` and
   `add_scan_record()` keep the immediate refresh. First confirm that nothing in
   `_add_prepared` reads `effective_clan_tribes` mid-batch.
2. **Invalidate caches once.** Call `_invalidate_query_caches()` once per batch instead
   of once per row.
3. **Skip pointless `plan_path.json` reads.** Today every row reads `plan_path.json`
   through `plan_path_marker` before `submitted_plan_artifact` checks its meta-only
   preconditions. Those are: no outcome, plan role suffix, a submission marker, not
   approved, and no feedback marker. Read the marker only when the preconditions hold
   (for example, pass a lazy marker callable, or check the preconditions first).
   `_add_prepared` uses only `planner_row_name`, which does not depend on the marker
   value, so results are identical.
4. **Other redundant I/O.** Remove other redundant per-row stats only where equivalence
   is exact. Example: carry the prepare-step `done.json` existence into `_add_prepared`
   instead of re-statting it. Leave anything that would change race behaviour alone.
5. **Parity test.** Over a fixture corpus, build one index with `add_many` and another
   with sequential `add()`. The corpus covers:
   - multi-member clans across generations, with and without a recorded clan tribe in
     the clan record;
   - posthoc tribes;
   - submitted and approved planner rows, with and without `plan_path.json`;
   - queued and dependency-parked rows;
   - `done.json`-less rows using the dismissed-archive fallback.

   Assert equal `named`, `workflows`, `agent_sessions`, `clans`, `tribes`,
   `effective_clan_tribes`, `artifacts`, and `artifacts_by_dir`.

6. **Measure.** Time a read-only
   `WaitDependencyIndex.empty().add_many(_filesystem_dependency_rows(sase_projects_dir()))`
   on athena before and after, and record both in a bead note.

## scope-keys

Goal: a tested API that computes, for a waiting marker, the minimal row set a view needs
to answer that marker exactly as the full view would. No caller changes in this phase.

1. **Shared identity helper.** Factor out one helper, for example `_row_identity(meta)`
   in `_index.py` or a new `_scope.py` beside it. It derives what `_add_prepared` files
   a row under:
   - `name`;
   - `workflow_name`;
   - `agent_session_base_from_meta(meta)`;
   - the clan name, including the legacy `agent_family` / parallel-session fallback;
   - the submitted-planner row name.

   Make `_add_prepared` use it. Then add `artifact_scope_keys(meta) -> frozenset[str]`
   returning those keys. For the planner row, return a **superset** computed from meta
   alone: whenever `canonical_plan_chain_suffix(meta["role_suffix"])` is the plan suffix
   and the name is usable, include
   `agent_session_phase_name(agent_session_base(name) or name, PLAN_CHAIN_PLAN_SUFFIX)`,
   regardless of outcome or marker. Extra keys only add rows, which never changes
   answers.

2. **Marker scope keys.** Add `marker_scope_keys(marker) -> frozenset[str] | None`. It
   returns `None` (unscopable, so the caller uses the full view) when the marker has any
   of:
   - non-empty `wait_for_artifacts`, `wait_for_fork_sources`, or `wait_for_hoods`;
   - armed epic follows (`armed_wait_epic_targets(marker)`, `wait_for_epics_of`, or
     `wait_epic_follows`);
   - any `waiting_for` entry that is not a string or starts with `@`.

   Otherwise it returns each `waiting_for` name plus
   `planner_row_name(name, include_legacy_dash=True)` when that differs. Bead-only
   markers return an empty set. `resolved_deps` needs no keys, because memoized names
   are skipped.

3. **Row selection.** Add `select_scope_rows(rows, keys, *, include_dirs)`. It keeps
   rows whose `artifact_scope_keys` intersect `keys`, plus rows whose artifact dir is in
   `include_dirs` (the waiters' own dirs, needed by `_excluded_member_name` and
   self-exclusion). Export the new names from the package `__init__` as the existing API
   is exported.
4. **Query audit.** Audit every index query reachable for a scopable marker:
   - `resolve_wait_release` (non-armed path) and `confirm_dependency_resolution`;
   - `dependency_member_dirs`;
   - `terminal_blockers` (`src/sase/scripts/_chop_wait_checks_terminal.py`);
   - `latest_member_finished_at`.

   Confirm each reads only entities keyed by a dependency name or the waiter's own row.
   If any reads a global structure (for example the tribe universe or all artifacts),
   either add the needed keys or make that shape unscopable. Record the audit in a bead
   note.

5. **Parity tests (the heart of this phase).** Write a helper that, for a corpus and a
   marker, compares full and scoped views on:
   - `resolve_wait_release` (`status.resolved`, `blocked_on`, `releasable`,
     `confirmation_failed`, with a `fresh_index` built the same way per view);
   - `dependency_member_dirs`;
   - `terminal_blockers`.

   Run it over every named-wait fixture already used by
   `tests/test_axe_chop_wait_checks*.py` and `tests/test_wait_*.py`. Also run it over
   targeted corpora:
   - retries and agent sessions (`X`, `X--1`, ...);
   - turn handoffs and plan-to-code chains, with follow-up present and absent;
   - clans across generations;
   - multi-step workflows;
   - submitted planner rows (`base--plan` and the legacy `base.plan` spelling);
   - queued and dependency-parked members;
   - a newer failed run after an older success (must stay blocked);
   - unknown terminal outcomes (terminal blocker);
   - cross-project names;
   - the waiter sharing its dependency's session.

   Every case must match exactly.

## scoped-wait-checks

Goal: `wait_checks` builds small views for scopable live waiters and the full view only
when an unscopable live waiter needs it.

1. **Read markers once.** After liveness classification, read each live waiter's
   `waiting.json` once and reuse the parsed dict in the per-waiter loop, so the marker
   is no longer read a second time. Compute `marker_scope_keys` for each.
2. **Resolving views.**
   - Load rows for all projects as today, but compute `artifact_scope_keys` for every
     row. Build the scoped index from
     `select_scope_rows(rows, union_of_scopable_keys, include_dirs=scopable_waiter_dirs)`.
   - Build the full index lazily, at most once per tick, only if some live waiter is
     unscopable. Use it for those waiters.
   - Keep per-path scope keys for all rows, but retain full meta dicts only for selected
     rows (and for the full view when one is built). That keeps peak memory proportional
     to the view.
3. **Confirmation views.** In scoped mode, `fresh_index` re-lists the confirmation
   projects' artifact dirs fresh, as today. It reads meta only for dirs this tick has
   not seen; that is the new-member signal the confirmation exists to detect. It builds
   the fresh index from rows matching **that waiter's** keys plus its own dir. Include
   the scope in the `fresh_indexes` memo key, so a scoped and a full confirmation never
   share an entry. Full-mode confirmation is unchanged.
4. **Counters.** Add `scoped_waiters` and `full_view_waiters`, and keep `view_rows` and
   the timing fields from tick-diagnostics.
5. **Host parity sweep (required, read-only).** With a throwaway script (not committed),
   evaluate every `waiting.json` marker on athena (live, dead, and already-ready; ~900)
   against both the full and the scoped view. Compare `resolve_wait_release` (with
   `fresh_index`), `dependency_member_dirs`, and `terminal_blockers`. Never publish or
   patch anything, and use a no-op bead `sync_hint`. **Zero mismatches are required.**
   Record marker counts per shape, mismatches (0), and before/after live-tick timings in
   a bead note.
6. **Tests.**
   - A tick with one scopable and one unscopable live waiter (fork source, hood,
     `@tribe`, epic follow, or identity dep) uses both views correctly. Spy on
     `add_many` sizes or view counters.
   - Existing confirmation tests pass in scoped mode.
   - A dependency member created between the resolving view and confirmation defers
     release. Cover a newer failed run of the dependency.
   - Bead-only waiters still resolve from bead state.
7. **Docs.** Update the `wait_checks` section of `docs/axe.md` (scoped view, which
   shapes fall back to the full view, new counters). Update the `wait_checks`
   description in `src/sase/default_config.yml`, which currently says it resolves "from
   filesystem agent metadata".

## scoped-runner-fallback

Goal: parked runners stop rebuilding the whole project history every 60 s.

1. **Scope the build.** In
   `src/sase/axe/run_agent_wait_deps.py::_resolve_marker_release`, when
   `marker_scope_keys(marker)` is not `None`, have `build_index()` build a scoped index
   over the **same project**. Use rows from that project's ace-run dirs, filtered by the
   marker's keys plus the runner's own `artifacts_dir`, for both the resolving index and
   the `fresh_index` confirmation. Keep `global_stored_tribes` as today. Unscopable
   markers, including markers with forwarded identity deps, keep the full
   `build_wait_dependency_index(project_name)` build.
2. **Keep everything else.** The startup fast path (`resolve_initial_wait_release`), the
   60 s interval, the project scoping, and the fail-closed exception handling stay as
   they are.
3. **`_release_satisfied_at`.** Check whether it builds another index. If it does, scope
   it the same way.
4. **Tests.**
   - Scoped and full runner decisions match for named waits, including the
     handoff-follow-up and newer-failed-run cases.
   - Unscopable markers still call `build_wait_dependency_index` (spy).
   - Existing `tests/test_run_agent_wait_deps*.py` pass.
5. **Measure.** Time a read-only scoped versus full runner resolution for a current live
   waiter, and record it in a bead note.

## scope-key-cache

Goal: remove the last O(history) file reads from live ticks and runner fallbacks.
Selection costs one `stat` per artifact instead of one JSON read.

1. **Cache module.** Add a cache module beside the scope helpers. Store one JSON file
   under `sase_home() / "cache"` (for example `wait_scope_keys.json`, with
   `schema_version`). It maps each ace-run artifact dir to its `agent_meta.json`
   `(st_mtime_ns, st_size, st_ino)` and its sorted `artifact_scope_keys`.
2. **Selection.**
   - For each walked dir, `os.stat` its `agent_meta.json`.
   - A fingerprint hit reuses the cached keys.
   - A miss or new dir reads the meta, computes keys, and updates the entry.
   - A missing meta means no row, as today.
   - Dirs that vanished are dropped.

   Selected rows always read their meta and markers fresh, so the cache can only affect
   **which** rows are selected, never their content.

3. **Writers and readers.** Only `wait_checks` writes the cache: temp file plus
   `os.replace`, and only when an entry changed. Runners read it read-only; on a miss
   they read meta themselves and never write. An unreadable, corrupt, or wrong-schema
   cache is treated as empty, so everything is read. It is never treated as "no rows".
4. **Confirmation.** New dirs still read meta (cache miss), which preserves the
   new-member signal.
5. **Tests.**
   - Hit, miss, and invalidation: a meta rewrite changes keys, so selection changes.
   - A corrupt cache falls back to reads.
   - Removed dirs are dropped.
   - A runner's read-only use never writes.
   - Scoped decisions are identical with a cold cache and with a warm cache.
6. **Measure and document.**
   - Re-run the host parity sweep from scoped-wait-checks with the cache warm; it must
     still show zero mismatches.
   - Time a read-only live tick and a runner fallback resolution, and record the numbers
     in a bead note.
   - Document the cache in `docs/axe.md`, including that deleting it is always safe.

## Landing notes

- **sase-1hn.** The land agent adds a note to task sase-1hn naming this epic as the
  implementation of its item 1, with the final timings. It leaves item 2 (scheduler
  event-wake) gated on post-deploy `wait_release_latency_s` telemetry and does not close
  the task unless that item is explicitly declined.
- **Expected outcome after deploy.**
  - A live `wait_checks` tick costs walk + stats + small views + bead cold open, about
    2–3 s on an idle host and far below 120 s under load. No timeout digests.
  - Release latency is about tick + run (~3–5 s) instead of ~10–15 s.
  - Parked runners' fallbacks cost well under a second each.
