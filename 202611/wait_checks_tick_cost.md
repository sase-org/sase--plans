---
tier: tale
title: Cut wait_checks per-tick work so host CPU spikes stop killing it
goal:
  A wait_checks tick does far less per-artifact work with identical results, so a
  saturated host no longer pushes it past its 2-minute job timeout.
size: medium
decisions:
  job_timeout:
    ask: Should agent_waits' job_timeout also be raised from 2m to 3m as extra headroom?
    choices:
      keep:
        Cost cuts alone restore headroom; a hung tick is still flagged after two minutes
      raise:
        Ticks on a saturated host finish instead of restarting, but a real hang surfaces
        later
    default: keep
    why:
      the measured cuts fit the observed overload in 2m; a longer timeout hides
      regressions
    answer: raise
proposed_by: bbugyi200.apollo.e3
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-15 06:47:56
status: wip
---

# Cut wait_checks per-tick work so host CPU spikes stop killing it

## Problem

The axe error digest reports `agent_waits` / `wait_checks` killed with
`timed out after 120s` (2026-11-15 06:00:43 EST, run `20261115T055842_910903`, exit -9).
Its progress lines show the marker walk took 21.4 s and the scoped resolving view 52.0 s
(21 rows, 3 scoped waiters, 0 full-view waiters). The per-waiter resolve loop was still
running when the job was killed.

### Root cause

**Trigger: the host was saturated.** At 06:00 EST, sysstat on apollo shows run queue 287
and a 1-minute load of 168 on 16 CPUs. Every lumberjack overran in the same minute:
`sidecar_sync` took 108 s, `waits` 89 s, `usage` 78 s, and `comments` 72 s.

**Why wait_checks is the job that dies: it has the least headroom.**

- A normal tick takes about 7 s of wall time: about 1.9 s of interpreter import plus
  about 4.8 s of in-process work. That matches the `Tick overrun: took 7.6s` logged just
  before the failure.
- At roughly 18x CPU oversubscription, 7 s becomes more than 120 s.
- The agent_waits log holds 7 such timeouts in its last 3 days (11-12, five on 11-14,
  and 11-15).
- Over the last 3,711 ticks, `view_ms` was p50 1.3 s, p90 6.3 s, p99 10 s, and max 51.5
  s.

**Most in-process cost scales with every artifact ever run, not with the waiters.**
Today there are 7,327 `ace-run` directories across 28 projects, up from 4,908 on
2026-10-08, while the scoped view returns about 21 rows. A cProfile of one tick on
apollo's real corpus, with all writes stubbed out, shows three per-artifact costs:

1. **`artifact_dir_key` resolves every listed directory.** Each call runs
   `Path(value).expanduser().resolve()`, which is a realpath (one `lstat` per path
   component, about 9 per directory) plus a trailing `stat` and heavy pathlib parsing.
   - This costs 1.5 s of the 2.5 s scoped view.
   - The existing `lru_cache` (commit 3b338c208a) only helps within one process, and
     every tick is a fresh subprocess.
   - None of the 7,327 artifact directories or their 155 distinct parents is a symlink.
2. **The "seed negatives" loop in `_run` saves nothing.** It runs `meta_path.exists()`
   on every walked directory, about 0.2–0.3 s.
   - The scoped selection never consults `meta_cache` for unselected directories: it
     stats `agent_meta.json` itself and drops a missing one without reading it.
   - The full-view and confirmation path already stores the `None` that `read_json_dict`
     returns for a missing meta.
   - So the seed spends 7,327 stats to spare about 15 failed `open`s that would each
     happen at most once anyway.
3. **The scoped resolving view lists every project's `ace-run` directories a second
   time**, right after the marker walk listed them. It rebuilds `Path` objects and
   re-hashes them into `seen_dirs`, about 0.4 s including hashing.

**A prototype confirms the main fix.** It resolves each distinct parent directory once
(155 of them), then runs one `lstat` on each leaf. It produced byte-identical keys for
all 7,327 directories and cut key time from 0.76 s to 0.09 s.

### Related work and scope

- Epic `sase-1jy` (landed) introduced the scoped view and the persistent scope-key
  cache.
- Open bead `sase-1hn` covers incremental resolution and scheduler event-wake.
- This plan only makes the existing scoped design cheaper per artifact. It does not
  change which rows are selected, when a waiter is released, the confirmation pass, or
  the fail-closed invariants.

## Goals

- Cut the per-artifact Python work of a `wait_checks` tick so a normal tick on apollo's
  current corpus takes clearly less than half of today's in-process time. Targets:
  scoped view about 2.5 s down to 1 s or less, walk about 0.9 s down to 0.5 s or less.
- Produce identical results:
  - the same keys
  - the same selected rows
  - the same scope-cache contents
  - the same release decisions

## Non-goals

- Interpreter import cost (about 1.7–1.9 s per tick): broad module-graph decoupling,
  captured as a follow-up bead in step 7.
- Porting the walk or the selection to `sase-core` Rust. This is a constant-factor
  optimization of existing Python code with no new shared behavior.
- The resolve-phase bead-store and project-alias lookups (about 0.5–0.8 s; they scale
  with project count, not artifact count).
- Changing scope-key semantics, the cache file schema, or the confirmation pass's fresh
  listing.

## Implementation

### 1. Resolve a normalized absolute artifact path through its memoized parent

File: `src/sase/core/wait_dependency_resolution/_artifact_state.py`

- Keep the public `artifact_dir_key` (`lru_cache(maxsize=65536)`) →
  `_artifact_dir_key_uncached(value)` structure.
  `tests/test_wait_dependency_index_caches.py` counts `_artifact_dir_key_uncached` calls
  and asserts at most one per distinct leaf. So parent resolution must **not** route
  through `_artifact_dir_key_uncached` or `artifact_dir_key`.
- Add a separate `@lru_cache(maxsize=4096) _resolved_parent_dir(head: str) -> str`. It
  returns `str(Path(head).resolve(strict=False))` and falls back to `head` on
  `(OSError, RuntimeError, ValueError)`, mirroring today's fallback.
- In `_artifact_dir_key_uncached`, add a fast path. It applies when `value` is absolute,
  does not start with `//`, and `os.path.normpath(value) == value` (so no trailing
  slash, no `.` or `..`, no doubled separators). Then:
  1. Split it into `head` and `name` with `os.path.split`. When `name` is empty, use the
     existing slow path.
  2. `os.lstat(value)`.
  3. If `lstat` raises `OSError` (missing leaf, non-strict semantics) or the mode is not
     a symlink, return `os.path.join(_resolved_parent_dir(head), name)`.
- Any other input uses today's `Path(value).expanduser().resolve(strict=False)` path
  unchanged, including its exception fallback. That covers a symlinked leaf, a relative
  path, a `~` path, and any non-normalized path.
- Docstring: state the equivalence (POSIX
  `realpath(parent/name) == realpath(parent)/name` whenever `name` is not a symlink,
  including a missing leaf). Note that parent memoization has the same staleness class
  as the existing per-value LRU in long-lived processes.

### 2. Remove the redundant seed-negatives loop

File: `src/sase/scripts/_chop_wait_checks_run.py`

- Delete the `for artifact_dir in scan.walked_dirs: ... meta_path.exists()` block and
  its comment. Explain the reason in the remaining comment near the `meta_cache`
  declaration:
  - scoped selection stats metas itself and never reads unselected ones;
  - full-view reads cache the `None` returned for a missing meta, so a missing meta
    costs at most one failed `open` per tick.

### 3. Build the resolving view from the marker walk's listing

Files: `src/sase/axe/wait_marker_scan.py`,
`src/sase/scripts/_chop_wait_checks_run_fs.py`,
`src/sase/scripts/_chop_wait_checks_run.py`

- `_WaitMarkerScan`: add `dirs_by_project: dict[str, list[Path]]`, keyed by the walked
  project directory name and kept in listing order. Keep `walked_dirs`, which
  `sidecar_auto_sync` and tests use, filled with the same `Path` instances.
- `filesystem_dependency_rows`: add a keyword-only
  `listed_dirs: Mapping[str, Sequence[Path]] | None = None`.
  - When it is given, both the scoped and the full branch iterate these projects and
    directories (still honoring `project_names`) instead of `projects_dir.iterdir()`
    plus `iter_agent_artifact_dirs`.
  - Update the docstring: the listing is fresh unless the caller passes this tick's walk
    listing, and confirmation passes never pass it.
- `_run`: pass `listed_dirs=scan.dirs_by_project` to the initial scoped view and to the
  initial full view.
- `fresh_index` in `src/sase/scripts/_chop_wait_checks_run_waiters.py` must keep calling
  without `listed_dirs`, so confirmation still re-lists fresh. That fresh listing is the
  new-member signal, and
  `test_stale_resolving_view_defers_when_newer_failed_run_appears` must still pass
  unchanged.
- The scoped call still adds every listed directory to `seen_dirs`, so `drop_missing`
  semantics do not change.

### 4. Trim per-artifact pathlib overhead in the hot loops

- `scan_waiting_markers`: test for `waiting.json` with
  `os.path.exists(os.path.join(dir_str, "waiting.json"))`. Build `WaitingMarker` `Path`s
  only for directories that have a marker.
- `stat_agent_meta` (`_scope_cache.py`): call `os.stat` on a joined string instead of
  `Path(artifact_dir) / "agent_meta.json"`.
- `select_dirs_with_cache`: compute `artifact_dir_key(str(artifact_dir))` once per
  directory and reuse it for:
  - the `included` check,
  - `ScopeKeyCache.lookup`, `store`, and `drop`,
  - `keys_for_artifact_dir`.

  Do this through optional precomputed-key parameters or private key-taking helpers.
  Keep the public behavior and the cache file contents identical.

### 5. Tests

- **New `tests/test_artifact_dir_key_fast_path.py`:**
  - On a tmp tree, check an equivalence table against
    `str(Path(v).expanduser().resolve(strict=False))` for these inputs:
    - plain directory
    - missing leaf
    - symlinked leaf (fallback; resolves to the target)
    - leaf under a symlinked parent directory
    - relative path
    - `~`-prefixed path (monkeypatch `HOME`)
    - trailing slash
    - `..` segment
    - `//` prefix
  - Guard: with N leaves under one parent,
    `_resolved_parent_dir.cache_info().misses == 1` after N distinct-leaf calls. Clear
    both LRUs in a fixture.
- **`tests/test_axe_wait_marker_scan.py`:** `dirs_by_project` matches `walked_dirs` and
  preserves listing order per project.
- **`tests/test_axe_chop_wait_checks_scoped_views.py`:** in a tick with only scoped
  waiters, spy on `iter_agent_artifact_dirs` in both `sase.axe.wait_marker_scan` and
  `sase.scripts._chop_wait_checks_run_fs`. The walk lists each project once, the
  resolving view lists nothing, and only the confirmation pass re-lists its confirmation
  projects.
- **Seed removal:** existing tests must pass. If one spies on meta reads for meta-less
  directories, update it to assert at most one read attempt per meta-less directory per
  tick, and say so in the commit.

### 6. Measure before and after, and report

Use a throwaway harness kept outside the repo (for example under `/tmp`; do not commit
it):

1. Import `sase.scripts._chop_wait_checks_run`.
2. Stub every write path with no-ops:
   - `publish_ready_marker`, `apply_wait_epic_follow_patch`, and
     `mark_bead_wait_sync_hint` in `_chop_wait_checks_run_waiters`
   - every `upsert_*` in `_chop_wait_checks_run_waiters`, and every `upsert_*` and
     `reconcile_*` in `_chop_wait_checks_run`
   - `ScopeKeyCache.save_if_changed`
3. Call the registered `_run` handler (the decorator returns it unwrapped) with a stub
   runtime that provides `log(msg)`, `log.debug`, and `emit_summary(dict, reason=...)`,
   under cProfile against the real `~/.sase/projects` corpus.

The harness must never write SASE state. Run it on the base commit and on the change, on
the same machine. Report `walk_ms`, `view_ms`, `resolve_ms`, and the total in the final
response. Also report `python -X importtime` totals for context only; import cost is out
of scope.

> [!decision] job_timeout = raise Also change the `agent_waits` routine's
> `job_timeout: "2m"` to `"3m"` in `src/sase/default_config.yml`. Update any docs page
> that states the agent_waits timeout (grep `docs/` first), following the docs_writing
> memory.

### 7. Docs and follow-up

- In `docs/reference/automation/scheduler-default-routines.md`, in the wait_checks
  paragraph that already says "Confirmation re-lists those projects fresh", add one
  sentence: the resolving view reuses the tick's marker-walk listing, and only
  confirmation re-lists. Read the `docs_writing` reference memory with
  `/sase_memory_read` before editing.
- Use `/sase_new_task` to file one follow-up task bead (size `large`) for the
  `wait_checks` subprocess import cost. Search for duplicates first; the related closed
  bead is `sase-yk`. Evidence:
  - about 1.7–1.9 s per tick on apollo, about a quarter of a normal tick;
  - `sase.core.wait_dependency_resolution` alone takes 1.74 s, through `_artifact_state`
    → `sase.gate_turn.state` → `sase.notification_gates` → `sase.macro`;
  - `sase.axe.wait_marker_scan` → `sase.agent.names` (whose package `__init__` pulls
    `_resume` → `sase.macro`) takes about 1.0 s.

## Verification

- Targeted tests:
  - `tests/test_artifact_dir_key_fast_path.py`
  - `tests/test_wait_dependency_index_caches.py`
  - `tests/test_wait_dependency_scope_cache.py`
  - `tests/test_axe_wait_marker_scan.py`
  - `tests/test_axe_chop_wait_checks_*.py`
  - `tests/test_axe_chop_incremental_scans.py`
- Equivalence: in the harness, compute `artifact_dir_key` for every real artifact
  directory with the new code and with the old expression. Assert they are identical,
  and that the scoped view's selected directory list matches the base commit's.
- Read the `lint_and_test` reference memory with `/sase_memory_read` and run the
  verification it prescribes before finishing.
