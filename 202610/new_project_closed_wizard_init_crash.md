---
tier: tale
title: Fix New Project wizard crash on late sase init outcome
goal: A sase init hand-off that finishes after the New Project wizard closed never
  touches the dismissed wizard, so the TUI no longer crashes with NoMatches.
size: small
proposed_by: bbugyi200.athena.10z
proposed_with: claude/opus
status: done
---

# Plan: Fix the TUI crash when the `sase init` hand-off finishes after the New Project wizard closed

## Symptom

The TUI crashed with:

```
_new_project_apply_flow.py:322 in _finish_ready  ->  self._show_stage("ready")
new_project_modal_shell.py:146 in _show_stage    ->  self.query_one("#new-project-stepper", Static)
NoMatches: No nodes match '#new-project-stepper' on NewProjectModal()
```

The locals show the crash came from `needs_init=True`: an init outcome of `False`
reached the wizard after a successful `project.add`
(`next_argv=('sase', 'init', '-p', ...)`).

## Root cause (confirmed by reproduction)

1. After a successful add, `_on_progress_result` calls `_start_init_handoff`. That runs
   `run_project_init(NewProjectInitHost(self), ...)`, which submits a
   `sase init --check` **app-level session worker**. The worker's `on_complete` callback
   outlives the wizard.
2. On the Progress stage, `Esc` closes the wizard ("esc close · setup continues in
   Procs"): `action_cancel` → `self.dismiss(None)`. Textual pops the screen from the
   stack right away, then prunes its children, dispatches `Unmount` (setting
   `_teardown_started`), and detaches it.
3. **Textual 8.0.1 never resets `is_mounted`.** `MessagePump._is_mounted` is set to
   `True` on mount and never cleared; `Widget._message_loop_exit` clears `_nodes` and
   detaches, but leaves `_is_mounted` alone. So a dismissed, pruned wizard still reports
   `is_mounted == True`.
4. When the init check finishes (it fails, or its preview is cancelled → `False`; or the
   project is already current → `True`), `NewProjectInitHost._finish` checks
   `modal.is_mounted` (always `True`), calls `modal._on_init_finished(...)` →
   `_finish_ready` → `_show_stage("ready")` → `NoMatches` on the empty DOM. The
   exception escapes from the session-worker completion and crashes the app.

A throwaway pilot test outside the repo reproduced this. It drove the wizard to
Progress, started the init hand-off, pressed `Esc`, then completed the init check:

- After close: `modal in app.screen_stack == False`, `modal.is_mounted == True`,
  `modal.is_attached == False`, `modal._teardown_started == True`.
- Failed check (`False`): exact user traceback. The crash goes through
  `_new_project_apply_init.py:76` → `_new_project_apply_flow.py:305` → `:322` →
  `new_project_modal_shell.py:146`, ending in `NoMatches '#new-project-stepper'`.
- "Already current" check (`True`): the same bug class, one line earlier, at
  `_new_project_apply_flow.py:302` (`query_one(NewProjectProgress)` → `NoMatches`). Its
  guard is `self.is_mounted and stage == "progress"`, and the stage stays `"progress"`
  after dismiss.

The bug is the stale `is_mounted` liveness check that every out-of-band completion path
of the wizard uses. The fix replaces that check with a real "wizard is still open"
check.

## Changes

### 1. Add a real liveness predicate — `src/sase/ace/tui/modals/_new_project_apply_shared.py`

Add a public-named helper (this private module already holds public names for siblings)
and export it in `__all__`:

```python
def wizard_is_open(modal: Any) -> bool:
    """Return whether the New Project wizard is still open and queryable.

    Textual sets ``is_mounted`` once and never clears it, so a dismissed
    wizard keeps reporting mounted after teardown prunes its widgets.
    Completions that outlive the wizard (durable add/create procs, the
    ``sase init`` hand-off) must use this instead. ``dismiss()`` pops the
    wizard off the screen stack synchronously, before pruning starts, so
    stack membership closes the whole teardown window.
    """
```

Semantics, in order:

- `modal is None` → `False`.
- `not getattr(modal, "is_mounted", False)` → `False` (not mounted yet).
- `getattr(modal, "_teardown_started", False)` → `False`.
- Read `modal.app.screen_stack` inside `try/except Exception` (return `False` on
  failure), then return `any(screen is modal for screen in stack)`.

The app uses no Textual modes, so `app.screen_stack` is the only stack. A wizard covered
by the init preview (`InitPlanModal` pushed on top) is still on the stack, so it is
still open. That is correct.

### 2. Use it in the init host — `src/sase/ace/tui/modals/_new_project_apply_init.py`

- `NewProjectInitHost.is_mounted`: replace `modal is not None and modal.is_mounted` with
  `wizard_is_open(modal)`. Keep the Projects-pane fallback unchanged. This makes the
  controller's "may completion effects still touch the surface" checks in
  `projects_pane_init_flow.py` (apply completion and terminal valve) honest when the
  wizard is gone and no Projects pane is mounted.
- `NewProjectInitHost._finish`: call `modal._on_init_finished(initialized)` only when
  `wizard_is_open(modal)`. Still set `self._finished = True` first. The controller has
  already toasted failures and the add itself is durable, so a closed wizard just drops
  the outcome.

### 3. Guard the wizard's out-of-band entry points — `src/sase/ace/tui/modals/_new_project_apply_flow.py`

Import `wizard_is_open` from `._new_project_apply_shared` (the module already imports
from it).

- `_on_init_finished`: return early unless
  `wizard_is_open(self) and self._state.stage == "progress"`. The `mark_initialized`
  branch then no longer needs its own `self.is_mounted and stage == "progress"` check.
  This is defense in depth, so the method stays safe even if another host calls it.
- `_on_add_handle`: replace `if not self.is_mounted: return` with
  `if not wizard_is_open(self): return`. Otherwise a proc-handle change after close
  would raise `NoMatches` from `query_one(NewProjectProgress)`.
- `_observe_add_completion`: replace `not self.is_mounted` with
  `not wizard_is_open(self)`. It is already safe thanks to the empty-query check; this
  is for consistency.
- Keep the file at or under the 500-line cap (it is 451 lines today; the change is net
  near zero).

### 4. Durable add/create completion — `src/sase/ace/tui/modals/_new_project_apply_shared.py`

In `on_project_apply_complete`, replace `getattr(modal, "is_mounted", False)` with
`wizard_is_open(modal)` before calling `_observe_add_completion`.

### 5. Ready-stage launch — `src/sase/ace/tui/modals/_new_project_apply_ready.py`

In `ensure_catalog_and_launch`, replace `if not modal.is_mounted: return` with
`if not wizard_is_open(modal): return`. Today a wizard closed during the
`ensure_project_tag_catalog` thread (before `on_unmount` cancels the task) would call
`modal.dismiss(...)` a second time. That calls `app.pop_screen()` and pops whatever
unrelated screen is now on top.

Leave `set_current_key` unchanged on purpose. It only sends a toast and reloads Projects
through `find_projects_pane(app)`, never touches wizard DOM, and the toast is still
useful after the wizard closes.

Out of scope: the `is_mounted` checks inside child widgets (`new_project_progress.py`,
`new_project_review.py`, and others). They gate pre-mount rendering inside the widget
itself, and their timers and tasks die with the widget. The provider mixin already pairs
`is_mounted` with `_teardown_started` and `_wizard_dom_live()`.

## Tests — `tests/ace/tui/modals/test_new_project_init_outcomes.py`

1. **Pilot regression (the reported crash).** Pull the setup that
   `test_wizard_init_success_marks_initialized` already does into a small local helper:
   `_make_add_flow`, the patched `NewProjectReview.start_check`, the `fake_submit`
   session worker, driving to Review, `y` to apply, and
   `modal._on_progress_result(_result("ok", next_argv=("sase", "init", "-p", "gh_sase-org__new")))`.
   Reuse that helper in the existing test. Then add
   `test_closed_wizard_ignores_late_init_outcome`, parametrized over two completions:
   - a failed check:
     `_completion(handle.proc_id, None, success=False, message="boom", error="boom")`.
     This is the reported `needs_init=True` path.
   - a current check:
     `_completion(handle.proc_id, check_payload(project_plan("gh_sase-org__new"), status="current"))`.

   After the init check is submitted, `await pilot.press("escape")` and
   `await pilot.pause()` (twice is fine). Assert `modal not in app.screen_stack`, then
   call `kwargs["on_complete"](completion)` and `await pilot.pause()`. The call must not
   raise, and `modal._state.stage` must still be `"progress"` (Ready never ran). Do
   **not** assert on `modal.is_mounted`; that is Textual-internal behavior the fix no
   longer depends on.

2. **Host unit test.** Add `test_closed_wizard_host_drops_outcome`. Use a fake modal
   with `is_mounted = True` whose `app` is `SimpleNamespace(screen_stack=[])` (the modal
   is not on the stack). Assert `host.is_mounted is False`. After
   `host.report_init_outcome(False)`, `_on_init_finished` was never called.
3. **Update `test_reload_does_not_finish_wizard_host`.** Its fake `_Modal` now has to
   sit on its app's stack, e.g. give the instance an
   `app = SimpleNamespace(screen_stack=[modal])` after construction. Otherwise
   `wizard_is_open` treats it as closed and the existing `finished == [True]` assertion
   fails. `find_projects_pane` tolerates the fake on the stack because it skips entries
   without a callable `query`.

Confirm that both new pilot parametrizations fail on the pre-fix code with `NoMatches`,
then pass after the fix.

## Verification

- `just fmt`
- `just check` (whole-repo lint gates plus the diff-scoped test lane). Do not run
  `just check-full`.

## Docs

No docs change. `docs/reference/tui/projects-tab.md` already says "`Esc` closes the
wizard while the operation continues in Procs". The fix makes that true for the init
hand-off: the init preview and its toasts still appear after the wizard closes, and the
wizard simply no longer tries to render Ready.
