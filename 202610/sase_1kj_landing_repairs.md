---
tier: tale
size: medium
title:
  Finish the sase-1kj landing - readiness-gate repairs, read-only dismissed-index
  queries, and closeout
goal:
  "Epic sase-1kj lands complete: dismissed-index reads stay read-only after the move to
  sase-core, the startup readiness gate never loses Agents updates or shows a false axe
  STOPPED, failed ToolRun ingests are retried, the AcePage startup test is
  deterministic, and the epic is closed with its plan marked done."
proposed_by: bbugyi200.athena.sase-1kj.land
bead: sase-1kj
status: done
---

- **PARENT:**
  [202610/apollo_tui_startup.md](https://github.com/sase-org/sase--plans/blob/main/202610/apollo_tui_startup.md)
- **BEAD:**
  [sase-1kj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kj/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1kj.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1kj.land.md)
- **COMMITS:**
  - [a89eac0](https://github.com/sase-org/sase-core/commit/a89eac09c29e2f493ef1152c0d6b0f94e3cce51c)
    — fix(agent-archive): keep dismissed-index queries read-only

# Finish the sase-1kj landing

Epic `sase-1kj` ("Stop sase TUI startup starvation on loaded hosts") has all eight
phases closed. The land audit checked every phase's code against the epic plan
(`plan:202610/apollo_tui_startup.md`). It found a few epic-caused defects and one
integration regression from a commit that landed after the epic started. This tale fixes
them and then closes the epic. It has no other context, so everything you need is below.

Keep the epic's DECISION: `fallback_seconds = sec_45`. Do not change
`_STARTUP_DEFERRED_FALLBACK_SECONDS = 45.0` or make the startup maintenance chain run
anything other than one step at a time.

Before you change TUI code, read `sase memory read tui.md -r "<why>"` and follow it.
Before you finish, read `sase memory read lint_and_test.md -r "<why>"` and follow it.
Shared backend behavior belongs in the linked `sase-core` repo. Open it with
`sase repo open sase-core -r "<why>"`, work only in the path it prints, and read its
`AGENTS.md`.

## Step 0 - Preconditions

1. Sync `master` and the linked sase-core checkout.
2. The nested landing `sase-1kj.8.1` (artifact-link index in sase-core) closed its beads
   before its final repair commit landed. Its session (`sase-1kj.8.1.land`) owns that
   commit, which is still being verified when this plan is written. Do not reimplement
   it, and do not edit the files it owns:
   - `src/sase/ace/tui/relations/link_index.py`, `_native_link_index.py`, and
     `artifact_links.py`;
   - `src/sase/ace/tui/actions/link_subject.py`;
   - `src/sase/ace/tui/actions/patch/_relation_index.py`;
   - `src/sase/core/agent_scan_facade.py` and `_agent_scan_shared.py`;
   - sase-core `crates/sase_core/src/artifact_link/`.

   Step 7 checks that the commit landed. None of Steps 1-5 depend on it.

## Step 1 - Dismissed-index queries must not write (integration with sase-1kk)

Phase `sase-1kj.3` made the dismissed-bundle index reads read-only. That fixed a cache
that never hit: reads ran `_ensure_schema`, which changes `index.sqlite`'s mtime, and
that mtime is part of the signature `dismissed_bundle_identities_snapshot` caches on.
Later, epic `sase-1kk` (commit `e8ec9cb291` plus sase-core `d30e1337`) moved the index
into sase-core. That move regressed this. In
`crates/sase_core/src/agent_archive/store.rs`, both `query_summaries` and
`query_summary_identities` open the index with `open_archive_store_writable`. So every
read:

- sets `PRAGMA journal_mode = WAL` in a busy-retry loop that can wait up to 30 s;
- takes `BEGIN IMMEDIATE`;
- runs `ensure_schema_inner`;
- rewrites `schema_version`.

On an old-version or incompatible index, a read even drops the tables
(`drop_index_tables`), which the phase plan forbade. Python worked around the mtime
change by re-reading the signature after the query (`src/sase/ace/dismissed_agents.py`,
the "Rust query binding may ensure the current schema on open" comment). That workaround
is racy: a write that commits between the query and the re-read caches stale identities
under the new signature.

**sase-core:**

1. Change `query_summaries` and `query_summary_identities` to open with
   `open_archive_store_read_only`. That function already exists and `index_signature`
   uses it. A read must take no write transaction, run no DDL, and set no pragma.
   `query_bundle_paths_by_suffixes` goes through `query_summaries`, so it inherits the
   change.
2. After opening, return `Ok(None)` without modifying anything when any of these holds:
   - the stored `schema_version` is not `ARCHIVE_STORE_SCHEMA_VERSION`;
   - `schema_compatible(conn)` is false;
   - any SQLite error occurs.

   Keep all write paths (`upsert_bundle_summary`, `delete_*`,
   `set_visibility_for_suffixes`, `rebuild_index`) on `open_archive_store_writable`.
   Migration stays a write-path job.

3. Add Rust tests in `crates/sase_core/src/agent_archive/store/tests.rs`:
   - On a populated current-version index (with WAL files present), both queries return
     rows and leave `index.sqlite`'s mtime and size unchanged.
   - On an old-version or incompatible index, both queries return `None` and every table
     survives.
   - A missing index returns `None`.
4. Run `sase tool run check` in the sase-core checkout. Commit sase-core before sase
   (declare both repos in your final declaration).

**sase:**

1. Declare both repos in your final declaration. Per `docs/rust_backend.md` ("The CI
   source revision pin"), the host commits sase-core first and writes its pushed SHA
   into `sase-core-revision.txt` before the sase commit, so do not hand-edit the pin.
   Rebuild the local `sase_core_rs` extension the way `lint_and_test.md` says, so the
   Python tests exercise the new Rust queries. Never run `just install` or
   `just install-dev`, because global installs are human-only.
2. In `src/sase/ace/dismissed_agents.py` `dismissed_bundle_identities_snapshot`, delete
   the signature re-read after the query and its comment. Cache under the signature read
   before the query. If a write lands concurrently, the next call sees a new signature
   and re-queries, which is the safe direction.
3. `query_summary_identities` in `src/sase/ace/dismissed_bundle_index/_api.py` already
   checks the version before calling Rust. `query_summaries` does not. Rust now returns
   `None` there, so a Python pre-check is optional.
4. Strengthen
   `tests/test_dismissed_bundle_index_verify.py::test_dismissed_bundle_identity_snapshot_real_index_is_read_only`.
   It currently builds an empty index, so the Rust query never runs. Rebuild the index
   from at least one real hidden bundle and assert:
   - the identity query runs exactly once across two snapshots and returns the row;
   - `dismissed_bundle_index_signature()` and the `st_mtime_ns` of `index.sqlite` are
     unchanged after the snapshots and after a direct `query_summaries(...)` call.

   Add a sibling test: an index with an old `schema_version` returns `None` from both
   queries and keeps its tables.

## Step 2 - Retry ToolRun event ingest after a failed ingest (phase sase-1kj.4)

`_ingest_events_if_changed` in `src/sase/tool/liveness.py` caches the events file's
`(st_size, st_mtime_ns)` even when `ingest_event_file` returned diagnostics.
`ingest_event_file` swallows append failures (for example, the store is busy) as
diagnostics. So a failed ingest is never retried in this process while the file stays
unchanged. A dead owner's file is exactly that case, and it is what lost-run recovery
exists for.

- Store the signature only when the ingest returned no diagnostics.
- Guard the bounded `OrderedDict` with a module `threading.Lock`, since reconcile can
  run from more than one thread.
- Add tests in `tests/tool/test_wait_polling.py`:
  - an ingest that returns diagnostics is retried on the next reconcile with an
    unchanged file;
  - a clean ingest is still skipped while the file is unchanged.

## Step 3 - Do not lose Agents updates across the startup release (phase sase-1kj.6)

Phase `.6` made two changes:

- Auto-refresh ticks before `_startup_deferred_loads_released` call
  `_accept_startup_hold_surface_tokens`
  (`src/sase/ace/tui/actions/event_refresh/_auto_refresh_tokens.py`). That accepts the
  probed token for every surface, `agents` included.
- `_release_startup_deferred_loads` (`src/sase/ace/tui/actions/_startup_loads_core.py`)
  clears `_dirty_agents`, `_dirty_patches`, `_dirty_axe`, and `_dirty_notifications`.

The startup Agents load is the visible surface and is in flight during the hold, which
lasts 11-14 s on apollo and up to the 45 s fallback. Suppose a watcher event
(`_on_artifact_change` → `_dirty_agents = True` plus queued delta paths) or Agents token
drift arrives after that load read disk. The release then clears the flag, or a hold
tick accepts the newer token without reloading.
`_should_refresh("_dirty_agents", "agents")` in `_auto_refresh_surfaces.py` then stays
false. The roster can show stale agents until the next 300 s sanity refresh
(`FULL_SANITY_REFRESH_SECONDS`). Before this epic, the `_agents_loading` early return
never accepted tokens, so updates were never lost.

Required behavior:

- A watcher event or token drift for Agents that happens after the startup Agents load
  started must still cause an Agents refresh (delta or broad, through the existing
  paths) after release.
- With no changes, the first post-release tick must still not redo the startup loads.
  Keep the existing tests that pin this green.

Suggested approach (the coder may choose an equivalent one):

1. Hold ticks must not accept the `agents` token. Seed the Agents baseline from a token
   probed before the startup Agents load reads disk, and accept it when that load
   succeeds. A stat-only probe off the loop is fine. If seeding is impractical, leave
   `agents` unseeded. The cost is one post-release Agents refresh, which happens after
   visible-ready, never before it.
2. Clear the init-default `_dirty_agents` when the startup Agents load starts, before
   its disk read, instead of at release. Watcher events during the load then survive the
   release. Patches, axe, and notifications start their startup loads at release, so
   clearing their flags immediately before those loads start stays correct. Keep that
   ordering.
3. Add tests next to the existing startup tests
   (`tests/ace/tui/test_startup_stopwatch_live_update.py` and its
   `_startup_stopwatch_live_update_*.py` helpers):
   - an agents watcher event during the startup Agents load leads to an Agents refresh
     after release;
   - agents token drift accepted by a hold tick still leads to a refresh;
   - the no-change case does not reload.

## Step 4 - Neutral axe chip until the first axe load (phase sase-1kj.6)

The footer chip shows `…` while `_axe_status_known` is false
(`src/sase/ace/tui/widgets/_keybinding_status.py`). But any `set_axe_running` call flips
it, and two paths push the default `axe_running=False` before the first axe load:

- `src/sase/ace/tui/actions/axe_display/_render_dashboard.py` (around line 227, reached
  by switching to Services before the first load);
- `src/sase/ace/tui/actions/axe_display/_render_footer.py` `_update_axe_keybinding`.

So the chip can show a false STOPPED early.

- Call `footer.set_axe_running(self.axe_running)` in both places only when
  `getattr(self, "_axe_first_load_done", True)`.
- Make sure the first axe load still pushes the real state afterwards.
- Add a test (`tests/test_keybinding_footer_status.py` or
  `tests/ace/tui/test_axe_startup_host_lifecycle.py`):
  - entering Services, or calling `_update_axe_keybinding`, before the first axe load
    keeps the `…` chip;
  - after the first load the chip shows RUNNING or STOPPED.

## Step 5 - Make the AcePage real-startup boundary test deterministic (epic note #1)

`tests/test_ace_testing.py::test_ace_page_real_startup_policy_invokes_production_boundaries`
asserts that `update-check` was recorded. Since phase `.6`, update-check is step 3 of
the sequential pump-free maintenance chain (`_run_deferred_startup_maintenance_chain` in
`src/sase/ace/tui/actions/_startup_loads_maintenance.py`). It runs after the real
prompt-catalog and link-index steps. `AcePage.__aenter__` returns at
`_mount_state_loads_done`, and `__aexit__` cancels pump-free tasks, so under load the
chain is cancelled before update-check. That fails at about line 313. Delaying
`load_artifact_links_snapshot` by 1.5 s reproduces it every time.

Fix the test, not production:

- Use `async with AcePage(startup_policy="real") as page:` and wait, bounded at 30 s,
  for `"update-check"` to appear in `calls` with `await page.pause()` between checks.
  Assert with a clear message if it never does.
- Optionally stub `_schedule_link_index_refresh` with `record("link-index")` and add
  `"link-index"` to the expected set. That removes real disk I/O and checks that
  boundary too. Confirm the chain's link-index wait still completes promptly with the
  stub.

## Step 6 - Verify

1. Run `just fmt`.
2. Run the focused tests:
   - the Step 1-5 tests;
   - `tests/test_dismissed_bundle_index*.py`;
   - `tests/tool/test_wait_polling.py`;
   - `tests/ace/tui/test_startup_stopwatch_live_update.py`;
   - `tests/ace/tui/test_axe_startup_host_lifecycle.py`;
   - `tests/test_ace_testing.py`;
   - `tests/ace/tui/test_patch_relation_index.py`;
   - `tests/ace/tui/test_link_index*.py`;
   - `tests/ace/tui/test_app_import_budget.py`.
3. Run `sase tool run check` in sase. The TUI import-budget test must pass (cap 3520).
   If it fails, attribute the growth with
   `tools/tui_import_closure --diff $(git merge-base HEAD origin/master)`. Defer any
   module this epic or this tale added. Growth from unrelated commits belongs to task
   `sase-1ic`: corroborate it there, and do not raise the cap. At planning time master
   measured 3526. The extra 5 modules come from the `sase.core.agent_scan_*` modules
   that the toobig split `456291d81d` added. The pending `sase-1kj.8.1` repair lazifies
   those imports and is expected to bring the count back under the cap; do not duplicate
   that work.
4. Triage other failures as the lint/test memory says. Known unrelated failures already
   have owners:
   - `tests/instructions/test_delivery_plan.py::test_render_failure_notifies_once` and
     the Symvision KNOWNs `_RunManifest` / `_render_sections_table` belong to epic
     `sase-1ko`;
   - also `sase-1jl`, `sase-xo`, `sase-1em`, `sase-1kr`.

   Do not run `just check-full`.

## Step 7 - Close out epic sase-1kj (final step)

1. Re-fetch master. Confirm the `sase-1kj.8.1` repair has landed:
   - a sase commit after `16252682b2` that changes
     `src/sase/ace/tui/relations/link_index.py`,
     `src/sase/ace/tui/actions/link_subject.py`, and
     `src/sase/core/agent_scan_facade.py` (bounded per-ref chip cache, async cold
     queries, lazified agent-scan facade imports);
   - a sase-core commit after `d776d751` that changes
     `crates/sase_core/src/artifact_link/index/compile.rs` (legacy Python string/count
     coercion for artifact-link rows).

   If either is missing, do not close the epic. Run
   `sase bead note sase-1kj "BLOCKED: ..."` naming the missing commit and any live
   `sase-1kj.8.1.land*` agent (`sase agent list -a`). Then stop and report this in your
   final response. Otherwise, skim that commit's diff against its bead notes (bounded
   chip cache, async cold queries, native build fences, Patch build counters, deferred
   native imports) and include it in the close note.

2. Run `sase bead epic-symbols sase-1kj`. Resolve every listed entry: wire it up,
   privatize it, add a non-test pragma, or delete it. Re-key a Justfile `--epic-symbol`
   line only if a still-open later bead needs it. At planning time the list was empty.
3. Close the epic normally. Never use `--force`. Adjust the note to what you actually
   verified:

   ```bash
   sase bead close sase-1kj --note "Lander audit: all 8 phases verified against plan:202610/apollo_tui_startup.md in source and commits 82aa8a7a88, ce1940f39f, 099fcee221, 731b7b18b5, 085f5eceb7, bda93bab64, 3c69bf5a97, bad0d0ac36, 16252682b2 plus the sase-1kj.8.1 landing repair; DECISION fallback_seconds=sec_45 honored (_STARTUP_DEFERRED_FALLBACK_SECONDS=45.0); coordinator invariants intact. Integration: restored read-only dismissed-index queries in sase-core after sase-1kk moved them to Rust (open_archive_store_read_only, version/compat check returns None, no table drops), removed the racy post-query signature re-read, pin written by the host to <sha>. Repairs: ToolRun ingest skip now retries after diagnostics; startup release no longer loses Agents watcher/token updates during the startup Agents load; axe chip stays neutral until the first axe load; AcePage real-startup test waits for the sequential maintenance chain. Follow-ups triaged in note #2 (sase-1kw created; sase-1jl and sase-xo +1; others declined with reasons). Verification: <focused test counts>, sase tool run check <run id/result>, sase-core check <result>."
   ```

4. Run `just symvision` and confirm the whitelist is clean.
5. Set `status: done` in the frontmatter of the epic's plan file. Find its path with
   `sase bead read sase-1kj -r "Need the PLAN path"`; it is
   `plan:202610/apollo_tui_startup.md` under the plans repo.
6. `sase-1kj` has no `parent_bead`, so no parent bead needs handling.
