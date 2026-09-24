---
tier: tale
title: Admin Center Updates tab opens from cache and refreshes only on r
goal:
  Opening or re-opening the Admin Center Updates tab never triggers a network refresh;
  it paints instantly from an in-memory session inventory or a network-free rebuild from
  the periodic update check's caches, and only the r keymap refreshes from the network.
size: medium
proposed_by: bbugyi200.athena.0qw
create_time: 2026-09-24 11:56:44
status: wip
---

# Plan: Admin Center Updates tab opens from cache, refreshes only on `r`

## Goal

Opening the SASE Admin Center **Updates** tab must never trigger a network refresh. The
tab shows cached data, taken from the caches the periodic automatic update check already
keeps warm. The only way to refresh from the network is the existing `r` binding
(`action_refresh`), plus the reloads that already follow a user-started mutation. Going
back and forth to the tab should be instant.

## Background (current behavior and root cause)

- `ConfigCenterModal` caches panes only per modal instance (`self._panes`). Switching
  tabs inside one open Admin Center reuses the pane. But `_open_config_center()`
  (`src/sase/ace/tui/actions/base.py`) pushes a **new** modal every time, so each close
  and reopen builds a new `PluginsBrowserPane` from scratch.
- `PluginsBrowserLayoutMixin.on_mount` (`plugins_browser_layout.py`) always calls
  `self._start_load(force=False)`. That runs `load_plugins_catalog_for_pane()`
  (`plugins_browser_loading.py`). Even with `refresh=False` it hits the network on every
  open:
  - core packages: `enrich_core_versions_latest(fetch_fn=fetch_latest_version)` makes
    **uncached** PyPI calls. Editable core packages run
    `detect_dev_latest(offline=False)`, which does a `git fetch` per checkout.
  - plugins: `enrich_with_latest()` refetches stale entries, and editable plugins
    `git fetch`.
  - agent CLIs: runs the `--version` probes and npm/JSON latest lookups whenever the
    cache is stale.
  - core incoming commits: GitHub (`gh`) lookups for index installs.
  - it builds, merges, and writes a new update-status snapshot.
- The periodic job (`UpdateToastMixin` in `src/sase/ace/tui/actions/update_toast.py`)
  keeps these local caches warm. The caches are: the update-status snapshot
  (`sase.updates.cache`), the plugin catalog cache, the shared latest-version cache
  (`sase.plugins.latest_cache`; `_make_cached_core_fetch_fn` in `sase/updates/status.py`
  also writes core entries there), the agent-CLI latest cache, and each editable
  checkout's remote-tracking refs, which its recompute fetches. The composite
  `UpdateStatus` snapshot lists only _outdated_ components. It cannot render the full
  inventory alone, so the pane has to rebuild its inventory from those lower-level
  caches.
- Per-pane lazy state is also lost on every reopen: the plugin incoming-commit LRU
  (`_incoming_commit_cache`) and lazily fetched plugin latest versions
  (`_apply_plugin_latest`). Re-highlighting a row fetches them from the network again.

## Design decisions

1. **Two load kinds for opening the tab.**
   - A **cache-only** load. It is network-free: no PyPI/GitHub/npm requests, no
     `git fetch`, no cache or snapshot writes. It is used when the tab opens and no
     usable in-memory inventory exists.
   - The existing **network** load. `r` uses it with `refresh=True`. Post-mutation
     reloads and the `o` offline toggle keep today's `force=False` network/offline
     behavior (unchanged scope).
2. **A session memo lets reopens paint instantly.** The last successfully applied
   online-mode inventory (a `PluginsLoadResult` with rows) lives in
   `UpdatesSessionState`. That object is already per-ACE-process and outlives Admin
   Center reopens (`app._admin_center_session_state`). A reopen hydrates synchronously
   from it and starts **no** worker at all.
3. **The memo defers to newer periodic data.** Each memo records the `checked_at` of the
   network evidence behind it. On open, if the app's in-memory
   `_automatic_update_status.checked_at` is newer, the periodic job has published newer
   results since then. The open then does a cache-only load, which is still
   network-free, instead of reusing the memo. This check reads memory only (no disk I/O
   on the event loop). While the tab is open, nothing reloads by itself.
4. **Local mutations invalidate the memo.** Every update, install, or uninstall
   completion path clears the memo, whether or not the pane is still mounted. The next
   open then reflects the new installed state. Successful code updates restart ACE
   anyway.
5. **`r` means "refresh everything".** It runs the existing forced network load. It also
   clears the session-shared incoming-commit cache. Network loads write the core latest
   versions through to the shared latest-version cache, so later cache-only opens see
   what `r` saw.
6. **The header states how old the data is.** `checked … ago` shows the age of the last
   network check behind the displayed data: the load time for network loads, and the
   update-status snapshot's `checked_at` for cache-only loads. It is measured against
   the current time when a memo is restored.
7. **No feature flag.** This is the requested final behavior. It is not a beta, and no
   old branch has to stay reachable (see the `sase_flags` memory).
8. **No sase-core change.** All update and cache logic touched here already lives in
   Python (`sase.updates`, `sase.plugins.latest`, `sase.agent_clis.latest`,
   `sase.uv_tool.versions`, `sase.dev_update.detect`). This plan extends those modules
   in place. No Rust/binding work and no `sase-core-revision.txt` bump.
9. **No keymap or config changes.** `r` stays as it is (hint text `r reload` unchanged),
   so `src/sase/default_config.yml` does not change. There are no new config keys and no
   new CLI options.

## Implementation steps

### 1. Cache-only primitives in the shared update helpers

Every cache-only mode below has the same meaning: _never touch the network or write a
cache. Use any cached evidence regardless of TTL. A cache miss leaves the item
"unchecked" instead of reporting an error or "offline"._ Pass the new keyword to
injected test doubles only when it is enabled, so existing doubles keep working.

a. `src/sase/dev_update/detect.py::detect_dev_latest(record, *, offline, fetch=True)`.
With `fetch=False` and `offline=False`, skip `fetch_git_upstream(...)` and classify the
existing remote-tracking ref through the normal branches
(`dirty`/`diverged`/`update_available`/`current`), never `offline` or `fetch_failed`.
Add `fetch: bool = True` to the `_DetectDevLatestFn` protocols in
`sase/uv_tool/versions.py` and `sase/plugins/latest.py`. b.
`src/sase/uv_tool/versions.py::enrich_core_versions_latest`. Add a cache-only mode; the
suggested seam is `cache_only: bool = False` plus an injected
`cached_latest_fn: Callable[[str], <object with .version> | None]`. Keep the module
independent of the plugin modules by using a duck-typed protocol, not an import of
`CachedLatest`. In cache-only mode `fetch_fn` is never called.

- Index packages: a cache hit sets `latest_checked=True`, `latest_version`,
  `update_available` via `is_newer`, and `latest_error=None`, or `"unavailable"` when
  the cached version is `None`. A miss leaves the package unchecked and only fills in
  `install_type`.
- Editable packages: call the detector with `fetch=False`. c.
  `src/sase/plugins/latest.py::enrich_with_latest(..., cache_only: bool = False)`. Never
  call `_fetch_misses` and never write the cache.
- Eager index entries: use any cached entry, fresh or stale.
- Misses: stay `LatestInfo.unknown()`, so the existing lazy highlighted-row fetch
  (`_ensure_plugin_latest`) can fill them on demand.
- Editable entries: detect with `fetch=False`.
- `git` entries: unchanged. d.
  `src/sase/agent_clis/latest.py::get_latest_versions(..., cache_only: bool = False)`.
  Never fetch or write. Any cached item becomes
  `LatestVersion(item.version,    cached=True)` with no error, fresh or stale. A miss
  becomes `LatestVersion(None)` with no error.
  `src/sase/agent_clis/operations.py::collect_agent_cli_statuses(...,    cache_only=False)`
  forwards the flag to `latest_fn`. Local `--version` detection is unchanged. e.
  `src/sase/updates/status.py`: public helpers over the latest-version cache that the
  periodic core check already uses. Export them from `sase/updates/__init__.py`.
- A cache-only core lookup, e.g. `make_cached_core_latest_lookup()`. It reads the cache
  lazily once, returns the cached entry for a distribution regardless of age, and makes
  no network calls.
- A write-through core fetcher, e.g. `make_core_latest_fetch_fn(now, *, force)`.
  `force=True` always fetches from PyPI and writes the result through. `force=False`
  keeps today's `_make_cached_core_fetch_fn` behavior; keep that private name as a thin
  wrapper so existing patches and tests keep working.

### 2. Cache-only mode in the pane loader

`src/sase/ace/tui/modals/plugins_browser_loading.py`:

- `PluginsLoadResult` gains `checked_at: float | None = None` and
  `cache_only: bool = False`.
- `load_plugins_catalog_for_pane(..., cache_only: bool = False)`. `cache_only` is used
  only with `refresh=False, offline=False`; `offline=True` keeps today's semantics. In
  cache-only mode:
  - Unchanged local work: the uv probe, install mode, dev root, and agent-CLI history.
  - Core: `collect_installed_core_versions()` plus the step 1b cache-only enrichment,
    using the step 1e lookup.
  - Agent CLIs: `collect_agent_cli_statuses(cache_only=True)`.
  - Core incoming commits: only specs with `source == "git"` (local `rev-list`/`log`
    against the existing upstream ref). Skip GitHub-sourced specs entirely. A missing
    key already makes `_core_incoming_section` omit the section.
  - Catalog: `load_plugin_catalog(refresh=False, offline=True)` (catalog cache only). If
    `PluginCatalogError` reports that no compatible cache exists, set `error` to a pane
    message such as `No cached plugin catalog yet — press r to refresh.`
  - Plugin latest: `enrich_with_latest(catalog, cache_only=True)`.
  - Do **not** build, merge, or write the update-status snapshot (`update_status=None`).
    Return `fresh_editable_roots=frozenset()` and `cache_only=True`. Set `checked_at` to
    `read_update_status_snapshot().checked_at` when readable, else `None`. Route the
    read through the module's existing `_read_update_status_snapshot` seam.
- Network loads (`cache_only=False`, `offline=False`): set `checked_at=load_now` and
  collect core latest versions with `make_core_latest_fetch_fn(load_now, force=True)`.
  This keeps today's always-live PyPI lookup and adds write-through.

### 3. Session memo state

`src/sase/ace/tui/modals/config_center_session.py::UpdatesSessionState`. Use
`TYPE_CHECKING`-only imports; this module is on the startup path.

- `inventory: PluginsLoadResult | None = None`: the last successfully applied
  online-mode inventory, rows included.
- `incoming_commit_cache: OrderedDict[IncomingCommitsCacheKey, IncomingCommits]`
  (`default_factory=OrderedDict`), shared across reopens. It stays bounded by the
  existing `_put_incoming_commit_cache` LRU cap.
- `invalidate_inventory()`: clears `inventory`.

### 4. Pane open, apply, and refresh paths

- `PluginsBrowserPane.__init__` (`plugins_browser_pane.py`):
  - Alias `self._incoming_commit_cache = self._session_state.incoming_commit_cache`.
  - When the session already holds an `inventory`, seed `_core_versions` from it instead
    of the synchronous `_collect_installed_core_versions()` probe. That avoids disk I/O
    on the event loop at construction.
  - Add `_checked_at: float | None = None` and `_cache_only = False`.
- Factor the `on_worker_state_changed` load-SUCCESS body (`plugins_browser_workers.py`)
  into `_apply_load_result(result, *, restored: bool = False)`. Both the worker and the
  memo path share this one apply path.
  - Always: assign every field as today, plus `_checked_at` and `_cache_only`; then
    `_rows`, `_rows_by_key`, and `_render_all()`. `_restore_key` keeps selection restore
    working.
  - `restored=True`:
    - Set `_now` from the current wall clock through a pane-module clock seam, so ages
      are measured against now.
    - Never push `update_status` to the updates indicator.
    - Never establish `_fresh_editable_roots_evidence` (set it to `None`).
  - `restored=False`:
    - Keep today's behavior, except that fresh-editable evidence also requires
      `not result.cache_only`.
    - After applying, remember the result when `not self._offline`
      (`self._session_state.inventory = result`). Offline-mode results are never
      remembered.
  - The ERROR branch leaves the memo untouched.
- `on_mount` (`plugins_browser_layout.py`): when `self._auto_load` is set, replace the
  unconditional `_start_load(force=False)`:
  - If the memo is usable, call `_apply_load_result(memo, restored=True)`. No worker
    starts.
  - Otherwise call `_start_load(force=False, cache_only=True)`.
  - "Usable" is a small pure helper that is easy to unit test: a memo exists, and either
    `getattr(self.app, "_automatic_update_status", None)` is `None`, or the memo has a
    `checked_at` and `status.checked_at <= memo.checked_at`.
  - Keep `_loading` consistent so the first paint shows the restored rows, not
    `Loading updates…`.
- `_start_load(*, force: bool, cache_only: bool = False)` forwards `cache_only` to the
  loader. Every other call site stays as it is.
- `_apply_plugin_latest` (`plugins_browser_latest.py`): after patching `_catalog` and
  `_rows`, patch the remembered inventory too
  (`dataclasses.replace(..., catalog=..., rows=...)`) when online. Lazily fetched latest
  versions then survive a reopen.
- `action_refresh` (`plugins_browser_controls.py`): clear the shared incoming-commit
  cache, then `_start_load(force=True)` as today.

### 5. Invalidate the memo on local mutations

- Pane completion handlers call `self._session_state.invalidate_inventory()` **before**
  their `is_mounted` or `_loading` checks, so invalidation also happens after the Admin
  Center was closed mid-proc. The handlers are:
  - `_handle_code_update_completion` (`plugins_browser_sase_update_procs.py`, shared by
    plugin update and uninstall);
  - the agent-CLI update completion (`plugins_browser_agent_clis_actions.py`);
  - the agent-CLI install completion (`plugins_browser_agent_clis_install.py`);
  - the combined install completion (`plugins_browser_install_combined.py`).

  Grep for any other completion that reloads via `_start_load(force=False)` or restarts
  after a mutation, and cover it too.

- App level: `_on_scoped_update_complete` in `src/sase/ace/tui/actions/update_run.py`
  (the `,U` / `,E` / global-shortcut path) invalidates
  `self._admin_center_session_state.updates` (getattr-guarded).

### 6. Header freshness wording

`plugins_browser_status.py`:

- `_cache_age_label()`: prefer `humanize_age(self._now - self._checked_at)` when
  `_checked_at` is known. Otherwise fall back to today's catalog age. The all-current
  banner's `Last checked … · press r to re-check` uses the same label.
- `_row_source_failure_message()`: when `_cache_only`, append ` — press r to check` to
  the `latest version unknown …` message.

With the default `checked_at=None` and `cache_only=False`, existing stubbed results
render as they do today, so visual snapshots should not move. If any do, refresh them
following the `tui_screenshot` memory.

### 7. Docs

- `docs/ace.md` → `## Updates Tab`, and `docs/configuration.md` → `### Updates tab`: add
  a short paragraph covering:
  - Opening the tab never refreshes from the network.
  - The first open in an ACE session builds the inventory from local caches the
    automatic update check maintains (plugin catalog, latest-version caches, editable
    checkouts' last-fetched upstream refs), without PyPI/GitHub/npm or `git fetch`.
  - Later opens reuse the in-memory inventory instantly. When the automatic check has
    published newer results, the open re-reads those caches instead.
  - `r` refreshes everything from the network.
  - `checked … ago` is the age of the last network check.

## Tests

Keep the existing pane-module monkeypatch seams (`_load_plugins_catalog`, etc.). New
loader kwargs must stay compatible with `lambda **_kw:` stubs.

- `tests/dev_update/test_detect.py`: `fetch=False` never calls `fetch_git_upstream`, and
  a behind checkout still reports `update_available`. `offline=True` still reports
  `offline`.
- `tests/uv_tool/test_versions.py`: cache-only never calls `fetch_fn`. A stale hit is
  used, including its update flag. A miss stays unchecked. Editable packages detect with
  `fetch=False`.
- `tests/test_plugin_latest.py`: cache-only makes no fetches and no cache writes, uses
  stale hits, leaves misses unknown, and detects editable entries with `fetch=False`.
- `tests/agent_clis/test_latest.py` and `test_operations.py`: cache-only semantics, and
  the flag is forwarded.
- Update-status helper tests (next to the existing `sase.updates` tests /
  `tests/_update_status_helpers.py`): the forced core fetcher writes through, and the
  cached lookup ignores TTL and never fetches.
- Loader tests (extend `tests/ace/tui/test_plugins_browser_loading_freshness.py` or add
  `test_plugins_browser_loading_cache_only.py`). In cache-only mode:
  - every network seam raises if called;
  - no snapshot write happens;
  - `update_status` is `None`;
  - fresh roots are empty;
  - `checked_at` comes from the snapshot;
  - GitHub core incoming specs are skipped;
  - a missing catalog cache produces the "press r" error.
- Pane tests, in a new `tests/ace/tui/test_plugins_browser_pane_cached_open.py` that
  reuses `_plugins_browser_pane_helpers.py`:
  1. The first open calls the loader once with `cache_only=True`, `refresh=False`.
  2. Reopening the Admin Center with the same `AdminCenterSessionState` does not call
     the loader, and rows plus the selected row are restored immediately.
  3. A reopen after the app's `_automatic_update_status.checked_at` becomes newer than
     the memo calls the loader with `cache_only=True`.
  4. `r` calls the loader with `refresh=True`, not cache-only. It clears the shared
     incoming-commit cache and updates the memo.
  5. An offline-toggle load does not overwrite the memo.
  6. A mutation completion that fires after the pane is unmounted invalidates the memo,
     so the next open loads.
  7. A lazily fetched plugin latest version survives a reopen.
  8. A restored memo neither pushes to the updates indicator nor establishes
     fresh-editable evidence.
  9. The header age label uses `checked_at` when present.
- Existing tests that assert mount-load kwargs (for example `_patch_catalog_recording`
  users), or that reopen the Admin Center and count loader calls, must be updated for
  the new cache-only and memo behavior.

## Verification

- Read the `lint_and_test` SASE memory and run the verification it prescribes
  (`just check` through the guarded-recipe flow). Fix any lint, type, or Symvision
  findings. New public helpers must be used, or exported and whitelisted, per the
  `symvision` memory.
- Optional live check (see the `tui` memories): open ACE, press `#` then `7`. Confirm
  the first open shows data without network activity. Close and reopen: it paints
  instantly with no `Loading updates…`. `r` shows the loading header and refreshes.

## Out of scope

- The global Update panel (`,U`), the periodic check cadence, and the `o` offline
  toggle.
- The network behavior of post-mutation reloads (they stay `force=False` network loads).
- Auto-updating an already-open tab when the periodic check lands.
- Moving update-status logic into sase-core.
