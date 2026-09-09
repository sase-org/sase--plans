---
tier: epic
title: Scale the Admin Center Updates > Plugins sub-tab to 1000+ community plugins
goal: "The Updates tab's Plugins sub-tab loads, filters, and navigates a catalog of
  1000+ community plugins without silent truncation, without a per-catalog-entry network
  storm, and with j/k key-to-paint p95 under 16 ms.

  "
phases:
  - id: bench
    title: Large-catalog bench harness and recorded baselines
    depends_on: []
    size: small
    description: "bench: add a slow-marked scale bench and large synthetic catalog
      fixture for the Updates > Plugins sub-tab, record p50/p95/max baselines at
      10/250/1000/2000 entries, and assert the fetch/enrich cost curves so later phases
      have a measuring stick.

      "
  - id: enrich
    title: Latest-version enrichment that scales with installed count, not catalog size
    depends_on:
      - bench
    size: medium
    description: "enrich: remove the quadratic installed-version lookup, scope eager
      PyPI enrichment to installed entries, add a lazy per-entry latest fetch for the
      highlighted row, bound the fetch budget, and prune the unbounded latest cache.

      "
  - id: fetch
    title: Catalog fetch past GitHub search's 1000-result cap
    depends_on:
      - bench
    size: medium
    description: "fetch: shard the topic search so results above GitHub's hard 1000-item
      search cap are still returned, surface truncation and incomplete_results as
      catalog warnings, and make the gh timeout scale with page count instead of a flat
      20 s.

      "
  - id: tui
    title: Constant-time render, filter, and navigation paths
    depends_on:
      - bench
    size: medium
    description: "tui: batch OptionList population, replace the per-keystroke linear
      scans with identity maps built during rebuild, debounce the filter through the
      existing detail debouncer, precompute filter haystacks, and bound the
      incoming-commit caches.

      "
  - id: guard
    title: Enforce the budgets and close out the epic
    depends_on:
      - bench
      - enrich
      - fetch
      - tui
    size: small
    description:
      "guard: flip the recorded baselines into enforced budgets, add the regression
      check alongside the other perf guards, verify the combined tree with check-full,
      and reconcile any feature flag opened during the epic."
proposed_by: bbugyi200.athena.075
status: done
bead_id: sase-qn
create_time: 2026-09-09 19:51:07
---

- **PROMPT:**
  [prompts/202608/plugin_catalog_scale.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/plugin_catalog_scale.md)
- **BEAD:**
  [sase-qn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-qn/README.md)

# Plan: Scale the Updates > Plugins sub-tab to 1000+ community plugins

## Problem

The Plugins sub-tab of the Admin Center's Updates tab (`PluginsBrowserPane`,
`src/sase/ace/tui/modals/plugins_browser_pane.py`) does **not** support a catalog of
1000+ community plugins. It has three independent scale ceilings, one of which is a hard
correctness bug rather than a slowness problem.

All numbers below were measured in this workspace on the current tree, not estimated.

### 1. The catalog is silently truncated at 1000 repositories (correctness)

`src/sase/plugins/github_source.py:28` defines the only registry query:

```python
_GH_SEARCH_ENDPOINT = f"search/repositories?q={GH_SEARCH_QUERY}&per_page=100"
```

fetched at line 82 with `gh api --paginate`. GitHub's REST search API returns **at most
1000 results for any query** (10 pages of 100); page 11 is a 422. So the moment the
`sase--plugin` topic passes 1000 repositories, the catalog either stops at 1000 with no
signal, or the 422 turns into `_GhCommandError` and the pane falls back to a stale
cache. `_items_from_value` (line ~150) discards the search envelope's `total_count` and
`incomplete_results`, so nothing downstream can even detect that it happened. A user
whose plugin is repo #1001 simply cannot find or install it.

The flat `GH_TIMEOUT_SECONDS = 20.0` (line 31) covers all ten sequential paginated
requests. Search is the slowest GitHub endpoint; ten pages plausibly exceed 20 s, which
turns a full catalog into a hard fetch failure.

### 2. Latest-version enrichment is quadratic _and_ issues one request per catalog entry

`enrich_with_latest` (`src/sase/plugins/latest.py`) enriches **every** catalog entry,
installed or not. Two separate problems compound:

**Quadratic CPU.** Line 218 calls `_installed_version_for_key(catalog, key)` inside the
loop over fetched misses, and that helper (line 414) rescans `catalog.entries` from the
top on every call. Measured with the network stubbed to zero latency:

| entries | `enrich_with_latest` |
| ------: | -------------------: |
|     100 |              0.008 s |
|     500 |              0.116 s |
|    1000 |              0.371 s |
|    2000 |              1.449 s |

That is a clean n² curve — pure CPU, before a single byte crosses the network.

**A per-entry network storm.** Every entry that is not installed resolves to
`source == "index"` and, on a cache miss, becomes one PyPI request. `_fetch_misses`
(line 354) runs them at `_MAX_WORKERS = 8`. At 1000 plugins that is 1000 requests eight
at a time — roughly 20–30 s at typical PyPI latency, most of it spent asking about
plugins the user has never installed and did not ask about. Worse, line 154:

```python
cached = {} if refresh and not offline else _safe_read_cache(read_cache_fn)
```

throws the whole cache away on refresh, so pressing `r` re-runs all 1000 every time. The
`latest_cache.json` this writes is never pruned, so it grows without bound.

### 3. The filter blows the 16 ms budget by ~3× at 1000 entries

End-to-end against the real pane (`AcePage` + `_open_plugins_pane`, catalog stubbed at
the `_load_plugins_catalog` seam so this measures the UI, not the load):

| entries | pane open | **filter keystroke** | j keypress | `'` jump scan | `I` mark |
| ------: | --------: | -------------------: | ---------: | ------------: | -------: |
|      10 |    192 ms |               0.6 ms |    0.09 ms |       0.01 ms |  0.14 ms |
|     250 |    321 ms |               4.8 ms |    0.19 ms |       0.07 ms |  0.13 ms |
|    1000 |    315 ms |          **46.7 ms** |    0.18 ms |       0.55 ms |  0.15 ms |
|    2000 |    315 ms |          **32.5 ms** |    0.38 ms |       0.74 ms |  0.41 ms |

Two findings worth stating plainly, because they redirect the work:

**j/k navigation is fine.** 0.18 ms at n=1000. The pane has six name-keyed linear scans
(`_option_index_for_plugin` :187, `_logical_row_for_plugin` :195, `_highlight_named`
:301, `_entry_by_name` :550, `_refresh_install_mark_row` :567) and
`_advance_install_mark_selection` (:581) is genuinely O(n²) — but they are all cheap
enough at this scale that none of them is a blocker. Pane open is flat at ~315 ms
regardless of catalog size, dominated by app startup. Do not spend the phase budget
here.

**The filter is the blocker.** `on_input_changed`
(`src/sase/ace/tui/modals/plugins_browser_controls.py:160`) does a full synchronous
rebuild on every keystroke — `_rebuild_groups()` + `_rebuild_options()` +
`_render_detail_now(force=True)` — and that lands ~3× over the 16 ms p95 key-to-paint
budget `tui_perf.md` sets for every tab. It is also the one interaction that _matters
most_ at 1000 plugins: filtering is how you find anything in a list that long. Three
contributors:

- `_matches` (`plugins_browser_rendering.py:134`) re-derives `needle` per entry and
  builds a fresh joined haystack string per entry — 1000 string joins per keystroke.
- `_rebuild_options` (line 145) adds options **one at a time** in a Python loop (line
  156). Measured against Textual 8.0.1: 7.2 ms for 1000 options via `add_option` in a
  loop vs **0.4 ms** for a single `add_options` call — an 18× penalty for no benefit.
- `_render_detail_now(force=True)` bypasses `DetailPanelDebouncer` entirely, which
  directly contradicts `sase/memory/tui_perf.md` rule 7 ("Debounce detail panels, never
  the highlight").

(The n=2000 filter number being _lower_ than n=1000 is a fixture artifact of which
prefix matches how many rows, not a real inversion. Both are far over budget; the phase
`bench` fixture should use a fixed match count across sizes so the curve reads cleanly.)

Finally, `_ensure_plugin_incoming_commits` (`plugins_browser_incoming.py:72`) writes
into `_incoming_commit_cache`, `_incoming_commit_loading`, and
`_incoming_commit_workers`, none of which are ever evicted. This is bounded by
_installed-and-updatable_ count rather than catalog size (`plugin_entry_commit_spec`
returns `None` unless `entry.update_available`), so it is the least urgent of the three,
but it still grows without limit across a session.

## Non-goals

- Rewriting the plugin catalog in Rust. See the boundary note below.
- Changing what a plugin _is_, the `sase--plugin` topic convention, or the install and
  update execution paths.
- Virtualizing the `OptionList`. Textual already renders lazily per visible line and
  caches per-option heights incrementally; the measured cost is in our own rebuild and
  scan code, not in Textual's rendering.

## Boundary note for implementers

`CLAUDE.md`'s `rust_core_backend_boundary` says shared backend and domain behavior
belongs in `../sase-core`, and the plugin catalog _is_ shared — `sase plugin list`,
`sase plugin show`, `mode_switch/repos.py`, and `updates/status.py` all consume it
alongside the TUI. It nonetheless lives in Python today under `src/sase/plugins/`.

**These phases fix existing Python in place and do not migrate it.** The changes here
are scale fixes to code that already exists on this side of the boundary, not new core
logic being reimplemented in Python. Whether `sase.plugins` should move to `sase-core`
is a real question, but it is a separate epic; do not start it here. If a phase finds
itself adding genuinely new domain behavior rather than fixing existing behavior, stop
and raise it rather than growing the epic.

## Feature flag

Phase `enrich` changes what users see: uninstalled rows will stop showing
`latest vX.Y.Z` until the row is highlighted and its version is lazily fetched. That is
user-reaching behavior whose old branch must stay reachable for rollback, so it needs a
flag. Read `sase/memory/sase_flags.md` with `/sase_memory_read` first, then create it
with `sase flag new <key>` (which also files the removal bead). Do not hand-roll a
config key. Phases `fetch`, `tui`, and `bench` are behavior-preserving and need no flag.

## Phases

### Large-catalog bench harness and recorded baselines

Measure first — `tui_perf.md` is explicit that perceived causes are usually wrong.

Build a `slow`-marked bench modeled on `tests/ace/tui/bench_tui_jk.py` (which already
establishes the p50/p95/max table shape and a `_AXE_P95_BUDGET_MS = 16.0` budget) and
the harness in `tests/perf/`. It needs:

- A synthetic catalog fixture parameterized over 10 / 250 / 1000 / 2000 entries, built
  on the existing helpers in `tests/ace/tui/_plugins_browser_pane_helpers.py`
  (`_patch_catalog`, `_patch_other_panes`, `_open_plugins_pane`, `_uv_tool`) so it stubs
  the same seams the functional tests already stub. Note that the pane fixture is
  entered via `async with AcePage() as page:`, not a `page` pytest fixture.
- Scenarios covering: initial pane open, one filter keystroke through the real
  `on_input_changed` path, 20 j presses, jump-hint allocation (`'`), and one install
  mark toggle (`I`). Drive j/k through the queued `OptionHighlighted` handler, not just
  `action_next_option` — the assignment is nearly free and the handler is where the work
  is, so measuring only the former understates it by ~5×.
- Hold the filter's _match count_ fixed across catalog sizes (e.g. a prefix that always
  matches 100 rows). The throwaway bench that produced the numbers above let match count
  vary with n, which made n=2000 read faster than n=1000.
- A separate non-TUI bench for `enrich_with_latest` (stub `fetch_fn`, so it isolates CPU
  from network) and for `fetch_catalog_payload` page count.
- Baselines recorded under `tests/perf/baselines/` following the existing convention.

Budgets are **recorded, not enforced** in this phase — phase `guard` flips them on once
the fixes land. Do not skip the 2000-entry case; the n² curve is only obvious across two
doublings.

### Latest-version enrichment that scales with installed count, not catalog size

In `src/sase/plugins/latest.py`:

1. **Kill the quadratic.** Build a `key -> installed_version` dict once before the miss
   loop and index it, instead of calling `_installed_version_for_key(catalog, key)`
   (line 218) per fetched key. Then delete `_installed_version_for_key` (line 414) if
   nothing else uses it. This alone should take n=2000 from ~1.45 s to near-linear;
   confirm against the phase `bench` numbers.
2. **Scope eager enrichment to what the UI actually needs up front.** The pane's eager
   pass exists to make update markers, `catalog.updates_available`, and the summary
   counts correct — all of which only depend on _installed_ entries. Restrict the eager
   miss set to `entry.installed.installed`; leave uninstalled entries at
   `LatestInfo.unknown()`. At a realistic 5-installed-of-1000 catalog this turns 1000
   requests into 5.
3. **Add a lazy per-entry latest fetch** so the highlighted uninstalled row still shows
   `latest vX.Y.Z`. Model it on the existing lazy incoming-commits path in
   `plugins_browser_incoming.py`: worker-backed, keyed by cache key, deduped against an
   in-flight set, repainting the detail on completion. Route it through the detail
   debouncer so a held j/k does not spawn a fetch per row.
4. **Bound the fetch.** Give the miss batch an overall deadline in addition to
   `_MAX_WORKERS` (line 48) so a slow or unreachable PyPI degrades to
   `error="unavailable"` rows instead of stalling the load worker.
5. **Stop discarding the cache on refresh.** Line 154 drops the entire cache when
   `refresh=True`. Refresh should re-fetch what it is refreshing, not everything ever
   cached. Keep the cache and force-expire the entries in scope.
6. **Prune `latest_cache.json`.** Evict entries older than a small multiple of
   `LATEST_TTL_SECONDS` on write in `src/sase/plugins/latest_cache.py`.

Keep `sase plugin list` consistent: it shares this path, so either it gets the same lazy
behavior or an explicit opt-in flag for full enrichment. Whichever you choose, say so in
`sase/memory/cli_rules.md` terms — read that note with `/sase_memory_read` before
touching CLI surface.

Put the behavior change from steps 2 and 3 behind the feature flag described above.

### Catalog fetch past GitHub search's 1000-result cap

In `src/sase/plugins/github_source.py`:

1. **Detect the cap.** Parse `total_count` and `incomplete_results` out of the search
   envelope in `_items_from_value` instead of discarding them, and thread them out of
   `fetch_catalog_payload` alongside the entries.
2. **Shard past it.** When `total_count > 1000`, split the query into disjoint ranges
   that each stay under the cap and union the results, deduping by `full_name`. `stars:`
   buckets or `pushed:` date ranges both work; prefer whichever produces stable,
   reproducible shard boundaries, since the cache key is the query string and
   `_compatible_cache` invalidates the whole cache when that string changes. Design the
   shard scheme so adding shards does not invalidate previously cached data more often
   than necessary.
3. **Surface truncation loudly.** If sharding is unavailable or still incomplete, the
   resulting `PluginCatalog.warnings` must say so — `_summary_hint()` in
   `plugins_browser_status.py:205` already renders `catalog.warnings[0]` under the
   counts line, so a warning there reaches the user with no new UI.
4. **Fix the timeout.** `GH_TIMEOUT_SECONDS = 20.0` (line 31) is a whole-command budget
   spanning every paginated page. Either scale it with expected page count or drop
   `--paginate` in favor of explicit per-page requests with a per-request timeout.
   Explicit paging is the better shape here anyway, since sharding needs per-request
   control and it lets a partial result degrade gracefully instead of failing the whole
   fetch.
5. Verify the cache envelope still round-trips: `write_cache` in
   `src/sase/plugins/cache.py` serializes with `indent=2`, which at 1000+ entries is a
   multi-MB file rewritten on every fetch. Drop the indent for the cache (it is
   machine-read) unless something depends on it.

### Constant-time render, filter, and navigation paths

The measured target is the filter keystroke: **46.7 ms → under 16 ms at n=1000**. Steps
1–3 are the fix; steps 4–5 are hygiene worth doing while in the file, but do not let
them crowd out the filter work.

In `src/sase/ace/tui/modals/plugins_browser_rendering.py` unless noted:

1. **Debounce the filter and stop forcing the detail.** `on_input_changed`
   (`plugins_browser_controls.py:160`) must not do a synchronous full rebuild plus
   `_render_detail_now(force=True)` per keystroke. Route the detail through the existing
   `DetailPanelDebouncer` (`src/sase/ace/tui/util/debounce.py`, 150 ms) per
   `tui_perf.md` rule 7, and debounce or coalesce the list rebuild itself. Respect the
   `NavigationGate` activity gate (rule 13) rather than inventing a new one. This is the
   single highest-leverage change in the phase.
2. **Batch the option population.** Replace the `add_option` loop at line 156 with a
   single `option_list.add_options(self._create_options())`. Measured 7.2 ms → 0.4 ms at
   n=1000.
3. **Precompute filter haystacks.** `_matches` (134) hoists nothing: derive `needle`
   once per filter pass, and cache each entry's casefolded haystack when the catalog
   loads rather than rejoining it per entry per keystroke.
4. **Build identity maps once per rebuild** (hygiene — measured at 0.18 ms/keypress, so
   this is about keeping the curve flat past 2000, not about a present-day stall).
   During `_rebuild_options` (line 145), populate `name -> option_index`,
   `name -> logical_row`, and `name -> entry` dicts, and rewrite
   `_option_index_for_plugin` (187), `_logical_row_for_plugin` (195), `_highlight_named`
   (301), `_entry_by_name` (550), `_refresh_install_mark_row` (567), and the O(n²)
   `_advance_install_mark_selection` (581) against them. Textual's
   `OptionList.get_option_index(option_id)` is already an O(1) dict lookup and is the
   natural fit for the index map. Invalidate the maps wherever the row set changes.
5. **Bound the incoming-commit caches.** Give `_incoming_commit_cache` and
   `_incoming_commit_workers` in `plugins_browser_pane.py` an LRU bound so a long
   session cannot grow them without limit. Note this is already bounded by
   installed-and-updatable count rather than catalog size, so it is a leak fix, not a
   scale fix.

Observe `tui_perf.md` rule 12 throughout: the pane's `ProgrammaticSelectionGuard`
interplay with `OptionHighlighted` echoes is subtle and already correct — do not disturb
it while swapping in the maps. Re-run the existing
`tests/ace/tui/test_plugins_browser_pane*.py` suite, which covers selection restoration,
jump hints, and marking behavior in detail.

### Enforce the budgets and close out the epic

1. Flip the phase `bench` baselines from recorded to enforced: filter keystroke and j/k
   p95 both < 16 ms at n=2000 (the filter is the one that starts out failing, at 46.7
   ms), `enrich_with_latest` demonstrably sub-quadratic, and eager network calls
   O(installed) rather than O(catalog).
2. Add the regression check next to the existing ones in `tests/perf/`
   (`check_view_hints_regression.py` and `test_view_hints_regression.py` are the closest
   model) so the budgets are checked, not just printable.
3. Add a test asserting the catalog fetch surfaces truncation rather than silently
   dropping repositories — that is the correctness bug in this epic, and it needs a test
   that fails loudly if it regresses.
4. Reconcile the feature flag opened in phase `enrich`: either flip it on by default
   with its removal bead updated, or record why it stays off.
5. Verify the combined tree with `just check-full` through `/sase_monitor` (it routinely
   outruns a single agent turn — never run it inline), handing a `--next` action so the
   follow-up agent acts on the result.

## Verification

- `just check` after each phase; `just check-full` via `/sase_monitor` on the combined
  tree before landing.
- `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` and the new scale bench, compared
  against the phase `bench` baselines.
- Manual smoke with a seeded 1000+ entry catalog cache: open the Updates tab, switch to
  Plugins, type a filter, hold `j`, press `'`, mark several rows with `I`, and press
  `r`. Nothing should stall; `SASE_TUI_PERF=1` p95 stays under 16 ms.
- `docs/perf_runbook.md` and `tests/perf/README.md` hold the capture/compare recipes.

## Risks

- **Shard boundaries churn the cache.** `_compatible_cache` keys on the exact query
  string, so a shard scheme that changes its boundaries as the catalog grows will
  invalidate the whole cache repeatedly. Pick stable boundaries.
- **Lazy enrichment changes what users see.** Hence the flag. Watch for surfaces that
  silently assumed every entry was enriched — `updates/status.py` and
  `render_catalog.py` are the ones to check.
- **Selection-restoration regressions.** The maps touch the same code that keeps the
  cursor on the right plugin across rebuilds. The existing pane tests are thorough;
  trust them, and add cases rather than relaxing them.
