---
tier: tale
title: Restore Admin Center entry selections within the ACE session
goal:
  Reopening SASE Admin Center restores each pane's last logical entry by stable identity, with deterministic fallback
  and no durable entry persistence.
proposed_by: bbugyi200.athena.qy
create_time: 2026-08-01 07:18:06
status: done
---

- **PROMPT:** [prompts/202608/admin_center_session_selection.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/admin_center_session_selection.md)
- **AGENTS:**
  - [bbugyi200.athena.qy](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.qy.md)
- **COMMITS:**
  - [460b8ff](https://github.com/sase-org/sase/commit/460b8ff2356cc7f35357d2207b14f23182eef3c9) — feat(ace): remember admin center entry selections

# Plan: Restore each Admin Center pane's selected entry within the ACE session

## Goal and user-visible contract

Extend the existing Admin Center resume behavior so returning to a working tab restores the logical entry most recently
selected on that tab during the current ACE process. With the default opener, the sequence is:

1. `#` opens the lightweight Admin Center home, as it does today.
2. The repeated `#` resumes the remembered top-level Admin Center tab, as it does today.
3. The newly-created pane for that tab reselects the same logical entry the user had highlighted before closing the
   previous Admin Center modal, if that entry is still available.

The feature applies to every Admin Center surface that has selectable entries:

- **Config:** the highlighted configuration field/section, identified by config path.
- **Logs:** the highlighted log source, identified by `LogSource.id` rather than its current index.
- **Projects:** the active Projects/Repos/Workspaces sub-tab and an independent selected row for each sub-tab. The Repos
  and Workspaces project scopes are also remembered because a scoped row may not be visible in the default inventory.
- **Tasks:** the current-session/all-sessions scope and the highlighted task, identified by
  `durable_task_id or task_id`.
- **Updates:** the active Core/Plugins/Agent CLIs sub-tab, plus independent selected plugin and agent-CLI identities.
  Core has no selectable row.
- **XPrompts:** the highlighted xprompt, identified by its resolved browser name.

The **Statistics** pane has no cursor-selectable entry: its views are rendered static reports with separate range,
grouping, project-filter, and xprompt-focus controls. Do not reinterpret those controls as an entry bookmark or expand
this request into restoring all Statistics query state. Its mounted instance continues to retain those controls while
the current modal is open, exactly as today.

This is selection continuity, not whole-pane suspension. New modal instances and fresh data must still be constructed on
reopen. Text filters, marks, detail scroll positions, pending operations, loaded models, Statistics scopes, and all
other pane-local state remain modal-lifetime-only unless explicitly listed above because it is required to make a
bookmarked entry visible.

## Current behavior and constraints

`ConfigCenterModal` lazily creates a pane on first entry and keeps it in `_panes` only until that modal closes. That
preserves most pane-local cursors when switching top-level tabs inside one open modal, but every pane is constructed
from defaults after closing and reopening. `TasksPane.focus_default()` is an additional exception: it currently rebuilds
with `highlight_index=0`, so even an in-modal tab switch snaps Tasks to the first row.

The existing top-level `AdminCenterTabHistory` is app-owned and written to `~/.sase/ace_admin_center_last_tab.txt`; it
may survive process restarts. Entry bookmarks must have a deliberately different lifetime: one `AceApp` instance only. A
new ACE process gets empty entry memory even if the persisted top-level tab causes `##` to resume a section.

All list restoration must follow the repository-wide invariant documented in `src/sase/ace/tui/util/selection.py`:
restore the same logical identity first; if it disappeared, retain the nearest valid visual row; if neither exists,
choose the first row; clear the bookmark when the authoritative data is genuinely empty. Never restore by index alone,
because Tasks, plugins, inventories, xprompts, and projects can reorder between modal instances or during refresh.

## Design decisions

### 1. Add one bounded, mutable, app-session state object

Create `src/sase/ace/tui/modals/config_center_session.py` containing small presentation-only dataclasses:

- `SelectionBookmark(identity: str | None, row: int | None)` stores a stable identity and its logical selectable-row
  ordinal. “Logical row” excludes disabled group headers and empty/loading placeholders.
- `ProjectsSessionState` stores the active sub-tab, one bookmark each for Projects/Repos/Workspaces, and independent
  Repos/Workspaces project-filter keys.
- `TasksSessionState` stores `all_sessions` and its task bookmark.
- `UpdatesSessionState` stores the active sub-tab and bookmarks for Plugins and Agent CLIs.
- `AdminCenterSessionState` owns bookmarks for Config, Logs, and XPrompts plus the three nested state objects above.

Keep the state typed and bounded to a fixed number of scalar fields; do not retain pane widgets, record objects, worker
results, rendered content, lists, or callbacks. Put the `ProjectsSubTab` and `UpdatesSubTab` literal types alongside
this state so the session module does not import concrete pane modules and create cycles.

Give a directly-constructed `ConfigCenterModal` a fresh default `AdminCenterSessionState`, preserving isolated tests and
non-ACE hosts. In `_state_init.py`, explicitly initialize `AceApp._admin_center_session_state` to a fresh instance. In
`BaseActionsMixin._open_config_center()`, pass that same object to every new modal, including direct-entry actions. The
tab catalog factories then pass only the relevant bookmark or nested state to each pane constructor.

Pane code may update the supplied mutable bookmark/state synchronously when its highlight or visibility context changes.
These updates are constant-time assignments on the UI thread. No persistence callback, dismissal-time scrape, disk
write, flush, timer, or background task is needed. This also means every close path—including Updates actions that close
the containing Admin Center—has already recorded the current cursor before teardown.

Do not alter `config_center_state.py`, the coalescing top-level history writer, its controlled-exit flush, or the
on-disk format. Top-level resume/alternate history and per-entry session bookmarks are intentionally separate concepts.

### 2. Centralize identity-first fallback behavior

Every list-bearing pane must use `restore_selection_by_identity()` while rebuilding. Add focused pane-local helpers for
converting between a logical selectable-row ordinal and the actual Textual widget index where a widget includes disabled
headers. After restoration settles, record the resolved identity and logical row back into the bookmark. If the saved
identity vanished, this replaces stale memory with the deterministic neighbor that is now visibly selected.

Programmatic `OptionList.highlighted = ...` assignments must be wrapped in synchronous guard flags, cleared in
`finally`, because Textual emits `OptionHighlighted` echoes. Explicitly synchronize the bookmark and detail pane after
the guarded assignment instead of depending on a queued echo. Existing `_syncing_options` guards in Logs, Projects,
inventory, and Tasks should be retained and used for this purpose; add equivalent protection to XPrompts and both
Updates lists. Config uses its tree path/detail update as the single synchronization point.

For asynchronous or staged loads, do not erase a requested bookmark merely because an initial/loading snapshot is empty.
Keep the desired identity and row until the pane receives authoritative data:

- Config and Logs resolve after their worker result is applied.
- Repo/Workspace inventories resolve after the inventory worker returns.
- Updates resolves plugin and agent-CLI bookmarks after the catalog result is rendered.
- Tasks may first show in-memory rows and later merge durable store rows. Preserve the requested bookmark through that
  first store load so a durable-only task is not replaced by row zero before it arrives; only the complete merged
  snapshot may finalize the fallback or clear an empty bookmark.

When a worker completes, re-read the current widget/session selection on the UI thread before applying the result rather
than trusting a pre-`await`/pre-worker index. Preserve the existing exclusive/coalesced/last-request-wins worker
behavior and never add I/O to a highlight, focus, render, or key handler.

### 3. Wire each pane to stable identities

#### Config

Seed `ConfigPane._selected_path` from its bookmark. In `_rebuild_tree()`, use the ordered visible path sequence with the
shared restoration helper; move the cursor to the identity hit or fallback node, update detail, and write the resolved
path/logical row. `on_tree_node_highlighted()` continues to update detail and now records the user-selected path and
row. If a saved path is gone after a completed inventory build, fall back deterministically. Do not preserve the filter,
modified-only toggle, edit mode, or loaded view.

#### Logs

Translate the bookmarked `LogSource.id` to the current `log_sources()` ordering before the off-thread detail read, and
return/retain stable source identity alongside any index needed for rendering. Rebuild under the existing guard, then
record the resolved source ID and logical row explicitly. Refresh and jump-mode navigation must update the same
bookmark; the jump back-stack itself remains pane-local.

#### Projects, Repos, and Workspaces

Initialize `ProjectsPane._active_subtab` from `ProjectsSessionState` before composition so the switcher and strip mount
on the prior sub-tab. Seed each list with its own bookmark and use project name, repo record ID, and workspace record ID
as the respective identities. Continue using the existing composite inventory IDs unless a focused test exposes a
collision.

Record active-sub-tab changes for keyboard and mouse navigation. Restore each inventory pane's saved project filter
before applying filters; validate it against current project records and clear it if the project disappeared. Keep a
saved inventory bookmark pending until the inventory worker returns, then resolve it within the restored scope. Rebuilds
after enable/disable/delete, scope changes, marks, filters, and reloads must update bookmarks to the actually
highlighted logical row while keeping the existing debounced detail rendering.

#### Tasks

Replace index-only rebuilding with identity plus prior-row restoration throughout mount, focus, periodic status changes,
store merges, scope changes, dismiss/kill completion, and manual refresh. `focus_default()` must focus and refresh
without forcing row zero, fixing the current in-modal cursor reset as part of the requested contract. Seed and update
`_all_sessions` from `TasksSessionState`; this is required for a task selected from another session to remain visible on
reopen. Preserve the existing 250 ms spinner cadence, off-thread durable-store reads, body cache, live-follow behavior,
and scroll semantics.

#### Updates

Initialize the active Core/Plugins/Agent CLIs sub-tab from `UpdatesSessionState` before composition and record every
sub-tab switch. Reuse plugin name and agent-CLI name as stable identities, but carry logical row ordinals too for
missing entry fallback. Seed the existing plugin reload restoration path from the session bookmark and add the
equivalent pending restore for the asynchronously populated agent-CLI list. Guard both lists during programmatic
rebuilds, then explicitly update hints/details/bookmarks. Marks, filter text, offline/verbose modes, incoming-commit
caches, and pending update operations remain modal-local.

#### XPrompts

Restore by xprompt name after the synchronous catalog/group build, mapping the logical xprompt row around disabled group
headers. Add a programmatic-highlight guard, explicitly refresh preview/meta/hints, and record the resolved name and
row. Filtering, reloads, adds, edits, and deletion must either retain the identity or choose the shared neighbor
fallback. Continue focusing the filter input on tab activation; “reselected” means the row highlight and preview are
restored, not that the filter loses its established browse-first behavior.

### 4. Preserve lazy opening and presentation boundaries

This feature must not change the first-`#` path: opening home constructs zero concrete panes and performs no pane data
loads. Session state initialization is O(1) and happens with app state, while each bookmark is read only by that pane's
existing lazy factory.

Keep the implementation in the Python/Textual frontend. Other frontends do not need to share an ACE modal cursor, so no
Rust-core API or binding belongs here. Do not add configuration fields, default keymaps, a second persistence file, JSON
state, database rows, or eager imports. Default initial selections remain the first valid entry, so no visual snapshot
golden should change when no bookmark exists.

## Implementation surfaces

Add:

- `src/sase/ace/tui/modals/config_center_session.py` for the bounded session state and sub-tab literal types.
- `tests/ace/tui/test_config_center_session.py` for pure state defaults/independence and shared-modal wiring coverage.

Modify the focused integration/lifecycle code:

- `src/sase/ace/tui/actions/_state_init.py` and `src/sase/ace/tui/actions/base.py` to own and pass one state object per
  `AceApp` instance.
- `src/sase/ace/tui/modals/config_center_modal.py` and `config_center_catalog.py` to accept the state and inject the
  correct portion only when a pane is lazily created.

Modify the selectable pane code and its existing focused test modules:

- Config: `config_pane_widget.py` and `test_config_pane_widget.py` / navigation tests.
- Logs: `logs_pane.py` and `test_logs_pane.py`.
- Projects/inventories: `projects_pane.py`, `project_list_controller.py`, `project_inventory_panes.py`,
  `test_projects_pane.py`, and `modals/test_project_inventory_subtabs.py`.
- Tasks: `tasks_pane.py` and `test_tasks_pane.py`.
- Updates: `plugins_browser_pane.py`, `plugins_browser_rendering.py`, `plugins_browser_controls.py`,
  `plugins_browser_agent_clis.py`, and the existing pane/loading/agent-CLI tests and helpers as needed.
- XPrompts: `xprompt_browser_pane.py` and the existing Admin Center XPrompt browser tests.
- End-to-end resume behavior: extend `test_config_center_resume.py` without weakening its zero-pane home, direct-entry,
  custom-opener, failure, or persistence assertions.

Update `docs/ace.md` and `docs/configuration.md`, replacing the statements that all selections end with the modal. Make
the lifetime split explicit: the top-level resume/alternate pair remains machine-local and durable; logical entry
bookmarks and the minimal scope/sub-tab context needed to show them last only for the current ACE process; all other
pane state still ends when the modal closes.

## Behavioral coverage

Add tests that exercise behavior rather than private assignment alone:

1. Two distinct Admin Center modal instances created by one app share the same session-state object; two different app
   instances do not. A home-only open/close neither creates panes nor mutates entry bookmarks.
2. Select a non-first task, close the modal, reopen home, repeat the opener, and assert the new `TasksPane` instance
   highlights the same task and renders its output. Verify the selected top-level tab still comes from the existing
   history path and no nested modal is created.
3. For every selectable surface listed in the contract, create/reopen or remount with reordered fixtures and assert the
   stable identity wins over its old index. Include active Projects/Updates sub-tab restoration and Tasks scope.
4. For every list family, remove the saved identity and assert `restore_selection_by_identity()` semantics: the prior
   logical row is clamped to the nearest surviving row, the bookmark is replaced with that row's identity, and an
   authoritatively empty result clears it without highlighting placeholders.
5. Cover staged loads explicitly: durable-only Tasks, Repo/Workspace inventory rows, plugin/agent-CLI rows, Config, and
   Logs do not lose the requested identity during loading/empty placeholders before the authoritative result arrives.
6. Switching away from and back to Tasks in one open modal no longer resets it to row zero. Other panes retain their
   existing modal-lifetime behavior.
7. Programmatic list rebuilds do not cause stale `OptionHighlighted` echoes to overwrite a newer user selection or
   schedule redundant detail work; user-driven keyboard and mouse highlights still update bookmarks immediately.
8. XPrompts and grouped plugin rows calculate fallback ordinals from selectable items, not disabled headers. Their
   restored row and detail/preview agree.
9. Existing navigation failures, tab-history persistence/coalescing, direct-entry actions, input key routing, refresh
   behavior, marks, and pane actions continue to pass unchanged.

No new PNG golden is expected. Run the visual suite to prove default fixtures remain byte-identical; if a golden
changes, investigate the default selection/focus regression rather than accepting it automatically.

## Performance and validation

Before implementation, run `just install` as required for the ephemeral workspace. During implementation, run focused
tests for each affected pane and the Admin Center lifecycle. Then run:

1. `pytest -s -m slow tests/ace/tui/bench_admin_center_open.py` and confirm both empty and populated cases still enforce
   zero working panes and fixture-size-independent home opening.
2. `just test-visual` with no snapshot-update flag; all existing Admin Center goldens should pass unchanged.
3. The repository-required `just check`.

Audit the final diff for synchronous I/O in handlers, new timer/pump work, accidental durable entry persistence, broad
state retention, or index-only restoration paths. Selection changes should remain instant; expensive detail panels must
continue through their existing debouncers.

## Acceptance criteria

- Within one ACE process, closing and reopening Admin Center and repeating its opener returns to the remembered
  top-level tab and reselects that tab's last logical entry when one exists.
- Config, Logs, all three Projects sub-tabs, Tasks, both selectable Updates sub-tabs, and XPrompts restore by stable
  identity; reordered rows do not change what is selected.
- The active Projects/Updates sub-tab, inventory project scope, and Tasks session scope are restored only where needed
  to make the selected entry visible.
- Missing entries use the shared nearest-row fallback and empty authoritative data clears the bookmark; loading
  placeholders never prematurely destroy it.
- Tasks no longer resets to its first row on an ordinary in-modal tab round trip.
- Entry bookmarks reset in a new ACE process and are never written to the existing Admin Center history file or any new
  durable store.
- First-`#` home opening remains lazy, zero-pane, and I/O-free; current keyboard, mouse, direct-entry, alternate-tab,
  update, filter, mark, and refresh behavior remains intact.
- Focused tests, the home benchmark, unchanged visual snapshots, and `just check` pass.
