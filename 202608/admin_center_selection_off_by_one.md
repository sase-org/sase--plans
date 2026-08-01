---
tier: tale
title: Fix Admin Center resumed-selection drift to the neighboring entry
goal:
  Resuming an Admin Center tab within one ACE process leaves the visible cursor on the entry that was selected before,
  on every selectable surface, with index fallback used only when the remembered entry is genuinely gone.
proposed_by: bbugyi200.athena.qy.f0
create_time: 2026-08-01 09:10:10
status: wip
---

- **PROMPT:** [prompts/202608/admin_center_selection_off_by_one.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/admin_center_selection_off_by_one.md)

# Plan: Fix the Admin Center resumed-selection drift ("entry below" bug)

## Verdict on the reported hypothesis

The suspicion "we remember an index that is off by one" is **denied as a literal description of the bookmark
bookkeeping, and confirmed as a description of the observable behavior**. Nothing in
`src/sase/ace/tui/modals/config_center_session.py` or any pane stores `row + 1`. Every pane records a stable identity
plus a logical row that is computed from the same sequence the restore reads back.

I drove the real user flow (`#` → switch to tab → `j` several times → `escape` → `#` `#` resume) against these surfaces
and the resumed pane highlighted exactly the previously selected entry every time:

- Projects (`#projects-list`): selected row 3 `delta` → resumed row 3 `delta`.
- Projects → Repos inventory sub-tab: selected row 3 `alpha:linked:repo-3:…` → resumed row 3, same record, correct
  sub-tab.
- Updates → Plugins (grouped list with disabled headers): selected widget row 3 / logical row 2 `telegram` → resumed
  widget row 3 `telegram`.
- XPrompts (grouped list with disabled headers): selected `item__p3` → resumed `item__p3`.
- Tasks with durable store rows merged in: selected `store-0` at row 3 → resumed row 3 `store-0`.

So the drift the user sees is **not** an arithmetic error. It is the shared helper degrading from identity restoration
to index restoration. `restore_selection_by_identity()` in `src/sase/ace/tui/util/selection.py` falls back to
`prior_visual_row` whenever `prior_identity` is not found in the new sequence, and that row now names whichever entry
slid into the vacated slot — normally the entry that was directly **below** the remembered one. The user's instinct is
right: we are restoring by index. The bug is that we drop to the index path far more often than the design intended.

Three independent defects push the panes onto that index path, and a fourth hides them from the test suite. All four
were reproduced or proven mechanically; details below.

## Confirmed root causes

### A. The Config tree cursor is never moved to the restored entry

`ConfigPane._rebuild_tree()` in `src/sase/ace/tui/modals/config_pane_widget.py` clears the tree, re-adds every visible
node, resolves `target` through `restore_selection_by_identity()`, and then calls `tree.move_cursor(target_node)`.

Textual's `Tree.move_cursor()` assigns `self.cursor_line = node._line`, and `TreeNode._line` is only populated by
`Tree._build()`. Every `TreeNode.add()` / `add_leaf()` call runs `Tree._invalidate()`, which clears the line cache, so
at the moment `move_cursor()` runs, `target_node._line` is still `-1`. The `cursor_line` reactive's
`validate_cursor_line()` clamps `-1` to `0` (and, as a side effect of reading `self._tree_lines`, is what finally
triggers `_build()`).

Reduced Textual-only reproduction (clear → re-add → `move_cursor` on the last node of a 7-line tree):

```
naive: target._line before move_cursor = -1
naive: cursor_line = 0 cursor = None
fixed: target._line before move_cursor = 6     # after touching tree.last_line
fixed: cursor_line = 6 cursor = b.y
```

Live Admin Center reproduction (real config schema, resume flow):

```
open#1:   cursor_line=0 cursor=ace    selected=ace.agents_sync.check_interval_minutes  bookmark=(…check_interval_minutes, row=2)
after 6j: cursor_line=6 cursor=ace.artifact_file_viewer.video  selected=ace.artifact_file_viewer.video  bookmark=(…video, row=6)
reopen#1: cursor_line=0 cursor=ace    selected=ace.artifact_file_viewer.video           bookmark=(…video, row=6)
reopen#1 + j: cursor_line=1 cursor=ace.agents_sync  selected=ace.agents_sync            bookmark=(ace.agents_sync, row=1)
```

So on Config the highlight always returns to the first row while the detail pane and the bookmark hold the correct
entry, and the very first `j`/`k` after resuming moves relative to row 0 and **overwrites the remembered entry with row
1**. The mismatch predates the session-bookmark feature (the old code moved the cursor the same way, and even a first
open lands the cursor on row 0 while the detail shows the first leaf at row 2), but the feature is what made it
user-visible.

### B. Every programmatic-rebuild guard is a no-op

`_syncing_options` (Logs, Tasks, Projects, inventory panes, XPrompts), `_syncing_plugin_options`, and
`_syncing_agent_cli_options` are set before a programmatic `OptionList.highlighted = …` assignment and cleared in a
synchronous `finally`. Textual's `OptionList.watch_highlighted()` publishes the change with `post_message()`, so
`OptionHighlighted` is dispatched from the message pump _after_ that `finally` has already run.

Measured directly on the Updates → Plugins pane by spying on `on_option_list_option_highlighted`:

```
guard set immediately after rebuild: False
guard values observed inside echo handler: [False]
```

Consequences:

1. The "do not clobber a bookmark while data is still loading" protections are bypassed. `TasksPane._rebuild_list()`
   computes `pending_missing_bookmark` and deliberately skips `_record_bookmark()`, but the echo then reaches
   `on_option_list_option_highlighted` with the guard already down and records the placeholder row anyway. The same
   applies to `_InventoryPaneBase._loaded_once` and the Updates agent-CLI loading guard.
2. `TasksPane` option ids are **positional** (`Option(..., id=str(i))`) and its handler resolves them with
   `int(event.option.id)` against the _current_ `self._tasks`. A late echo from a previous rebuild therefore indexes a
   different list and records a neighboring task's identity under the old row — precisely the "entry below" shape when
   an entry above the selection disappeared between rebuilds.
3. Detail-panel work the guard was meant to skip is scheduled on every rebuild.

### C. The Tasks identity is not stable within a session

`TasksPane._task_identity()` returns `task.durable_task_id or task.task_id`. Per the comment on
`TaskInfo.durable_task_id` in `src/sase/ace/tui/task_queue.py`, that field is _minted later_, when the task mirror
starts tracking an in-TUI task. A task highlighted shortly after launch is bookmarked under `task_id`; by the time the
user resumes, the same row answers `durable_task_id`. The lookup misses and the pane falls back to the row.

### D. No test asserts the visible cursor

`test_config_pane_restores_session_bookmark_by_path` in `tests/ace/tui/test_config_pane_widget.py` asserts
`pane._selected_path` and `state.config.identity` — exactly the two values that stay correct while the cursor is wrong.
The `OptionList` tests assert `option_list.highlighted`, so B and C can silently reroute a bookmark without failing
anything, and no existing test walks the real `#`/`##` resume path for more than one tab.

## Fixes

### 1. Restore the Config tree cursor for real

In `ConfigPane._rebuild_tree()`, force Textual to rebuild its line cache before moving the cursor, then move it:

```python
_ = tree.last_line          # public property; triggers Tree._build() and assigns every node._line
tree.move_cursor(target_node)
```

`last_line` is documented public API (`len(self._tree_lines) - 1`), so this stays inside Textual's supported surface and
needs no private access. Do the same anywhere else the pane moves the cursor immediately after mutating the tree;
`_move_cursor()` (jump/edit paths) already runs against a built tree and can stay as it is, but verify that with a test
rather than by inspection.

After the cursor genuinely lands on the restored node, revisit `_suppress_post_rebuild_highlight`. It exists only to
swallow the bogus row-0 `NodeHighlighted` echo caused by defect A; once the cursor is correct the post-rebuild echo
names the node we already selected. Prefer deleting the flag and its four clear-sites (`action_cycle_cursor_down`,
`action_cycle_cursor_up`, `_tree_action`, `_move_cursor`) and keeping only the existing
`event.node is not tree.cursor_node` staleness check, which is the correct and self-describing guard. If removing it
regresses a test, keep it, but document precisely which echo it still absorbs — do not leave an unexplained suppression
flag behind.

Also drop the now-unnecessary asymmetry in `_rebuild_tree()` where `prior_row` is only honored when
`prior_identity is not None`; with the cursor fixed, the two should be passed to `restore_selection_by_identity()`
exactly as recorded.

### 2. Make the rebuild guards work against asynchronous echoes

A synchronous flag cannot gate an asynchronously posted message, so replace the flag pattern rather than widening it.
Introduce one small shared helper (a good home is `src/sase/ace/tui/util/selection.py`, next to the invariant it
protects) that panes use around a programmatic highlight:

- Before assigning, record the _intended_ selection as `(identity, row)` on the pane.
- In `on_option_list_option_highlighted`, resolve the event to an identity from the pane's current data (never from a
  positional option id), compare it with the intended selection, and if they match treat the event as the echo of that
  assignment: clear the intent and return without re-recording a bookmark the rebuild deliberately withheld. If they
  differ, it is user-driven (or a genuinely newer state) and follows the normal record-and-render path.
- Clear the intent when the pane rebuilds again, so a stale intent can never swallow a real user move.

Apply it to every list that currently uses a `_syncing_*` flag for this purpose: `logs_pane.py`, `tasks_pane.py`,
`project_list_controller.py`, `project_inventory_panes.py`, `plugins_browser_rendering.py`,
`plugins_browser_agent_clis.py`, `xprompt_browser_pane.py`. Keep the flags where they guard something other than
bookmark recording (for example suppressing debounced detail repaints) only if a test proves that they still do.

While in `tasks_pane.py`, stop resolving highlights positionally: give each option a stable id derived from the task
identity (as Logs already does with `log__{source.id}` and plugins with `plugin__{name}`) so a late echo can never name
a different task. Keep any numeric index the jump/hint code needs internal to the pane.

### 3. Preserve a pending bookmark through staged loads

With guards that actually work, the existing "don't clobber during loading" logic becomes effective, but make the intent
explicit rather than implicit in flag bookkeeping. Give `SelectionBookmark` a way to distinguish _what the user asked
for_ from _what is currently displayed_: keep the requested identity/row until an authoritative snapshot either finds it
or proves the list no longer contains it. Concretely, the requested value must survive:

- `TasksPane` mounting with only in-memory rows before the first durable-store merge.
- `_InventoryPaneBase` showing its "Loading … inventory…" placeholder before the worker returns.
- The Updates pane's agent-CLI list before the catalog result is rendered.
- `LogsPane` and `ConfigPane` before their worker results are applied.

An authoritatively empty result still clears the bookmark; a loading placeholder never does.

### 4. Make the Tasks identity stable

`_task_identity()` must not change value for the same row over the life of a session. Fix it at the source: when a task
gains a `durable_task_id`, re-key any bookmark that still points at its `task_id` (a single assignment in the pane the
next time it rebuilds), or make the lookup accept either key. Prefer the re-key: it keeps one identity per bookmark and
avoids teaching `restore_selection_by_identity()` about alternate keys. Cover it with a test that bookmarks a task
before its durable id is minted, mints the id, rebuilds, and asserts the same row stays selected.

## Tests

The existing suite passes today while the bug is live, so the coverage change is as important as the code change.

1. **Visible-cursor assertions for Config.** Extend `tests/ace/tui/test_config_pane_widget.py` (and
   `tests/ace/tui/_config_pane_widget_helpers.py` if a helper is needed) so bookmark restoration asserts
   `tree.cursor_line` and `tree.cursor_node.data`, not only `pane._selected_path`. Add the round trip: select a
   non-first path, close, reopen, assert the cursor is on that path, then press `j` once and assert the cursor moved to
   the _next_ row after it — the assertion that fails today.
2. **One shared real-flow resume harness.** Add a test module that walks `#` → activate tab → move the highlight with
   real key presses → `escape` → `#` `#`, and asserts the resumed pane's _visible_ selection is the same entry, for
   every selectable surface: Config, Logs, Projects, Projects→Repos, Projects→Workspaces, Tasks, Updates→Plugins,
   Updates→Agent CLIs, XPrompts. Parameterize it so a new selectable surface is one table entry. This is the regression
   net that would have caught A and would catch any future divergence between the bookmark and the widget.
3. **Echo-timing regression.** A focused test that asserts a programmatic rebuild does not record a bookmark it
   deliberately withheld: seed a bookmark whose identity is absent from the first (loading) snapshot, pump the event
   loop so the echo is dispatched, assert the requested identity survives, then deliver the authoritative snapshot and
   assert it is restored. Do this for Tasks and for one inventory pane.
4. **Stale-echo regression for Tasks.** Rebuild the task list twice in a row with a different ordering, pump, and assert
   no bookmark names a task other than the one highlighted.
5. **Identity-miss fallback stays correct and documented.** Keep asserting the intended fallback: when the remembered
   entry is genuinely gone, the nearest surviving row is selected and the bookmark is replaced with that row's identity.
   This is the behavior that _looks_ like the reported off-by-one, so name it explicitly in the test so a future reader
   does not "fix" it.
6. **Durable-id re-key test** as described in fix 4.

Re-run the modules the feature commit touched: `tests/ace/tui/test_config_pane_widget.py`,
`tests/ace/tui/test_config_center_session.py`, `tests/ace/tui/test_logs_pane.py`, `tests/ace/tui/test_projects_pane.py`,
`tests/ace/tui/modals/test_project_inventory_subtabs.py`, `tests/ace/tui/test_tasks_pane.py`,
`tests/ace/tui/test_plugins_browser_pane_agent_clis.py`, `tests/ace/tui/test_plugins_browser_pane_loading.py`,
`tests/ace/tui/test_xprompt_browser_load_keymap.py`, and `tests/ace/tui/test_config_center_resume.py`.

## Scope boundaries

- Do not change the durable top-level tab history (`config_center_state.py`, `config_center_history.py`, or
  `~/.sase/ace_admin_center_last_tab.txt`). Entry bookmarks stay process-local.
- Do not add configuration fields, keymap entries, or a second persistence file.
- Do not widen the feature: Statistics still has no cursor-selectable entry, and no additional pane state becomes
  session-scoped.
- Keep the change in the Python/Textual frontend. This is presentation-layer cursor behavior, so nothing belongs in
  `../sase-core`.
- Preserve lazy opening: the first `#` must still construct zero working panes and perform no pane data loads.
- Do not add I/O to a highlight, focus, render, or key handler, and keep the existing debouncers and exclusive/coalesced
  worker behavior.

## Validation

1. `just install` first (ephemeral workspace).
2. Focused runs for each affected module during implementation.
3. `pytest -s -m slow tests/ace/tui/bench_admin_center_open.py` — both cases must still enforce zero working panes and
   fixture-size-independent home opening.
4. `just test-visual` with no update flag. A Config golden **may** legitimately change here: today's snapshots encode
   the tree cursor sitting on row 0. If a golden moves, confirm the new image shows the cursor on the restored/first
   leaf row and matches the detail panel before accepting it with `--sase-update-visual-snapshots`, and say so in the
   summary. Any other golden change is a regression.
5. `just check`.

## Acceptance criteria

- On every selectable Admin Center surface, resuming a tab within one ACE process leaves the **visible** cursor on the
  same entry that was selected before, and the detail/preview panel agrees with it.
- The Config tree cursor lands on the restored path; pressing `j` immediately after resuming moves to the row after that
  path, not to row 1.
- A programmatic rebuild never records a bookmark the rebuild deliberately withheld, and a late echo can never record an
  entry other than the one currently highlighted.
- A task bookmarked before its `durable_task_id` is minted is still restored afterwards.
- When the remembered entry is genuinely gone, the documented nearest-row fallback still applies and the bookmark is
  replaced with the surviving row's identity.
- Entry bookmarks remain process-local; a new ACE process starts with empty entry memory.
- Focused tests, the new resume harness, the home-open benchmark, the visual suite, and `just check` all pass.
