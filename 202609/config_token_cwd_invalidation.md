---
tier: tale
title: Invalidate the config-token cache on chdir
goal:
  A chdir invalidates the process-wide config-token cache, so `sase init --all` plans
  every project against its own config and stops deleting the `flag` task type from the
  sase project's committed `sase/task_types.json` snapshot.
size: medium
proposed_by: bbugyi200.kellys_mbp.b
create_time: 2026-09-03 07:08:44
status: wip
---

# Invalidate the config-token cache on `chdir` so `sase init --all` stops corrupting `sase/task_types.json`

## Problem

`sase init -a` reports the `sase` project as fully initialized on one machine (a Mac
with two enabled projects) while the same command against the same commit on `apollo`
(one enabled project) wants to rewrite `sase/task_types.json`, re-adding the
project-local `flag` task type.

The two machines have identical `sase`, `sase-core-rs`, `sase-github`,
`sase-research-artifacts`, and `sase-telegram` versions, identical checkouts, and
identical live task-type catalogs. So the split is not a version or plugin skew.

## Root cause

`current_config_token()` in `src/sase/config/core.py` is a **process-wide cache whose
key does not include the current working directory, even though its value does**.

- `_compute_current_config_token()` (`src/sase/config/core.py:167`) folds
  `str(Path.cwd())` and the stat tokens of
  `resolve_project_layout(discover_project_root()).config.candidates` into the token.
  The token payload is therefore CWD-sensitive by design.
- `current_config_token()` (`src/sase/config/core.py:224`) only recomputes when the
  cached value is `None` or when `CONFIG_DIR` has been rebound. Otherwise it serves the
  cached token, and on deadline expiry it serves the **stale** token immediately while a
  daemon worker recomputes off-thread (`_CONFIG_TOKEN_REFRESH_INTERVAL_SECONDS = 5.0`).
- A `chdir` is invisible to that cache. Every token-keyed cache therefore keeps
  answering with the previous directory's state: `_merged_config_cache`,
  `_agent_owner_config_cache`, and — the one that bites here —
  `get_task_type_registry()` in `src/sase/task_types/registry.py:31`, whose cache token
  is `(current_config_token(), SASE_DISABLE_PLUGINS, SASE_DISABLE_PLUGIN_TASK_TYPES)`.

`sase init --all` chdirs between projects: `_working_directory()` at
`src/sase/main/init_onboarding.py:429` wraps each target's `_run_init_onboarding_result`
call. So every project after the first is planned against the first project's cached
config.

`sase/task_types.json` is rendered from
`committed_task_type_records(get_task_type_registry())`
(`src/sase/main/init_memory/root_rendering_task_types.py:_project_task_type_records`),
which includes every `builtin` and `project` record. The `sase` project declares a
`project`-source `flag` task type in `sase/sase.yml` under `bead.task_types`. Under a
stale token the registry is the _other_ project's, which has no `flag`, so the rendered
snapshot silently drops it — and `compare_expected_memory_files()` then sees the
truncated on-disk file as correct.

Confirmed directly against the installed build:

```text
# cwd=bob-cli -> token+registry cached; chdir to sase; no 5s wait
bob-cli records: ['bug', 'ci', 'feature', 'flake', 'memory', 'github']
token stale (t1 == t2)?              True
freshly computed token differs?      True
sase records (served from cache):    ['bug', 'ci', 'feature', 'flake', 'memory', 'github']   # 'flag' missing
```

The same run also shows `load_merged_config()` returning the **identical object** across
the `chdir`, so the contamination is not limited to task types.

### Why the two machines disagree

|        | enabled projects  | first project in the batch | registry used for `sase` |
| ------ | ----------------- | -------------------------- | ------------------------ |
| Mac    | `bob-cli`, `sase` | `bob-cli` (sorts first)    | `bob-cli`'s — no `flag`  |
| apollo | `sase` only       | `sase`                     | `sase`'s — has `flag`    |

Nothing is machine-specific: the variable is _how many projects the batch visits before
reaching `sase`_. apollo is the machine reporting the truth.

### The corruption already landed

`07f5898ab chore: run sase init memory` is a 96-line **deletion** from
`sase/task_types.json` — the whole `flag` entry — written by a batch run on the Mac.
`master` is therefore already wrong, and the single-project gate already rejects it on
both machines:

```text
$ sase validate
  fail   init memory --check
  run  init memory  update `flag` is not in the committed snapshot (sase installed)
```

`just check` runs `just validate` → `sase validate`, so `just check` is red on `master`
today. The gate is not missing; `sase init --all` is producing output the gate rejects,
and re-running the batch on the Mac re-deletes the entry, so the two machines flip-flop
the file.

### Blast radius beyond `init --all`

`src/sase/xprompt/workflow_executor_utils.py:182` (`apply_chdir_output`) chdirs
mid-process on a workflow `_chdir` output and rebinds `SASE_ACTIVE_PROJECT_DIR`. It does
not clear config caches either, so a workflow that hops workspaces reads the pre-hop
project's merged config. Fixing the cache key fixes both call sites; fixing only the
`init --all` loop would not.

## Fix

Make a `chdir` a cache-generation event for the config token, exactly as rebinding
`CONFIG_DIR` already is. All changes are Python process-cache plumbing in this repo; the
merge semantics, the token payload, and the stale-while-revalidate policy are unchanged,
and no `sase-core` (Rust) wire or domain behavior is involved.

### 1. `src/sase/config/core.py`

- Add a module global `_current_config_token_cache_cwd: str | None = None`, documented
  alongside the existing `_current_config_token_cache_dir` as the second half of the
  cache key.
- Add a private helper that returns the CWD component of the key:

  ```python
  def _current_config_token_cache_cwd_key() -> str | None:
      """Return the cwd half of the token cache key, or None when unused."""
      if not _include_local_config:
          return None
      try:
          return str(Path.cwd())
      except OSError:
          return None
  ```

  Returning `None` when `_include_local_config` is false mirrors
  `_compute_current_config_token()`, which already omits the CWD from the payload in
  that mode — so `sase ace` and other local-config-disabled processes pay nothing for
  this.

- In `current_config_token()`, read the key inside `_current_config_token_cache_lock`
  and treat a mismatch as a synchronous recompute, extending the existing `CONFIG_DIR`
  branch:

  ```python
  cwd_key = _current_config_token_cache_cwd_key()
  cached = _current_config_token_cache_value
  if (
      cached is None
      or _current_config_token_cache_dir is not CONFIG_DIR
      or _current_config_token_cache_cwd != cwd_key
  ):
      ...
      _current_config_token_cache_cwd = cwd_key
      return token
  ```

  `Path.cwd()` is one `getcwd(2)` call with no filesystem stat, so the hot path keeps
  its stat/glob-free character; only an actual directory change pays for a recompute.

- Clear `_current_config_token_cache_cwd` in
  `_reset_current_config_token_cache_locked()` so explicit invalidation and
  `clear_config_cache()` reset the full key.
- Pass the scheduling CWD key into the refresh worker
  (`args=(_current_config_token_cache_epoch, cwd_key)`) and have
  `_refresh_current_config_token(cache_epoch, cache_cwd)` publish **only** when the
  epoch matches and the CWD key is unchanged both in the cache and when re-read after
  the compute:

  ```python
  if cache_epoch == _current_config_token_cache_epoch and (
      _current_config_token_cache_cwd == cache_cwd
      and _current_config_token_cache_cwd_key() == cache_cwd
  ):
      ...publish token, extend deadline...
  ```

  A worker that raced a `chdir` computed its token against a directory the cache is no
  longer keyed to. It must neither publish that token nor extend the deadline; declining
  is safe because the next synchronous read sees the CWD mismatch and recomputes.
  Deregistering the worker thread stays unconditional, as today.

### 2. Do not patch the call sites

`_working_directory()` in `src/sase/main/init_onboarding.py` and `apply_chdir_output()`
in `src/sase/xprompt/workflow_executor_utils.py` stay as they are. Adding
`clear_config_cache()` calls there would paper over the same defect at two of N chdir
sites and would bump `_config_cache_generation`, needlessly discarding the
process-static default/plugin config layers.

## Tests

### 3. `tests/test_config_cache_token.py`

- `test_current_config_token_recomputes_after_chdir`: with `time.monotonic` pinned so
  the refresh deadline never expires, read the token in one temp directory,
  `monkeypatch.chdir` to a second, and assert the second read returns a different token
  and that `_compute_current_config_token` was called twice. Pinning the clock is what
  proves the recompute comes from the CWD key rather than from expiry.
- `test_config_token_refresh_worker_declines_to_publish_after_chdir`: reuse the module's
  existing gated-`compute` pattern (`refresh_started` / `release_refresh` events) to
  block a worker mid-compute, `chdir` while it is blocked, release it, and assert the
  worker's token is never published and that the next read recomputes synchronously for
  the new directory.
- Keep `test_current_config_token_serves_stale_while_refreshing` passing without
  weakening it: it must still observe one compute inside the interval and exactly one
  off-thread recompute after expiry when the CWD does not change. Its `compute` stub
  takes no arguments, so only the thread `args` change affects it.

### 4. `tests/task_types/` — registry rebuild across `chdir`

Add a test (new module `test_registry_cwd.py`, or extend
`tests/task_types/test_project_config.py` if it already carries live-registry cases)
that builds two temp project roots, each with a `sase/sase.yml` declaring a distinct
`bead.task_types` entry, points `CONFIG_DIR` at an isolated config dir, then asserts
`get_task_type_registry().by_slug` reflects the root it is standing in — first in root
A, then after `monkeypatch.chdir` to root B, with no cache reset in between. This is the
assertion that fails today.

### 5. `tests/main/test_init_onboarding_all.py` — batch regression

Add a test in the style of `test_batch_check_isolates_failures_and_restores_cwd`: two
real temp project workspaces with different `bead.task_types`, a fake `InitCommandSpec`
whose `plan` callback records `sorted(get_task_type_registry().by_slug)` for the
directory it runs in, then assert each project saw **its own** slugs. This is the
end-to-end lock on the reported symptom and must fail before the fix in
`src/sase/config/core.py` lands.

## Data repair

`sase/task_types.json` on `master` is missing the `flag` entry that `07f5898ab` deleted.
After the source fix, restore it from the project's own catalog:

```bash
sase init memory      # regenerates sase/task_types.json; re-adds the 96-line `flag` entry
```

Do not hand-edit the file — it is a generated snapshot. The regeneration is expected to
touch `sase/task_types.json` only; if it wants to rewrite `AGENTS.md`, a provider shim,
or anything under `sase/memory/`, stop and report that instead of committing it, because
that would mean a second generator is drifting and is out of scope here.

## Verification

1. `just install` first — this workspace's virtualenv may predate the current pins.
2. `just check`. `sase validate`'s `init memory --check` line must flip from `fail` to
   `ok`; that single assertion is the proof the data repair worked.
3. Prove the fix on the multi-project path, from the `sase` project's primary workspace
   on a machine with more than one enabled project:

   ```bash
   sase init -a -c -d      # must report the sase project current, with no sase/task_types.json diff
   sase init memory -c     # must agree with the batch run
   ```

   Before the fix these two disagree; agreeing is the point of the change.

4. `just check-full` through `/sase_monitor` before landing — the change touches a
   process-wide config cache that essentially every module reads through, which is
   exactly the "touches the broadening set" case the two-speed rule reserves the full
   suite for.

## Out of scope

- Any change to the token payload, the merge order, or the 5-second
  stale-while-revalidate policy. Serving a stale token within its window is a deliberate
  TUI-latency choice; the defect is that a `chdir` never ends the window.
- Any change to which records `committed_task_type_records()` admits. The
  builtin/project/required- plugin membership rule is correct; it was fed the wrong
  registry.
- Reconciling `sase-github`, `sase-telegram`, or any other plugin catalog across
  machines. Both machines already agree there.
