---
tier: tale
title: Restore the post-commit push for sase bead close
goal:
  sase bead close publishes its bead-store commit again by pushing the sidecar repository that actually received it, and
  a new -P/--no-push flag lets a caller keep the commit local when batching mutations.
create_time: 2026-07-30 16:06:07
status: done
---

- **PROMPT:** [prompts/202607/bead_close_push.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/bead_close_push.md)
- **AGENTS:**
  - [bbugyi200.athena.po](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.po.md)
- **COMMITS:**
  - [e3a898b](https://github.com/sase-org/sase/commit/e3a898b6a32609979d79ce03d31f6dbf0d7dbc16) — fix(beads): route deferred close pushes correctly

# Plan: Restore the post-commit push for `sase bead close`

## Problem

`sase bead close` commits the bead-store mutation locally but never pushes it. `sase bead doctor` reports the
accumulating drift directly, e.g. `WARNING: bead store has 4 unpushed local bead commit(s)`.

The defect is a wrong push target introduced with the deferred-push refactor in `59930584c`
(`fix(beads): recover from transient sync divergence`).

`bead_store_mutation()` in `src/sase/bead/cli_common.py` deliberately commits with `push_after_commit=False` so that no
network work happens while the bead-store write lock is held, then pushes after the lock is released:

```python
committed = auto_commit(mutation.commit_message, push_after_commit=False, already_locked=already_locked)
...
if committed:
    _push_committed_bead_store()
```

`_push_committed_bead_store()` then resolves `location.store` and hands that store straight to
`push_sdd_store_after_commit()`, which pushes `store.repo_root`. For `sidecar_repos` storage `location.store.repo_root`
is the **plans** sidecar, not the **beads** sidecar that actually received the commit. Verified in a live workspace:

```
store.repo_root        = <workspace>/sase/repos/plans
_find_git_root(...)    = <workspace>/sase/repos/plans
resolve_beads_dir(...) = None
semantic beads dir     = <workspace>/sase/repos/plans
```

So every deferred push runs the managed sync worker against the plans repo, and the bead commit sitting in
`<workspace>/sase/repos/beads` is never published.

`commit_sdd_store_files()` already solves exactly this routing problem: it partitions the requested paths across
sidecars with `sdd_commit_targets()` and calls `push_sdd_store_after_commit()` once per **target** store. That is the
routing the pre-regression code got for free, because `handle_bead_close` used to call `auto_commit_bead_store(message)`
with `push_after_commit=None` and let `commit_sdd_store_files()` push each target.

### Why `close` is the command that exposes it

`try_handle_bead_fast_path()` returns `None` for `close` (`src/sase/main/bead_fast_path.py:31-32`), so `close` always
takes the Python slow path through `bead_store_mutation()`. Fast-pathed verbs (`create`, `update`, `open`, `rm`, `dep`,
`ref`) call `auto_commit_bead_store(message)` with the default `push_after_commit=None`, which keeps the correct
per-target push. The other slow-path handlers (`note`, `doctor --fix-*`, `history`) share the bug but are used less.

The failure is intermittent rather than absolute because `src/sase/main/entry.py:129-134` runs
`schedule_current_bead_refresh()` after every bead handler. That helper does push the beads repo, but only in
`background` refresh mode and only once the integration TTL (`sdd.bead_refresh.ttl_seconds`, default 120) has expired,
so a close inside the TTL window publishes nothing.

## Goal

1. `sase bead close` publishes its commit by default again, by pushing the repository that actually received it.
2. The push is overridable per invocation with `-P, --no-push`, mirroring `sase bead work`, so a caller batching many
   bead mutations can push once at the end.

## Approach

### 1. Route the deferred push through `sdd_commit_targets()`

In `src/sase/bead/cli_common.py`, rewrite `_push_committed_bead_store()` to resolve push targets the same way the commit
resolved them:

```python
from sase.sdd._commit_store import push_sdd_store_after_commit, sdd_commit_targets
...
for target_store, _paths in sdd_commit_targets(store, [location.beads_dir]):
    push_sdd_store_after_commit(target_store, push_after_commit=None)
```

This guarantees the push target can never diverge from the commit target again. `sdd_commit_targets()` returns
`[(store, paths)]` unchanged for non-sidecar storage, so `separate_repo` and `local` stores keep their current behavior.
Keep the existing `location is None or location.is_in_tree` guard, and keep the broad `except Exception` warning wrapper
so a push failure still cannot undo a successful local commit.

Keep `push_after_commit=None` so the configured `sdd.push_after_commit` mode (default `async`) still decides _how_ the
push runs. That keeps `sase bead close` returning immediately instead of blocking on remote latency, and users who want
a blocking push already have `sdd.push_after_commit: true`.

### 2. Add a `no_push` override to the shared mutation scope

Give `bead_store_mutation()` a keyword-only `no_push: bool = False` parameter that skips `_push_committed_bead_store()`
while still committing. Every existing call site keeps its current behavior because the default is `False`.

### 3. Wire `-P, --no-push` into `sase bead close`

- `src/sase/main/parser_bead.py`: add `-P, --no-push` to `bead_close_parser`, placed alphabetically by long option
  between `--force` and `--note`. Reuse the `sase bead work` help wording, adapted: "Commit the close locally but skip
  the post-commit push".
- `src/sase/bead/cli_crud.py`: `handle_bead_close` passes `no_push=getattr(args, "no_push", False)` into
  `bead_store_mutation`. The `getattr` default is required — existing tests construct `argparse.Namespace` literals
  without that attribute.

Only `close` gets the flag; that is what was asked for. The routing fix in step 1 benefits every slow-path mutation
regardless.

## Non-goals

- No new output. `sase bead close` keeps its current stdout, so the `tests/test_bead/golden/cli/close*.stdout` goldens
  stay untouched. Unlike `sase bead work`, an async push has no synchronous result worth reporting.
- No switch to a synchronous push. `sase bead work` pushes synchronously because it must guarantee publication before
  spawning detached workers; `close` has no such barrier requirement.
- No `--no-push` flag on other bead mutation subcommands.
- The dead `bead.push_after_commit` config field stays as-is. `docs/configuration.md` already documents it as a
  compatibility field that the current code does not read.
- No attempt to repair the already-diverged local bead store; `sase bead doctor` and `sase bead sync` own that.

## Files

| File                                        | Change                                                                                      |
| ------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `src/sase/bead/cli_common.py`               | Fix `_push_committed_bead_store()` target routing; add `no_push` to `bead_store_mutation()` |
| `src/sase/bead/cli_crud.py`                 | Thread `no_push` from `args` in `handle_bead_close`                                         |
| `src/sase/main/parser_bead.py`              | Add `-P, --no-push` to the close parser, alphabetically placed                              |
| `docs/beads.md`                             | Document close push behavior and the new flag row                                           |
| `tests/test_bead/test_cli_mutation_push.py` | New focused module for the push-routing and `--no-push` regressions                         |

## Tests

Add `tests/test_bead/test_cli_mutation_push.py`, following the existing per-behavior split convention in
`tests/test_bead/`. Patch `sase.sdd._commit_store.push_sdd_store_after_commit` (or the name as imported inside
`_push_committed_bead_store`) and assert on the store it receives.

1. **Sidecar routing regression.** With a `sidecar_repos` store whose `repo_root` is the plans sidecar and whose
   `beads_dir` is the beads sidecar, a committed mutation pushes exactly one store, whose `repo_root` is the **beads**
   repo and whose `sidecar_role` is `"beads"`. This test fails on `master`.
2. **Non-sidecar storage unchanged.** A `separate_repo` store still pushes exactly once with its own `repo_root`.
3. **In-tree stores still skip.** An in-tree location pushes nothing.
4. **`no_push` suppresses only the push.** `bead_store_mutation(..., no_push=True)` still calls `auto_commit` with
   `push_after_commit=False`, but never calls `_push_committed_bead_store()`.
5. **`sase bead close --no-push`.** `handle_bead_close(Namespace(..., no_push=True))` commits and does not push.
6. **Legacy namespace.** `handle_bead_close(Namespace(ids=..., reason=...))` with no `no_push` attribute still pushes,
   pinning the `getattr` default.
7. **Parser.** `sase bead close X -P` and `sase bead close X --no-push` both parse to `no_push=True`; the flag defaults
   to `False`.

Existing assertions in `tests/test_bead/test_cli_auto_commit.py` that the commit call uses
`push_after_commit=False, already_locked=False` remain correct and must keep passing — the commit call is unchanged.

## Docs

In `docs/beads.md`, under `### sase bead close`:

- Add a `-P, --no-push` row to the flag table, alphabetically after `-f, --force`.
- Add a short paragraph stating that a close that changes the store is committed and then published per
  `sdd.push_after_commit` (default `async`), and that `--no-push` keeps the commit local so a batch of bead mutations
  can be published once with a later `sase bead sync`.

## Verification

- `just install` first (ephemeral workspace), then `just check`.
- Targeted: `pytest tests/test_bead/test_cli_mutation_push.py tests/test_bead/test_cli_auto_commit.py`
  `tests/test_bead/test_cli_close_note.py tests/test_bead/test_cli_close_phases.py`
  `tests/test_bead/test_cli_close_resolution.py tests/main/test_bead_fast_path_mutations.py`.
- Manual smoke: run `sase bead doctor` and note the unpushed-commit count, run a real `sase bead close`, then rerun
  `sase bead doctor` and confirm the count does not grow.
