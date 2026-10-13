---
tier: tale
size: small
title: Finish the sase-1kk SQLite single-owner landing gaps and close the epic
goal: 'Epic sase-1kk''s remaining verification gaps are closed: the TUI faulthandler
  setup has its required unit test, docs no longer call the dismissed-bundle summary
  index Python-owned, no test or benchmark opens the Rust-owned artifact index through
  in-process stdlib sqlite3, and epic sase-1kk is closed with its plan marked done.'
proposed_by: bbugyi200.athena.sase-1kk.land
bead: sase-1kk
status: done
---

- **PARENT:**
  [202610/tui_sigbus_single_sqlite_owner.md](https://github.com/sase-org/sase--plans/blob/main/202610/tui_sigbus_single_sqlite_owner.md)
- **BEAD:**
  [sase-1kk](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kk/README.md)

# Plan: Finish the sase-1kk SQLite single-owner landing gaps and close the epic

## Context

Epic **sase-1kk** ("Stop TUI SIGBUS crashes by giving every SQLite file one SQLite
library per process", plan `plan:202610/tui_sigbus_single_sqlite_owner.md`) has all six
phases closed (sase-1kk.1 … sase-1kk.6). Its commits are:

- sase: 96e003ce12, 432dd1f759, e8ec9cb291, 73a61e6944
- sase-core: d30e1337, 6306bd2b

`sase-core-revision.txt` already pins d30e1337, which contains both sase-core commits.
The land audit confirmed the core work:

- the Rust archive store with its `(dev, ino)` quarantine guard;
- the Python adapters;
- removal of the stopgap lock;
- the static guard `tests/test_sqlite_single_owner.py`;
- the "SQLite Ownership" docs section.

Its targeted tests pass (120 passed). `sase bead epic-symbols sase-1kk` lists **no**
entries.

The audit found these gaps against the epic plan. They are the only remaining work:

1. **Missing stopgap test.** The plan's `stopgap` step 5 requires a unit test that the
   TUI faulthandler setup writes its session header and enables faulthandler, with
   `faulthandler.enable` monkeypatched. The implementation exists
   (`_enable_tui_faulthandler` in `src/sase/main/ace_handler.py`, called from
   `handle_ace_command`), but nothing under `tests/` references it.
2. **Stale docs.** `docs/rust_backend.md` still says the dismissed-bundle summary index
   is Python-owned, in two places. That contradicts the new "SQLite Ownership" table
   (`dismissed_bundles/index.sqlite` → Rust core).
3. **In-process stdlib sqlite3 on the Rust-owned artifact index in tests.** The plan's
   `py-index-readers` phase says: "Tests must not open the artifact index with stdlib
   `sqlite3` in-process." These sites remain:
   - `tests/agent_artifact_index_lifecycle/test_projection_sync.py`:
     `_write_index_schema_version` creates a `meta` table in
     `agent_artifact_index.sqlite` with `sqlite3.connect`. It is then read by
     `read_agent_artifact_index_schema_status` /
     `refresh_agent_artifact_index_if_schema_stale`, whose
     `_read_stored_index_schema_version` phase 5 rewired to Rust.
   - `tests/test_core_agent_scan_records_index.py`: three
     `with sqlite3.connect(index_path)` reads, at about lines 57, 88 and 123, of an
     index built by `rebuild_agent_artifact_index`.
   - `tests/perf/bench_agent_catalog.py`: `_write_synthetic_artifact_index` writes the
     synthetic `agent_artifact_index.sqlite` with `sqlite3.connect`. The catalog then
     reads it through Rust.
4. **Leftover cleanups from phase 5:**
   - In `src/sase/history/chat_catalog_provenance/artifacts.py`, `_indexed_records`
     wraps `load_agent_artifact_index_projection` in an outer
     `agent_artifact_index_operation_lock()`. The facade function in
     `src/sase/core/agent_scan_facade.py` already takes that same RLock, so the outer
     lock is redundant. The plan said to drop ad-hoc wrapping the facade now provides.
     Do **not** touch the bounded `try_agent_artifact_index_operation_lock(2.0)` in
     `src/sase/agents/catalog/_sources.py`. It is intentional, because it gives the UI
     thread a timeout.
   - `tests/history/test_chat_catalog_provenance.py` still has an unused
     `import sqlite3`, left over after phase 5 removed its last use. Ruff ignores F401
     project-wide, so lint does not catch it.

Follow-up triage for the epic is already finished and recorded on sase-1kk:

- duplicates sase-1ic, sase-1gz and sase-17n each got a +1;
- new task sase-1kp was filed;
- a DISCOVERED ISSUE note was added to sase-x7.

Do not create beads for those again.

The epic's reviewer decision `memory_sqlite_owner = no` stands. Do **not** edit any
memory note, including the decisions web.

## Steps

### 1. Unit-test the TUI faulthandler setup

Add tests to `tests/main/test_ace_handler.py`, which already imports
`from sase.main import ace_handler`. The file is about 330 lines, so these additions
keep it under 500. Cover at least:

- **Header and enable call.**
  - Monkeypatch `ace_handler.sase_home` to return `tmp_path`.
  - Monkeypatch `ace_handler.faulthandler.enable` with a recording fake that captures
    `file` and `all_threads`.
  - Monkeypatch `ace_handler._TUI_FAULT_LOG` to `None`.
  - Call `ace_handler._enable_tui_faulthandler()`. Assert that
    `tmp_path / "logs" / "tui_faults.log"` exists and that its text contains one header
    line with `pid=<os.getpid()>` and `sase=`.
  - Assert the fake was called once with `all_threads=True`, and that `file` is the
    object stored in `ace_handler._TUI_FAULT_LOG`.
- **Idempotence.** A second call does not call `enable` again and does not append a
  second header.
- **Failure is non-fatal (optional).** When `faulthandler.enable` raises `RuntimeError`,
  the function returns without raising and `_TUI_FAULT_LOG` stays `None`.

Never let the real `faulthandler.enable` run in the test process. Close any file object
the test opened, for example via `ace_handler._TUI_FAULT_LOG.close()` in a `finally` or
fixture teardown, before monkeypatch restores `_TUI_FAULT_LOG`.

### 2. Correct the stale ownership wording in `docs/rust_backend.md`

**About line 76.** In the "Agent cleanup planning plus deterministic cleanup mutations"
bullet, the sentence "In the current sase's TUI host path, dismissed-bundle JSON
persistence and its summary SQLite index are Python-owned." is stale. Rewrite it to say:

- dismissed-bundle JSON persistence stays on the Python host path;
- the summary SQLite index (`dismissed_bundles/index.sqlite`) is owned by
  `sase_core::agent_archive::store` and reached only through `sase_core_rs` bindings
  (see "SQLite Ownership").

**About line 157.** The bullet "Agent cleanup process signalling, dismissed-bundle
persistence, dismissed-bundle summary indexing, and TUI orchestration stay on the Python
host path. …" is also stale. Remove "dismissed-bundle summary indexing" from the
Python-host list, and add the summary index to what the Rust boundary owns.

Keep the surrounding style and line wrapping (about 88 columns). Grep `docs/` for any
other sentence that calls the dismissed summary index Python-owned, and fix it the same
way.

### 3. Stop tests from opening the Rust-owned artifact index with in-process stdlib sqlite3

**`tests/agent_artifact_index_lifecycle/test_projection_sync.py`.**

- Make `_write_index_schema_version(index, version)` build the meta-only fixture in a
  **subprocess**. Follow the pattern of `_create_legacy_index` in
  `tests/_dismissed_bundle_index_fixtures.py`: run `sys.executable -c <script>` with
  `check=True, capture_output=True, text=True`.
- The script must create `meta(key TEXT PRIMARY KEY, value TEXT NOT NULL)`, upsert
  `('schema_version', str(version))`, commit, and close.
- Drop the module's `import sqlite3` if it becomes unused.

**`tests/test_core_agent_scan_records_index.py`.**

- Replace the three in-process reads with the epic's Rust projection binding:
  `load_agent_artifact_index_projection` from `sase.core.agent_scan_facade`. Its
  signature is `(index_path, columns, *, busy_timeout_ms=250)`, and it returns
  `{"schema_version": int, "rows": [[...], ...]}` with each row in column order.
- Request `("artifact_dir", "status", "started_at")` for the two status tests and
  `("artifact_dir", "hidden")` for the hidden test. Every column is in the Rust
  allowlist in `agent_scan/index/readers.rs`.
- Select the row whose `artifact_dir` equals the expected directory, then assert the
  same values as before: `("starting", None)`, `("running", None)`, and hidden `1`.
- Rows come back as lists, so compare slices or tuples accordingly.
- Remove `import sqlite3`.

**`tests/perf/bench_agent_catalog.py`.**

- Move the body of `_write_synthetic_artifact_index` into a subprocess writer with the
  same pattern. Pass the path, the schema version, the column list, and the row tuples
  as JSON, through argv or stdin.
- The resulting file must be identical: a `meta` table holding `schema_version`, and an
  `agent_artifacts` table with exactly `catalog_sources._ARTIFACT_INDEX_COLUMNS` and the
  same rows.
- Remove `import sqlite3` if it becomes unused.
- This is a benchmark, not a collected test, so keep it import-clean. If it has a quick
  or smoke entry point, run it once to confirm the synthetic index still loads.

**Final grep.** Afterwards, this grep should show no in-process stdlib `sqlite3` access
to `agent_artifact_index.sqlite` or `dismissed_bundles/index.sqlite` in Python tests or
benchmarks:

```bash
grep -rn "sqlite3.connect" tests
```

Inspect each remaining hit. Python-owned databases (bead store, chat catalog cache, agy)
and code that runs inside subprocess scripts are fine.

### 4. Remove phase-5 leftovers

- In `src/sase/history/chat_catalog_provenance/artifacts.py`, `_indexed_records`: drop
  the outer `with agent_artifact_index_operation_lock():` block and dedent its body.
  Keep the `try/except (OSError, RuntimeError, TypeError, ValueError)` and
  schema-version check unchanged. Remove the then-unused
  `agent_artifact_index_operation_lock` import.
- Keep `import sqlite3` in that module, because it annotates the Python-owned chat
  catalog cache connection.
- In `tests/history/test_chat_catalog_provenance.py`, remove the unused
  `import sqlite3`.

### 5. Verify

- Read the `lint_and_test` reference memory with `/sase_memory_read` and follow it. Run
  `just check` through `sase tool run` as that note and the guarded-recipes rule
  require. Do **not** run `just check-full`.
- A known unrelated failure such as the TUI import-budget assertion tracked by sase-1ic
  may remain, but only if it reproduces identically on a clean base. Record that in the
  close note and do not "fix" it here.
- Also run these targeted tests directly while iterating:
  - `tests/main/test_ace_handler.py`
  - `tests/agent_artifact_index_lifecycle/`
  - `tests/test_core_agent_scan_records_index.py`
  - `tests/history/test_chat_catalog_provenance.py`
  - `tests/test_sqlite_single_owner.py`
  - the `tests/test_dismissed_bundle_index*.py` files

### 6. Close out epic sase-1kk (final step, same turn)

Do this in the same turn as the code changes. Do not wait for this work's own commit,
push, or CI.

1. Run `sase bead epic-symbols sase-1kk`. The land audit found no entries. If any
   appear, resolve each one per the Symvision epic-whitelist policy: wire it up,
   privatize it, add a non-test pragma, or delete it. Re-key a Justfile line only if a
   still-open later bead needs the exemption.
2. Close the epic normally:

   ```bash
   sase bead close sase-1kk --note "<verification>"
   ```

   Never use `--force`. The note should summarize:
   - all six phases verified against the plan and source;
   - sase commits 96e003ce12, 432dd1f759, e8ec9cb291, 73a61e6944, and sase-core commits
     d30e1337 and 6306bd2b contained in the `sase-core-revision.txt` pin;
   - the integration review of commits since the epic started, which found no conflicts.
     78e967bd3d's dismissed-index test split and 099fcee221's read-only index tests were
     preserved on the Rust backend;
   - this tale's fixes: faulthandler test, docs wording, test sqlite3 sites, redundant
     lock, and unused import;
   - the `just check` result;
   - that follow-up triage is already recorded on the epic (+1 sase-1ic, sase-1gz,
     sase-17n; new sase-1kp; DISCOVERED ISSUE on sase-x7).

3. Run `just symvision`, through `sase tool run` if it is guarded, and confirm the
   whitelist is clean.
4. Run `sase bead read sase-1kk -r "Need the PLAN path to mark the plan done"` to get
   the epic's PLAN file path. It resolves to
   `plan:202610/tui_sigbus_single_sqlite_owner.md` under the plans sidecar. Change its
   frontmatter `status: wip` to `status: done`, and change nothing else in that file.
5. Epic sase-1kk has no `parent_bead`, so the landing ends here.
