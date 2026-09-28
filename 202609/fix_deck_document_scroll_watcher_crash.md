---
tier: tale
title: Fix the FINAL/Tools deck scroll-watcher crash
goal:
  Scrolling the Tools or FINAL deck no longer crashes `sase tui`, and scroll-derived
  spread block-cursor sync actually runs for those decks, pinned by a real-Textual pilot
  regression test.
size: small
proposed_by: bbugyi200.apollo.2r
create_time: 2026-09-28 10:41:43
status: wip
---

# Plan: Fix the FINAL/Tools deck scroll-watcher crash

## Problem

`sase tui` crashes with the following error whenever the user scrolls the **Tools** deck
or the **FINAL** deck in the Agents tab:

```
TypeError: DeckPanelCardDocumentsMixin._watch_document_scroll.<locals>.<lambda>()
missing 2 required positional arguments: '_o' and '_n'
```

The stack enters Textual's `Widget._scroll_to` on
`VerticalScroll(id='agent-deck-panel-0-tools-scroll')`. Main and Files are not affected.

## Root cause

`src/sase/ace/tui/widgets/decks/card_documents.py`,
`DeckPanelCardDocumentsMixin._watch_document_scroll`, registers the FINAL/Tools
`scroll_y` watcher like this:

```python
handler = lambda _o, _n, _d=deck: self._on_document_scroll_y(_d, _o, _n)  # noqa: E731
self.watch(scroll, "scroll_y", handler, init=False)
```

Textual picks the arguments for a watcher by **arity**, not by required-ness.
`textual._callback.count_parameters` returns `len(inspect.signature(fn).parameters)`,
and that count includes defaulted parameters. `textual.reactive.invoke_watcher` then
passes `(old, new)` for 2 parameters, `(new,)` for 1, and **nothing** for anything else.
The lambda has 3 parameters because of the `_d=deck` default-binding trick, so every
`scroll_y` change calls it with no arguments and it raises `TypeError`. Textual runs the
watcher synchronously inside the reactive setter. `scroll_to` itself is deferred through
`call_after_refresh`, so the exception fires inside the app's message pump and takes the
whole TUI down.

Regression origin: commit `6702105da8` ("clear phase-owned symvision failures from
card-document-view extraction", sase-1b2.11). It changed `_watch_deck_scrolls` from
watching only the Main scroll, through a 2-parameter bound method, to looping over
`CARD_DOCUMENT_DECKS` (MAIN, FINAL, TOOLS). It routed FINAL and TOOLS through this
lambda. The watcher sits on the whole `-tools-scroll` container, so any real scroll of
the Tools deck crashes, whichever card is active (Runs or the legacy LLM-calls card).
Any real scroll of the FINAL deck crashes too.

The `_d=deck` binding isn't needed anyway. `deck` is a parameter of
`_watch_document_scroll`, not a loop variable, so an ordinary closure captures it
correctly.

Verified empirically with a pilot that mounts a bare `DeckPanel(0)`, shows a deck's
scroll, mounts 200 rows, and runs `scroll.scroll_to(y=22, animate=False)`:

- current code: TOOLS → the exact `TypeError` above; FINAL → same `TypeError`; MAIN →
  OK.
- with a 2-parameter closure: TOOLS/FINAL/MAIN → OK, and the handler receives
  `(DeckId.TOOLS, 0.0, 22)` / `(DeckId.FINAL, 0.0, 22)`.

### Latent second defect on the same path (fix together)

`_on_document_scroll_y` ends with:

```python
self._sync_document_spread_block_cursor(deck)  # type: ignore[attr-defined]
```

No method by that name exists. The real method is the **public**
`sync_document_spread_block_cursor(deck)` on `DeckPanelDocumentRailMixin`
(`src/sase/ace/tui/widgets/decks/document_rail.py`). `DeckPanel` inherits it via
`DeckPanelDocumentBlocksMixin`. Main's equivalent call in
`panel_interaction.py::_sync_spread_block_cursor_from_scroll` already uses the right
name. The `AttributeError` is swallowed by the surrounding `except Exception`, and the
`type: ignore` hides it from mypy. Fixing only the crash would leave FINAL/Tools
scroll-derived spread block-cursor sync a silent no-op. That sync is the feature the
watcher exists for: rail/chrome following the scroll position across card blocks, as
Main does.

## Changes

### 1. `src/sase/ace/tui/widgets/decks/card_documents.py`

In `_watch_document_scroll`, replace the 3-parameter lambda with a nested 2-parameter
function that closes over `deck`:

```python
            else:

                def handler(old: float, new: float) -> None:
                    self._on_document_scroll_y(deck, old, new)

                self.watch(scroll, "scroll_y", handler, init=False)  # type: ignore[attr-defined]
```

- Add a short comment next to the nested function. It should say that Textual dispatches
  watchers by signature arity, counting defaulted parameters, so the callback must take
  exactly `(old, new)` and must not use `_x=value` default binding. This is the
  non-obvious gotcha that caused the crash, so the note stops it being reintroduced.
- Remove the `# noqa: E731`, which no longer applies.
- Look up `self._on_document_scroll_y` at call time inside the closure. Do **not** use a
  pre-bound `functools.partial`. Call-time lookup keeps instance-level spies and patches
  effective.
- Leave the MAIN branch (`self._on_main_scroll_y`, a 2-parameter bound method)
  unchanged.

In `_on_document_scroll_y`, change the final call from
`self._sync_document_spread_block_cursor(deck)` to
`self.sync_document_spread_block_cursor(deck)`. Keep the surrounding `try/except` and
the `# type: ignore[attr-defined]` style that the rest of the file uses.

No other `self.watch(...)` callback in `src/sase` has the problem. The rest use bound
methods with `(old, new)`, single-parameter methods, or `*_args`. There's no need to
touch them.

### 2. New regression test: `tests/ace/tui/widgets/decks/test_deck_document_scroll_pilot.py`

Write an async pilot test, in the style of `test_deck_spread_pilot.py`, that exercises
the **real Textual watch dispatch path**. Calling `_on_document_scroll_y` directly is
not enough, because the bug lives in how the callback is registered.

- Define a minimal `App` with `CSS_PATH = <repo>/src/sase/ace/tui/styles.tcss`. Compute
  the repo root with `Path(__file__).resolve().parents[5]`, as the sibling pilot tests
  do. `compose` yields `DeckPanel(0)` from `sase.ace.tui.widgets.decks.panel`.
- Parametrize over `DeckId.FINAL` and `DeckId.TOOLS`. Optionally include `DeckId.MAIN`
  as a no-crash control with no spy assertion. For each deck:
  1. `async with app.run_test(size=(100, 30)) as pilot:`, then `await pilot.pause()`.
  2. `panel = app.query_one(DeckPanel)`, then use `monkeypatch.setattr` to replace the
     **public** `panel.sync_document_spread_block_cursor` with a spy that records the
     deck argument. This avoids touching private names.
  3. `panel.set_deck(deck)`.
  4. Query `#agent-deck-panel-0-{deck.value}-scroll` as `VerticalScroll`. Then
     `scroll.add_class("-shown")`: an empty deck has `-shown` removed by `set_deck`, and
     a hidden scroll has `max_scroll_y == 0`, so the scroll would clamp to a no-op.
     Leave a one-line comment explaining why.
  5. `await scroll.mount(Static("\n".join(f"row {i}" for i in range(200))))`, then two
     `await pilot.pause()` calls.
  6. Assert `scroll.max_scroll_y > 22` as a precondition. Then run
     `scroll.scroll_to(y=22, animate=False)` and two more `await pilot.pause()` calls.
  7. Assert `int(scroll.scroll_y) == 22` and that the spy recorded `deck`. Put the spy
     assertion after the `async with` block, or inside it; either works.
- On the unfixed code this test fails in two ways. The pump's `TypeError` is re-raised
  when `run_test` exits. And the spy is never called, because of the misspelled method
  name. Before the fix, confirm locally that both failures happen: temporarily revert
  one change at a time, or at least reason it through against the reproduction above.
  Then confirm it passes after the fix.

## Verification

- Run the new test file directly, then run `sase tool run check`, which is `just check`
  through the tool control plane (`sase memory read lint_and_test.md` has the details).
  Don't run `just check-full`.
- No PNG goldens should change. The fix only restores scroll handling and doesn't touch
  rendering. `just fix-tui-screenshots` isn't needed.
- Don't edit `CHANGELOG.md`; release-please generates it from the commit subject. Use a
  `fix(ace-tui): ...` style subject when finalizing.

## Out of scope / notes

- The broader pattern in the deck mixins, `# type: ignore[attr-defined]` plus a bare
  `except Exception` around cross-mixin calls, is what hid the misspelled method name.
  Hardening it, for example with a test that every cross-mixin method name called on
  `DeckPanel` resolves, is a separate follow-up. Don't do it in this change.
- A running `sase tui` keeps the code it imported at start. The user has to restart the
  TUI, after updating the uv-tool install, before the fix takes effect.
