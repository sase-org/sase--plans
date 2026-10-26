---
tier: tale
title: Make bead read-model version recovery converge
goal:
  Recover an old-version bead cache durably, preserve concurrent updates, and serve
  subsequent reads without repeated rebuilds.
size: medium
proposed_by: bbugyi200.athena.sase-1oh.3
proposed_with: codex/gpt-6-astra
bead: sase-1oh.3
create_time: 2026-10-26 14:29:30
status: wip
---

- **PARENT:**
  [202610/backlog_epic_top_five_task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202610/backlog_epic_top_five_task_beads.md)
- **BEAD:**
  [sase-1oh.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1oh/sase-1oh.3.md)

# Make bead read-model version recovery converge

Implement the `read-model` phase, `sase-1oh.3`, of
`plan:202610/backlog_epic_top_five_task_beads.md`. The reported defect is tracked by
`sase-1nw`: readers repeatedly rewrite a cache marked `crate_version=0.40.0`, then
apparently discard the rebuild instead of leaving a usable cache. The required result is
a durable recovery followed by warm reads, including when readers race.

This is one bounded coding task in the Rust read-model subsystem, with focused tests and
integration through sase's core revision pin. The phase has no prior work or
remaining-work notes as of planning. No implementation files were changed during
planning, and no reproducer has yet been executed.

## Scope and evidence

Start by re-reading `sase-1oh.3` and `sase-1nw` through `sase bead read`. Read the
parent design through `sase artifact read`. Its final decisions are `secret_scan=build`
and `dormant_flakes=close`; neither changes this phase. Do not re-ask them. Do not edit
memory notes, versions, changelogs, or unrelated domains.

Open `sase-core` through `/sase_repo` and read its `AGENTS.md`. All paths below are
relative to the appropriate checkout; use the path that `sase repo open` prints. Shared
behavior belongs in `crates/sase_core`, with Python remaining a thin caller.

Investigation at core `4121de03` found:

- `crates/sase_core/src/bead/read_model/store.rs` owns `rebuild`, `ensure_fresh`,
  `ensure_fresh_forced`, `rebuild_from_replay`, `write_snapshot_in_txn`,
  `open_and_serve`, and `invalidate_cache_in_place`.
- `invalidate_cache_in_place` already resets the schema, reducer, and crate version
  keys, advances generation counters, clears the token, and removes `streams_known`
  inside an immediate transaction. Commit `5c5bcdc4` added this on October 9, before the
  October 24 bug report; it is also present at tag `v0.40.0`. Simply adding a version
  assignment is not an established fix.
- Snapshot replacement writes all rows before checking the generation CAS. On a lost CAS
  it rolls back and calls `open_and_serve`, whose check currently verifies versions but
  does not establish freshness. Invalidation also advances the generation, so a CAS loss
  alone does not prove another writer committed fresh content.
- `drop_cache_file` deliberately invalidates valid SQLite files in place and discards
  invalidation errors. Preserve the prohibition against unlinking a database or its WAL
  companions while connections can still hold them.
- `meta_keys.rs` owns defaults/backfill; `refresh.rs` owns tail-to-rebuild fallback;
  `queries.rs::with_fresh_cache_at` provides the indexed read entry point. Recovery must
  work for both snapshot readers and indexed/forced-freshness callers.
- Existing substrate tests are in `read_model/tests.rs`. They cover warm reads, in-place
  invalidation with a held connection, schema-less recovery, and freshness races, but
  have no crate-version mismatch regression.

The original report's artifact, `file:explicit:f62a0f258447dcd1b8c4e942`, resolves as
missing. The task bead retains its observations. Treat the suggested CAS/write-fault
mechanism as unverified. The report examined the main database with `immutable=1`; the
reproducer must inspect committed state through an ordinary SQLite read-only connection
that observes WAL state, not infer rollback from main-file mtime or cancelled-write
counters alone.

## 1. Establish the failing transition

Extend the existing Rust test fixtures using temporary event stores and explicit cache
paths. Keep helpers beside the tests; split to a neighboring test module if needed to
respect the core repository's file-size convention.

1. Build a valid cache, seed a prior crate version (`0.40.0`, while asserting it differs
   from the running crate), close the seed connection, and read through the normal cache
   API. Record metadata before and after using reopened connections: version, token,
   streams marker, generation, content generation, and rebuild count.
2. Exercise `cached_store_snapshot_at`, `ensure_cache_ready_at` with an indexed read,
   and `ensure_cache_ready_for_mutation_at` on independent fixtures. Require usable
   results and parity with forced event replay, not a successful replay fallback
   masquerading as a cache hit. Inspect the stored version directly: the status wire
   reports the running crate version and is not proof that the metadata was persisted.
3. Race readers against the same old-version cache using deterministic barriers or
   narrow test hooks. Include a replay prepared against a generation that loses to
   invalidation, and one that loses to a successful competing rebuild. Verify that
   invalidation cannot be mistaken for a successful publication and that the winner is
   never overwritten by an older replay.
4. Capture the actual failure stage before changing production behavior. If an ordinary
   old-version fixture already converges, inspect the swallowed invalidation error and
   `WriteFault::Other` separately from `CasLost`, and reproduce the relevant state with
   a minimal fixture. Check old-schema/missing-key cases only when the evidence points
   there. Do not introduce an unrelated schema-migration framework.

If synthetic fixtures do not reproduce the reported persistent failure, follow the
parent design's read-only snapshot route: obtain a copy of the stuck cache and the
matching event-store inputs through the supported audited repository/artifact access
mechanisms, then run the experiment entirely on disposable copies. Never directly open
or mutate another checkout without `/sase_repo`, and never repair or delete the live
primary cache. Do not mix Python's SQLite library and Rust's SQLite library on the same
database in one process. Preserve only a minimal synthetic regression in git, not a live
bead database or its contents.

If newer landed work has already fixed the defect, identify the fixing commit and prove
the actual failure condition now converges. An already-passing version-string test alone
is insufficient to declare the October 24 report fixed.

## 2. Repair recovery and publication

Make the smallest Rust change supported by the failing regression. Keep one coherent
recovery rule across the snapshot, indexed-read, and mutation-admission paths.

- A version mismatch must establish a valid rebuild baseline. Do not proceed as if
  invalidation succeeded when it failed, or carry a generation captured before an
  invalidation into a replacement write.
- The successful rebuild must atomically publish compatible metadata, replayed rows,
  signatures, frontier, and freshness state. Version metadata alone cannot make retained
  old rows eligible for a fast-path serve.
- Preserve monotonic generations and all existing protections against overwriting
  concurrent mutations. A CAS loser may use a competing result only after proving it is
  usable and fresh; otherwise take the bounded cache-failure/replay path. Avoid
  unbounded retries, longer busy timeouts, and speculative serialization of every read.
- If the regression confirms wasted writes after a known generation loss, check the
  expected generation within the write transaction before clearing and rewriting tables,
  while retaining the atomic commit guard. Do not count committed-rebuild telemetry as
  evidence that losing attempts did no work.
- Preserve the existing distinction between cache faults (fallback) and event-store
  errors (propagate). Preserve in-place invalidation and held-reader safety.

Keep diagnostics/test hooks narrow. No new CLI options, wire schema, Python fallback, or
public binding is expected. If the proven fix requires a changed binding contract,
update the Rust binding and its consumers/tests together; otherwise leave them alone.

## 3. Prove convergence and verify

The regression tests must demonstrate these observable properties:

1. With one reader and no store changes, an old-version cache requires one successful
   recovery rebuild; a reopened connection sees the current stored crate version and
   current rows.
2. Later reads are cache hits: no additional content generation or rebuild count, and no
   additional full replay/replacement writes. Cover a subsequent freshness sweep by
   aging the fixture's sweep timestamp instead of sleeping for a minute.
3. Competing readers converge to one valid published result for a stable store. A racing
   reader may fall back during bounded contention, but later reads cannot repeatedly
   invalidate and rebuild an already-current cache.
4. A real concurrent mutation survives recovery, cache results match replay, and a stale
   CAS loser cannot return invalidated rows as fresh. Held connections remain usable
   throughout recovery.

Use committed metadata and appropriately scoped test instrumentation rather than
timing/IO thresholds. Existing `store_io_stats` counters are thread-local, and its
`full_replays` counter measures mutation-load replay, not every read-model rebuild; do
not use that counter alone to assert this path performed no replay. Add a small
test-only counter at the actual rebuild/write boundary if needed.

Run `just test -p sase_core bead::read_model` through the core repository's supported
wrapper, plus the affected indexed-read/mutation tests. Never invoke bare `cargo`.
Retain the existing invalidation and freshness-race tests unchanged as regression
coverage. Format, then run `sase tool run check` in core and sase. Read
`lint_and_test.md` through `/sase_memory_read` before finishing. Route long checks
through `/sase_monitor`; do not start a command and end the turn while it is running. Do
not run `check-full`, weaken assertions, add suppressions, or skip failures.

For a check failure that reproduces identically on a clean base, record the exact
command and matching failure as a `PROPOSED FOLLOW-UP:` note on `sase-1oh.3`, citing an
existing tracking task if one is known. Such a failure does not keep the completed phase
open. Failures caused by this change must be fixed before completion.

## 4. Integrate and hand the live rollout to the lander

Sase's source pin was `f95690648dadfbe9a1f02e3282f1840f81ed6589` during planning. Follow
`docs/contribute/rust-backend.md`, especially its CI source revision pin section.
Refresh against master before integration because `sase-1oe.3` also changes the pin, and
never move it backward.

For new core changes, use `/sase_final` to declare the linked core change and the
primary integration obligation. The host commits/pushes the linked core first and writes
that pushed SHA into `sase-core-revision.txt` before the primary commit. Do not create
commits or guess the future SHA yourself. For a fix that is already landed, use
`just ratchet-core-revision`, verify the selected SHA contains it, and treat exit 2 as
the documented applied/pending-change result. Verify the relevant tests against the
fixed local core and record both repository checks for the lander.

Live installation is a post-landing action. Before closing this phase, append a
`PROPOSED FOLLOW-UP:` entry on `sase-1oh.3` assigning the parent lander the required
`/sase_gate` for an athena reinstall once master contains the fix and pin. Include the
fix/test evidence and this acceptance procedure:

- Confirm the installed core includes the fix and identify any parked runners still
  using an older loaded extension.
- Read cache metadata with a normal read-only connection, trigger the first normal bead
  read, and verify the stored version and replay parity become current.
- Observe later reads and an existing parked runner across at least two fallback
  intervals. Confirm rebuild/content generations stay stable without store changes and
  the reported full-cache WAL churn is gone; ordinary telemetry writes are not failures.

The gate is required by `global-install-is-human-only` and the approved parent design.
Do not reinstall globally or publish a gate claiming the fix is landed before the host
has landed it. The lander owns this post-master gate and records its identity when
settling `sase-1nw`.

## Completion

Honor the direct phase-worker instruction over the broader parent plan: close only
`sase-1oh.3`. Leave `sase-1nw`, the parent epic, and any ancestor plan bead for their
owners; provide verification evidence on this phase for the lander to close the task. Do
not set statuses by hand or create beads. Record discovered work only through
`sase bead note sase-1oh.3 'PROPOSED FOLLOW-UP: ...'`.

Before closing, run `sase bead epic-symbols sase-1oh.3` again. Planning found none;
resolve any newly introduced entries or re-key genuinely pending symbols to a still-open
owning bead before closing. Then run
`sase bead close sase-1oh.3 --note "<reproduction, fix or fixing commit, targeted tests, both check results, and post-landing gate handoff>"`.
Finish through `/sase_final`, declaring every repository changed. Report verified local
convergence separately from the live acceptance still awaiting the human gate.
