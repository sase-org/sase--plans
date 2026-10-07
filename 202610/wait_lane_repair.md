---
tier: epic
title: Repair the wait lane so waiting agents wake on completion, not on the fallback
goal: 'Agent dependency waits are released by wait_checks within seconds of the dependency
  finishing (instead of by the runner''s 60 s fallback), ready.json publication is
  race-free, every release records its source and latency, and wait_checks runs in
  its own fast lane resolving only live waiters.

  '
phases:
- id: atomic-ready
  title: Race-free ready.json publication and reading
  depends_on: []
  size: small
  description: 'atomic-ready: publish ready.json via temp file plus no-clobber link,
    skip publishing once the waiter is gone, and make the runner treat unreadable
    or malformed ready.json as not ready.'
- id: pulse-trigger
  title: Point wait_checks and bead_claim_checks at the completion pulse
  depends_on: []
  size: small
  description: 'pulse-trigger: replace the blind ace-run/* fs glob with the per-project
    .ace_refresh_pulse, touch the pulse on dependency waiting-marker writes, audit
    done.json writers, and rewrite the trigger tests on the real YYYYMM/DD/<run> layout.'
- id: release-telemetry
  title: Record wait release source and latency (research Phase 0)
  depends_on:
  - atomic-ready
  size: medium
  description: 'release-telemetry: stamp wait_release_source, dependency-satisfied
    time, release latency, admission latency, and runner-slot wait into agent_meta.json;
    wait_checks adds released_by and dependencies_satisfied_at to ready.json.'
- id: live-waiters
  title: Resolve only live waiters from a filesystem view
  depends_on:
  - release-telemetry
  size: medium
  description: 'live-waiters: shared waiting-marker walk plus tri-state runner liveness,
    skip dead waiters and the dependency-view build when no live waiter is pending,
    build the resolving view from filesystem rows instead of the slow index query,
    add backlog counters, and reuse the walk in sidecar_auto_sync.'
- id: lane-split
  title: Give wait_checks and sidecar_auto_sync their own routines
  depends_on:
  - pulse-trigger
  - live-waiters
  size: small
  description: 'lane-split: new agent_waits (2 s) and sidecar_sync (30 s) routines,
    post-sync beads pulse, 0.5 s ready.json poll in the runner, test and docs updates,
    and the consolidated PROPOSED FOLLOW-UP list.'
proposed_by: bbugyi200.athena.0y0
create_time: 2026-10-07 14:45:40
status: wip
bead_id: sase-1hf
---

- **PROMPT:** [prompts/202610/wait_lane_repair.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/wait_lane_repair.md)
- **BEAD:** [sase-1hf](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hf/README.md)

# Plan: Repair the wait lane (research Phases 0 and 1)

## Context

Source: the research report
`research:202610/wait_wakeup_jobs_vs_service_procs/wait_wakeup_jobs_vs_service_procs.md`
("Should Builtin Jobs Become Service Procs? Waking Waiting Agents Sooner"). Its verdict:
do **not** migrate `wait_checks` (or any builtin job) to a service proc; instead fix the
`waits` lane inside the job model. This epic implements that report's **Phase 0
(instrument)** and **Phase 1 (fix the lane)**. Phases 2 (scheduler event-wake) and 3
(resolver service proc) are explicitly out of scope.

Today's defects (verified in the report and re-verified while planning, on athena,
2026-10-07):

1. **Blind trigger.** `wait_checks` and `bead_claim_checks` watch
   `{path: projects, glob: "*/artifacts/ace-run/*"}` (`src/sase/default_config.yml`,
   waits routine). The fs token
   (`src/sase/axe/chop_policy_snapshots.py:_fs_watch_token`) is a shallow
   `name:mtime:size:child_count` per match, so it only sees month-shard dirs and ~2.1k
   leaked `..gate-shell-*` / `..monitor-start-*` / `..gate-turn-*` lock files in
   `ace-run/`. `done.json` / `waiting.json` writes never change it; the job mostly runs
   on its 120 s `max_quiet` backstop or incidentally.
2. **Dead waiters.** ~900 `waiting.json` markers exist; only ~12–19 belong to live
   runners. `wait_checks` resolves all of them every run (plus terminal-blocker
   notification upserts for dead ones). A recent run:
   `artifacts=16875 waiting=900 already_ready=92 invalid=134 unresolved=674`, 24 s.
3. **Tick convoy.** The `waits` routine runs `wait_checks`, `bead_claim_checks` and
   `sidecar_auto_sync` in one `ThreadPoolExecutor` tick that waits for all of them
   (`src/sase/axe/lumberjack.py`), so the "10 s" lane averages ~24 s.
4. **`ready.json` race.** `src/sase/scripts/_chop_wait_checks_run.py` writes
   `ready.json` with `open(path, "w")` + `json.dump`, and `read_ready_result`
   (`src/sase/axe/run_agent_wait_deps.py`) returns **True** on `JSONDecodeError` /
   `OSError`, so a runner that observes a half-written file crosses the barrier.

Result: ~82% of dependency releases come from the runner's 60 s direct-resolution
fallback (`_WAIT_DEPENDENCY_FALLBACK_INTERVAL` in `src/sase/axe/run_agent_wait.py`),
with release latency p50 ~21–33 s.

### New measurements taken while planning (they change part of the report's Phase 1)

All read-only, on athena, same load conditions:

| Operation                                                                                                                     | Time                 |
| ----------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| Python walk of every project's `ace-run` dirs (`iter_agent_artifact_dirs`) checking `waiting.json` (16,875 dirs, 901 markers) | **0.25 s**           |
| Liveness check of all ~894 waiting records (`is_process_alive` with process identity)                                         | **0.008 s**          |
| Rust `query_agent_artifact_index` full-history query (15.7k ace-run rows), any option set                                     | **~11–12 s**         |
| ...plus `agent_scan_wire_from_dict` decode / `asdict` rows / `WaitDependencyIndex.add_many`                                   | ~2.5 s / ~2 s / ~3 s |
| Active-tier-only index query                                                                                                  | 2.3 s                |
| Rust `scan_agent_artifacts` with `include_waiting` (used by `sidecar_auto_sync`)                                              | **~12 s**            |
| Filesystem dependency rows (`_filesystem_dependency_rows`, reads all `agent_meta.json`) + `add_many`                          | **~2–2.5 s + ~3 s**  |
| Confirmation rebuild for one project with warm meta cache                                                                     | ~3 s                 |
| Lumberjack per-tick patch load + serialization (`find_all_patches`, 309 patches)                                              | ~30 ms               |

Consequences:

- The report's step "enumerate waiters from the artifact index instead of walking 16.9k
  directories" is **reversed**: the walk is 0.25 s, every index route costs ≥2.3 s. Keep
  the walk.
- `wait_checks` currently builds its resolving view from the full-history index query
  (introduced by sase-wn.4, benchmarked on a 400-artifact fixture). At today's 16.8k
  artifacts that route costs ~19 s versus ~5.5 s for the filesystem rows it already uses
  for confirmation. Switch the resolving view to filesystem rows.
- Even after this epic, resolution remains O(history) whenever at least one live waiter
  is pending: expect a release run of roughly walk 0.3 s + view ~5.5 s + confirmation ~3
  s. Expected named-agent release latency is therefore **~7–12 s** (pulse → ≤2 s tick →
  ~6–9 s run → ≤0.5 s runner poll), not the report's 2–6 s estimate (which assumed a
  ~1–3 s index-backed resolve). That is still a 2–3× improvement over p50 21–33 s and
  removes the fallback as the primary release path. Closing the rest is a follow-up (see
  the follow-up list in `lane-split`).

### Deliberate adjustments to the report's Phase 1 (call these out in bead notes)

- **Keep the filesystem walk for waiter enumeration** (measured above).
- **Do not lengthen the runner fallback yet.** The report gates it on Phase 0 data
  ("once ≥90% of releases come via `ready.json`"); that data cannot exist until this
  epic is deployed and has run for a while. It becomes a recorded follow-up.
- **Defer the hourly "archive dead `waiting.json`" sweep.** After the liveness filter,
  dead markers cost `wait_checks` ~0.1 s, so the sweep is hygiene, not latency. But
  `waiting.json` is treated as a _live_ marker by continuation retention
  (`src/sase/core/continuation_retention.py`), project lifecycle
  (`src/sase/main/project_handler_lifecycle.py`) and the TUI (`WAITING` status in
  `src/sase/ace/tui/models/_loaders/_meta_enrichment_filesystem.py`); removing or
  renaming it changes those behaviours and needs its own decision. Record it as a
  follow-up.
- **Do not change the sidecar sync cadence** (stays 30 s); only isolate it and add the
  post-sync pulse.

### Boundaries and invariants

- **No sase-core changes.** The fs trigger token is computed in Python
  (`chop_policy_snapshots.fs_snapshot`); the Rust config validator only requires `glob`
  to be a string, and the Rust decision compares tokens. `agent_meta.json` is written
  only by Python merge writers (sase-core writes it only in tests), so new keys are
  additive. Wait-dependency resolution already lives in Python
  (`src/sase/core/wait_dependency_resolution/`); small helpers added beside it keep that
  placement. Porting resolution to Rust is out of scope.
- **Fail closed on release, fail open on liveness.** Never publish `ready.json` without
  the existing resolution + fresh-membership confirmation. A waiter whose runner
  liveness cannot be proven dead is resolved as if live.
- **Polling backstops stay.** `max_quiet: "120s"` stays on both triggers; the runner's
  60 s fallback stays unchanged.
- **Keep incidental duties.** Terminal-blocker logging/notifications and the
  `unknown_outcome` counter keep working for live waiters.
- Do not restart the scheduler or run `sase update`; new routines activate when the user
  next updates. Do not run the real `wait_checks` job against the live host to measure
  it (it would publish real `ready.json` markers); time read-only snippets instead.
- Every phase finishes with `sase tool run check` (see the lint-and-test memory) and
  updates `docs/axe.md` for its behaviour change.

## atomic-ready

Goal: `ready.json` can never be observed half-written, never clobbers an existing
marker, and a torn or malformed file never releases a runner.

1. Add a publish helper in `src/sase/axe/run_agent_wait_markers.py` (the module that
   owns wait markers), e.g. `publish_ready_marker(artifacts_dir, payload) -> bool`:
   - Write the JSON to a `tempfile.mkstemp` file in the same directory (hidden prefix
     such as `.ready.json.`, `.tmp` suffix), flush + `fsync`.
   - Immediately before publishing, return `False` if `waiting.json` no longer exists
     (the runner already released and cleaned up; this closes the window where a late
     `wait_checks` leaves a stray `ready.json`).
   - Publish with `os.link(temp, ready_path)` so the first writer wins; on
     `FileExistsError` return `False`. If `os.link` fails for another reason (filesystem
     without hard links), fall back to "`ready.json` absent → `os.replace`".
   - Always unlink the temp file (`finally`). Return `True` only when this call
     published.
2. In `_process_one_waiter` (`src/sase/scripts/_chop_wait_checks_run.py`) replace the
   `open(..., "w")` + `json.dump` block with the helper. `True` → `ready_written += 1`;
   `False` → count as `already_ready` (lost race or waiter gone); `OSError` → keep the
   existing `skipped_invalid` accounting.
3. In `read_ready_result` (`src/sase/axe/run_agent_wait_deps.py`): `JSONDecodeError` /
   `OSError` → return `False` (not ready; the runner retries on its next poll); a
   non-dict payload → `False`. Keep the legacy `cancelled` handling (unlink + `False`).
   A persistently corrupt marker degrades to the runner fallback, which never reads
   `ready.json`.
4. Leave the TUI run-now writer (`_write_ready_marker` in
   `src/sase/ace/tui/actions/agents/_directive_persistence.py`) as is: it is already
   temp-file + `os.replace` behind an existence check under the directive lock.
5. Tests (extend `tests/test_run_agent_wait_deps*.py`,
   `tests/test_axe_chop_wait_checks*.py` or a new focused module): torn / empty /
   non-dict `ready.json` keeps the runner parked until a valid marker appears; legacy
   cancelled marker unchanged; helper does not clobber an existing marker and leaves no
   temp files; helper skips when `waiting.json` is gone; `wait_checks` with a
   `ready.json` created between scan and publish counts it as already-ready and leaves
   its content untouched.
6. Docs: in the `wait_checks` part of `docs/axe.md`, state that `ready.json` is
   published atomically, first writer wins, and the runner treats an unreadable or
   malformed marker as not ready.

## pulse-trigger

Goal: a dependency completing (or a new dependency waiter parking) changes the
`wait_checks` trigger token within one tick.

1. `src/sase/default_config.yml`, `waits` routine: for both `wait_checks` and
   `bead_claim_checks`, replace the `*/artifacts/ace-run/*` path entry with
   `{path: projects, glob: "*/artifacts/.ace_refresh_pulse"}`; keep `max_quiet: "120s"`.
   (The glob is project-level only, so per-agent pulses written by
   `touch_agent_refresh_pulse` inside run dirs do not match.) Leave the job placement in
   `waits` for now; `lane-split` moves `wait_checks`.
2. `write_waiting_marker` (`src/sase/axe/run_agent_wait_markers.py`): after the index
   refresh, when the payload carries any dependency field (non-empty `waiting_for`,
   `wait_for_artifacts`, `wait_for_fork_sources`, `wait_for_beads`, or
   `wait_for_hoods`), touch the project pulse best-effort, the same way
   `write_done_marker_and_update_index` does
   (`touch_turn_refresh_pulse(project_name_from_artifacts_dir(...))` from
   `sase.turns.settlement`, lazy import, swallow exceptions). Runner-slot queue markers
   (written by `run_agent_wait_slots.py` / `run_agent_wait_slot_candidate.py`, no
   dependency fields) must **not** touch the pulse, so slot-queue republishes do not
   cause wake churn.
3. The TUI wait editor's `_write_waiting_marker`
   (`src/sase/ace/tui/actions/agents/_directive_persistence.py`) must also touch the
   pulse when the rewritten marker carries dependency fields, so changed wait targets
   wake the resolver.
4. Audit every writer of a terminal `done.json` that can satisfy a wait (agent runs via
   `write_done_marker_and_update_index`, gate/monitor turn settlement, plan propose,
   handoff, repeat-stop, finalizers) and confirm each touches the project pulse; add the
   touch where one is missing. Record the audit result in a bead note.
5. Rewrite the artifact-glob part of `tests/test_axe_default_chop_triggers.py` on the
   real `ace-run/YYYYMM/DD/<run>` layout (update the module docstring and the
   `_ARTIFACT_GLOB_CHOPS` naming). For both chops cover: idle tick skips; a done marker
   written through `write_done_marker_and_update_index` into a real-layout run dir
   fires; a dependency `write_waiting_marker` fires; a slot-queue-style marker (no
   dependency fields) does not fire; creating `..gate-shell-*.lock` /
   `..monitor-start-*` files or a new day dir under `ace-run/` does not fire; the
   `max_quiet` sweep still fires. Keep the hooks-lane tests intact.
6. Docs: replace the `docs/axe.md` sentence promising a re-scan "when a project gains a
   new agent artifact" with the pulse behaviour (what touches the pulse, `max_quiet`
   backstop, dead-owner claim release still relies on `max_quiet`).

## release-telemetry

Goal (report Phase 0): each crossed wait barrier records how it was released and how
long each segment took, in `agent_meta.json`, so the Phase 1 exit criteria (fallback
share < 10%; named-agent release p50 ≤ 5 s / p95 ≤ 15 s) can be measured without
grepping runner logs.

New `agent_meta.json` keys (all optional, additive):

| Key                              | Meaning                                                                                                                                                                                     |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `wait_release_source`            | `startup` (resolved before parking), `ready_json` (wait_checks marker), `manual` (marker with `unwait: true`, i.e. TUI run-now), `runner_fallback`, or `timer` (duration/until-only waits)  |
| `wait_dependencies_satisfied_at` | epoch seconds when the last relevant dependency member finished, when known                                                                                                                 |
| `wait_release_latency_s`         | dependency-release instant minus `wait_dependencies_satisfied_at`, clamped ≥ 0; only for `ready_json` / `runner_fallback` releases with a known satisfied time and **no bead dependencies** |
| `admission_latency_s`            | `run_started_at − wait_completed_at`, for runs that crossed a wait barrier (covers post-wait re-exec, bootstrap, deferred macro expansion and slot admission)                               |
| `runner_slot_wait_s`             | `run_started_at` minus the instant the runner entered `wait_for_runner_slot`                                                                                                                |

Implementation:

1. `src/sase/core/wait_dependency_resolution/`: add a small exported helper, e.g.
   `latest_member_finished_at(member_dirs) -> float | None`, returning the max numeric
   `finished_at` (tolerate ISO strings) across the members' `done.json`, ignoring
   members without one; `None` when nothing is known. Unit-test it.
2. `wait_checks`: `_process_one_waiter` already computes `member_dirs` via
   `dependency_index.dependency_member_dirs(...)`. Publish
   `{"resolved_deps": waiting_for, "released_by": "wait_checks", "dependencies_satisfied_at": <float>}`
   (omit `dependencies_satisfied_at` when unknown or when the waiter has bead
   dependencies).
3. Runner resolution helpers (`src/sase/axe/run_agent_wait_deps.py`): make
   `initial_dependencies_resolved` and `waiting_marker_dependencies_resolved` return a
   small frozen result (e.g. `DependencyResolution(resolved, satisfied_at)`) whose
   `__bool__` is `resolved`, so existing `if ...` call sites and the many
   truthiness-based tests and `patch(..., return_value=True)` stubs keep working.
   Compute `satisfied_at` from the index those functions already built, only when
   resolved and confirmed, and `None` when bead dependencies are present. Do the same
   for `read_ready_result` (truthy result carrying `released_by`, `unwait`,
   `dependencies_satisfied_at` from the payload).
4. `wait_for_dependencies` (`src/sase/axe/run_agent_wait.py`): pick the source at each
   release branch (startup fast path → `startup`; ready marker → `manual` if `unwait`
   else `ready_json`; fallback → `runner_fallback`; time-only and duration-only paths →
   `timer`), capture the dependency-release instant (`time.time()` at loop exit, before
   any post-dependency duration floor), and pass them to `record_wait_completed_at`.
5. `record_wait_completed_at` (`src/sase/axe/run_agent_wait_markers.py`): accept the
   optional release info and stamp the new keys **only** in the branch that first stamps
   `wait_completed_at`; the existing disk-stamp short-circuit (refreshed runner) writes
   nothing new, keeping it idempotent.
6. `record_run_started_at` (`src/sase/axe/run_agent_markers.py`): add an optional
   `slot_wait_started_at` kwarg; when first stamping `run_started_at`, also stamp
   `admission_latency_s` (if the merged meta has `wait_completed_at`) and
   `runner_slot_wait_s` (if given). `_admit_and_launch`
   (`src/sase/axe/run_agent_runner.py`) captures `time.time()` immediately before
   `wait_for_runner_slot` and passes it through the `claim` lambda. Other callers (e.g.
   `tests/fakey/_runner_slot_harness.py`) stay valid unchanged.
7. Verify the artifact index / scanner and TUI loaders tolerate the extra keys (they are
   additive; no wire change expected).
8. Tests: every source is stamped correctly (startup, ready_json, manual,
   runner_fallback, timer); latency derives from member `done.json` `finished_at`;
   absent for bead waits and for startup/manual; idempotent across a refreshed runner;
   admission fields computed; `wait_checks` payload contents; helper unit tests.
9. Docs: add a short "Wait release telemetry" subsection to `docs/axe.md` listing the
   keys and a one-line `jq` (or Python) recipe that tallies `wait_release_source` over
   recent `ace-run` `agent_meta.json` files.

## live-waiters

Goal: a `wait_checks` run with no live pending waiter costs well under a second, and a
run with live waiters never spends time on dead ones or on the slow index route.

1. Add a shared module (e.g. `src/sase/axe/wait_marker_scan.py`) with:
   - A walk that reuses today's `iter_agent_artifact_dirs(project, "ace-run", ...)` loop
     and returns the counts (`projects`, `artifacts`, `waiting`, `already_ready`), the
     pending markers (`waiting.json` present, `ready.json` absent) and the walked dirs.
   - `waiting_runner_liveness(artifact_dir, meta) -> "alive" | "dead" | "unknown"`:
     `unknown` when `agent_meta.json` is missing/unreadable or no integer pid is
     recorded (and `running.json` has none); `dead` when `stopped_at` is set or
     `is_process_alive(meta, artifact_dir)` (`sase.agent.names`, which guards PID reuse
     with `process_identity` and boot time) is `False` for a recorded pid; otherwise
     `alive`; any exception → `unknown`. `unknown` is treated as live.
2. Rework `_run` in `src/sase/scripts/_chop_wait_checks_run.py`:
   - Walk; read each pending marker's `agent_meta.json` into the run's `meta_cache` and
     classify liveness.
   - If no pending marker is `alive`/`unknown`, emit the summary and return without
     building any dependency view (`no_op`).
   - Otherwise build the resolving view from
     `_filesystem_dependency_rows(projects_dir, meta_cache=meta_cache)` (keep the
     existing negative-seeding for walked dirs and the confirmation pass unchanged), and
     resolve only the `alive`/`unknown` waiters. Dead waiters get no resolution, no
     terminal-blocker notifications, and no logs beyond the counter.
   - Remove `wait_checks`' use of `query_ace_run_index_records` /
     `wait_rows_from_index_records` and its `full_walk` toggle; keep
     `chop_scan_full_walk` and the index pre-pass for `bead_claim_checks`.
   - Summary counters: keep the existing keys and add `live_waiting`, `dead_waiting`,
     `unknown_liveness` (the dead-waiter backlog counter from report Phase 0). Document
     that `unresolved` now counts only live/unknown waiters.
3. Update `tests/test_axe_chop_incremental_scans.py` and the `wait_checks` suites: no
   pending → zero `agent_meta.json` reads; only-dead pending → only those waiters' meta
   reads and no dependency-view build; dead waiter with a terminal blocker produces no
   notification; live and unknown-pid waiters still resolve exactly as before (existing
   fixtures write no `agent_meta.json`, so they classify as `unknown` and keep passing);
   PID-reuse (live pid, mismatched `process_identity`) counts as dead; results on a
   populated tree match the previous behaviour for live waiters.
4. Verify the report's open question and record the answer in a bead note: no restart or
   revive path relaunches a runner in an existing artifact dir relying on a `ready.json`
   published while it was dead (the runner records `pid` + `process_identity` at
   bootstrap in `src/sase/axe/run_agent_runner_bootstrap.py` before
   `write_waiting_marker`, and re-resolves via `initial_dependencies_resolved` before
   parking). If such a path exists, stop and treat those waiters as `unknown`.
5. `sidecar_auto_sync._projects_with_live_bead_waits`
   (`src/sase/scripts/sase_chop_sidecar_auto_sync.py`) currently calls the Rust
   `scan_agent_artifacts` (~12 s per run). Reimplement it with the shared walk +
   liveness helper with identical semantics (marker has non-empty `wait_for_beads`, no
   `ready.json`, runner not provably dead). Keep the function name and signature:
   several tests monkeypatch it.
6. Record read-only before/after timings in a bead note (time the walk, classification
   and view build in a Python snippet against `~/.sase/projects`; do not execute the job
   against the live host).
7. Docs: update the `wait_checks` description in `docs/axe.md` (live-waiter filter, new
   counters, filesystem resolving view) and the `wait_checks` job description in
   `default_config.yml` if it mentions scanning every marker.

## lane-split

Goal: `wait_checks` no longer convoys behind slow jobs, and bead closes wake it
immediately after the beads clone fast-forwards.

1. `src/sase/default_config.yml`:
   - New routine `agent_waits`: `interval: 2`, `job_timeout: "2m"`, only `wait_checks`
     (with the pulse trigger from `pulse-trigger`).
   - New routine `sidecar_sync`: `interval: 30`, only `sidecar_auto_sync` (keep
     `timeout: "2m"`; drop its `run_every`, which the routine interval now provides;
     keep the `_CHOP_TIMEOUT_SECONDS` comment in sync).
   - `waits` keeps `bead_claim_checks` and `epic_launch_flush`. Rewrite all three
     routine descriptions and the moved jobs' descriptions where they mention lanes or
     cadence.
2. Post-sync pulse: in `sase_chop_sidecar_auto_sync._run`, after a `refreshed` result
   for the beads role (`BEADS_SIDECAR_ROLE`), touch that project's pulse best-effort
   (`touch_turn_refresh_pulse(target.project_key)`), so `wait_checks` re-evaluates bead
   waits on its next 2 s tick. Keep the existing per-tick bead hint for live bead
   waiters.
3. Runner: in the dependency loop of `wait_for_dependencies`, poll `ready.json`
   existence every 0.5 s instead of 2 s (a single `stat`). Leave duration/until sleeps
   and the 60 s fallback cadence unchanged.
4. Verify an idle `agent_waits` tick stays cheap: per-tick fixed work (~30 ms patch
   serialization measured) plus ~40 pulse `stat`s, and skipped-preflight run records
   stay bounded by the existing run-history retention (the 5 s hooks lane already relies
   on this). Adjust nothing unless that is false.
5. Tests: move `wait_checks` to `("agent_waits", "wait_checks")` in
   `tests/test_axe_default_chop_triggers.py`; rewrite
   `test_epic_launch_flush_and_sidecar_auto_sync_are_unaffected` for the new placement;
   add shipped-config assertions (`agent_waits` = exactly `wait_checks` at interval 2;
   `sidecar_sync` = exactly `sidecar_auto_sync` at interval 30; `waits` =
   `bead_claim_checks` + `epic_launch_flush`); an idle `agent_waits` tick spawns nothing
   once warmed (mirror the hooks-lane idle test); the post-sync pulse fires only for a
   refreshed beads role; the runner poll constant. Fix any other test that pins
   `sidecar_auto_sync` or `wait_checks` to `waits`.
6. Docs: `docs/axe.md` "The scheduler ships with seven default routines" → nine; add
   `agent_waits` and `sidecar_sync` sections; update the `waits` section and table; note
   that the routines activate on the scheduler restart `sase update` performs, and that
   the moved jobs start with fresh per-routine state (trigger checkpoint fires once;
   `sidecar_sync` re-checks every role once because its backoff schedule file lives in
   the routine state dir). Grep other docs for routine lists that need the same change.
7. Record one consolidated `PROPOSED FOLLOW-UP:` bead note listing:
   - Lengthen `_WAIT_DEPENDENCY_FALLBACK_INTERVAL` from 60 s to 180–300 s once the new
     telemetry shows ≥90% of dependency releases are non-fallback over a representative
     window (saves ~5 s of index building per live waiter per minute).
   - sase-core: the full-history `query_agent_artifact_index` takes ~11–12 s for 15.7k
     rows regardless of options, and `scan_agent_artifacts` ~12 s; this also likely
     explains `bead_claim_checks`' ~33 s runs (it uses the same full-history query).
   - Leaked `..gate-shell-*` / `..monitor-start-*` / `..gate-turn-*` lock files in
     `ace-run/` (~2.1k in the sase project).
   - Archiving `waiting.json` for runners dead longer than N hours, after deciding its
     effect on TUI status, continuation retention and project-lifecycle live markers.
   - If measured release latency still misses p50 ≤ 5 s / p95 ≤ 15 s: scoped or
     incremental wait resolution (resolution is still O(history), ~6–9 s with a live
     waiter), then the report's Phase 2 event-wake.
   - The report's admission-latency parallel track (~11 s capacity-only slot scan under
     the host-wide lock); the new `admission_latency_s` / `runner_slot_wait_s` keys
     quantify it.

## Verification for the epic as a whole

- Each phase: `sase tool run check` green, plus the targeted suites named above.
- After landing (user action, outside this epic): `sase update` restarts the scheduler;
  then tally `wait_release_source` and `wait_release_latency_s` over a day of runs to
  evaluate the exit criteria (fallback share < 10%; named-agent release p50 ≤ 5 s, p95 ≤
  15 s; expect ~7–12 s until the O(history) follow-up lands).
