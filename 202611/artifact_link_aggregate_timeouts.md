---
tier: epic
title: Stop artifact-link aggregate rebuilds from SIGKILLing apollo's publication
  and backfill chops
goal: agent_publication and artifact_link_backfill finish inside their budgets because
  aggregate previews build one cheap snapshot, write retries respect the deadline,
  and slow drains leave progress breadcrumbs.
decisions:
  publish_deferral:
    ask: If a link publish runs out of budget before its aggregate rebuild lands,
      keep its outbox rows queued?
    choices:
      retain: Rows stay queued and republish idempotently next tick, like today's
        aggregate failures
      ack: Rows settle now; the local aggregate lags until the next rebuild or hourly
        reconcile
    default: retain
    why: It reuses the existing failure path unchanged, and deferrals should be rare
      once snapshots are cheap
    answer: retain
phases:
- id: snapshot-reuse
  title: Build the artifact-link event snapshot once and reuse its inputs
  size: medium
  depends_on: []
  description: Make preview_aggregate build one event snapshot instead of two (resolves
    task bead sase-1rt). Add a scoped, opt-in memo for durable event objects and the
    parsed pending outbox, so repeated snapshots inside one publish, drain, or reconcile
    stop re-reading and re-validating unchanged inputs.
- id: deadline-aware-aggregate
  title: Gate aggregate write retries and publish rebuilds on the caller deadline
  size: medium
  depends_on:
  - snapshot-reuse
  description: Make _write_merged_aggregate stop retrying after a lost CAS race when
    the next preview cannot finish before the caller's deadline. Thread the deadline
    through rebuild_aggregate, reconcile_aggregate, and the publish-path apply_events_to_aggregate,
    with typed deferral handling in both chops. Add per-sub-phase publish timings.
- id: publication-drain-progress
  title: Stream per-stage progress from agent_publication project drains
  size: small
  depends_on:
  - deadline-aware-aggregate
  description: Give drain_project a progress callback, so a slow or killed agent_publication
    tick leaves breadcrumbs for the stage it was in. Log the link drain's phase timings,
    including the new publish sub-phases, the same way artifact_link_backfill already
    logs its drain phases.
proposed_by: bbugyi200.athena.1ew
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-21 10:59:28
status: wip
bead_id: sase-1uu
---

- **PROMPT:** [prompts/202611/artifact_link_aggregate_timeouts.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/artifact_link_aggregate_timeouts.md)
- **BEAD:** [sase-1uu](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1uu/README.md)

# Stop artifact-link aggregate rebuilds from SIGKILLing apollo's publication and backfill chops

## Why this keeps happening

An apollo error digest (2026-11-21 10:37) reported two 300 s axe timeouts within a
minute: `housekeeping/artifact_link_backfill` was killed inside sase's
`reconcile/aggregate` step, and `publication/agent_publication` was killed with no
output at all. Neither is a one-off:

- `agent_publication` has hit its 300 s timeout about 20 times since 2026-11-16. A
  healthy tick takes 150–240 s against its 240 s soft budget
  (`PUBLICATION_JOB_WORK_BUDGET_SECONDS`,
  `src/sase/agents_sync/_publication_job_run.py`).
- `artifact_link_backfill` runs on apollo took 146–160 s overnight, then 222 s, 265 s,
  and finally 300 s (killed) as the morning's agent load grew. The 08:13 run already
  overran its own 240 s budget: sase's reconcile/repair took 177 s and logged
  `reconcile deadline expired`.

### Measured root cause

Every slow step is a **full artifact-link aggregate rebuild**, and the code builds that
rebuild's most expensive input several times per operation.

1. **One `preview_aggregate()` costs ~55–65 s for bob-cli and sase on apollo.** Profiled
   read-only on apollo (load average ~9). The table shows bob-cli's breakdown:

   | Step                                                                       | bob-cli     |
   | -------------------------------------------------------------------------- | ----------- |
   | `artifact_link_event_snapshot` (built **twice**)                           | 52 s        |
   | — durable event objects: read + Rust-validate every `link-events/**.json`  | ~7.7 s each |
   | — pending outbox parse (7,172 queued v2 events), one Rust reduce per entry | ~8 s each   |
   | — `reduce_artifact_link_event_inputs` (collision partition + reduce)       | ~10 s each  |
   | `projected_rows()`                                                         | 6.9 s       |

   The second snapshot is pure duplication: `preview_aggregate`
   (`src/sase/sdd/_artifact_link_store_aggregate.py`) builds one, then calls
   `_load_store_truth_rows` (`src/sase/sdd/_artifact_link_store_rows.py`), which builds
   an identical one. Task bead **sase-1rt** already reports exactly this.

2. **A single link publish builds the snapshot about four times.**
   `_publish_event_objects_locked` (`src/sase/sdd/_artifact_link_event_publish.py`)
   builds it once in admission (`_artifact_link_event_publish_persist.py`), once in the
   bead projection (`apply_events_to_beads`, `_artifact_link_event_project.py`), and
   twice in the aggregate rebuild (`apply_events_to_aggregate` → `rebuild_aggregate` →
   `preview_aggregate`). Apollo's 10:37 backfill logged bob-cli
   `drain phases: ... publish=112.40s` while draining just **2** outbox rows. The
   agent_publication drain for bob-cli takes the same path every tick. Its hood
   publishes finish within ~20–100 s, and then bob-cli's link drain eats the next
   100–170 s before `drain_project(sase)` starts.
   (`~/.sase/logs/agent_publication_drains.jsonl` and the progress lines in
   `lumberjack-publication.log` confirm this tick by tick.)

3. **Lost write races repeat the whole preview, ignoring the deadline.**
   `_write_merged_aggregate` retries `compute_preview()` up to
   `_MAX_AGGREGATE_WRITE_ATTEMPTS = 5` times whenever another writer advanced the
   aggregate generation. A 50–100 s preview window makes those races likely: both chops
   and agent finalizers write the same `artifact-links.json`. No retry checks the
   deadline. `preview_reconciled_aggregate` checks the deadline only between stores, so
   one lost race in sase's reconcile is enough to blow through the 240 s budget into the
   300 s SIGKILL. That is the 10:00 backfill failure.

4. **agent_publication cannot say where it died.** Its slow-pass progress lines
   (`_maybe_progress`, 60 s threshold) fire only between projects. A kill inside one
   project's drain leaves `[output absent]`, which is what the digest showed.

The bob-cli outbox backlog (7,172 never-publishable `read` observations, up to ~11 weeks
old) inflates every snapshot. It is intentional, though: the approved
`plan:202611/artifact_link_backfill_timeouts.md` decided `outbox_retention = keep`
(90-day retention). This epic does not revisit that decision. It makes each snapshot
cheaper and stops rebuilding it redundantly.

### Rust boundary

These are I/O, memoization, and orchestration changes inside existing Python modules.
The reduction, collision-partition, and validation logic stays in the existing
`sase_core_rs` bindings, unchanged. No sase-core change and no `sase-core-revision.txt`
bump. A later sase-core change could batch the per-entry `artifact_link_events_reduce`
and `artifact_link_collision_partition` calls (~14k each per bob-cli preview). If
measurements after this epic still justify it, record it as a `PROPOSED FOLLOW-UP`. It
is out of scope here.

## Goal and acceptance criteria

- `preview_aggregate()` builds exactly one event snapshot and parses the outbox once.
- Inside one publish, drain, or reconcile scope, repeated snapshots do not re-read or
  re-validate unchanged durable event files, and do not re-parse an unchanged outbox.
  Outside a scope, behavior is byte-for-byte what it is today: TUI and CLI processes get
  no long-lived cache.
- After a lost CAS race, an aggregate write gives up with a typed deferral instead of
  starting a preview it cannot finish before the caller's deadline. Callers without a
  deadline (interactive `sase artifact doctor --fix`, CLI writes) keep today's
  5-attempt, last-attempt-forced loop.
- Both chops turn that deferral into their existing logged-deferral paths, never a
  SIGKILL.
- A killed or slow agent_publication tick leaves a progress line naming the project and
  stage it was in. Link drain phase timings, including publish sub-phases, appear in
  both chops' logs.
- Same published, retained, and dropped rows as today. The one exception is the
  `publish_deferral` decision when a deferral actually happens.
- Expected effect on apollo after deploy: a bob-cli link publish drops from ~100 s to
  ~20–30 s, and an agent_publication tick drops from 150–240 s to well under 120 s.
  sase's reconcile preview roughly halves, and the digests stop reporting these two
  timeouts.

## Phase `snapshot-reuse`: one snapshot per preview, plus a scoped input memo

Resolves task bead **sase-1rt**. Close it with
`sase bead close sase-1rt --note "<what you verified>"` once this phase's checks pass.

### 1. Single snapshot per preview (sase-1rt)

- Add a keyword-only `event_snapshot: ArtifactLinkEventSnapshot | None = None` to
  `_load_store_truth_rows` and `_load_sidecar_truth_rows`
  (`src/sase/sdd/_artifact_link_store_rows.py`). When it is given, use it instead of
  calling `self.artifact_link_event_snapshot(...)`.
- `preview_aggregate` passes the snapshot it already built. The arguments are identical
  (`include_pending=True`, `strict=True`, same `exclude_pending_event_ids`).
  `_authoritative_source_was_consulted_for_pass(event_snapshot=...)` keeps receiving
  that same object.
- Audit the other `artifact_link_event_snapshot(` call sites in
  `src/sase/sdd/_artifact_link_store_rows.py` and `_artifact_link_store_reconcile.py`
  for the same build-then-rebuild pattern inside one method. Fix any exact duplicates
  the same way. Do not change a call whose arguments (`include_pending`, `strict`,
  exclusions) differ.
- Update the mixin `Callable[...]` attribute annotations that declare these methods.

### 2. Scoped snapshot-input memo

New module `src/sase/sdd/_artifact_link_snapshot_memo.py`:

- `artifact_link_snapshot_memo()` is a context manager backed by a `ContextVar`. It is
  re-entrant: a nested scope reuses the outer memo, and only the outermost exit discards
  it. With no active scope, every lookup is a miss and nothing is stored.
- **Durable entries.** Key:
  `(kind, resolved root, relpath, st_ino, st_size, st_mtime_ns)`. Value: the
  `ArtifactLinkEventCandidate`, or the `ArtifactLinkEventValidationFinding` that
  `_durable_candidates` (`src/sase/sdd/_artifact_link_event_store.py`) produced for that
  file. Event objects are content-addressed (`link-events/v1/<xx>/<sha256>.json`), so an
  unchanged stat tuple means unchanged bytes. Take the `stat` result before
  `read_bytes`.
- **Pending outbox.** Key: `(project_key, outbox path, st_ino, st_size, st_mtime_ns)`.
  Value: the full, unfiltered tuple of pending candidates that `_pending_candidates`
  would build, plus their `created_at` values. Apply `exclude_operation_ids` after the
  lookup, by `operation_id`. The stat must describe the bytes that were parsed. Stat
  inside the same shared lock that `read_artifact_link_outbox_entries` /
  `_read_outbox_records_unlocked` (`src/sase/sdd/_artifact_link_outbox_io.py`) holds, or
  stat before and after the read and store only when they match. Appends change size and
  mtime, and atomic rewrites change the inode, so both invalidate the entry.
- Do not memoize the reduction itself (`reduce_artifact_link_event_inputs`): its inputs
  differ per call. Do not memoize quarantine records.

Activate the scope (outermost wins) around:

- the body of `publish_artifact_link_events`
  (`src/sase/sdd/_artifact_link_event_publish.py`), covering admission, bead projection,
  and aggregate rebuild;
- `drain_artifact_link_outbox` (`src/sase/sdd/_artifact_link_outbox_drain.py`);
- `reconcile_and_repair_artifact_links` (`src/sase/sdd/artifact_link_backfill.py`).

Nothing else activates it. TUI, CLI, and finalizer behavior outside those calls is
unchanged.

### Tests

- `tests/sdd/test_artifact_link_store_aggregate.py` (or a new sibling): one
  `preview_aggregate()` calls `ArtifactLinkEventStoreAdapter.snapshot` once and
  `_read_outbox_records_unlocked` once (counting monkeypatches). The rows it returns are
  equal to the pre-change result for a fixture with durable events, pending events, and
  legacy and bead rows.
- New `tests/sdd/test_artifact_link_snapshot_memo.py`:
  - Inside a scope, two snapshots validate each durable file once (count
    `artifact_link_event_validate_bytes` or `Path.read_bytes`) and parse the outbox
    once.
  - A durable file added inside the scope is picked up. An outbox append inside the
    scope is picked up. `exclude_pending_event_ids` still excludes on a memo hit.
  - With no scope, every snapshot re-reads.
  - Nested scopes share one memo and clear on outer exit.
  - A scoped snapshot equals an unscoped snapshot for the same state (rows, edges,
    findings, pending stats).
- Existing suites stay green: `tests/sdd/test_artifact_link_event_store*.py`,
  `test_artifact_link_store_reconcile*.py`, `test_artifact_link_event_publisher.py`,
  `test_artifact_link_outbox_drain_scaling.py`,
  `tests/test_axe_chop_artifact_link_backfill_*.py`.
- Read-only re-timing on the local machine: `store.preview_aggregate()` for the sase
  project's primary store, before and after. Record both numbers in a bead note.

## Phase `deadline-aware-aggregate`: deadline-gated retries and typed deferrals

### 1. `_write_merged_aggregate` honors a deadline

In `src/sase/sdd/_artifact_link_store_aggregate.py`:

- Add `class ArtifactLinkAggregateWriteDeferred(Exception)` carrying `attempts: int` and
  `last_compute_seconds: float`.
- `_write_merged_aggregate(compute_preview, *, deadline: float | None = None)`:
  - Time each `compute_preview()`.
  - After a lost CAS (`_write_aggregate_if_current` returned `None`), raise the deferral
    when `deadline is not None` and
    `time.monotonic() + last_compute_seconds >= deadline`. Otherwise retry as today.
  - When `deadline` has already passed before the first compute, raise immediately
    (`attempts=0`).
  - With `deadline=None` the loop is unchanged: 5 attempts, last one forced.
  - When a deadline-bearing caller does reach the final attempt within budget, it is
    still forced, as today.
- `rebuild_aggregate(*, exclude_pending_event_ids=(), deadline: float | None = None)`
  forwards the deadline.

### 2. Reconcile path

- `reconcile_aggregate(deadline=...)` (`src/sase/sdd/_artifact_link_store_reconcile.py`)
  passes its deadline to `_write_merged_aggregate`. On
  `ArtifactLinkAggregateWriteDeferred`, it returns the current `load_aggregate()`
  document with `skip_diagnostics` containing
  `"reconcile deadline expired; aggregate write deferred after a concurrent writer"`.
  The substring `reconcile deadline expired` is `_RECONCILE_CUT_SHORT_DIAGNOSTIC` in
  `src/sase/scripts/_artifact_link_backfill_jobs.py`, so the backfill already treats it
  as a deferral: the project goes first next tick and its estimate is not recorded.
- `reconcile_and_repair_artifact_links` (`src/sase/sdd/artifact_link_backfill.py`)
  returns early, skipping `dangling_scan` and `repair`, when the reconcile result
  carries that diagnostic. Today it returns early only when
  `time.monotonic() >= deadline`.

### 3. Publish path

- `apply_events_to_aggregate(store, objects, *, deadline=None)`
  (`src/sase/sdd/_artifact_link_event_project.py`) forwards to
  `rebuild_aggregate(deadline=...)`.
- `_publish_event_objects_locked` passes its `deadline`. It catches
  `ArtifactLinkAggregateWriteDeferred` **before** the generic `except Exception`, and
  appends the diagnostic `"artifact-link aggregate projection deferred past deadline"`.

> [!decision] publish_deferral = retain Set `aggregate_ready = False` (so
> `published_ids = ()`, exactly like today's aggregate-failure path) and set the
> report's `deferred=True` (OR-ed with the bead-projection deferral).
> `drain_artifact_link_outbox` already propagates `event_report.deferred`. As a result,
> agent_publication's `drain_project` retains the link hints and marks
> `deferred_budget`, and the backfill counts a deferral. The outbox rows stay queued.
> The durable event objects are already written, so next tick's republish is idempotent.

> [!decision] publish_deferral = ack Keep `aggregate_ready = True` with
> `aggregate_rows = ()`, so published ids and event keys settle and the drain rewrites
> those rows out of the outbox. Do not set `deferred`. Use the diagnostic
> `"artifact-link aggregate projection deferred past deadline; next rebuild converges"`.
> Add a test proving that the next `rebuild_aggregate()` (or reconcile) includes the
> settled rows from their durable events.

### 4. Publish sub-phase timings

- Add `phase_seconds: tuple[tuple[str, float], ...] = ()` to
  `_ArtifactLinkEventPublishReport`. Fill it in `_publish_event_objects_locked` for
  `admission`, `states`, `roots`, `local_receipts`, `beads`, and `aggregate`.
- `drain_artifact_link_outbox` appends them to its own `phase_seconds` as
  `publish.<name>`, after `publish`.
- Make sure the backfill's `drain phases:` log line (built in
  `src/sase/scripts/_artifact_link_backfill_jobs.py`) prints the extra keys in order.

### Tests

- Aggregate unit tests (`tests/sdd/test_artifact_link_store_aggregate.py`):
  - A lost CAS with insufficient remaining budget raises the deferral after one compute.
  - A lost CAS with enough budget retries.
  - An already-expired deadline raises before computing.
  - `deadline=None` keeps the 5-attempt / forced-last behavior.
- Reconcile (`tests/sdd/test_artifact_link_store_reconcile*.py`): a deferred write
  returns the on-disk aggregate plus the diagnostic, and
  `reconcile_and_repair_artifact_links` skips repair.
- Backfill (`tests/test_axe_chop_artifact_link_backfill_budgets.py`): that diagnostic
  marks the project deferred and does not call `record_reconcile`.
- Publish (`tests/sdd/test_artifact_link_event_publisher.py`): a deferral produces the
  `publish_deferral` outcome chosen above, and `phase_seconds` lists the six sub-phases.
  Outbox drain (`tests/sdd/test_artifact_link_outbox_drain_scaling.py` or a sibling):
  `publish.*` keys appear in the drain report.

## Phase `publication-drain-progress`: breadcrumbs inside agent_publication drains

- `drain_project(target, *, deadline, git_runner, progress=None)`
  (`src/sase/agents_sync/_publication_job_drain.py`). Emit a stage line **before** each
  stage starts, so a SIGKILL leaves the last stage that started:
  - `hood batch <n> (<k> keys)`
  - `link store resolve`
  - `link drain (<m> agents)`
  - `referenced-by drain`
  - `quarantine sweep`
- After the link drain, emit one line in the backfill's format:
  `link drain phases: read=..s evidence=..s ... publish=..s publish.aggregate=..s ...`,
  built from `link_report.phase_seconds`. Skipped phases print `skipped`, as the
  backfill does.
- `_run_publication_job` (`src/sase/agents_sync/_publication_job_run.py`) passes a
  callback that reuses `_maybe_progress` gating
  (`PUBLICATION_PROGRESS_THRESHOLD_SECONDS`) and the `project=<key>` suffix, so fast
  ticks stay quiet.
- Confirm that `runtime.log.info` output from a builtin chop
  (`src/sase/scripts/sase_chop_agent_publication.py`, `sase.chops.builtin`) reaches the
  run's `.log` while the process is still running. A SIGKILL diagnostic captures only
  what was already written. If the output is buffered, flush per line.
- Tests: extend `tests/agents_sync/test_publication_slow_pass_progress.py`:
  - Past the threshold, stage lines appear in order and before each stage's work.
  - Under the threshold, no stage lines appear.
  - The link-phase line formats `publish.*` keys and `skipped` phases.

## Verification for every phase

Read `sase memory read lint_and_test.md -r "<why>"` before finishing, and run the
verification it requires (`just check` through `sase tool run`). Do not run
`just check-full` unless instructed.

After the epic lands, the owner pulls apollo's `~/projects/github/sase-org/sase`. AXE
code-swaps its routines automatically. Then compare `lumberjack-publication.log` tick
durations and the `artifact_link_backfill` run JSON `duration_ms` against the numbers
above. Agents do not touch apollo.
