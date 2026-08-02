---
tier: tale
title: Restore the Tasks tab selection for durable-store rows
goal:
  Reopening the Admin Center Tasks tab reselects the entry the user had selected, including durable-store rows that
  arrive after the pane mounts.
proposed_by: bbugyi200.athena.s1
create_time: 2026-08-02 10:25:56
status: done
---

# Plan: Restore the Tasks tab selection for durable-store rows

## Problem

In the SASE Admin Center, select an entry on the **Tasks** tab, close the panel with `Esc`, then reopen the tab with
`##`. The previously selected entry is supposed to be reselected. It is not: the cursor lands on a different row and the
Output pane shows a different task.

The failure is not random. It happens **exactly when the selected row is a durable-store row** — any `◆ detached` task,
or any row that this TUI process does not own in its in-memory `TaskQueue`. Rows the TUI owns in memory resume
correctly, which is why the bug looks intermittent.

A second symptom shares the same cause: opening the Tasks tab fresh, with no prior bookmark, does not land on the newest
task (row 0). It lands on whichever row happened to be first before the durable store was read.

## Reproduction

Verified against the current tree by driving `ConfigCenterModal(initial_tab="tasks")` twice with a shared
`AdminCenterSessionState`, an in-memory `TaskQueue` of three TUI-owned tasks, and a patched `load_store_task_rows`
returning three `◆ detached` store rows. Selecting each row in turn and reopening:

| selected row                     | identity | row after reopen | result   |
| -------------------------------- | -------- | ---------------- | -------- |
| 0 `Epic launch A` (store)        | `d-epic` | 1 (`mem-1`)      | mismatch |
| 1 `Plan response: epic` (memory) | `mem-1`  | 1                | ok       |
| 2 `launch sase` (memory)         | `mem-2`  | 2                | ok       |
| 3 `Epic launch B` (store)        | `d-old1` | 2 (`mem-2`)      | mismatch |
| 4 `Task launch C` (store)        | `d-old2` | 2 (`mem-2`)      | mismatch |

Every store-backed row fails; every in-memory row passes. This matches the reported screenshots, where the selected
running `◆ detached Epic launch · just_test_contention_flakes` (row 0) resumed onto `Plan response: epic` (row 1) — row
1 in the merged list is exactly row 0 of the memory-only list.

## Root cause

`TasksPane` is rebuilt from scratch on every modal open. `TasksPaneStoreMixin._request_store_reload` reads the durable
store on a **thread worker**, so at `on_mount` time `self._store_rows` is empty and `_merged_tasks()` returns in-memory
tasks only. The bookmarked store row simply does not exist yet.

`src/sase/ace/tui/modals/config_center_session.py` already models this: `SelectionBookmark.display()` records a
_provisional_ row without disturbing `identity`/`row`, while `record()` commits an authoritative selection.
`TasksPaneSelectionMixin._rebuild_list` uses it correctly — `pending_missing_bookmark` is true when the bookmarked
identity is absent and the store has not loaded, so the pre-store paint calls `display()` and the requested identity
survives.

The defect is that **nothing ever reads that preserved request back**. Instrumenting `SelectionBookmark` during the
reproduction shows the whole lifecycle of one reopen:

```
DISPLAY 'mem-1' row=0                     <- on_mount, provisional; bookmark.identity stays 'detached-epic'
DISPLAY 'mem-1' row=0                     <- focus_default, provisional; still preserved
RECORD  'mem-1' row=1   via _apply_store_snapshot -> _rebuild_list -> _record_bookmark
```

`TasksPaneStoreMixin._apply_store_snapshot` (`src/sase/ace/tui/modals/tasks_pane_store.py:173`) computes

```python
prior_identity = self._selected_task_identity()
```

which is the **provisional stand-in** the pane is currently painting, not the request the bookmark is holding. It passes
that to `_rebuild_list`, where `identity = self._rekey_task_identity(prior_identity or bookmark.identity)` prefers the
non-`None` argument, so `bookmark.identity` is never consulted. The rebuild therefore restores the stand-in, finds it at
its new merged index, and — because the store has now loaded, making `pending_missing_bookmark` false — commits it with
`record()`. The user's real selection is destroyed on the very rebuild that was supposed to honor it.

Three secondary defects fall out of the same confusion between _requested_ and _displayed_ selection:

1. `_refresh_running_output` (`tasks_pane_store.py:97`) and `action_toggle_scope` (`tasks_pane_store.py:203`) read
   `_selected_task_identity()` the same way, so a status tick or an `a` scope toggle landing inside the pre-store window
   clobbers a pending request too.
2. `_rebuild_list`'s `pending_missing_bookmark` guard requires `identity is not None`, so a **fresh** open with no
   bookmark commits its pre-store row 0 authoritatively. When the store rows arrive that stale commit is restored by
   identity, which is why a fresh Tasks tab does not land on the newest task.
3. `on_option_list_option_highlighted` records authoritatively for every highlight message whose identity/row match the
   widget. `ProgrammaticSelectionGuard` remembers only one intended selection, and the reopen path rebuilds twice
   (`on_mount` then `focus_default`), so a second echo can slip past the guard and promote the provisional stand-in.
   This did not fire in the captured trace, but it is a live second path to the same corruption and must be closed with
   the primary fix.

Existing coverage misses all of this because `tests/ace/tui/test_admin_center_selection_resume.py` and
`tests/ace/tui/test_tasks_pane_store.py::test_tasks_session_restores_selected_task_across_modal_instances` both call
`patch_store_loader(monkeypatch, [])`. With zero store rows every task is in memory at mount, the identity restore
succeeds immediately, and the asynchronous-arrival window never opens.

## Fix

Make "the displayed row is a stand-in for an unsatisfied request" an explicit, readable state, and make every rebuild
caller aim at the request rather than at the stand-in.

### 1. `src/sase/ace/tui/modals/config_center_session.py`

Add a `provisional: bool = False` field to `SelectionBookmark`. `record()` clears it; `display()` sets it. Update the
class docstring so the flag reads as part of the requested-vs-displayed contract. `rekey()` needs no change.

### 2. `src/sase/ace/tui/modals/tasks_pane_selection.py`

- Add a small helper on `TasksPaneSelectionMixin` that returns the `(identity, row)` a rebuild must aim at:

  ```python
  def _restore_target(self) -> tuple[str | None, int | None]:
      """Return the entry a rebuild must restore: request beats stand-in."""
      bookmark = self._session_state.task
      if bookmark.provisional:
          return bookmark.identity, bookmark.row
      return self._selected_task_identity(), self._highlighted_row()
  ```

  Returning `(None, None)` for a provisional paint with no prior bookmark is deliberate: it lets
  `restore_selection_by_identity` fall through to row 0, which is the newest task.

- In `_rebuild_list`, drop the `identity is not None` requirement from `pending_missing_bookmark` so a pre-store paint
  is provisional whether or not a bookmark exists:

  ```python
  pending_missing_bookmark = not any(
      self._task_identity(task) == identity for task in self._tasks
  ) and (self._store_load_pending or not self._store_loaded_once)
  ```

- In `on_option_list_option_highlighted`, do not let a highlight message promote a stand-in. When `bookmark.provisional`
  is set and the message describes exactly the row the last rebuild painted
  (`(identity, row) == (bookmark.displayed_identity, bookmark.displayed_row)`), record it with `authoritative=False`. A
  genuine keypress moves the cursor off that row, so it still records authoritatively and correctly cancels the pending
  restore.

### 3. `src/sase/ace/tui/modals/tasks_pane_store.py`

Replace the three `prior_identity = self._selected_task_identity()` / `highlighted = self._highlighted_row()` pairs with
`prior_identity, highlighted = self._restore_target()` in `_refresh_running_output`, `_apply_store_snapshot`, and
`action_toggle_scope`. In `_apply_store_snapshot` this also subsumes the `snapshot.unchanged` branch's hand-rolled
`self._session_state.task.identity` lookup, so that branch can use the same pair and lose its special case. Add
`_restore_target` to the mixin's `TYPE_CHECKING` protocol block alongside the other forward declarations.

## Verification of the fix

The design above was prototyped by monkeypatching the four methods and rerunning the reproduction. Every row resumes to
itself, including all three store-backed rows, and a fresh open lands on row 0. The nearest-neighbor fallback also
becomes correct: when the bookmarked store row is deleted from the store between opens, the cursor lands on the row that
took its slot (the behavior `src/sase/ace/tui/util/selection.py` documents) instead of an arbitrary pre-store row.

## Tests

Add regression coverage to `tests/ace/tui/test_tasks_pane_store.py`, using `patch_store_loader(monkeypatch, [...])` with
**non-empty** store rows so the asynchronous-arrival window is actually exercised:

1. `test_tasks_session_restores_selected_store_row_across_modal_instances` — queue holds in-memory tasks, the store
   loader serves at least one `store_task(...)` row. Select the store row, `action_close()` the modal, reopen with the
   same `AdminCenterSessionState`, and assert both the highlighted index and the Output pane content come back to that
   store row. This test fails on the current tree.
2. `test_tasks_fresh_open_selects_newest_row_after_store_rows_arrive` — no prior bookmark; assert the pane settles on
   row 0 of the merged list rather than on the pre-store row 0.
3. `test_tasks_scope_toggle_preserves_store_row_selection` — select a store row, press `a`, and assert the selection
   returns to that row once the re-scoped store read lands.

Extend `tests/ace/tui/test_tasks_pane_selection.py` with a unit-level test that a provisional highlight echo cannot
promote a stand-in over a pending request, so defect (3) has coverage independent of the store timing.

In `tests/ace/tui/test_admin_center_selection_resume.py`, give the `tasks` case real store rows via
`_patch_store_loader` instead of `[]`, so the shared resume matrix stops being blind to store-backed rows. Keep the
other surfaces unchanged.

## Non-goals

- No change to `restore_selection_by_identity` in `src/sase/ace/tui/util/selection.py`. Its contract is already correct;
  it was being fed the wrong identity.
- No caching of `_store_rows` across modal instances. The store read must stay off the event loop, so the pre-store
  window is inherent and the fix has to be correct in the presence of it.
- No change to the Projects, Repos, Workspaces, Logs, Plugins, or XPrompts panes. Their loaders do not paint a
  provisional selection before an asynchronous data source lands, so they do not share this failure. If a reviewer finds
  one that does, file a follow-up task bead rather than widening this change.

## Validation

```bash
just install
just lint
just test
```

Targeted runs while iterating:

```bash
pytest tests/ace/tui/test_tasks_pane_store.py tests/ace/tui/test_tasks_pane_selection.py \
       tests/ace/tui/test_tasks_pane.py tests/ace/tui/test_admin_center_selection_resume.py
```

`just check` is required before reporting completion, per the repo's agent instructions.
