---
tier: tale
title: Persist the SASE Admin Center resume tab
goal: The Admin Center resumes the last successfully chosen tab across TUI and machine
  restarts without blocking navigation on disk I/O.
create_time: 2026-07-24 18:20:01
status: done
---

- **PROMPT:** [202607/prompts/persist_admin_center_resume_tab.md](prompts/persist_admin_center_resume_tab.md)

# Persist the SASE Admin Center resume tab

## Objective

Make the SASE Admin Center's home-screen resume target survive ACE TUI and machine restarts. The first configured Admin
Center opener must remain home-first, while the second opener shown by the `# resume <Tab>` affordance must use the last
successfully activated Admin Center tab from the current or a previous TUI process.

All writes triggered by Admin Center navigation must stay off Textual's event loop and serial message pump. Failed,
malformed, or stale state must degrade to the existing no-history behavior without preventing the Admin Center or TUI
from opening.

## Current behavior and constraints

- `StateInitMixin._init_app_state()` initializes `_last_admin_center_tab` to `None`, so history is process-local.
- `BaseActionsMixin._open_config_center()` passes that value into `ConfigCenterModal` as `resume_tab`.
- `ConfigCenterModal._switch_to()` is the authoritative success boundary: it returns `False` for pane
  construction/mount/switch failures and only exposes a tab after the pane is active.
- The app currently learns the final tab only from the modal dismissal result. That loses a choice if ACE or the machine
  stops while the modal is open.
- The first opener must continue to mount the bounded home view with zero concrete panes. No disk access may be added to
  opener dispatch, tab-switch handlers, render methods, or Textual pump/timer callbacks.
- This is TUI presentation state, so it belongs in the Python TUI layer rather than the shared Rust backend. It is
  bounded machine-local runtime state, not a configurable option, so it should not be added to `default_config.yml` or
  the config schema.

## Implementation plan

1. Add a focused persistence module beside the Admin Center modules, for example
   `src/sase/ace/tui/modals/config_center_state.py`.
   - Store one catalog tab ID plus a newline in a bounded top-level state file such as
     `sase_home() / "ace_admin_center_last_tab.txt"`. Resolving the path through `sase.core.paths.sase_home()` preserves
     `SASE_HOME` test and deployment overrides and makes the preference survive normal TUI and machine restarts.
   - Keep `config_center_catalog.validated_center_tab()` as the only authority for accepted IDs; do not duplicate the
     catalog in the persistence layer.
   - Load with a small hard byte limit. Missing, unreadable, oversized, non-UTF-8, empty, or non-catalog content returns
     `None` and retains the existing "resumes after your first section visit" behavior.
   - Save through a same-directory temporary file, flush/fsync it, and atomically replace the destination. Clean up a
     temporary file on failure and let the caller log the error. This prevents partial/corrupt reads across crashes or
     concurrent readers.

2. Seed the in-memory resume target during app construction in `src/sase/ace/tui/actions/_state_init.py`.
   - Replace the unconditional `None` with the defensively loaded catalog tab. Perform only the persistence module's
     single bounded read, before Textual's event loop starts; no first-opener or render-path read is allowed.
   - Initialize explicit writer state: latest pending tab/generation, completed generation, and the one active writer
     task. Keep panes and all pane-local state scoped to a modal lifetime exactly as they are today.

3. Report successful tab activation at its source in `src/sase/ace/tui/modals/config_center_modal.py`.
   - Add an optional synchronous `on_tab_activated(CenterTab)` callback to the modal constructor.
   - Invoke it only after `_switch_to()` has mounted the target, committed the active tab/switcher/header, activated and
     focused the pane, and is about to return success. Do not invoke it for invalid digits, failed construction, failed
     mounting, failed switching, the home view, or an idempotent re-focus of the already-active tab.
   - Isolate callback failures with logging so persistence can never turn a successful UI navigation into a failed one.
   - This one success hook must cover numbered keys, Tab/Shift+Tab, landing-row clicks, repeated-opener resume, and
     direct-entry commands without adding persistence logic to each input path.

4. Add app-owned, latest-value non-blocking persistence in `src/sase/ace/tui/actions/base.py` or a small private action
   mixin composed by `BaseActionsMixin`.
   - Pass the app callback into every `ConfigCenterModal` created by `_open_config_center()`.
   - On activation, update `_last_admin_center_tab` immediately in memory, then enqueue an immutable tab/generation for
     a pump-free async task whose blocking atomic save runs via `asyncio.to_thread()`. Never await the write from modal
     navigation.
   - Allow only one writer at a time. Coalesce switches made while a write is in flight to the newest pending
     generation, then drain that generation after the active write completes. This prevents a slow older write from
     overwriting a newer tab.
   - Log write failures without rolling back the in-memory selection or interrupting the user with a modal failure. A
     later successful activation remains able to enqueue a fresh generation.
   - Preserve dismissal behavior for updates-indicator revalidation and home-only closes. Route a valid dismissal result
     through the same remember helper only as an idempotent fallback; do not perform a second disk write when the
     successful activation callback already recorded the same generation.

5. Extend the controlled-exit path in `src/sase/ace/tui/actions/lifecycle.py`.
   - Add a bounded, best-effort flush method for the Admin Center writer that waits for the newest queued generation
     while its disk work remains in `asyncio.to_thread()`.
   - Drain it alongside the existing Agents-fold persistence before cleanup cancels pump-free tasks and exits. A timeout
     or write error must be logged and must never trap the user in ACE.
   - Keep ordinary tab activation fully optimistic and non-blocking; the bounded wait is only after a controlled
     quit/restart has already begun.

6. Add focused persistence, race, lifecycle, and integration coverage.
   - In a new state-module test file, cover valid round trips, the exact newline-terminated wire value, missing files,
     invalid catalog IDs, malformed/non-UTF-8 and oversized files, atomic replacement, and temporary cleanup/write
     failure. Use the per-test `SASE_HOME`; never touch the real home directory.
   - In `tests/ace/tui/test_config_center_tabs.py`, assert the activation callback fires only after successful switches
     and remains silent for construction/mount failures and idempotent re-focus.
   - Add an ACE integration test that writes a valid state file before app construction, opens the home view, observes
     `resume Tasks` (or another chosen tab), presses the configured opener again, and reaches only that pane. Also
     assert invalid persisted content yields the no-history home.
   - Gate the first save with a `threading.Event`, continue navigating while that write is blocked, switch through
     multiple tabs, release the gate, and assert the UI remained responsive and the final durable value is the newest
     tab. This proves both off-thread execution and last-write ordering.
   - Cover choosing a tab and persisting it before modal dismissal, direct entry, failed pane activation retaining the
     prior target, home-only close retaining it, write failure remaining non-fatal, and controlled exit waiting for the
     latest queued generation.
   - Run the existing Admin Center open benchmark structurally to confirm the generic opener still paints home with no
     concrete panes and no data-scaled work; no PNG golden should change because the already-covered `resume Tasks`
     presentation is unchanged.

## Validation

Run, in order:

1. `just install`
2. Focused tests for the new state module, `tests/ace/tui/test_config_center_tabs.py`,
   `tests/ace/tui/test_log_panel_keymap.py`, and the controlled-exit tests.
3. `pytest -s -m slow tests/ace/tui/bench_admin_center_open.py`
4. `just check`

The implementation is complete when a tab selected by any successful Admin Center entry path becomes the next process's
home-screen resume target, rapid switches cannot persist an older tab last, corrupt/unwritable state is non-fatal,
opener/tab navigation performs no synchronous disk I/O, controlled quit makes a bounded attempt to drain the newest
write, the Admin Center remains home-first and lazy, and the full repository check passes.
