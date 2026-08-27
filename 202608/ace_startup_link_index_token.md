---
tier: tale
title: Fix ACE startup regression from uncached provider_source_token
goal:
  ACE startup returns to its pre-2026-08-27 timings by making provider_source_token
  cached, so the mount-time link-index build stops spending ~105s of GIL-bound work
  starving the agents disk load.
size: medium
proposed_by: bbugyi200.athena.0em
---

- **AGENTS:**
  - [bbugyi200.athena.0em](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0em.md)
- **COMMITS:**
  - [f49030d](https://github.com/sase-org/sase/commit/f49030db66b134281bc447c9d367cd7aabad6d9c)
    — perf(ace-tui): cache provider_source_token and cascade tab cache invalidation

# Fix the ACE startup regression caused by uncached `provider_source_token()`

## Problem

`sase ace` startup regressed hard on 2026-08-27. Startup telemetry
(`~/.sase/logs/tui_startup.jsonl`) shows the break between the last run on 2026-08-26
and the first run on 2026-08-27, with no growth in data volume to explain it:

| date       | median `agents_ready_s` | median `visible_ready_s` | median `axe_ready_s` |
| ---------- | ----------------------- | ------------------------ | -------------------- |
| 2026-08-24 | 4.81                    | 3.78                     | 2.27                 |
| 2026-08-25 | 4.86                    | 3.88                     | 2.29                 |
| 2026-08-26 | 4.88                    | 3.77                     | 2.33                 |
| 2026-08-27 | 12.18                   | 11.31                    | 3.70                 |

`process_start_to_on_mount_seconds` (0.93 -> 0.91) and `on_mount_to_first_paint_seconds`
(0.31 -> 0.30) did **not** move, so the app still paints on time; everything after first
paint is what got slow. Individual 2026-08-27 runs reached `agents_ready_seconds` of
16.4 and 25.1 while carrying _fewer_ agents than the fast 2026-08-26 runs (229-389 vs
703).

`~/.sase/logs/tui_agent_loads.jsonl` isolates the regression to the `disk` stage of the
startup agents load:

- 2026-08-26 startup loads: `disk` 2.06s - 3.88s (703 agents at the high end)
- 2026-08-27 startup loads: `disk` 5.22s, 6.56s, 11.88s, 8.44s, 21.04s, 8.25s, 11.76s
  (229-389 agents)

Running that same loader standalone on the same machine takes **1.3s**, so the disk
stage is not itself slower. It is being starved.

## Root cause

`_build_link_index()` (`src/sase/ace/tui/relations/link_index.py:91`) calls
`_build_chip()` once per link-graph edge endpoint. `_build_chip()` calls
`accent_and_icon_for_ref()` (`src/sase/ace/tui/relations/link_subject.py:38`), which
calls `descriptor_for_artifacts_pane_id()` (`src/sase/ace/tui/artifact_tabs.py:116`),
which calls `resolve_artifacts_subtabs()` (`src/sase/ace/tui/artifact_tabs.py:68`).

`resolve_artifacts_subtabs()` caches its descriptors in `_ARTIFACTS_TAB_CACHE`, but it
computes its cache key **before** every lookup by calling `provider_source_token()`
(`src/sase/ace/tui/_artifact_tab_discovery.py:212`), which is not cached at all. Each
call does, per configured project:

- `list_project_records()` -> Rust `list_project_records` plus a per-record
  `is_valid_sase_project_name()` that runs two `Path.resolve()` syscalls
- `project_file.stat()`
- `resolve_project_config_read_path()` -> `resolve_project_layout()` ->
  `_resolve_content_layout()` -> Rust `sase_content_layout` plus a full
  `content_layout_from_mapping()` Python wire re-parse
- another `Path(config_path).stat()`
- `current_config_token()`

Measured at ~2ms per call. The descriptor cache never pays off because validating the
cache costs far more than the work it guards.

Measured on this machine against the real aggregate
(`~/.sase/projects/gh_sase-org__sase/artifact-links.json`, 5.9MB, 13,926 rows; 14,802
rows across all projects):

```
_build_link_index                                       105.77 s
  _build_chip                                           105.24 s
    accent_and_icon_for_ref                             103.78 s
      descriptor_for_artifacts_pane_id                  103.61 s
        resolve_artifacts_subtabs                       103.45 s
          provider_source_token                         102.18 s
            resolve_project_config_read_path             69.73 s
            list_project_records                         28.36 s
              is_valid_sase_project_name                  7.76 s
```

(pyinstrument, 5ms sampling. `link_index_for_snapshot()` memoizes on the snapshot
`source_key`, so a repeat call in the same process is 0.000s — the cost is paid once per
process, at startup, every time.)

`src/sase/ace/tui/actions/_startup_mount.py:108` calls
`self._schedule_link_index_refresh(source="mount")` during `on_mount`. That schedules a
pump-free task which runs `load_artifact_links_snapshot` and `link_index_for_snapshot`
through `asyncio.to_thread`. The work is correctly off the event loop and off the pump,
but it is ~100s of **GIL-bound Python** in a worker thread, concurrent with the agents
disk load in its own `asyncio.to_thread` worker. The two serialize on the GIL, which is
exactly why a 1.3s disk stage takes 8-21s in-process while the standalone loader stays
at 1.3s.

`~/.sase/logs/tui_stalls.jsonl` corroborates this: of the 17 non-recovery stall and
hitch records on 2026-08-27, 9 sample the main thread inside `list_project_records` /
`is_valid_sase_project_name`, two of them entered from `resolve_artifacts_subtabs` ->
`provider_source_token` directly, plus one in `artifact_link_edges`.

## Regression window

The last fast startup was 2026-08-26T23:38:06Z (19:38 local); the first slow one was
2026-08-27T10:36:58Z (06:36 local). Landing in that window:

- `48e019af8` (2026-08-26 22:22 local) `feat(ace): add read-only link rail` — added
  `link_index.py` and the `_schedule_link_index_refresh(source="mount")` call in
  `_startup_mount.py`
- `a7b702863` (2026-08-27 01:50 local)
  `feat(tui): move links to app-level rail, drop link_rail flag and L bindings` —
  deleted `src/sase/ace/tui/link_rail_flag.py` and the `link_rail` feature flag, making
  the rail (and therefore the mount-time index build) unconditional.
  `~/.sase/feature_flags.json` never opted into `link_rail`, so this commit is what
  first put the index build on this user's startup path.

This also violates `sase memory read tui_perf.md` rule 8 ("Cache disk reads keyed by
mtime; render paths never stat/glob") and rule 9 ("Keep startup off data-scaled work").

## Fix

Memoizing `provider_source_token()` for the duration of one index build was measured end
to end:

| variant                                            | `_build_link_index` |
| -------------------------------------------------- | ------------------- |
| today                                              | 105.77 s            |
| memoized `provider_source_token()`                 | **0.71 s**          |
| plus memoized `descriptor_for_artifacts_pane_id()` | **0.56 s**          |

Both produce the identical index (`refs=38931`).

### Step 1 — Give `provider_source_token()` a time-gated cache

In `src/sase/ace/tui/_artifact_tab_discovery.py`, wrap `provider_source_token()` in a
process-local cache guarded by a lock and a monotonic deadline, modelled on the existing
`current_config_token()` cache in `src/sase/config/core.py:222` (read that
implementation first and match its shape and naming). Requirements:

- Rename the current uncached body to a private `_compute_provider_source_token()` and
  keep `provider_source_token()` as the cached public entry point, so existing callers
  and tests are unaffected.
- A `None` result (transient discovery failure) MUST NOT be cached — the existing
  docstring is explicit that a `None` token is uncacheable so a degraded four-tab answer
  cannot get pinned. Preserve that.
- Pick a refresh interval in the same ballpark as
  `_CONFIG_TOKEN_REFRESH_INTERVAL_SECONDS`; do not invent a longer one. Reuse that
  constant's value or define a sibling constant next to it. A synchronous recompute on
  expiry is acceptable here — do not add a background revalidation thread unless
  matching `current_config_token()` makes it fall out naturally.
- Export a `reset_provider_source_token_cache()` and call it from
  `reset_artifacts_subtabs_cache()` (`src/sase/ace/tui/artifact_tabs.py:60`) so every
  existing reset call site — including `src/sase/artifact_cli/pane.py:55` and the ~8
  test modules that call it — invalidates the new cache too. Missing this will produce
  stale-descriptor test failures, so wire it before running the suite.

### Step 2 — Hoist the descriptor lookup out of the per-chip loop

In `src/sase/ace/tui/relations/link_index.py`, `_build_link_index()` resolves an
accent/icon pair per chip through `accent_and_icon_for_ref()`, but the result only
depends on `(neighbor_kind, neighbor_target.pane_id)`, of which there are under a dozen
distinct values across ~29k chips. Memoize it for the duration of one build (a local
`dict` threaded into `_build_chip`, or a build-scoped helper) rather than calling
through `resolve_artifacts_subtabs()` per chip. This is the defence-in-depth half of the
fix: Step 1 alone gets 105.77s to 0.71s, and Step 2 takes it to 0.56s and stops the
rebuild from being re-coupled to token cost later.

Do not change what `accent_and_icon_for_ref()` returns, and do not memoize it at module
scope — descriptors are palette-hash-assigned at runtime and a process-lifetime cache
would go stale exactly the way `reset_artifacts_subtabs_cache()` exists to prevent.

### Step 3 — Stop `is_valid_sase_project_name()` doing filesystem I/O

`is_valid_sase_project_name()` (`src/sase/core/paths.py:143`) is a name-shape check that
costs 42us per call because it runs two `Path.resolve(strict=False)` syscalls to confirm
`projects_root / name` stays inside `projects_root`. It is called once per record inside
`list_project_records()` (`src/sase/core/project_lifecycle_facade.py:69`), which sits on
multiple pump paths (`filtered_project_records`, `iter_patch_project_file_records`,
`provider_source_token`) — it accounted for 7.76s of the 105s profile and appears
directly in six of the 2026-08-27 stall samples.

The function already rejects empty names, `.`, `..`, dotfiles, NUL, and both path
separators by string inspection _before_ it touches the filesystem. Those checks are
what actually enforce the containment property; the `resolve()` pair adds nothing for
any input that reaches it. Drop the two `resolve()` calls and the containment
comparison, keeping the string-level checks and the existing signature and return type.

If — and only if — you find a case the string checks miss, keep a filesystem check but
hoist the `projects_root` resolve to a module-level memo so it is paid once per process
instead of once per record, and say so in the stitch message.

## Verification

1. **Reproduce before the fix.** From a scratch directory, against the real aggregate:

   ```python
   from sase.ace.tui.relations.artifact_links import load_artifact_links_snapshot
   from sase.ace.tui.relations import link_index as LI
   import time
   snap = load_artifact_links_snapshot(None)
   t = time.perf_counter(); idx = LI._build_link_index(snap)
   print(len(snap.rows), len(idx.by_ref), time.perf_counter() - t)
   ```

   Record the baseline seconds and `len(idx.by_ref)`. Note `_build_link_index` is called
   directly on purpose — `link_index_for_snapshot` memoizes.

2. **Re-run after each step** and confirm `len(idx.by_ref)` is byte-identical to the
   baseline while the time drops to roughly the table above. An index that changed shape
   means Step 2 memoized on too coarse a key.

3. **Add a regression test.** Add to `tests/ace/tui/test_link_index.py` a test that
   builds an index from a synthetic snapshot of a few hundred rows with a counting
   wrapper installed over `provider_source_token` (or over
   `_compute_provider_source_token`), and asserts the call count stays bounded by a
   small constant instead of scaling with row count. Assert on call counts, not
   wall-clock seconds — a timing threshold will flake on a loaded host.
   `tests/ace/tui/test_link_index.py:129` already calls
   `reset_artifacts_subtabs_cache()`; follow that fixture pattern.

4. **Add a cache-invalidation test** in `tests/ace/tui/test_artifact_tab_discovery.py`:
   mutate a project file's mtime/size, call `reset_artifacts_subtabs_cache()`, and
   assert `provider_source_token()` returns the new token rather than the cached one.
   Also assert a `None` token is not cached — force the discovery failure path and
   confirm the next call recomputes.

5. **Gates.** Run `just check`. Because this touches `src/sase/core/paths.py` and the
   shared artifact-tab cache, also run `just check-full` through `/sase_monitor` with
   the `TESTING` / `TESTED` status pair before landing.

6. **Confirm in the real app.** Start `sase ace`, quit, then read the newest record in
   `~/.sase/logs/tui_startup.jsonl`. Expect `agents_ready_seconds` back near the
   2026-08-26 median of ~4.9s (host load permitting) and the `disk` stage in
   `~/.sase/logs/tui_agent_loads.jsonl` back under 4s. Check
   `~/.sase/logs/tui_stalls.jsonl` for new records naming `provider_source_token` or
   `is_valid_sase_project_name`; there should be none from the new pid.

## Out of scope

Note these as `PROPOSED FOLLOW-UP:` rather than fixing them here:

- `~/.sase/projects/gh_sase-org__sase/artifact-links.json` has reached 5.9MB / 13,926
  rows with no retention policy, and `artifact-link-projection.json` is 6.2MB alongside
  it. Even the fixed build is O(rows) at every startup.
- `~/.sase/agent_artifact_index.sqlite` is 195MB and `~/.sase/agent_name_registry.json`
  is 17MB, with several multi-MB orphaned `.agent_name_registry.json.*.tmp` files left
  in `~/.sase/`.
- `bead.sync.fetch` git operations jumped from 55 (2026-08-26) to 1384 (2026-08-27,
  partial day), 1358 of them from
  `/home/bryan/projects/github/sase-org/sase/sase/repos/beads`, totalling 373s of git
  fetch. That is background agent load rather than a TUI startup cost, but it is a large
  new source of host contention.
- `agents_sync.object_read` has a 1.34s median across 185 operations.
