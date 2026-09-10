---
tier: tale
title: Fix pager forward history and display the Tab shortcut
goal: "Tab advances the pager's retained forward history in standalone and embedded use,
  and the pager consistently advertises the shortcut as <tab>.

  "
size: small
proposed_by: bbugyi200.athena.0i3
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0i3](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0i3.md)
  - [bbugyi200.athena.toobig-54.agent_list_build.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-54.agent_list_build.0/README.md)
  - [bbugyi200.athena.toobig-54.fleet.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-54.fleet.0/README.md)
  - [bbugyi200.athena.toobig-54.machines_pane.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-54.machines_pane.0/README.md)
- **COMMITS:**
  - [0523874](https://github.com/sase-org/sase/commit/0523874af159e7f029ec342218d08d5d2f5b8570)
    — fix(tui): bind pager tab history
  - [2da11eb](https://github.com/sase-org/sase/commit/2da11eb28790d10291b1b5912b70f03f1e2cd169)
    — refactor(agents): split fleet mixin modules
  - [024e01b](https://github.com/sase-org/sase/commit/024e01b70c379aa4ad2bd87c8b8b62d8ef96bbd1)
    — fix(check): restore split-module public APIs
  - [9378903](https://github.com/sase-org/sase/commit/93789035bbff78e168c06ee9bdfa089e9cf87411)
    — refactor(tui): split machines pane helpers
  - [16001bb](https://github.com/sase-org/sase/commit/16001bb368d7a24de1ac0ee4743b6d511eef7b62)
    — refactor(tui): split agent list build helpers

# Fix pager forward history and display the Tab shortcut

The user reports that Ctrl+I does not advance the pager's breadcrumb/history stack and
requests `<tab>` as the displayed shortcut. Implement this as one focused change to
pager key dispatch, its visible legends, documentation, and regression coverage. The
confirmed cause and small implementation surface justify a `tale` of size `small`; there
are no independently useful implementation phases.

## Confirmed cause

`src/sase/pager/screen.py` binds only `ctrl+i` to `trail_forward`. In the installed
Textual 8.0.1, the terminal byte `0x09` decodes to `Key("tab", "\t")`. Although Textual
defines a Ctrl+I alias for key handlers, its binding lookup uses the event's exact key
name. `PagerScreen` therefore inherits the screen's `tab` binding to `app.focus_next`
instead of routing that key to history.

A read-only headless reproduction followed a document and returned with Ctrl+O:

| Input                             | Current document | Remaining forward entries |
| --------------------------------- | ---------------- | ------------------------- |
| Ctrl+O after following the target | source           | 1                         |
| `pilot.press("tab")`              | source           | 1                         |
| `pilot.press("ctrl+i")`           | target           | 0                         |

The existing tests in `tests/pager/test_app_history.py`,
`tests/pager/test_bead_live_links.py`, and
`tests/pager/test_rendered_link_navigation.py` inject `ctrl+i` directly, bypassing the
terminal decoding that exposes the bug.

Replacing the Tab binding in memory with an ordinary, non-priority `trail_forward`
binding fixed the reproduction. That probe also passed with search typing, goto input,
help-sheet isolation, an empty forward stack, the existing Ctrl+I binding, and an
embedded host using ACE's priority Tab action and actual modal-availability guard. No
implementation files were changed during investigation.

## Implementation

1. **Bind the terminal key at the shared pager screen.** In `src/sase/pager/screen.py`,
   explicitly bind `tab` to `trail_forward` and retain `ctrl+i` as a compatibility
   alias. A combined `tab,ctrl+i` binding is sufficient; give it `key_display="<tab>"`
   so any Textual-generated display also uses the requested spelling. Keep ordinary
   binding priority. The screen's existing `on_key` ordering lets goto and search typing
   consume keys before normal bindings run, and prevents handling beneath the help
   modal. A priority binding would bypass those protections.

2. **Reuse the existing history action.** Route both key names to
   `PagerTrailMixin.action_trail_forward` in `src/sase/pager/_screen_trail.py`. Preserve
   one-entry-per-press traversal, bounded stacks, forward-branch clearing after
   following a new target, view restoration, and the empty-forward no-op. These existing
   Textual view-state mechanics already work when the action is reached. Keep the change
   at this presentation boundary; no shared backend API, resolver, or Rust core change
   is needed.

3. **Use literal `<tab>` in every pager shortcut display.** Update:

   | File                                   | Surface                                           |
   | -------------------------------------- | ------------------------------------------------- |
   | `src/sase/pager/_chrome.py`            | Conditional footer: `<tab> forward`               |
   | `src/sase/pager/_trail_chrome_band.py` | Full breadcrumb direction hint: `<tab> forward N` |
   | `src/sase/pager/_trail_chrome_help.py` | Key-guide row: `<tab>` / Walk forward             |
   | `docs/pager.md`                        | Forward-history key in the keys table             |

   Retain the footer's forward-history availability condition and the breadcrumb
   renderer's width-based fallback to compact counts. The longer label must still fit or
   degrade through those existing renderers. Use Rich `Text` as today so the angle
   brackets are literal. Leave unrelated Ctrl+I commands elsewhere alone.

4. **Confirm configuration and host integration.** Pager bindings are declared directly
   on `PagerScreen`; `src/sase/default_config.yml` currently has only syntax settings
   under `pager`, with no pager keymap scope. This fix therefore needs no new
   configuration setting. ACE's `next_tab` binding is priority-based, but
   `check_app_action` in `src/sase/ace/tui/_app_action_availability.py` already disables
   `next_tab` and `prev_tab` whenever a `ModalScreen` is active. Exercise that guard in
   the embedded regression instead of changing global Tab behavior.

## Regression coverage

Extend existing focused tests, using `pilot.press("tab")` for the main behavioral
regressions so they fail on the original implementation. Keep a focused case for the
explicit `ctrl+i` alias; changing all tests to synthetic Ctrl+I would miss the reported
failure again.

- In `tests/pager/test_app_history.py`, cover follow, back, and Tab forward with the
  body scroll focused. Assert the destination, trail position/counts, and restored view
  state. Exercise repeated forward steps and the existing branch truncation scenario
  through Tab. Check that Tab at the latest visit, including an initially empty history,
  leaves the pager open with unchanged document, stacks, and focus.
- With forward history available, press Tab while typing a search and while the goto
  prompt is open. Assert that the current document and history stay put and the prompt
  remains active with unchanged text. With help on top, Tab must not navigate the
  underlying pager. After dismissing the prompt or help, Tab must advance normally.
  Preserve the existing committed-search passthrough behavior and stored search-state
  restoration; this fix does not redefine search history.
- Update the existing real-link round trips in `tests/pager/test_bead_live_links.py` and
  `tests/pager/test_rendered_link_navigation.py` to exercise Tab, including their
  existing forward-history assertions and footer text expectations.
- Add an embedded-screen regression alongside
  `tests/ace/tui/actions/test_view_files_pager_screen.py`. Use a small host with the
  priority `next_tab` binding and the production modal guard, extending the existing
  harness or adding a local variant. Assert Tab advances the pager once and does not
  invoke the host tab-switch action. Also cover exhausted forward history so an
  unavailable destination cannot leak Tab to the host.
- Update `tests/pager/test_help.py` and the history/footer assertions for `<tab>`. Cover
  the conditional footer and full breadcrumb hint in `tests/pager/test_chrome.py` and
  `tests/pager/test_trail_chrome.py`; assert the old `^I`/`ctrl+i` labels are absent
  from those rendered shortcut surfaces. Retain the existing breadcrumb cell-width sweep
  and short-height rendering checks, which protect against overflow from the longer
  label. Existing PNG cases do not show forward history, so no blanket golden
  regeneration is expected.

## Verification and acceptance

Before implementation, consult `tui_perf.md` and `lint_and_test.md` through
`sase memory read`; their relevant requirements were reviewed during planning. Keep key
handling free of new I/O, subprocesses, and blocking work.

1. Demonstrate the new Tab regression failing before the binding fix, then passing after
   it. A lightweight terminal-decoder probe can confirm that `0x09` becomes `tab`; use
   the public Pilot key path for durable pager regressions rather than testing Textual's
   private parser internals.
2. Run the pager and embedded-screen tests, for example:

   ```bash
   .venv/bin/pytest -q tests/pager tests/ace/tui/actions/test_view_files_pager_screen.py
   ```

   The default pytest configuration excludes visual snapshots. If a changed surface
   requires a PNG update, run the relevant visual cases explicitly and inspect
   differences before accepting intentional changes.

3. Run `just check` after implementation, as required by repository guidance. Use
   `/sase_monitor` if verification becomes long-running; use that skill for
   `just check-full` if the repository's escalation rules require the full lane.
4. Review the diff and search the pager source, tests, and documentation for stale
   Ctrl+I display strings. Keep `ctrl+i` only where the input alias or its compatibility
   coverage is intentional. If a terminal session is available, smoke-test both physical
   Tab and Ctrl+I after following a link and going back, and inspect `<tab>` in the
   footer, breadcrumb band, and help sheet.

Acceptance requires working Tab traversal in both pager hosts, retained Ctrl+I
compatibility, unchanged prompt/modal ownership and history edge behavior, consistent
`<tab>` shortcut displays, and passing focused tests plus `just check`.
