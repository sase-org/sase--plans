---
create_time: 2026-07-09 13:52:52
status: done
prompt: 202607/prompts/multi_commit_view_navigation.md
tier: tale
---
# Plan: Multi-Commit Navigation in the Commit View Modal

## 1. Problem and Goal

The current Agents-tab `v` hint flow treats commit hints as valid view targets, but default selection is effectively
single-select: when the user enters multiple commit hint numbers, ACE opens the first commit in `CommitViewModal` and
shows a warning toast that the remaining commit selections were ignored.

Goal: make multi-commit selection useful. When the user selects more than one commit hint, ACE should open one commit
viewer modal containing the selected commits as an ordered navigation set. Inside that modal:

- `ctrl+n` moves to the next selected commit.
- `ctrl+p` moves to the previous selected commit.
- Navigation wraps, matching the existing file-panel `ctrl+n` / `ctrl+p` behavior.
- The modal never blocks the Textual event loop while loading commit diffs.

The existing single-commit behavior should continue to work, and the `%` and `@` suffix flows should keep their current
multi-selection behavior: `%` copies all selected commit SHAs, and `@` opens all available raw diff files.

## 2. Current Findings

- Multi-commit default handling is centralized in `src/sase/ace/tui/actions/hints/_processing.py`: `_open_commit_hint()`
  opens the first selected commit and emits the "ignored extra commit selection(s)" toast.
- Commit metadata is already available as `CommitViewSpec` values from `_hint_commit_views`.
- `CommitViewModal` currently owns a single `CommitViewSpec`, starts one thread worker on mount, and updates the diff
  body when the worker reaches a terminal state.
- Existing TUI performance rules require disk reads / VCS fallback calls to stay off the event loop. The current modal
  already follows this by using a Textual worker; the multi-commit design should preserve that shape.
- `ctrl+n` / `ctrl+p` are established local navigation bindings in ACE modals and file panels. File-panel navigation
  wraps around, so the commit modal should do the same.

## 3. User Experience

1. User presses `v` on the Agents tab and enters multiple commit hints, for example: `3 4 8` or `3-5`.
2. ACE opens one `CommitViewModal`, initially showing the first selected commit in the parsed selection order.
3. The modal title/footer show that this is a navigable commit set, for example:
   `COMMIT 2/3  sase  abcdef123456 - subject`
4. `ctrl+n` switches to the next selected commit; `ctrl+p` switches to the previous selected commit.
5. On navigation, the modal updates the title and message immediately, resets scroll to the top, and shows either the
   cached diff or a "Loading diff..." state while the selected commit's diff is loaded off-thread.
6. With only one selected commit, `ctrl+n` and `ctrl+p` are no-ops. The footer can omit or de-emphasize commit
   navigation in the single-commit case.

No warning toast should be shown merely because multiple commit hints were selected.

## 4. Technical Design

### 4.1 Dispatch multiple commit specs instead of dropping extras

Update `_process_view_input()` / `_open_commit_hint()` in `actions/hints/_processing.py` so default commit selection
builds an ordered `tuple[CommitViewSpec, ...]` from the selected commit hint numbers and passes the entire tuple into
the modal.

Important details:

- Preserve the parser's selected-hint order, including range expansion order, instead of sorting by SHA or repo.
- Keep duplicate hint numbers deduped by the existing parser behavior.
- Preserve existing suffix behavior:
  - `1 2%` copies both SHAs.
  - `1 2@` opens both available raw diff paths and warns only for commits without a diff path.
- Keep mixed commit+file default behavior scoped to current behavior unless tests reveal an existing bug. The commit
  sequence should include only selected commit hints; selected file hints should continue through the normal file
  routing path.

### 4.2 Make `CommitViewModal` sequence-aware

Change `CommitViewModal` from "one spec" to "one or more specs":

- Store `_commit_specs: tuple[CommitViewSpec, ...]`.
- Store `_current_index: int`.
- Add a small `_current_spec` property used by title/content/copy/diff loading helpers.
- Validate that the sequence is non-empty and clamp/validate the initial index.

The constructor can be updated to a sequence-based API, such as:

```python
CommitViewModal(commit_specs: Sequence[CommitViewSpec], initial_index: int = 0)
```

Then update all existing call sites and tests to pass a one-item tuple/list for single-commit usage.

### 4.3 Add modal-local commit navigation bindings

Add modal-local bindings to `CommitViewModal.BINDINGS`:

- `("ctrl+n", "next_commit", "Next Commit")`
- `("ctrl+p", "prev_commit", "Previous Commit")`

Implement:

- `action_next_commit()`
- `action_prev_commit()`
- a shared `_select_commit(index: int)` helper that wraps around with modulo arithmetic when more than one commit is
  present.

No `default_config.yml` keymap change is needed because these are modal-local Textual bindings, consistent with other
ACE modals that bind `ctrl+n` / `ctrl+p` locally.

### 4.4 Keep diff loading asynchronous and race-safe

Refactor the current single-worker state into per-selected-commit state:

- `_diff_text_by_index: dict[int, str | None]` caches loaded diffs, including unavailable results.
- `_loading_index: int | None` tracks the commit currently being loaded.
- `_diff_worker: Worker[str | None] | None` remains the active worker handle.

When selecting a commit:

1. Update title/message immediately from in-memory `CommitViewSpec`.
2. Reset `_diff_loaded` / `_diff_text` from cache if present.
3. If uncached, show "Loading diff..." and start a new thread worker for `load_commit_diff_text(current_spec)`.
4. Ignore stale worker completions if the worker is no longer the active worker or its captured index no longer matches
   the selected commit.
5. Cache successful `str | None` results by index and update the body only if the modal is still attached.

This avoids wrong-diff races when a user presses `ctrl+n` repeatedly while earlier diff loads are still in flight. It
also avoids repeat disk/VCS reads when cycling back to a previously viewed commit.

### 4.5 Refresh title, body, footer, and scroll position

On commit navigation:

- Rebuild `#commit-view-title`.
- Rebuild `#commit-view-content`.
- Reset `#commit-view-scroll` to the top.
- Keep `y` copying the full SHA for the currently visible commit.
- Update the footer to include `ctrl+n/p commit` when more than one commit is available.

The existing message header, change summary, `lazy_renderable(..., "diff", theme="monokai")`, and unavailable-diff
states should be reused unchanged.

## 5. Files to Touch

Expected production files:

- `src/sase/ace/tui/actions/hints/_processing.py`
  - Replace the "open first and warn" behavior with sequence construction.
  - Keep `%` and `@` suffix paths intact.
- `src/sase/ace/tui/modals/commit_view_modal.py`
  - Make the modal sequence-aware.
  - Add `ctrl+n` / `ctrl+p` actions.
  - Add diff cache and stale-worker guards.
- `src/sase/ace/tui/modals/__init__.py`
  - Only if constructor/export usage needs adjustment.

Expected tests / snapshots:

- `tests/ace/tui/actions/test_view_files_image.py`
- `tests/ace/tui/modals/test_commit_view_modal.py`
- `tests/ace/tui/visual/test_ace_png_snapshots_preview_panel.py`
- `tests/ace/tui/visual/snapshots/png/commit_view_modal_120x40.png` or a new multi-commit golden

No Rust core changes are needed. No generated memory files, `AGENTS.md`, or provider instruction shims should be
modified.

## 6. Testing Plan

Focused tests:

- Dispatch:
  - Selecting `1 2` where both hints are commits pushes one `CommitViewModal` with both specs in order.
  - No "ignored extra commit selection" warning is emitted.
  - `%` still copies all selected SHAs.
  - `@` still opens all available raw diff paths.
- Modal navigation:
  - A modal initialized with two commits starts at index 0.
  - `ctrl+n` moves to commit 2 and loads/displays its diff.
  - `ctrl+p` moves back to commit 1.
  - Navigation wraps at both ends.
  - `y` copies the SHA for the currently visible commit.
  - Single-commit `ctrl+n` / `ctrl+p` is a no-op.
- Diff loading:
  - Navigating back to a previously loaded commit uses the modal's cache.
  - A stale worker completion cannot overwrite the currently selected commit's diff.

Visual coverage:

- Update or add a PNG snapshot that shows the commit modal in a multi-commit state with the `1/N` title/footer
  affordance.

Verification:

- Run the focused dispatch/modal/visual tests first.
- Because implementation will modify files in this repo, run `just install` if needed and finish with `just check`.

## 7. Risks and Mitigations

- Race risk: a slower diff worker could finish after the user navigates. Mitigate with active-worker and captured-index
  checks before applying results.
- UX ambiguity for mixed file+commit default selections: keep the existing file routing behavior scoped and only change
  the multi-commit modal behavior requested here.
- Repeated expensive fallback reads: cache loaded diff results per modal index.
- Keybinding conflict: modal-local bindings should take precedence, as in other ACE modals using `ctrl+n` / `ctrl+p`.

## 8. Success Criteria

- Multi-commit `v` selections open a single navigable commit modal.
- `ctrl+n` / `ctrl+p` cycle through exactly the selected commits in selection order.
- No extra-selection warning appears for multi-commit default selections.
- Diff loading remains asynchronous, cached, and race-safe.
- Focused tests, visual snapshot coverage, and final `just check` pass.
