---
tier: tale
title: Retire Python's beads.db mirror
goal:
  Remove same-process SQLite lock loss by making Rust the sole owner of the beads.db
  lock file.
size: medium
proposed_by: bbugyi200.athena.sase-1tc.4
proposed_with: codex/gpt-6-astra
bead: sase-1tc.4
create_time: 2026-11-19 05:29:34
status: wip
---

- **PARENT:**
  [202611/backlog_epic_cmdsub_retries_launches.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_cmdsub_retries_launches.md)
- **BEAD:**
  [sase-1tc.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tc/sase-1tc.4.md)

# Retire Python's beads.db mirror for sase-1tc.4

Complete phase `sase-1tc.4` of `sase-1tc`, resolving the defect tracked by `sase-1kp`.
Python must never open the bead store's `beads.db` with stdlib SQLite. Rust retains its
existing mutation lock on that file; canonical events and Rust's separate read-model
database continue to serve reads and mutations.

## Scope and accepted decision

The parent design is `plan:202611/backlog_epic_cmdsub_retries_launches.md`. Its final
decision is **mirror_fix = retire_mirror**. Do not introduce a new lock filename,
dual-lock rollout, Python fallback, feature flag, or memory change. This tale is one
bounded compatibility cleanup: production allocation and export already exist in Rust,
and most dependent test changes are import/fixture changes. There are no earlier phase
notes describing unfinished implementation.

Read the phase and task with `sase bead read`, and the parent design with
`sase artifact read`, before implementing. Recheck `sase agent list` and
`sase bead list -t plan` for live overlapping work. At planning time the only other
running sase work was a projected-record refactor verification monitor; the next phase
and parent lander were waiting. Follow the parent's coordination rule if ownership has
changed.

## Verified starting point

At sase commit `150fce3735efec9d7c3a53ac104a69151f432250`:

- `src/sase/bead/project.py` opens Python SQLite during `BeadProject.init` and keeps
  `_conn_cache` on project instances.
- `_project_store.py` lazily rebuilds and opens the mirror through `_conn`.
  `_max_local_child_counter` queries it, and `_export` falls back to it after Rust
  export errors. The child-counter helpers have no production callers; real creation
  delegates to Rust.
- `_sync_git.py:rebuild_from_jsonl`, re-exported by `bead/sync.py`, recreates the mirror
  from JSONL. Its remaining callers are the compatibility property and tests.
- `bead/db.py`, `_db_*.py`, and `bead/jsonl.py` contain the old SQLite fixture
  implementation. `bead/work_plan.py` also accepts a SQLite connection and converts it
  into issues; its production entry point already passes a path to Rust. Tests and two
  performance fixtures still use these APIs.
- In linked sase-core, `bead/mutation/store.rs` still locks `beads.db`.
  `mutation/create.rs` creates the file if absent and allocates children through
  `MutationView::next_child_id`. `bead_export_jsonl` is already bound to Python.
- `tests/test_sqlite_single_owner.py` still allows `bead._db_schema` to open stdlib
  SQLite. The ownership table in `docs/contribute/rust-backend.md` still identifies
  `beads.db` as Python-owned.

A disposable repro was run and recorded on the phase. In one Python process, initialize
a project, create a plan, retain `conn = project._conn`, and read an issue row. A fresh
subprocess opens the database and attempts
`fcntl.lockf(fd, LOCK_EX | LOCK_NB, 510, 1073741826)`, the main-file SQLite shared-lock
range. It reports **blocked**. Call `sase.core.bead_mutation_facade.update` directly
while retaining that Python connection; the same probe then reports **acquired**, and
`_conn_cache is conn` remains true. `PRAGMA journal_mode` reports `wal`.

This proves loss of the main-file POSIX lock. It does not demonstrate SIGBUS, database
corruption, or loss of locks on different WAL/SHM inodes. Preserve that distinction in
tests and documentation.

## Implementation

### 1. Remove production mirror lifecycle and fallbacks

In `project.py`, `_project_store.py`, `_sync_git.py`, and `sync.py`:

- Remove Python SQLite initialization, `_conn`, `_conn_cache`, `_close_connection`, and
  mirror rebuild/import/export fallbacks.
- Keep `BeadProject` context-manager behavior, with no SQLite resource to close. Retain
  configuration refresh after Rust mutations; update its name/docstring and callers if
  needed so it no longer claims to refresh a database.
- Make `_export` call the required Rust export and propagate failures. In particular, an
  unavailable binding or a corrupt-store `ValueError` must not produce an apparently
  successful export from stale data.
- Delete `_max_local_child_counter` and `_next_child_id` after confirming their only
  callers are tests. Test real child creation through Rust instead. Do not add a binding
  solely to preserve these unused private methods.
- Preserve legacy JSONL-only store reads and their existing Rust import path,
  event-store authority when the projection is stale, config counters, and
  git-sync/no-op behavior.
- Preserve the existing `beads.db` inode. Do not unlink, truncate, replace, or clean up
  old `beads.db`, `-wal`, or `-shm` files during adoption; older processes can still be
  using them. New processes simply stop opening them through SQLite. A newly initialized
  store can retain Rust's empty lock file.

No new backend behavior is expected. If a real missing domain operation is found, use
`/sase_repo` to open `sase-core`, implement and bind it there, and let host completion
update the containing `sase-core-revision.txt` pin (or use the approved ratchet recipe).
Never implement it as a Python fallback. Do not change Rust's lock protocol as part of
this decision.

### 2. Remove the production dependency on legacy SQLite fixtures

Move still-needed compatibility helpers from `bead/db.py`, `_db_*.py`, and
`bead/jsonl.py` into a clearly named package under `tests/test_bead/` and update their
imports. Keep those fixture helpers out of the shipped `src/sase` package. Preserve
useful existing codec, historical schema, and golden-fixture assertions without
redesigning the fixture library. Any helper with no remaining use can be deleted. Audit
textual import and monkeypatch targets as well as normal imports.

Remove the SQLite overload from the production work-plan adapter. Give existing
SQLite-seeded work-plan tests a test helper that reads their fixture issues and calls
the existing Rust-backed `_build_epic_work_plan_from_issues` adapter. Keep the
production path-based entry point and Rust scheduling semantics intact.

Known consumers include `tests/test_bead/conftest.py`, `work_test_helpers.py`,
`db_test_helpers.py`, the `test_db*`, `test_jsonl*`, work-plan/rendering, close, snooze,
task, flag and evidence tests, `test_cross_project.py`, and
`tests/test_autonomy_roles.py`. Use `rg` to find every remaining consumer. Update stale
production docstrings that mention the removed fixture modules.

Adapt integration tests that directly inspect `project._conn`, particularly
`test_project_storage.py`, `test_project_lifecycle.py`, and `test_cli_dep_rm.py`, to
assert canonical issues, dependencies, exports, and no-op behavior. Replace
`test_sync_rebuild.py`'s removed mirror contract with legacy-store/export coverage.
Retain dependency-removal idempotence and rejected-operation nonmutation assertions; do
not drop those contracts with the mirror.

Update `tests/perf/bench_bead.py` and `bench_artifacts_first_paint.py` to create their
JSONL corpora directly or via isolated test fixtures. They must not keep a Python SQLite
connection on `beads.db` while invoking Rust. Preserve corpus size, dependency shape,
and measured behavior. Fixture-only SQLite files should use a separate name such as
`fixture.sqlite`, not the production lock filename.

### 3. Enforce and demonstrate single ownership

- Remove `bead._db_schema` from the production SQLite allowlist and classify `beads.db`
  as a Rust-owned lock-file marker in `tests/test_sqlite_single_owner.py`. Keep the
  existing protections for every other Rust-owned database. The new guard must fail on
  the old production connection site, including its indirect path through `init_db`.
- Add a same-process integration regression covering initialization, open/read, real
  Rust mutation, export/sync, and context exit. Spy on or reject stdlib SQLite opens of
  `beads.db` so the old implementation fails deterministically. Run it for a new store
  and an existing legacy mirror prepared in a separate completed subprocess. Reopen the
  project across mutations and assert current canonical data. Do not simulate success by
  mocking Rust mutations.
- Use a coordinated subprocess writer to exercise real mutations against the same
  disposable store and confirm unique child IDs, retained updates, and working mutation
  exclusion. Use handshakes and bounded waits rather than timing sleeps. Reuse existing
  contention helpers where possible.
- An intentionally manually opened old mirror remains unsafe with the unchanged Rust
  lock. Do not assert that this unsupported combination becomes safe. The regression
  proves that supported new application paths cannot create it; retain the before-fix
  lock witness as phase evidence.
- Keep read-only legacy-store coverage (including the 1,500-row fixture), event
  authority over a stale export, export failure propagation, and successful explicit
  exports without a Python database. Child allocation tests must continue to cover
  local-store counters and existing direct children.

### 4. Update ownership documentation

Read `docs_writing.md` and follow `docs/contribute/writing-docs.md`. Update only the
relevant ownership and bead-backend paragraphs in `docs/contribute/rust-backend.md`:
`beads.db` is the Rust mutation lock file, not a Python SQLite mirror; the SQLite read
model is separate and Rust-owned. Explain that preexisting mirror bytes are left in
place and unused by new Python paths, preserving the lock inode and mixed-process
rollout. No memory notes are authorized for editing.

## Verification and completion

Read `lint_and_test.md`. Run the ownership regression and the affected bead store,
work-plan, CLI, JSONL, core-facade, and fixture consumers, including
`test_autonomy_roles.py`. Smoke-test the changed performance fixture builders without
running full performance sweeps. Run formatting and `sase tool run check` in sase; run
the linked repository's required checks too if it changed. Do not run `just check-full`.
Use `/sase_monitor` for long verification and wait for the staging command itself to
exit before declaring.

Record the before/after evidence, base revision, verification results, and limitations
on `sase-1tc.4`. If a failure reproduces identically on the clean base, record
`PROPOSED FOLLOW-UP:` with that evidence and any existing tracking bead; it does not
keep the completed phase open. Preserve existing assertions, budgets, and safety guards.
Do not add skips, xfails, suppressions, or weakened timeouts to make checks pass. Phase
workers create no task beads.

Once the scope is implemented and verified, append
`TASK RESOLVED: sase-1kp — <evidence>` to this phase. Run
`sase bead epic-symbols sase-1tc.4` immediately before closing and resolve or re-key any
entries to a still-open owning bead; the planning check found none. Close only
`sase-1tc.4` with `sase bead close sase-1tc.4 --note "<verification>"`. Do not close
`sase-1tc`, `sase-1kp`, or an ancestor plan: the parent lander owns those completions.
Finish through `/sase_final` with the correct repository declarations.
