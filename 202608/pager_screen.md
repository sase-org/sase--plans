---
tier: tale
title: Host the pager as a screen so `v` stops crashing ACE
goal:
  Pressing `v` in the ACE Agents/Patches tabs opens the SASE pager instead of killing
  the TUI, with the standalone `sase pager` / `sase bead show` / `page_or_print` entry
  points rendering byte-identically to today.
size: medium
proposed_by: bbugyi200.athena.0eu
create_time: 2026-09-09 20:00:16
status: wip
---

# Plan: Host the pager as a screen so `v` stops crashing ACE

## 1. The symptom

On current `master`, selecting a file hint after pressing `v` in `sase ace` kills the
whole TUI with:

```
RuntimeError: asyncio.run() cannot be called from a running event loop
```

The traceback bottoms out in `textual/app.py:2350` inside `App.run`, with
`self = SasePager(...)`. Because ACE's own `App.run_async()` task dies with it, the
crash takes the entire control surface down, not just the pager.

## 2. Root cause

`FileViewingMixin._view_files_with_sase_pager`
(`src/sase/ace/tui/actions/hints/_files.py`) runs a second Textual `App` inside the
first one's suspend block:

```python
with self.suspend():
    pager.run()
```

Three facts combine into the crash:

1. `App.run()` is a _synchronous_ entry point. In Textual 8.0.1 it unconditionally calls
   `asyncio.run(run_app())`, which raises `RuntimeError` when a loop is already running
   in the calling thread.
2. `App.suspend()` does **not** stop the event loop. Reading Textual's source: it
   publishes a suspend signal, calls `driver.suspend_application_mode()` (stop
   application mode + close the driver), redirects stdout/stderr, and yields. The loop
   is still installed and still "running" — today it merely happens to be _blocked_,
   because every other `suspend()` caller in this tree does synchronous `subprocess.run`
   work inside the block.
3. `_view_files_with_sase_pager` is reached from
   `InputProcessingMixin._finish_view_request`
   (`src/sase/ace/tui/actions/hints/_processing.py`), an `async` method running on ACE's
   event loop thread. So `asyncio.events._get_running_loop()` returns ACE's loop and
   `asyncio.run` refuses.

Reproduced in isolation against the pinned Textual (no SASE code involved):

```python
async with Outer().run_test():
    Inner().run()   # -> RuntimeError: asyncio.run() cannot be called from a running event loop
```

### 2.1 Why no test caught it

Both view-file tests stub the parts that matter:

- `tests/ace/tui/actions/_view_files_helpers.py` defines `_ViewApp`, a plain object
  whose `suspend()` returns a bare recorder — not a Textual `App`, no driver, no loop
  semantics.
- `tests/ace/tui/actions/test_view_files_pager.py::test_view_files_with_sase_pager_runs_under_suspend`
  monkeypatches `SasePager` with `_FakeSasePager`, whose `run()` just sets a flag.

So the suite asserts the _shape_ of the call (`suspend` entered, `run` called) and never
executes the one line that fails. Closing this blind spot is part of the work (§5).

### 2.2 The design doc's premise was wrong

`plan:202608/link_traversing_pager.md` section D1 chose "one Textual `App`, run
in-process by both hosts" and justified it with:

> `MemoryReviewTuiApp` already proves both halves of exactly this pattern in exactly
> this tree.

Only the CLI half is proven. `sase memory review` calls `MemoryReviewTuiApp().run()`
from sync `main()`, where there is no running loop — that works. The ACE half,
`handle_memory_review` in `src/sase/ace/tui/actions/agents/_notification_handlers.py`,
does `with app.suspend(): MemoryReviewTuiApp(...).run()` from a `push_screen` dismiss
callback — i.e. from ACE's message pump — and has exactly the same defect. Its test
(`tests/ace/tui/test_memory_review_notification_action.py`) stubs `run()` the same way.
D1 generalized from a pattern that was never exercised. See §7 for how that second
instance is handled.

D1 also rejected "a modal inside the running ACE app" for two stated reasons:

> It solves only ACE; `bead show` and `sase pager` would need a second host, and it puts
> pager crashes inside the control surface.

The first reason assumed the modal would be a _duplicate_ implementation. The design in
§4 avoids that: there is exactly one pager implementation (a screen), and the standalone
`App` shrinks to a ~30-line host that pushes it. The second reason is a real cost, and
§6.4 mitigates it — but it is not a trade-off against a working alternative, because
there is no way to run a nested Textual `App` in this process at all (§3).

## 3. Rejected repairs

Establish these as dead ends before implementing, so the fix is not re-litigated:

- **Run `pager.run()` in a worker thread inside `suspend()`.** Fails. Textual's
  `LinuxDriver.start_application_mode` installs `SIGWINCH`/`SIGTTOU`/`SIGTTIN` handlers
  with `signal.signal`, which raises
  `ValueError: signal only works in main thread of the main interpreter` off the main
  thread. Verified against the pinned Textual in a pty: the app dies on a crash screen
  and never paints. The same applies to `asyncio.to_thread` and
  `run_worker(..., thread=True)`.
- **`await pager.run_async()` inside `suspend()`.** Fails worse — it hangs. Awaiting
  inside the suspend block leaves ACE's pump live while its driver is closed. ACE keeps
  rendering into `LinuxDriver.write`, which enqueues onto a stopped `WriterThread` whose
  `Queue` is bounded at `MAX_QUEUED_WRITES = 30`. After 30 writes `Queue.put` blocks
  forever on the event-loop thread, deadlocking ACE _and_ the pager (which shares the
  loop). There is also no public API to pause an App's render pipeline.
- **Shell out to the `sase pager` subcommand under `suspend()`.** Loses the two things
  the ACE integration exists for: the `AttachedTarget` commit handler (which dispatches
  live `CommitViewSpec` objects into `CommitViewModal`) and the `_link_index`-backed
  `resolve_ref_fn`. Both pass live Python objects; a subprocess would need a versioned
  wire format. D1 already priced the process hop at 288–318 ms.

## 4. The fix

Split the pager's UI out of `App` and into a `ModalScreen`, then give it two hosts:

```
PagerScreen(ModalScreen[PagerExit])   <- all current SasePager behavior
   ^                        ^
   |                        |
SasePager(App[PagerExit])   AceApp.push_screen(PagerScreen(...))
  (thin host; CLI)            (v keymap; no suspend at all)
```

`ModalScreen` specifically, not plain `Screen`: a modal screen's binding chain excludes
the host App's non-priority bindings. Verified against the pinned Textual — with a modal
pushed, the host's `s` binding does not fire and the screen's own `q` wins. That matters
because the pager paints _letter_ jump-hint labels; on a plain `Screen`, an unmatched
label key would bubble to `AceApp` and fire an ACE action underneath the pager.

## 5. Implementation steps

### 5.1 Extract `PagerScreen`

Move the body of `SasePager` from `src/sase/pager/app.py` into a new
`src/sase/pager/screen.py` as `PagerScreen(ModalScreen[PagerExit])`. Keep
`_PagerBodyScroll` with it (retarget its `isinstance(app, SasePager)` check at the
screen — reach it via `self.screen`, or make the widget call a method on its parent
screen). The behavior is unchanged; only the host bindings move:

| Today (`App`)                                                 | On `PagerScreen`                                   |
| ------------------------------------------------------------- | -------------------------------------------------- |
| `self.exit(PagerExit(...))`                                   | `self.dismiss(PagerExit(...))`                     |
| `self.push_screen(PagerHelpScreen(...))`                      | `self.app.push_screen(...)`                        |
| `len(self.screen_stack) > 1` (modal-on-top guard in `on_key`) | `self.app.screen is not self`                      |
| `suspend_for_external_tool(self, ...)`                        | `suspend_for_external_tool(self.app, ...)`         |
| `CSS = PAGER_CSS`                                             | `DEFAULT_CSS` / `CSS` on the screen, scoped (§6.1) |
| `ENABLE_COMMAND_PALETTE = False`                              | stays on `SasePager`                               |
| `BINDINGS = [...]`                                            | move verbatim to the screen                        |

These need no change and should be left alone:

- `self.notify(...)` — `Widget.notify` exists and forwards to the app.
- `self.query_one(...)`, `self.call_after_refresh(...)`, `on_mount`, `on_unmount`.
- `spawn_pump_free_task(self, ...)` / `cancel_pump_free_tasks(self)` — the owner is
  duck-typed and the registry attrs live on the owner.
- `schedule_copy_delivery(self, ...)` — `_delivery.py` already resolves `owner.app` for
  the driver and `owner.notify` for toasts.
- The whole `vim_search_*` host protocol — `VimSearchController` only calls
  `vim_search_*` methods on its host. `ZoomPanelModal` is the existing proof that this
  controller works from a `ModalScreen` inside ACE.

### 5.2 Shrink `SasePager` to a host

`src/sase/pager/app.py` keeps `PagerExit`, `AttachedTargetHandler`, `ResolveRef`,
`PendingAction`, and `_PENDING_ACTION_KEYS` (wherever they are cleanest to import from —
do not duplicate them), and reduces `SasePager` to:

- the same
  `__init__(document, *, links_enabled=True, attached_handlers=None, resolve_ref_fn=None)`
  signature, stored for the screen;
- `ENABLE_COMMAND_PALETTE = False`;
- `on_mount` → `self.push_screen(PagerScreen(...), callback=self.exit)` so `App.run()`
  still returns the `PagerExit`.

Do not change the three call sites' signatures: `src/sase/cli_pager.py:_run_sase_pager`,
`src/sase/main/pager_handler.py:_run_pager_app`, and the ACE site in §5.3 are the only
constructors in the tree.

Keep `src/sase/pager/__init__.py` exporting `SasePager` and add `PagerScreen`.

### 5.3 Point ACE at the screen

In `src/sase/ace/tui/actions/hints/_files.py`, replace `_view_files_with_sase_pager`
with a push (rename it — `_view_files_with_pager_screen` — and update the docstring,
which currently talks about `suspend()`):

```python
def _view_files_with_pager_screen(self, document: PagerDocument) -> None:
    handlers: dict[str, AttachedTargetHandler] = {}
    screen = PagerScreen(
        document,
        attached_handlers=handlers,
        resolve_ref_fn=lambda ref: _resolve_ref_from_link_index(self, ref),
    )
    handlers[_COMMIT_TARGET_KIND] = lambda target, action: (
        _handle_commit_attached_target(screen, target, action)
    )
    self.push_screen(screen)
```

Update the caller in
`src/sase/ace/tui/actions/hints/_processing.py::_finish_view_request`. The
`await asyncio.to_thread(build_pager_document, ...)` hop stays exactly as it is — file
I/O must remain off the loop (`tui_perf` rule 1).

`_handle_commit_attached_target` now receives a `PagerScreen` rather than a `SasePager`.
Retype its first parameter and change `pager.push_screen(CommitViewModal((spec,)))` to
`screen.app.push_screen(...)`, and `suspend_for_external_tool(pager, ...)` to
`suspend_for_external_tool(screen.app, ...)`. `pager.notify(...)` and
`schedule_copy_delivery(pager, ...)` work unchanged.

Delete the now-dead `suspend()` usage from this path; leave every other `suspend()`
caller in the tree alone (they all shell out to real subprocesses and are correct).

### 5.4 Tests

Existing suites, updated for the new object graph:

- `tests/pager/test_app.py` — keep driving `SasePager(...).run_test()` (that is still
  the real standalone entry point). Roughly 25 accesses of the form `app._body`,
  `app._search`, `app._back_trail`, `app._label_layer`, `app._last_activated_label`,
  `app._apply_resolution`, `app._update_subject` need to go through the pushed screen.
  Add one small helper (e.g. `_pager_screen(app)`) rather than editing each site ad hoc.
- `tests/pager/visual/test_app_png_snapshots.py` — same treatment for `app._body` and
  `app._update_subject()`. **The committed PNG goldens must not change.** A diff here
  means the CSS scoping in §6.1 is wrong; fix the CSS rather than rebaselining.
- `tests/ace/tui/actions/test_view_files_pager.py` — the routing and
  `build_pager_document` assertions stay; the `_FakeSasePager`/`suspend` assertions
  become push-screen assertions.

New coverage that would have caught this bug — this is the point of the exercise, so do
not skip it:

1. **A real host-app integration test.** Build a minimal `App` (or reuse an existing ACE
   test app if one is already wired for `run_test`), push a `PagerScreen` inside
   `async with app.run_test(...)`, drive real keys through `Pilot` (`j`, `ctrl+n`, `q`),
   and assert the screen dismissed with a `PagerExit` and the host app is still running.
   This executes the exact nesting that crashes today.
2. **A guard against the regression's shape.** Assert that the ACE view-files path never
   calls a Textual `App.run` — e.g. drive `_finish_view_request` on a `run_test`-hosted
   app with a real `PagerScreen` and assert no `RuntimeError`, and/or assert
   `AceApp.screen` is a `PagerScreen` afterwards. A stubbed pager object must not be
   able to satisfy this test.
3. **Modal isolation.** With the pager screen on top of a host app that binds a letter
   key the pager uses as a jump-hint label, assert the host's action does not fire.

## 6. Risks and things to verify while implementing

### 6.1 CSS bleed and scoping

`PAGER_CSS` (`src/sase/pager/_styles.py`) currently uses bare
`Screen { layout: vertical; }` plus `#pager-*` id rules, written for an app that owns
the whole stylesheet. Inside ACE it becomes one screen among many under a 9,300-line
`src/sase/ace/tui/styles.tcss`.

- Scope every rule under `PagerScreen` (`PagerScreen { layout: vertical; }`,
  `PagerScreen #pager-body { ... }`, …) so ACE's rules cannot win on specificity.
- Override `ModalScreen`'s default `background: $background 60%;` to an opaque
  background, or the standalone PNG goldens shift and ACE shows through the pager.
- `.hidden { display: none; }` exists in both stylesheets with the same meaning; keep
  the pager's copy scoped rather than relying on ACE's.

### 6.2 `tab` / `shift+tab` still reach ACE

`src/sase/ace/tui/bindings.py` declares `tab`/`shift+tab` as `priority=True` App
bindings, and Textual checks priority bindings from the App down even under a modal
(verified). So `tab` inside the pager will still switch ACE's tab underneath. This is
pre-existing behavior shared by every ACE modal and the pager binds neither key, so
**accept it**; do not add an ACE-wide guard on this plan.

### 6.3 The pager now runs on ACE's message pump

Under `suspend()` the pump was frozen anyway, so this is not a new stall — but it is
newly _visible_ to the stall watchdog (`src/sase/ace/tui/util/stall_watchdog.py`), which
the suspend-signal hookup used to silence for this path.

- `PagerScreen.on_mount` calls `_ensure_body()` → `compose_body` + `build_label_layer`
  synchronously. Keep the existing `tui_trace("pager.open", ...)` span and check
  `~/.sase/perf/tui_trace.jsonl` against D1's 150 ms open-to-first-paint budget on a
  realistic multi-file selection.
- Everything already off-pump must stay off-pump: `build_pager_document` behind
  `asyncio.to_thread` (§5.3) and `_resolve_and_dispatch`'s `spawn_pump_free_task` +
  `asyncio.to_thread(self._resolve_ref, ...)` (`tui_perf` rules 1 and 2).
- If the open span exceeds budget, move the body compose behind `call_after_refresh`
  with a placeholder paint rather than reverting the design.

### 6.4 A pager crash now lands inside ACE

This is D1's second rejection reason and it is real. Mitigate rather than ignore:

- Keep the existing defensive posture in `_resolve_and_dispatch` ("a press must never
  crash the pager" — the bare `except Exception` around resolution).
- Wrap the `self.push_screen(screen)` in §5.3 so a failure constructing or mounting the
  screen toasts (`severity="error"`) instead of propagating into ACE's pump.

### 6.5 Symvision

The refactor moves public symbols between modules. After it, re-check that `SasePager`,
`PagerScreen`, `PagerExit`, `AttachedTargetHandler`, and `PAGER_CSS` each still have a
real non-test consumer, and that nothing became a cross-file-imported `_private`.
Test-only references do not keep a public symbol alive. Run `just _lint-symvision`
directly if `just check` flags it.

## 7. Out of scope (file, do not fix here)

`handle_memory_review`
(`src/sase/ace/tui/actions/agents/_notification_handlers.py:141-142`) has the identical
defect: `with app.suspend(): MemoryReviewTuiApp(...).run()` from ACE's pump. Opening a
`memory_review` notification from ACE crashes the TUI the same way. It is a different
feature with a different owner surface and a different cheapest fix (shelling out to
`sase memory review --proposal-id …` under `suspend()` is likely enough, since that path
needs no live in-process objects). **File it as a task bead via `/sase_new_task`** —
type `bug`, and state the root cause and the two stubbed tests in the description —
rather than widening this plan.

Also out of scope: everything already tracked by `sase-uq`, `sase-ur`, `sase-us`, and
`sase-ut`.

## 8. Definition of done

- [ ] `PagerScreen` exists; `SasePager` is a thin host over it; no `App.run()` anywhere
      on the ACE `v` keypath.
- [ ] `v` → hint → selection opens the pager inside ACE; `q`, `escape`, and a
      trail-exhausted `backspace` all return to the same tab, selection, and scroll
      position as before; `ctrl+n`/`ctrl+p`, `/` search, `y`/`yy`, `E`/`EE`, and a
      commit label press into `CommitViewModal` all still work.
- [ ] `sase pager`, `sase bead show`, and `page_or_print` are unchanged in behavior.
- [ ] `just test-visual` passes with the committed pager PNG goldens **unmodified**.
- [ ] The new integration test fails against pre-fix code (check this explicitly — a
      test that passes both ways is the bug that got us here).
- [ ] `just check` passes; `just check-full` run through `/sase_monitor` before landing.
- [ ] A task bead is filed for the `handle_memory_review` instance (§7).
