---
tier: tale
title: Install-scoped update-status snapshot
goal:
  ACE's updates badge and Update panel reflect only the running install, so a TUI
  launched from another venv (workspace soak, screenshot harness) can no longer make the
  badge disagree with the `,U` preview.
size: small
proposed_by: bbugyi200.athena.1de
proposed_with: claude/opus
create_time: 2026-11-17 10:09:41
status: wip
---

# Scope the update-status snapshot to the running install

## Problem

ACE's top-bar badge (`updates: ⬆1`) and the Update panel rows (`,U`) read one
machine-wide cache, `~/.sase/updates/status_snapshot.json`
(`src/sase/updates/cache.py::_cache_path`). That snapshot's contents depend on the
install. For editable installs, each `OutdatedComponent` records the checkout's
`source_root`, and "behind upstream" is a property of that one checkout.

Any `sase tui` started from a different install writes its own install's verdict into
that same file. Examples are a workspace `.venv` used by GC census or soak runs, or a
screenshot or perf harness. That happens through `get_cached_update_status()` (startup
and periodic checks in `src/sase/ace/tui/actions/update_toast.py`) and through
`plugins_browser_loading.py` (`merge_update_status` + `write_update_status_snapshot`).

Observed on 2026-11-17: the `sase-1t3.9` GC census soak runs
`<workspace>/.venv/bin/python -m sase tui -x -t agents` against the real `~/.sase`. Its
checkout was one commit behind `origin/master`. At 09:57:15 it wrote a snapshot whose
only component is `sase` (editable, `source_root=<that workspace>`,
`0.19.0+119 → 0.19.0+120`). The user's TUI, installed from
`~/projects/github/sase-org/sase`, was already at the newest commit. It then:

1. read that snapshot, which was inside the TTL;
2. revalidated it with `revalidate_update_status`, which re-classifies git at the
   **component's** `source_root` (the soak's workspace, still behind), so the row
   survived;
3. showed `⬆1`.

Pressing `,U` → Everything builds a fresh preview from the TUI's own uv-tool receipt
(`load_receipt_for_summary` → `make_sase_dev_update_preview`). That proves the user's
real checkout is current, so it toasts "Everything in the captured comprehensive update
is already current." Both answers are correct for their own install. The bug is that
they share one cache file.

`merge_update_status` also carries the other install's last-known-good rows forward when
a source fails. So the leak can outlive a successful recompute, and two live TUIs on
different installs keep overwriting each other's verdicts.

## Fix

### 1. Key the snapshot by install identity (`src/sase/updates/cache.py`)

- Add `current_install_prefix() -> str`, returning `str(Path(sys.prefix).resolve())`. It
  is the venv that owns the running interpreter: the uv tool venv for the user's
  install, `<checkout>/.venv` for workspace runs. Keep it a module-level function so
  tests can monkeypatch it. It is cheap, with no subprocess and no import-budget impact.
- `_cache_path()` becomes `sase_subdir("updates") / f"status_snapshot.{key}.json"`,
  where `key` is the first 12 hex characters of `sha256(prefix)`. Keep `CACHE_FILENAME`
  as the legacy name, documented as legacy, and add a `CACHE_FILENAME_TEMPLATE` (or
  similar) for the keyed name. `SASE_HOME` handling is unchanged because the path still
  goes through `sase_subdir`.
- Envelope: write `"install_prefix": <prefix>`. On read, return `None` when the field is
  missing or differs from `current_install_prefix()`. This guards against hash
  collisions or copied files. When an explicit `path=` is passed (tests), still enforce
  the prefix match against `current_install_prefix()`. Tests monkeypatch it.
- Bump `SCHEMA_VERSION` to 5, since the envelope gained a required field. Update the
  existing `test_previous_update_status_schema_is_treated_as_cache_miss` pattern to
  match.
- Housekeeping on write, best effort, never raising, only when `path is None`:
  - unlink the legacy unkeyed `status_snapshot.json` if present;
  - unlink sibling `status_snapshot.*.json` files (not the current one) whose mtime is
    older than 7 days, so dead workspace venvs do not leave files behind.

  This runs only in the existing writer paths, which are already off the UI thread
  (worker threads and procs). Do not add any read-path globbing (TUI perf rule 8).

The callers `get_cached_update_status`, `read_update_status_snapshot`,
`write_update_status_snapshot`, and the `plugins_browser_loading.py` read/merge/write
need no signature changes. They pick up the keyed path through `_cache_path()`.
`merge_update_status` then only merges same-install history.

### 2. Reconcile the badge after a no-op preview (`src/sase/ace/tui/actions/update_run.py`)

In `_on_update_preview_complete`, when `not preview.runnable`, call
`_handle_comprehensive_noop(preview)` as today. Then, if it exists, call
`self._schedule_updates_indicator_revalidation()`. That is the same hook
`_on_scoped_update_complete` already uses. If a fresh preview proves the install is
current while the badge still shows a cached row (for example, the checkout was
fast-forwarded outside ACE), the badge drops the row instead of disagreeing with the
toast. The revalidation is drop-only and does no network work. It already runs on a
worker thread.

## Tests

- `tests/test_update_status_snapshot.py` (extend):
  - two monkeypatched prefixes under a tmp `SASE_HOME` write to distinct files. Reading
    under prefix A never returns prefix B's components.
  - an envelope whose `install_prefix` differs from the current prefix, or is missing,
    reads as `None`.
  - a legacy `status_snapshot.json` is ignored on read and removed by a default-path
    write. A stale (>7 days, `os.utime`) sibling keyed file is pruned, and a fresh
    sibling is kept.
  - the round trip still preserves every field (existing round-trip test, updated for
    the new envelope field).
- `tests/ace/tui/test_update_run_actions.py`: a non-runnable preview completion toasts
  the no-op message and schedules indicator revalidation. A runnable preview does not
  schedule it from this path.
- Run any existing `get_cached_update_status` / `plugins_browser_loading` tests
  (`tests/ace/tui/test_plugins_browser_loading_*.py`) unchanged. They must pass with the
  keyed path. Fix only fixtures that hard-code the legacy filename.

## Out of scope

- Disabling update checks in non-primary installs. Keying makes every install's badge
  truthful for itself, so a special case for workspace TUIs is unnecessary.
- Other `~/.sase` caches. The plugin and core PyPI latest-version cache is keyed by
  distribution and is truly global. Agent-CLI status is global by nature.
- No docs or memory notes reference the snapshot path, so none need edits.
