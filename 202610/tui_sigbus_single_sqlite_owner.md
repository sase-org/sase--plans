---
tier: epic
title: Stop TUI SIGBUS crashes by giving every SQLite file one SQLite library per
  process
goal: 'The sase TUI no longer dies with `bus error (core dumped)` or corrupts its
  SQLite indexes, because no process ever opens the same SQLite database file through
  both Python''s stdlib `sqlite3` and the Rust core''s bundled SQLite; the existing
  corrupt dismissed-bundle index heals itself, future fatal signals leave Python stacks
  behind, and a regression guard keeps the invariant from silently returning.

  '
phases:
- id: stopgap
  title: Stop the crash now with in-process serialization, self-heal, and fault logs
  depends_on: []
  size: small
  description: 'stopgap: serialize every Python-sqlite3 connection and Rust call on
    the dismissed-bundle index and artifact index inside one process, quarantine and
    rebuild a corrupt dismissed index, and enable faulthandler logging in the TUI.'
- id: core-archive-store
  title: sase-core owns the dismissed-bundle archive index store
  depends_on: []
  size: medium
  description: 'core-archive-store: port the dismissed-bundle index schema, writes,
    queries, rebuild, signature, and corrupt-file quarantine from Python into sase_core::agent_archive
    with PyO3 bindings and Rust tests.'
- id: core-index-readers
  title: sase-core bindings for the remaining Python artifact-index reads and alias
    writes
  depends_on: []
  size: small
  description: 'core-index-readers: add read-only no-migrate schema-version and column-projection
    bindings plus alias upsert/delete/list bindings for agent_artifact_index.sqlite.'
- id: py-archive-adapter
  title: Python dismissed-bundle index becomes a thin adapter over Rust
  depends_on:
  - stopgap
  - core-archive-store
  size: medium
  description: 'py-archive-adapter: move the sase-core pin, replace all stdlib sqlite3
    use in ace/dismissed_bundle_index with the new bindings while keeping public signatures
    and degrade semantics, and drop the stopgap dismissed-index lock.'
- id: py-index-readers
  title: Route Python artifact-index readers and alias writes through Rust
  depends_on:
  - stopgap
  - core-index-readers
  size: small
  description: 'py-index-readers: move the sase-core pin and switch the lifecycle
    schema check, catalog projection, chat provenance reader, and layout alias CLI
    to the new bindings.'
- id: guard
  title: Regression guard, ownership docs, and follow-ups
  depends_on:
  - py-archive-adapter
  - py-index-readers
  size: small
  description: 'guard: add a static test that forbids stdlib sqlite3 connections outside
    an allowlist of Python-owned databases, document SQLite ownership, and record
    follow-ups for the remaining CLI-only hazards.'
decisions:
  memory_sqlite_owner:
    ask: Add a decisions-web record stating the one-SQLite-library-per-database-file
      rule?
    memory:
    - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.104
decided_by: reviewer
decided_via: tui
create_time: 2026-10-13 12:24:21
status: wip
bead_id: sase-1kk
---

- **PROMPT:** [prompts/202610/tui_sigbus_single_sqlite_owner.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/tui_sigbus_single_sqlite_owner.md)
- **BEAD:** [sase-1kk](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kk/README.md)

# Plan: Stop TUI SIGBUS crashes by giving every SQLite file one SQLite library per process

## Diagnosis (root cause, proven)

The sase TUI process loads **two independent copies of SQLite**:

| Copy                                                                  | Used by                      | Version on athena |
| --------------------------------------------------------------------- | ---------------------------- | ----------------- |
| Python stdlib `sqlite3` (uv-managed CPython)                          | Python code under `src/sase` | 3.53.1            |
| Bundled in `sase_core_rs` (`rusqlite` 0.32, `features = ["bundled"]`) | `sase_core` Rust code        | its own (3.46.x)  |

Both copies open the **same WAL-mode database files in the same process**:

- `~/.sase/dismissed_bundles/index.sqlite`: Python owns the schema and all writes
  (`src/sase/ace/dismissed_bundle_index/_schema.py` `_connect`, which sets WAL). Rust
  reads it on the TUI's archive-corpus worker
  (`sase_core/src/agent_archive/corpus/compile.rs` `compile_agent_archive_corpus`,
  reached via `src/sase/core/agent_archive_facade.py` and
  `src/sase/core/agent_archive_corpus_cache.py`). Meanwhile other TUI threads run Python
  read-write connections: record decks, identity snapshots, dismiss/kill upserts, revive
  visibility writes, and the startup rebuild. Nothing serializes them.
- `~/.sase/agent_artifact_index.sqlite`: Rust owns it (`agent_scan/index/storage.rs`,
  WAL). Python opens it directly in
  `src/sase/core/agent_artifact_index_lifecycle_schema.py` (TUI startup),
  `src/sase/agents/catalog/_sources.py` (TUI catalog, **outside** the existing
  `agent_artifact_index_operation_lock`), and, from the CLI,
  `src/sase/history/chat_catalog_provenance/artifacts.py` and
  `src/sase/agents/cli_artifacts_layout.py`. In addition, `src/sase/stats/query.py`
  calls Rust on the index without that lock.

POSIX advisory locks belong to a (process, file) pair, and **locks held by the same
process never conflict with each other**. SQLite's unix VFS tracks its own connections
per copy, so neither copy can see the other's locks:

1. **Open → truncate → SIGBUS.** When copy B attaches to the `-shm` file, `F_GETLK` on
   the DMS byte reports "unlocked" whenever no _other process_ holds it. B then decides
   it is the first connection and `ftruncate`s `-shm` to 3 bytes while copy A still has
   it `mmap`ed. A's next access to a hash-table page is past EOF, which raises
   **SIGBUS**.
2. **Close → checkpoint and unlink.** When B closes what it thinks is the last
   connection, its EXCLUSIVE lock on the database "succeeds" in-process. B then
   checkpoints and **deletes `-wal` and `-shm`** while A is still mid-transaction. A
   keeps using an unlinked wal-index, and the database is corrupted.

Evidence:

- **Deterministic repro (athena, installed `sase_core_rs`).**
  1. Open a Python `sqlite3` connection on a WAL database created by
     `sase_core_rs.write_agent_artifact_index_meta`, and keep it open.
  2. In the same process, call `write_agent_artifact_index_meta` once more.
  3. `-wal` and `-shm` disappear from disk, and `/proc/self/maps` shows
     `agent_artifact_index.sqlite-shm (deleted)`.

  This is exactly the "(deleted) -shm mapping" the apollo research note caught live.

- **Stress repro (same process).** One Python thread writes and reads while two threads
  call Rust open/close on the same file, for 25 s.
  - Run 1 died with exit **135 (128+SIGBUS)**.
  - Runs 2 and 3 ended with `database disk image is malformed`.
- **Control.** The identical workload with the Python writer in a **separate process**
  ran for 2×25 s with 0 errors and `integrity_check = ok`. The hazard is strictly
  intra-process.
- **The athena data itself.** `~/.sase/dismissed_bundles/index.sqlite` (48 MB, last
  written 2026-10-12 15:07) has lost its `SQLite format 3` header. Page 1 now holds a
  b-tree leaf page, and every open fails with `file is not a database`. That is a
  checkpoint applied through a stale or reset wal-index. The TUI silently falls back to
  slow filesystem scans because every Python API catches `sqlite3.Error`.
  `ensure_dismissed_archive_ready` cannot repair it, because `rebuild_index` itself
  fails to open the corrupt file.
- **No core file.** The crashed TUI ran with `RLIMIT_CORE` soft limit 0 and
  `core_pattern=core`. There is also no `faulthandler`, so the crash itself left no
  stack. The stopgap phase fixes that.

This is consistent with the apollo note `research:202610/apollo_tui_bus_error_crash.md`.
That note blamed only the Python reads of the artifact index. The bigger exposure is the
dismissed-bundle index, where Python is the writer and Rust reads concurrently.
Contention from the startup-slowness work (sase-1kj) only widens the race window; it
neither causes nor fixes the bug.

## The invariant this epic establishes

**Every SQLite database file is opened by exactly one SQLite library per process.**

- Rust-owned files are opened only through `sase_core_rs` bindings:
  - `agent_artifact_index.sqlite`
  - `dismissed_bundles/index.sqlite` (after this epic)
  - `tools/runs.sqlite`
  - `telemetry/metrics.sqlite`
  - the bead read model
- Python-owned files are opened only by stdlib `sqlite3` and never by Rust:
  - `beads.db`
  - `chats_catalog.sqlite`
  - external agy conversation DBs

Linking `rusqlite` against a shared system SQLite was rejected. CPython from uv /
python-build-standalone links its SQLite statically, so there is no shared library to
unify on, and dropping `bundled` would hurt wheel portability. Process-wide
serialization alone was also rejected as the end state. It is fragile, because any new
call site that skips the lock re-opens the bug (two such sites already exist). It also
stalls UI-thread writers behind long Rust corpus compiles. It is used here only as the
stopgap.

## Phase `stopgap`: stop the crash now (sase repo only)

Goal: land in one small commit so the crash and the corruption stop before the Rust port
is done. Keep it minimal; later phases delete most of it.

1. **Dismissed-index process lock.** Add `src/sase/ace/dismissed_bundle_index/_lock.py`,
   mirroring `src/sase/core/agent_artifact_index_lock.py`: one module-level `RLock`, a
   blocking context manager, and a bounded try-lock variant.
   - Hold it for the **entire lifetime** of every Python connection opened by
     `_schema.connection()` and `_schema.write_connection()`. Every Python open goes
     through `_connect`; confirm that with a grep.
   - Hold it around the Rust call in `src/sase/core/agent_archive_facade.py` that
     invokes `compile_agent_archive_corpus` (the `require_rust_binding(...)` call
     returning the corpus handle).
   - Rust `py.allow_threads` releases the GIL, but the Python `RLock` stays held by the
     calling thread, which is the point.
   - Callers reachable from the Textual UI thread (dismiss/kill persistence, record
     decks, revive) must not block indefinitely. Use the bounded acquire (about 2 s) in
     the `_api.py` entry points, and on timeout take the function's existing failure
     return (`None` or `False`) exactly as a `sqlite3.Error` does today.
   - Scope the lock to the connection or Rust call only. Never call into the
     artifact-index lock while holding it, and vice versa, so the two locks can never
     deadlock.
2. **Artifact-index sites that skip the existing lock.**
   - Wrap `load_artifact_index_projection` in `src/sase/agents/catalog/_sources.py` with
     `try_agent_artifact_index_operation_lock`. On timeout, return the existing
     diagnostic-style empty projection, for example "agent artifact index is busy".
   - Wrap the Rust call in `src/sase/stats/query.py` with
     `agent_artifact_index_operation_lock`.
   - In `src/sase/core/agent_artifact_index_lifecycle_schema.py`,
     `_read_stored_index_schema_version` uses `with sqlite3.connect(...) as conn:`. That
     commits but does **not** close. Wrap it in `contextlib.closing` so the connection
     is closed inside the lock its callers already hold.
   - Wrap `_indexed_records` in `src/sase/history/chat_catalog_provenance/artifacts.py`
     with the artifact-index lock as well.
3. **Self-heal a corrupt dismissed index.**
   - In `rebuild_index` (`src/sase/ace/dismissed_bundle_index/_api.py`), check whether
     opening the existing file fails with `sqlite3.DatabaseError` whose
     `sqlite_errorcode` is `SQLITE_NOTADB` (26) or `SQLITE_CORRUPT` (11).
   - If it does, quarantine the file: rename `index.sqlite` and any `-wal`/`-shm`
     siblings to `index.sqlite.corrupt-<UTC yyyymmddTHHMMSSZ>`. First re-`stat` the path
     and only rename if `(st_dev, st_ino)` still matches the file that failed, so a
     concurrent healer never quarantines a freshly rebuilt index.
   - Then rebuild into a fresh file. Log one warning with the quarantine path.
   - `ensure_dismissed_archive_ready` and the startup path in
     `src/sase/core/agent_artifact_index_lifecycle.py` already call `rebuild_index` when
     `index_signature` returns `None`, so the athena file heals on the next TUI start.
4. **Fatal-signal stacks.**
   - At TUI startup, before the Textual app runs, open `<sase_home>/logs/tui_faults.log`
     in append mode. Find the entry point behind `sase tui` / `sase -p tui`.
   - Write a one-line session header (UTC time, PID, sase version), then call
     `faulthandler.enable(file=<that file>, all_threads=True)`.
   - Keep the file object referenced for the process lifetime.
   - The TUI owns stderr, so `PYTHONFAULTHANDLER` output is lost. This log is how the
     next SIGBUS or SIGSEGV gets a stack.
5. **Tests.**
   - Unit-test the quarantine-and-rebuild path: write garbage bytes into a temp
     `index.sqlite`, call `rebuild_index`, and assert the quarantine file exists and the
     rebuilt index answers `index_signature`.
   - Unit-test that `_connect`'s callers hold the new lock. Patching the lock with a
     recording double is enough.
   - Unit-test that faulthandler setup writes the header and enables it. Monkeypatch
     `faulthandler.enable`.

## Phase `core-archive-store`: sase-core owns the dismissed-bundle archive index (sase-core repo)

Open the linked checkout with `sase repo open sase-core`. Work in
`crates/sase_core/src/agent_archive/` and expose bindings in
`crates/sase_core_py/src/agent_custody/`, next to the existing archive bindings. The
Python reference implementation to port, verbatim in behavior, is
`src/sase/ace/dismissed_bundle_index/` in the sase repo: `_schema.py`, `_api.py`,
`_summary.py`, `_bundle_io.py` and `_models.py` (`SCHEMA_VERSION = 3`).

1. **One store module owns `<root>/index.sqlite`**, for example
   `agent_archive/store.rs`.
   - Opening for writes: create `root` when asked, set `busy_timeout` to 30 s, run
     `PRAGMA journal_mode = WAL` and `PRAGMA synchronous = NORMAL`, then ensure the
     schema.
   - The schema must be **byte-for-byte compatible** with the current Python DDL:
     - tables `dismissed_bundle_index_meta`, `dismissed_bundle_summaries` (all columns,
       defaults and NOT NULLs), and `archive_visibility_projection` (with its CHECK);
     - the FTS5 virtual table
       `dismissed_bundle_search_fts(bundle_path UNINDEXED, archive_search_text)`;
     - the five `idx_dismissed_bundle_*` indexes;
     - the `schema_version` meta row;
     - the same drop-and-recreate behavior when the stored version differs or
       `_schema_compatible` fails.

     Existing on-disk indexes written by Python must open **without** being dropped.

   - Read-only opens stay read-only and never migrate.
   - Route the existing `open_existing_index` in `agent_archive/mod.rs` and the open in
     `corpus/compile.rs` through this module's open helpers, so there is one open path.

2. **Operations**, each a `sase_core` function plus a PyO3 binding that uses
   `py.allow_threads`, returns JSON-shaped wire data, and maps errors to `RuntimeError`:
   - **upsert bundle summary** `(root, bundle_path, bundle_json)`.
     - Port `summary_from_bundle` and its helpers (`_is_workflow_child`, `_meta_patch`,
       `_archive_payload_hash`, `_project_name_from_file`, `_missing_requirements`).
       `archive_payload_sha256` must be identical to Python's; Rust already has an
       `archive_payload_hash` in `mod.rs`, so reconcile the two.
     - Take the file signature `(mtime_ns, size)` from the bundle file.
     - Run the same `INSERT … ON CONFLICT(bundle_path) DO UPDATE` and the
       `archive_visibility_projection … DO NOTHING` insert inside `BEGIN IMMEDIATE`.
   - **delete summaries for suffixes** `(root, suffixes)`.
   - **query summaries**
     `(root, suffixes?, cl_name?, project_name?, top_level_only, visibility = "hidden" | null, limit?, offset?)`.
     Port the filters, ordering, and column set from `_api.query_summaries`, and return
     summary wire rows. Return `None` when the index file is absent, matching Python.
   - **query bundle paths by suffixes** `(root, suffixes)`.
   - **set visibility for suffixes**
     `(root, suffixes, visibility, revived_at?, dismissed_at?, pinned_at?)`. Port
     `_schema.set_visibility_for_suffixes`; it returns the changed count.
   - **query summary identities** `(root)`.
   - **index signature** `(root)`, returning
     `(schema_version, mtime_ns, size, row_count)` or `None`.
   - **rebuild** `(root)`, returning `{indexed_rows, skipped_corrupt}`.
     - Iterate bundle paths exactly like `_bundle_io.iter_bundle_paths`, including the
       sorting.
     - Skip unreadable or invalid bundles, and clear the FTS table like Python does.
     - Expose progress so the Python `dismissed_bundle_index_progress` contract (polled
       from other threads during a rebuild) still works. A process-global
       `root → indexed_rows` map readable through a small binding is fine.
   - **verify**: make `verify_agent_archive_index` return the fields Python's
     `verify_index` / `DismissedBundleIndexVerifyResult` report. Extend the wire if
     needed.
3. **Corruption self-heal in Rust.** Port the stopgap behavior. Only **rebuild**
   quarantines a file that fails with `SQLITE_NOTADB` or `SQLITE_CORRUPT`, using the
   same `index.sqlite.corrupt-<UTC>` name and `(dev, ino)` guard, then builds fresh.
   Reuse or share the quarantine helper in `agent_scan/index/storage.rs` if it fits.
4. **Tests (Rust).**
   - Round-trip for every operation.
   - A parity test: an index created by the exact Python DDL (embed the DDL as a test
     fixture) is opened by the Rust store without a schema drop and returns the same
     rows.
   - Payload-hash parity against a Python-computed fixture value.
   - FTS5 table presence (the bundled build enables `SQLITE_ENABLE_FTS5`).
   - Quarantine-and-rebuild on a garbage file.
   - A concurrent test where several threads upsert and compile the corpus on one root
     in one process, then `verify` reports ok.
5. Add a CHANGELOG entry, and run the sase-core repo's own checks per its AGENTS.md.

## Phase `core-index-readers`: Rust bindings for the remaining artifact-index access (sase-core repo)

Add these in `crates/sase_core/src/agent_scan/index/` with bindings in
`crates/sase_core_py/src/agent_scan/mod.rs`:

1. **`read_agent_artifact_index_schema_version(index_path) -> int | None`.** Open with
   `SQLITE_OPEN_READ_ONLY` and **never** migrate or fall back to the read-write open
   (unlike `open_index_read_only`). Return `None` when the file is absent or unreadable,
   `0` when the `meta` table or `schema_version` key is missing, and the parsed integer
   otherwise. This mirrors `_read_stored_index_schema_version`.
2. **`load_agent_artifact_index_projection(index_path, columns, busy_timeout_ms=250)`**,
   returning `{schema_version, rows}`.
   - Read-only, never migrates.
   - `columns` is validated against an allowlist of real `agent_artifacts` columns.
     Reject anything else; never interpolate unvalidated names.
   - It must serve both current callers: the catalog column set in
     `src/sase/agents/catalog/_sources.py` (`_ARTIFACT_INDEX_COLUMNS`) and the chat
     provenance set
     (`artifact_dir, project_name, workflow_dir_name, timestamp, record_json`).
   - Rows come back as lists in column order.
3. **Alias bindings** for the table Rust already defines (`agent_artifact_aliases` in
   `storage.rs`):
   - `upsert_agent_artifact_alias(index_path, alias_path, artifact_dir)`
   - `delete_agent_artifact_alias(index_path, alias_path, artifact_dir)`, which deletes
     where `alias_path = ? OR artifact_dir = ?`
   - `list_agent_artifact_aliases(index_path)`

   These mirror `_write_alias`, `_delete_alias` and `_read_aliases` in
   `src/sase/agents/cli_artifacts_layout.py`, using the normal read-write open.

4. Rust tests for each binding, including:
   - a stale-schema index, where the version read must not migrate it;
   - a missing meta table;
   - a rejected column name.

   Add a CHANGELOG entry and run the repo checks.

## Phase `py-archive-adapter`: Python dismissed-bundle index becomes a thin adapter (sase repo)

1. Move `sase-core-revision.txt` (and any version window `docs/rust_backend.md`
   requires) to a sase-core master SHA that contains the `core-archive-store` commit. If
   a sibling phase already moved the pin, move it to the newest SHA that contains both.
2. Rewrite the internals of `src/sase/ace/dismissed_bundle_index/`:
   - Every function exported from `__init__.py` keeps its signature, return types and
     degrade semantics (`None` / `False` instead of raising). That covers
     `archive_index_exists`, `index_signature`, `upsert_bundle_summary`,
     `delete_bundle_summaries_for_suffixes`, `query_summaries`,
     `query_bundle_paths_by_suffixes`, `set_archive_visibility_for_suffixes`,
     `rebuild_index`, `query_summary_identities`, `verify_index` and
     `dismissed_bundle_index_progress`.
   - Each one calls the new bindings through the `require_rust_binding` facade pattern
     used in `src/sase/core/agent_archive_facade.py`.
   - Convert wire rows into the existing `DismissedBundleSummary` model.
   - Delete the stdlib `sqlite3` code: `_schema.py`'s connection helpers and SQL,
     `summary_from_row`, `summary_from_bundle` if it becomes unused, and
     `archive_index_connection` (no external callers; confirm with a grep).
3. Replace the `sqlite3.Error` catches that guarded these calls with the binding's error
   type (`RuntimeError`/`OSError`/`ValueError`). They appear in
   `src/sase/ace/dismissed_agents_bundles.py`, `src/sase/ace/dismissed_agents.py`, the
   TUI deck, modal and action callers listed by
   `grep -rn "dismissed_bundle_index" src/sase`, and
   `src/sase/agents/catalog/_sources.py`.
4. **Delete the stopgap dismissed-index lock** and its call sites, including the one
   around the corpus compile. Every access is now inside one SQLite copy, which handles
   concurrency itself, and keeping the lock would only stall UI-thread writers.
5. Tests:
   - Update the existing tests under `tests/` for the dismissed index and archive to the
     new backend.
   - Add a threaded in-process smoke test: concurrent `upsert_bundle_summary` calls and
     corpus compiles on one temp root, then assert `verify_index().ok`.
   - Tests must not open these files with stdlib `sqlite3`. If a test needs raw SQL, run
     it in a subprocess.
   - Follow the `lint_and_test` reference memory before finishing.

## Phase `py-index-readers`: route the remaining artifact-index access through Rust (sase repo)

1. Move the sase-core pin to include the `core-index-readers` commit. Same rule as above
   if a sibling moved it first.
2. In `src/sase/core/agent_artifact_index_lifecycle_schema.py`,
   `_read_stored_index_schema_version` calls `read_agent_artifact_index_schema_version`.
   Keep the current `None`/`0` semantics and the caller-held lock.
3. In `src/sase/agents/catalog/_sources.py`, `load_artifact_index_projection` calls
   `load_agent_artifact_index_projection` with `_ARTIFACT_INDEX_COLUMNS`.
   - Keep the `SUPPORTED_AGENT_ARTIFACT_INDEX_SCHEMA_VERSIONS` check and the existing
     diagnostic strings.
   - Call it through the artifact-index lock wrapper, like the other facade calls.
   - Drop the stopgap's ad-hoc wrapping if the facade now provides it.
4. In `src/sase/history/chat_catalog_provenance/artifacts.py`, `_indexed_records` uses
   the same projection binding with the provenance column set.
5. In `src/sase/agents/cli_artifacts_layout.py`, `_write_alias`, `_delete_alias` and
   `_read_aliases` use the alias bindings, and `_LAYOUT_ALIAS_DDL` is removed.
6. Remove the now-unused `sqlite3` imports from these modules and update their tests.
   Tests must not open the artifact index with stdlib `sqlite3` in-process. Follow the
   `lint_and_test` reference memory.

## Phase `guard`: regression guard, docs, and follow-ups (sase repo)

1. **Static guard test**, for example `tests/test_sqlite_single_owner.py`.
   - AST-scan `src/sase/**/*.py` for calls to `sqlite3.connect` (including
     `from sqlite3 import connect`) and for `sqlite3.Connection(...)` construction.
   - Assert that the set of modules containing them equals an explicit allowlist. Each
     entry carries the Python-owned database it serves:
     - the bead store (`beads.db`, `src/sase/bead/_db_schema.py`);
     - the chat catalog cache (`chats_catalog.sqlite`,
       `src/sase/history/chat_catalog_provenance/cache.py`);
     - external agy conversation DBs (`src/sase/llm_provider/_subprocess_agy.py`);
     - `src/sase/migration_kit/sqlite_backup.py`, CLI-only and flagged as a follow-up.
   - Also assert that no module in the allowlist mentions a Rust-owned filename:
     `agent_artifact_index.sqlite`, the dismissed index `INDEX_FILENAME`, `runs.sqlite`,
     `metrics.sqlite`, `bead-read-model`.
   - The failure message explains the one-SQLite-library-per-file rule and points at the
     docs section below.
   - Importing `sqlite3` only for exception types stays allowed.
2. **Docs.** Add a short "SQLite ownership" section to `docs/rust_backend.md` with:
   - the hazard: two SQLite copies in one process lose POSIX locks, truncate or unlink
     `-shm`/`-wal`, and cause SIGBUS and corruption;
   - the owner table from "The invariant" above;
   - the rule that new Python access to a Rust-owned file must go through a binding.
3. **Follow-ups.** As an epic phase worker, record `PROPOSED FOLLOW-UP:` notes on this
   phase's bead instead of creating beads:
   - Rust uses `beads.db` as its mutation **lock file**: `bead/mutation/store.rs`
     `BEAD_MUTATION_LOCK_FILENAME`, opened and closed per mutation in `store_lock.rs`.
     Python uses the same file as a WAL SQLite mirror. Closing the Rust handle drops
     Python's POSIX locks on it. Today this is CLI-only, through
     `_project_store._conn_cache`.
   - `migration_kit` opens arbitrary live databases, including Rust-owned ones, with
     stdlib `sqlite3` (`sqlite_backup.py`). It also reads their headers with plain
     `open()`, raw-copies live `-wal`/`-shm` files (`backup.py`), and hashes live DB
     files (`restore.py`). All of these are CLI-only.
4. > [!decision] memory_sqlite_owner
   >
   > If accepted, use the `/sase_memory_write` skill to add a decisions-web strand
   > `one-sqlite-per-file`, titled "One SQLite Library Per Database File". Give it a
   > Claim / Why / Cost / Reopens-when body drawn from this plan's Diagnosis and
   > Invariant sections, link `[[decisions:rust-core-required]]`, and run
   > `sase memory init`.

## Verification (whole epic)

- `just check` passes in sase after each sase phase. The sase-core checks pass after
  each sase-core phase.
- `grep -rn "sqlite3.connect" src/sase` matches only the guard test's allowlist.
- Manual check on athena after the human reinstall below:
  - Start `sase tui`. `~/.sase/dismissed_bundles/index.sqlite.corrupt-*` appears once,
    `index.sqlite` reopens with a valid header, and the Agents Archive shelf shows
    dismissed agents.
  - `~/.sase/logs/tui_faults.log` has a session header.
  - While the TUI runs, no `-shm (deleted)` mapping appears in `/proc/<tui pid>/maps`.

## Rollout notes for the user

- The TUI runs from editable checkouts, and `sase_core_rs` is a locally built wheel.
  Rust changes take effect only after a rebuild or reinstall. Global install is
  human-only, so the user runs `just install` (or the sase-core dev build) after the
  sase-core phases land.
- The stopgap phase is pure Python. It takes effect at the next TUI restart once it
  lands. On that start, it self-heals athena's corrupt dismissed index, and apollo's if
  it is corrupt too.
- On apollo, also move aside the stale Oct 9 apport report
  (`/var/crash/_usr_bin_python3.12.1000.crash`), as the research note suggests, so a
  future crash can be captured. The faulthandler log makes this less critical.
