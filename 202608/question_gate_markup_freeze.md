---
tier: tale
title: Fix the frozen ACE TUI when opening a question gate whose options contain markup
goal:
  Pressing <enter> on a UserQuestion notification opens the answer modal and renders
  option labels verbatim, even when the agent's option text contains bracket sequences
  like `[/]`; and any future unhandled TUI exception exits cleanly with a logged
  traceback instead of beeping and freezing the terminal.
size: medium
proposed_by: bbugyi200.athena.01l
create_time: 2026-09-09 20:00:24
status: wip
---

# Plan: Fix the frozen ACE TUI when opening a question gate whose options contain markup

## Symptom

In `sase ace`, opening the Notifications modal, highlighting a `UserQuestion` row (right
pane shows "● Awaiting your answer / press Enter to answer"), and pressing `<enter>`
produces:

1. a single terminal **beep**, and then
2. the **entire TUI freezes** — no repaint, no keys accepted, terminal left in raw mode
   on the alternate screen. No traceback is printed and nothing is written to
   `~/.sase/logs/tui.log`, so the failure leaves no trace at all.

Reproduced with the real, still-on-disk request bundle
`~/.sase/interaction_requests/question/dd035a3a-c08a-418a-ad36-ff2cdc935611`
(notification `7a98b078-6536-4f47-aef7-16081b4c5638`, sent 2026-08-14 13:37:57 from the
`bob-cli` patch).

## Root cause

Two independent defects compose: **defect A** crashes the modal, **defect B** turns that
crash into a hang instead of a clean exit and traceback.

### Defect A — unescaped agent text is parsed as Textual markup

`UserQuestionModal._build_selections()`
(`src/sase/ace/tui/modals/user_question_modal.py:276-289`) builds each answer row as a
**plain `str`**:

```python
display = f"{label} — {desc}" if desc else label
items.append((display, label))
```

and hands the resulting `(str, value)` tuples to `_WrappingSelectionList` (a
`SelectionList` subclass) both in `compose()` (line 200) and in `_load_question()`
(lines 377-378, on every `n`/`p`/left-pane question switch).

Textual's `SelectionList` treats a `str` prompt as **content markup**:
`Selection.__init__` → `Content.from_text(prompt)` → `Content.from_markup(...)` →
`textual.markup.to_content(...)`.

The question in question asks about Obsidian checkbox syntax, so its option descriptions
contain literal bracket tokens:

> "Jump targets become exactly `[ ]`, `[/]`, `[*]`. Blocked `[?]` tasks are skipped."

`[/]` is Textual's _auto-closing tag_, and nothing is open, so the parse raises:

```
textual.markup.MarkupError: auto closing tag ('[/]') has nothing to close
```

This is raised inside `UserQuestionModal.compose()`, so Textual routes it to
`App._handle_exception()` → `App._fatal_error()`.

**Why this survived the earlier hardening pass.** Commit `ca024d640` ("fix: Escape Rich
markup in user-facing strings to prevent bracket consumption (sase-g8d)") fixed the two
_other_ untrusted-text sinks in this exact file — it added `markup=False` to the
`#user-question-text` `Static` and `escape_markup()` to `_build_question_header()` — but
did not touch `_build_selections()`. The option labels and descriptions have been the
one unescaped sink in this modal ever since.

### Defect B — Textual's fatal-error path can itself raise, and it bells first

`App._fatal_error()` (textual 8.0.1, `textual/app.py:3212-3224`) runs in this order:

```python
self.bell()                                                    # 1. the beep
traceback = Traceback(show_locals=True, ..., suppress=[rich])   # 2. raises
self._exit_renderables.append(...)                              # 3. never runs
self._close_messages_no_wait()                                  # 4. never runs
```

Step 2 explodes here because `show_locals=True` makes rich pretty-print the crashed
frame's locals, and one of those locals is the **half-constructed `Selection`**:
`Selection.__init__` calls `Content.from_text(prompt)` _before_ `super().__init__()`, so
`self._prompt` was never assigned. `Option.__rich_repr__` is a generator that yields
`self._prompt`, and rich guards only the _call_ to `__rich_repr__`
(`rich/pretty.py:656-661`, `try/except Exception: pass`) — not the iteration of what it
returns (`args = list(iter_rich_args(rich_repr_result))`, line 666, unguarded). Calling
a generator function raises nothing, so the failure lands outside the guard:

```
AttributeError: 'Selection' object has no attribute '_prompt'
```

escapes `_fatal_error()` after the bell but before the shutdown.
`_close_messages_no_wait()` is therefore never called: the driver stays in raw mode on
the alternate screen, the message pump is dead, and no traceback is ever queued for
display. **That is the freeze.**

Textual's crash path also never goes through the `sase` logger, which is why
`~/.sase/logs/tui.log` (installed by `install_tui_file_logging()`) has no record of it.

### Third sink, same class, same flow

`AceApp.notify()` defaults to `markup=True`, and `Toast.render()`
(`textual/widgets/_toast.py:100-121`) calls `Content.from_markup(notification.message)`
unconditionally at render time. Several `notify()` calls on this very flow interpolate
agent- or exception-supplied text, e.g. `handle_user_question()`'s
`f"Error reading question request: {e}; press d on the notification to debug"`
(`src/sase/ace/tui/actions/agents/_notification_question_modal.py:135-138`) and
`_finish_question_response_task()`'s `app.notify(outcome.message)` (line 272). A bad
bracket in any of those crashes the app the same way — and, via defect B, freezes it.

## Evidence (measured, not assumed)

All runs below are from this workspace's venv against the real on-disk bundle.

1. **Defect A reproduces end to end.** Pushing `UserQuestionModal` with the real payload
   under `App.run_test()` fails in `compose()`:

   ```python
   from sase.user_question_actions import read_user_question_request
   from sase.ace.tui.modals.user_question_modal import UserQuestionModal
   questions = read_user_question_request(
       Path("~/.sase/interaction_requests/question/"
            "dd035a3a-c08a-418a-ad36-ff2cdc935611").expanduser()
   )["questions"]
   # push UserQuestionModal(questions) -> MarkupError inside compose()
   ```

2. **The trigger is markup parsing of the prompt, nothing else.** Constructing the
   offending row directly:

   | prompt passed to `Selection(...)` | result                                      |
   | --------------------------------- | ------------------------------------------- |
   | raw `str`                         | `MarkupError: auto closing tag ('[/]') ...` |
   | `rich.markup.escape(...)`         | OK, text preserved                          |
   | `textual.markup.escape(...)`      | OK, text preserved                          |
   | `textual.content.Content(...)`    | OK, text preserved                          |
   | `rich.text.Text(...)`             | OK, text preserved                          |

3. **The candidate fix works on the real payload.** With `_build_selections()`
   monkeypatched to return `(Content(display), label)` tuples, the modal composes,
   `space` toggles, `n` advances to Q2 (exercising the `_load_question()` rebuild), a
   second `space` toggles, `<enter>` submits and dismisses — `app._exception is None`.
   Force-rendering `#user-question-options` shows the literal tokens intact:

   ```
   ▐X▌ Yes, drop Blocked — Jump targets become exactly
       ]`, `[/]`, `[*]`. Blocked `[?]` tasks are skippe
   ```

4. **Defect B is exactly "bell, then no shutdown".** A minimal app whose `compose()`
   raises with a local object that mimics `Option.__rich_repr__` (generator yielding a
   missing attribute) instruments both methods:

   ```
   bell rang: 1
   _close_messages_no_wait called: 0
   _exit_renderables: 0
   ```

   The same harness with a local whose `__repr__`/`__rich_repr__` raises _on call_
   (rather than mid-iteration) shows `_close_messages_no_wait called: 1` — rich guards
   that case, which is why this only bites on partially constructed `Option`/`Selection`
   objects.

5. **The toast sink is real.**
   `Toast(Notification("... bad `[/]` token ...", markup=True)).render()` raises
   `MarkupError`.

6. **Nothing in this flow was logged.** `~/.sase/logs/tui.log` has no entry near 13:39
   on 2026-08-14 (last record 13:29), confirming the crash bypasses the `sase` logger
   entirely.

## Fix

Three changes: kill the crash (A), stop _any_ crash from freezing the terminal (B), and
close the toast sink of the same defect class.

### Change 1 — `UserQuestionModal` must never markup-parse agent text

File: `src/sase/ace/tui/modals/user_question_modal.py`.

- `_build_selections()`: return `list[tuple[Content, str]]` instead of
  `list[tuple[str, str]]`, wrapping each display string in
  `textual.content.Content(...)`. `Content.from_text()` returns a `Content` instance
  unmodified, so this bypasses markup parsing entirely — no escape sentinels, no stray
  backslashes, and the option text renders byte-for-byte as the agent wrote it. Wrap the
  `"Other..."` sentinel row the same way for type uniformity.

  Prefer `Content` over `rich.markup.escape()` here. `escape()` also works (evidence row
  2), but this call site interpolates **no** style tags of its own, so escaping would be
  round-tripping text through a markup encoder for no reason and would reintroduce the
  bug the moment someone appends a styled prefix. The existing `escape()` usages in
  `gate_branch_layout.py` and `custom_gate_modal.py` are correct _because_ those sites
  do interpolate their own `[dim]`/`[bold]` tags — leave them alone.

- `_load_question()` (lines 377-378) already consumes `_build_selections()` output via
  `sel_list.add_option((label, value))`, so it is fixed by the same change; confirm the
  tuple still type-checks against `SelectionList.add_option`.

- Update the `_build_selections()` docstring and return annotation, and add a short
  comment naming the hazard (`SelectionList` markup-parses `str` prompts) so the next
  person does not "simplify" it back to an f-string.

Audit the rest of the file while there — the remaining untrusted sinks
(`#user-question-text` `Static(markup=False)`, `_make_question_label()`'s `rich.Text`,
`_build_question_header()`'s `escape_markup`) are already safe and need no change.
`Static.update()` respects the constructor's `markup` flag, so the Q2 text update path
is safe as-is; state that in the plan's test coverage rather than changing it.

### Change 2 — a failed fatal-error path must still restore the terminal

File: `src/sase/ace/tui/app.py` (`AceApp`).

Override the crash entry point so a broken traceback render can never leave the terminal
wedged:

- Override `_handle_exception(self, error)`:
  1. `log.exception("Unhandled exception in sase ace", exc_info=error)` **first**, so
     the `sase` logger's rotating handler puts a real traceback in
     `~/.sase/logs/tui.log` regardless of what Textual does next. This is the piece that
     made this bug invisible.
  2. Call `super()._handle_exception(error)` inside `try/except Exception`.
  3. In the `except`, log that secondary failure too, then guarantee shutdown by calling
     `self._close_messages_no_wait()` directly (guarded by its own `try/except`), so the
     driver restores the terminal and the process exits.

  `App._handle_exception()` already sets `self._return_code`, `self._exception`, and
  `self._exception_event` before dispatching to `panic()`/`_fatal_error()`, so
  `run_test()`-based tests still see the original exception.

- Keep the override thin and free of function-body imports that could themselves fail
  during a crash; `app.py` already has a module-level `log` (line 89), so use that.

Do **not** vendor or monkeypatch `rich.traceback` / `textual.app._fatal_error`. The
point is to bound the blast radius of _any_ failure in Textual's crash path, not to fix
rich's `iter_rich_args()` from here.

### Change 3 — `AceApp.notify()` degrades bad markup instead of crashing

File: `src/sase/ace/tui/app.py`, existing `AceApp.notify()` override (line 256).

There are ~1000 `notify()` call sites in `src/sase/ace/tui`, many interpolating
exception text, agent names, paths, and bead/patch text. Sweeping them all is neither
tractable nor desirable — most intentionally use markup. Fix it once at the single choke
point instead: `Widget.notify()` delegates to `self.app.notify()`, so every modal and
mixin already funnels through this override.

When `markup` is `True`, pre-validate with
`textual.content.Content.from_markup(message)` inside `try/except MarkupError`; on
failure, call `super().notify(...)` with `markup=False` so the message renders
literally. `record_toast(...)` keeps recording the original message unchanged either
way. Keep the happy path allocation-light — bail out of the check immediately when `"["`
is not in `message`, mirroring the same early-out `Content.from_markup()` itself uses.

## Tests

`asyncio_mode = "auto"` is set in `pyproject.toml`, so `async def` tests need no
decorator. Reuse the `_TestApp`/`run_test` harness style already in
`tests/ace/tui/test_user_question_modal_vim.py`.

1. **The exact regression** — new `tests/ace/tui/test_user_question_modal_markup.py`:
   - A question whose option `label` and `description` contain the real-world tokens
     (`[ ]`, `[/]`, `[*]`, `[?]`, `[x]`, `[-]`) plus a plausible style-tag lookalike
     (`[bold]`): pushing `UserQuestionModal` composes without raising and
     `app._exception is None`.
   - The rendered prompt text is **verbatim** — assert on
     `sel_list.get_option_at_index(i).prompt.plain` (or the rendered strips) that `[/]`
     and `[bold]` survive as literal characters and that no backslash was introduced.
     This is what distinguishes the `Content` fix from a silent-truncation regression.
   - Two such questions: `n` (`action_next_question`) rebuilds the list through
     `_load_question()` without raising, and `p` returns.
   - `#user-question-text` still shows Q2's raw text after the switch (locks in that
     `Static.update()` keeps `markup=False`).
   - Toggle + submit produces a `UserQuestionResult` whose `answers[i].selected` holds
     the original, unescaped labels — the label doubles as the `SelectionList` _value_
     and is written into `question_response.json`, so an escaping regression here would
     corrupt the answer the agent reads back.
2. **Crash containment** — new `tests/ace/tui/test_app_crash_shutdown.py`:
   - An `AceApp`-derived (or `AceApp`-mixin-applied) app driven so that
     `_handle_exception()` runs with a stubbed `super()._handle_exception` that raises:
     assert `_close_messages_no_wait()` was still called exactly once, and that `caplog`
     captured the original exception at `ERROR` on the `sase` logger.
   - The normal path (`super()` succeeds) must not double-close.
   - If instantiating a full `AceApp` in a unit test is too heavy, extract the guard
     into a small module-level helper in `src/sase/ace/tui/` that takes the app plus the
     two callables, and test that helper directly — but keep `AceApp._handle_exception`
     as the only production caller.
3. **Toast fallback** — extend the nearest existing toast/notify test module (or add
   `tests/ace/tui/test_app_notify_markup.py`):
   - `notify("bad `[/]` token")` records the toast and renders literally rather than
     raising; assert the underlying `Toast.render()` succeeds for the notification the
     app produced.
   - A message with intentional markup (`"[bold]done[/bold]"`) still renders **styled**
     — i.e. the fallback is not a blanket `markup=False`.
   - `record_toast` receives the original, unmodified message in both cases.

## Verification

- `just install` first (workspace venvs go stale), then `just check`.
- `just check-full` before landing — Change 2 and Change 3 touch `AceApp`, which most
  TUI tests instantiate.
- `just test-visual` is not expected to be affected (no renderer or styling changes).
- Manual smoke, which is what actually proves this out: the failing bundle is still on
  disk at `~/.sase/interaction_requests/question/dd035a3a-c08a-418a-ad36-ff2cdc935611`.
  It is cancelled (`cancellation.json`, `reason: requester_cancelled`), so re-answering
  it is a no-op — but the modal must still **open and render** rather than freeze.
  Verify by opening `sase ace`, pressing the notifications keymap, selecting the row,
  and pressing `<enter>`. If the store no longer surfaces it, have a test agent ask a
  question whose option descriptions contain `` `[/]` `` and repeat.

## Notes / proposed follow-ups (not in scope here)

- **Upstream bug worth reporting.** Textual's `Selection.__init__` parses the prompt
  before `super().__init__()` assigns `_prompt`, so a `MarkupError` there leaves an
  object whose `__rich_repr__` raises — which in turn breaks `App._fatal_error()`'s
  `show_locals=True` traceback. Either ordering fix (assign `_prompt` first) or guarding
  `rich.pretty.iter_rich_args()` would prevent the freeze class entirely. Worth a
  `Textualize/textual` issue; not a reason to delay this fix.
- **Other untrusted-text sinks outside this flow.** This plan closes the question modal
  and the central `notify()` choke point. A broader audit of `Static`/`Label`/`Button`
  construction from agent-authored text across `src/sase/ace/tui/modals/` (92
  `Option(...)` sites, most of which already build `rich.Text` and are safe) belongs in
  its own task bead.
- **Crash visibility.** Once Change 2 lands, unhandled TUI exceptions appear in
  `~/.sase/logs/tui.log`. A follow-up could route them to the launch-failure JSONL
  (`sase.logs.log_launch_failure`) so `sase doctor` can surface repeat crashers.
