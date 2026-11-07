---
tier: tale
title: Make the artifact_link_backfill chop finish reliably inside its budget
goal:
  The hourly artifact_link_backfill job stops hitting its 300 s SIGKILL, with drain cost
  linear in queue size and every long step deadline-guarded.
size: medium
decisions:
  outbox_retention:
    ask:
      Should a terminal agent's never-publishable outbox rows leave the live queue 14
      days after it finishes, instead of 90?
    choices:
      keep:
        No visible change; sase's queue still levels off near 30,600 rows that reconcile
        must overlay
      days14:
        Hot queue shrinks ~6x, reconcile gets faster; read-only agents' reads leave
        local view at 14 days
    default: keep
    why:
      The required fixes bound the job on their own, without changing what users see
      locally
    answer: keep
proposed_by: bbugyi200.athena.19m
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-07 10:21:12
status: wip
---

# Make `artifact_link_backfill` finish reliably inside its budget

## Why this keeps happening

Source: `research:202611/artifact_link_backfill_timeouts.md` (read it with
`sase artifact read`). The hourly `housekeeping/artifact_link_backfill` chop has been
SIGKILLed at its 300 s axe timeout 21 times since 2026-10-29. Every one of those runs
died inside the **sase** outbox drain. Successful runs already take 240–262 s against a
240 s soft budget.

Confirmed causes:

1. **The eligibility check is O(rows × evidence).** `_entry_is_eligible`
   (`src/sase/sdd/_artifact_link_outbox_drain.py:383`) calls
   `artifact_link_run_has_release_evidence`
   (`src/sase/sdd/artifact_link_release_evidence.py:82`) once per queued row. Each call
   takes the lock and re-reads and re-parses the whole evidence JSONL. For sase that is
   12,889 rows × 4,172 lines, about 108 s, with 0 matches.
2. **The deadline is checked only once** (`_artifact_link_outbox_drain.py:124`), before
   the slow loops, so a slow tick runs straight into the hard kill.
3. **The agent catalog snapshot** (`build_agent_catalog_snapshot`, 4–44 s) is rebuilt
   once per project per run (`_terminal_agent_finished_times`, `:437`), only to find
   rows old enough to drop. No row can be old enough before 2026-12-08.
4. **The queue only grows.** 99.9 % of rows are `read` observations from read-only runs
   that never earn evidence. They are dropped only 90 days after their agent finishes,
   so sase plateaus near 30,600 rows.

Additional measurements taken on athena while planning (read-only, shared locks):

| Measurement (sase, 12,898 queued rows)                                          | Value                                                                                                                                          |
| ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| One full outbox parse (`_read_outbox_records_unlocked`)                         | 4.7 s; the drain does **three** (inspect, read, rewrite), and the rewrite also computes a Rust digest per row while holding the exclusive lock |
| Building a validated `(run_id, agent_id)` set from all 4,173 evidence lines     | **47 ms**; 0 queued rows match                                                                                                                 |
| `artifact_link_event_snapshot(include_pending=True)` vs `include_pending=False` | 24.6 s vs 9.3 s; `_pending_candidates` reads the outbox **twice**                                                                              |
| `store.preview_aggregate()` (the reconcile's core step)                         | 59 s, reading the outbox 4 times                                                                                                               |
| sase `reconcile_repair` when it runs / bob-cli                                  | 37–59 s / ~26 s                                                                                                                                |

Once the drain is fast, the next long uninterruptible step is sase's reconcile preview
(about 59 s). The chop only refuses to start reconcile after the deadline has _already
passed_ (`sase_chop_artifact_link_backfill.py:465`), so a slow earlier project could
still push a reconcile that starts at about 230 s into the 300 s kill. This plan closes
that gap too.

## Goal and acceptance criteria

- A drain parses the outbox **once**, loads release evidence **once**, and costs
  O(rows + evidence lines). No queued row triggers its own file read.
- Unchanged semantics: the same rows publish, the same rows stay queued, and the same
  rows are dropped. The only difference: a drop may now happen up to 24 h later than
  before, because of the daily stale-drop cadence.
- The 240 s soft budget becomes real. The drain checks its deadline inside its loops,
  and neither the catalog build nor reconcile starts when it is expected to overrun. A
  slow tick produces logged deferrals, never a SIGKILL.
- The agent catalog is built at most once per chop run, only when some row is old enough
  to drop, and at most once per 24 h per project.
- A project whose work was deferred goes first on the next tick, so sase's
  reconcile/repair stops being skipped.
- Drain phases are timed in the chop log, so the next regression can be diagnosed from
  logs alone.
- Expected steady state: sase drain ~150 s → ~5 s, bob-cli drain 30–150 s → ~3 s,
  small-project drains ~4–66 s → <1 s. A whole run drops from ~250 s to ~110–130 s, with
  sase reconcile running every tick. Every agent commit finalization for sase also gets
  ~10 s faster, because `_drain_artifact_link_outbox_for_agent` uses the same drain.

Rust boundary: these changes are I/O and orchestration fixes inside existing Python
modules. The eligibility rule itself is still the sase-core validator
(`validate_artifact_link_release_evidence` in
`crates/sase_core/src/artifact_link_eligibility/policy.rs`, a pure per-record check on
schema, run/agent equality, and a non-empty `qualifying_repo_ids`). No sase-core change
and no `sase-core-revision.txt` bump.

## Implementation

### 1. Release-evidence index — `src/sase/sdd/artifact_link_release_evidence.py`

- Add a public
  `load_artifact_link_release_evidence_index(project_key: str) -> frozenset[tuple[str, str]]`:
  - Read the file once, under the existing shared `flock`.
  - For each line, skip blanks, invalid JSON, and non-dicts; build
    `ArtifactLinkReleaseEvidence`; validate it with
    `expected_run_id`/`expected_agent_id` set to its own ids.
  - Include `(run_id, agent_id)` only when both are non-blank and validation passes.
  - Return an empty set when the file is missing or raises `OSError`.
  - Self-validation is equivalent to today's per-query validation, because the validator
    is pure per record.
- Reimplement `artifact_link_run_has_release_evidence` on the same private line parser,
  as blank-guard plus `(run_id, agent_id) in index`, so there is one parsing path. Add
  the new function to `__all__`.

### 2. Drain — `src/sase/sdd/_artifact_link_outbox_drain.py` plus a new sibling module

The drain module is 524 lines, so put the new helpers in a new
`src/sase/sdd/_artifact_link_outbox_drain_support.py`:

- **`AgentTerminalTimes`.** A lazily built, reusable provider with
  `finished_times(names) -> dict[str, float]`.
  - It builds `build_agent_catalog_snapshot()` once on first use, injectable for tests,
    with the same terminal `state`/`status` filtering as today's
    `_terminal_agent_finished_times`.
  - It exposes `built: bool`, `ok: bool`, and `build_seconds: float`.
  - A build failure is cached and yields `{}`. That fails closed: nothing is dropped.
- **A small deadline helper.** A private `_DrainDeadline` exception plus a
  `check(index)` helper that tests `time.monotonic() >= deadline` every 256 iterations.

New `drain_artifact_link_outbox` flow:

1. Run `_requeue_ready_parked_entries` (fixed in step 4).
2. Read the outbox **once** with the new `read_artifact_link_outbox_queue` (step 3),
   which yields the entries, `legacy_queued`, `invalid_queued`, and `has_duplicate_ids`.
   Drop the separate `_inspect_artifact_link_outbox` call. Keep `parked_total` from
   `_inspect_parked`. Keep the early return for an empty queue.
3. `_partition_selected` and the existing up-front deadline check are unchanged.
4. Load the evidence index **once**, only if some selected entry is not a
   `_entry_is_trusted_machine_event`. If loading raises, use an empty index: rows stay
   queued, as today.
5. Compute eligibility **once per selected entry**, reusing it for both the stale and
   the publish partitions. Change the signature to
   `_entry_is_eligible(entry, evidence_index)` and keep it as the single,
   monkeypatchable eligibility seam. Check the deadline inside the loop.
6. Stale terminal drop, only when `drop_stale_terminal`:
   - Compute the cutoff from `observed_at` (the `now` parameter), not a fresh
     `time.time()`. The retention-days source is unchanged unless the decision is
     `days14`.
   - Candidates are the selected entries that are not eligible and have
     `created_at <= cutoff`. An agent finishes after it records its rows, so a newer row
     can never be stale. When `created_at` was reset later (parked requeue,
     host-appended rows), skipping only delays the drop; it can never drop a row
     wrongly.
   - No candidates: `stale_checked = True`, with no catalog.
   - The catalog is not built yet and
     `deadline - now < _STALE_CATALOG_MIN_HEADROOM_SECONDS` (60): skip stale evaluation
     for this drain, with `stale_checked = False` and a skip diagnostic
     `deferred stale terminal drop: insufficient budget`.
   - Otherwise, `stale` is the candidates whose agent's terminal
     `finished_at <= cutoff`, and `stale_checked = terminal_times.ok`.
7. `publishable` is the eligible entries. The rest of the flow is unchanged: the
   machine-writable partition, the event-drainable partition, and the deadline-aware
   `publish_artifact_link_events`.
8. Call `_rewrite_without_ids` **only** when there are settled event keys, stale rows,
   or `has_duplicate_ids`. A no-op drain no longer rewrites and fsyncs the whole file
   under the exclusive lock, which today blocks every agent's append for seconds.
9. If `_DrainDeadline` is raised before publishing, return the same deferred report
   shape as the existing early return (`:124-135`). Nothing has been mutated at that
   point.
10. The report (`_ArtifactLinkOutboxDrainReport`) gains `stale_checked: bool = False`
    and `phase_seconds: tuple[tuple[str, float], ...] = ()`, covering `read`,
    `evidence`, `eligibility`, `catalog`, `publish`, and `rewrite`.

New keyword: `terminal_times: AgentTerminalTimes | None = None`. `None` means a private
provider per call, which is today's behavior. Other callers (commit workflow
`workflow_publication.py:531`, `artifact_link_derivation.py:207`,
`agents_sync/publication_job.py:360`) need no change and still benefit.

### 3. Outbox I/O — `src/sase/sdd/_artifact_link_outbox_io.py`

- Add a public `read_artifact_link_outbox_queue(project_key)`. It returns a frozen
  dataclass with `entries`, `legacy_queued`, `invalid_queued`, and `has_duplicate_ids`,
  built from one `_read_outbox_records` pass. Duplicates are detected by entry id, with
  no digests.
- Add a public
  `pending_artifact_link_outbox_snapshot(project_key, *, exclude_operation_ids) -> tuple[tuple[dict, ...], tuple[float, ...]]`
  that returns the events and their `created_at` values from a **single** read.
- Re-export both from `src/sase/sdd/artifact_link_outbox.py`.
- `inspect_artifact_link_outbox` and `rewrite_artifact_link_outbox_without_ids` stay as
  they are; doctor and link-health still use `inspect`.

### 4. Parked recheck persistence (bug) — drain plus `_artifact_link_outbox_parked.py`

Today `_requeue_ready_parked_entries` (`_artifact_link_outbox_drain.py:465-500`)
rewrites the ledger only when something is requeued, so every touched `last_checked_at`
is lost and the 24 h throttle never works. The fix is a read-then-blind-rewrite, which
could also lose an entry parked concurrently by another drain. So:

- Add
  `update_artifact_link_parked_entries(project_key, *, touched: Mapping[str, float], removed: Collection[str]) -> None`.
  Under the parked file's exclusive lock it re-reads the ledger, sets `last_checked_at`
  for the touched ids, drops the removed ids, and keeps every other entry, including any
  appended after the caller's read.
- `_requeue_ready_parked_entries` collects the touched and requeued ids. It calls the
  helper only when either set is non-empty, after the outbox appends; keep today's
  append-then-remove order. Delete `rewrite_artifact_link_parked_entries` if nothing
  else uses it (symvision flags unused symbols).

### 5. Pending overlay single read — `src/sase/sdd/_artifact_link_event_store.py:303`

Make `_pending_candidates` use `pending_artifact_link_outbox_snapshot` instead of the
two separate reads. This removes one full outbox parse from every pending snapshot (~4.7
s for sase today; reconcile's `preview_aggregate` goes from 4 reads to 2), and the
events and stats now come from the same consistent read. Remove
`pending_artifact_link_outbox_events` and
`pending_artifact_link_outbox_event_created_at` if no other caller remains.

### 6. Chop — `src/sase/scripts/sase_chop_artifact_link_backfill.py`

- **Shared catalog.** Create one `AgentTerminalTimes` per `_run` and pass it through
  `_run_project` to `drain_artifact_link_outbox(..., terminal_times=...)`.
- **Schedule state.** Add a new `artifact_link_backfill_schedule.json` in
  `runtime.context.state_dir` with `schema_version: 1`,
  `stale_checked_at: {project: epoch}`, and `reconcile_seconds: {project: float}`.
  - Write it atomically, like `_write_state`, and once at the end of `_run` alongside
    the other state files.
  - Prune the keys of projects that are no longer enabled.
  - An unreadable or invalid file reads as empty.
- **Daily stale drop.** Set `drop_stale_terminal` to whether the project is due: no
  entry, or `time.time() - stale_checked_at >= _STALE_DROP_INTERVAL_SECONDS` (86,400).
  When `report.stale_checked`, record the current time.
- **Reconcile start headroom.** Before starting reconcile, set
  `estimate = reconcile_seconds.get(project, _DEFAULT_RECONCILE_ESTIMATE_SECONDS)` (60).
  - If this is not the first project started this tick and
    `chop_deadline - now < estimate`, skip reconcile. Log the warning
    `deferred reconcile/repair: needs ~{estimate:.0f}s, {remaining:.0f}s left`, and
    count the project as deferred.
  - The first project of a tick always runs reconcile, which guarantees progress.
  - After a reconcile that was not cut short (no `reconcile deadline expired` skip
    diagnostic and `deferred_refs == 0`), store
    `max(observed, 0.7 * previous + 0.3 * observed)`, or just `observed` when there is
    no previous value. The estimate rises at once on a slow run and decays slowly.
- **Deferred-first cursor.** `_run_project` returns whether its drain or reconcile work
  was deferred or skipped for budget.
  - The next cursor is the first project, in this tick's order, whose work was deferred.
  - If no project was deferred, keep the current `_next_cursor_after` behavior.
  - The existing sweep-backlog early return (`:421-427`) does not count as deferred.
- **Phase log.** After each drain, log one info line:
  `[artifact_link_backfill] <project>: drain phases: read=…s evidence=…s eligibility=…s catalog=…s publish=…s rewrite=…s`.
  The catalog value is `skipped` when no build was needed, or `shared` when an earlier
  project already built it.
- **Counter.** Add a `stale_checked_projects` summary counter.

> [!decision] outbox_retention = days14 Add
> `artifacts.link_outbox.unpublished_max_age_days: 14` to `src/sase/default_config.yml`,
> next to `artifacts.retention`. Comment it as: days after a terminal agent finishes
> before its never-publishable outbox rows move to the dropped audit; `0` disables
> dropping. Add `get_artifact_link_outbox_unpublished_max_age_days()` to
> `src/sase/config/_settings_artifacts.py`, using the same validation pattern as
> `get_artifact_retention_max_age_days`, and export it like its siblings. The drain's
> cutoff uses it instead of `get_artifact_retention_max_age_days()`. Update
> `docs/reference/artifact-links.md` (~line 343, "remain unpublished for 90 days") and
> add a getter test. The first due run after landing drops roughly 10k old sase rows in
> one rewrite plus an audit append, which is expected.

> [!decision] outbox_retention = keep No config or retention change; the drain keeps
> using `get_artifact_retention_max_age_days()`.

## Tests

- `tests/sdd/test_artifact_link_release_evidence.py`: the index holds only valid,
  non-blank pairs. Lines with invalid JSON, blank ids, or empty `qualifying_repo_ids`
  are skipped. `artifact_link_run_has_release_evidence` agrees with the index.
- A new drain-scaling test, for example
  `tests/sdd/test_artifact_link_outbox_drain_scaling.py`. Queue ~300 reads and ~300
  evidence lines; one drain loads the evidence index exactly once (counting wrapper) and
  never calls `artifact_link_run_has_release_evidence`. The publish and retain outcomes
  match the existing tests.
- Catalog:
  - No build when no ineligible row is older than the cutoff.
  - One build shared by two drains using one provider.
  - Stale rows are still dropped (existing drop tests keep passing).
  - A build failure drops nothing and sets `stale_checked=False`.
  - With less than 60 s left before the deadline, nothing is built or dropped.
- Deadline: a monotonic clock that crosses the deadline mid-eligibility gives
  `deferred=True`, the outbox bytes are unchanged, and nothing is published.
- No-op rewrite: a drain that settles nothing leaves the outbox inode and mtime
  unchanged. A queue with a duplicate id still gets rewritten and deduped.
- Parked:
  - `last_checked_at` persists.
  - A second drain within 24 h does not call `_parked_endpoints_are_local`.
  - An entry appended to the parked ledger during the recheck survives (inject it from a
    monkeypatched `_parked_endpoints_are_local`).
- Event store: `_pending_candidates` reads the outbox once.
- Chop (`tests/test_axe_chop_artifact_link_backfill_budgets.py` and `_selection.py`):
  - stale-drop due logic and schedule persistence and pruning;
  - the same provider object reaches every drain;
  - reconcile is skipped when the estimate exceeds what remains (and the project is not
    first), and runs when the project is first;
  - the estimate update rule;
  - the cursor moves to the first deferred project;
  - the phase log line is present.
- Update the existing monkeypatches of `_entry_is_eligible`
  (`tests/sdd/test_artifact_link_machine_authorize.py:314,363`) and
  `_terminal_agent_finished_times`
  (`tests/agents_sync/test_referenced_by_events.py:143`) to the new seams.

## Docs and changelog

Read `sase memory read docs_writing.md` first.

- `docs/reference/automation/scheduler-housekeeping.md` (~line 252) should say:
  - the stale terminal drop runs at most once per 24 h per project;
  - drain or reconcile work deferred for budget is logged, and that project goes first
    on the next tick;
  - the drain logs per-phase timings.
- Add a matching sentence about drop cadence at `docs/reference/artifact-links.md`
  (~line 218/343).
- Update the chop description in `src/sase/default_config.yml` (~line 1830) if it
  describes cadence.
- Follow the repo's changelog gate.

## Verification

- `sase tool run check` (not `check-full`).
- A read-only timing probe on the real queue with `.venv/bin/python`. Time
  `read_artifact_link_outbox_queue("gh_sase-org__sase")`,
  `load_artifact_link_release_evidence_index(...)`, and the eligibility pass. Do not
  publish or rewrite. Expect ~5 s, <0.1 s, and <0.1 s.
- The hourly job runs from the globally installed sase, and global installs are
  human-only. The final response must tell the user to reinstall, and to look for the
  new `drain phases:` log lines and the absence of `artifact_link_backfill` timeouts in
  the next error digests.

## Out of scope and follow-ups

- The research's cold-ledger idea (fix #4) is covered by the `outbox_retention` decision
  instead: a shorter drop window reuses the existing dropped-audit path, with no third
  ledger.
- `preview_aggregate` still builds both an event snapshot and the store-truth rows, so
  the outbox is read twice per preview even after step 5. Collapsing them is a separate
  reconcile performance change. File it with `/sase_new_task` after checking for
  duplicates.
