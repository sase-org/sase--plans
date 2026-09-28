---
tier: tale
title: Stop Tab in the prompt input from switching TUI tabs
goal:
  Pressing Tab or Shift+Tab while a prompt input bar is open never switches the TUI tab,
  even when a background Agents-tab refresh happens around the keypress.
size: small
proposed_by: bbugyi200.athena.0to
create_time: 2026-09-28 15:10:39
status: wip
---

# Plan: Fix `Tab` in the prompt input sometimes switching the TUI tab

## Problem

While typing in the prompt input bar on the Agents tab, pressing `Tab` occasionally
switches the whole TUI to the next tab (Agents -> Artifacts) instead of doing prompt
work (snippet expansion, tabstop jumps, list indent). It is intermittent.

## Root cause (reproduced)

1. `tab` / `shift+tab` are **priority** app bindings (`next_tab` / `prev_tab` in
   `src/sase/ace/tui/bindings.py`). In Textual 8, `App.on_event` runs priority bindings
   _before_ forwarding the key to the focused widget, so the prompt's own `Tab` handling
   in `_prompt_text_area_key_handling.py` never gets a chance when the binding is
   allowed.
2. The only thing stopping that binding while a prompt is open is in `check_app_action`
   (`src/sase/ace/tui/_app_action_availability.py`): `next_tab`/`prev_tab` return
   `False` only when `isinstance(app.focused, VimTextArea)`. It is keyed on _current
   focus_, not on "a prompt bar is mounted".
3. Background Agents-tab repaints steal Textual focus from the prompt:
   `PanelLayoutMixin._focus_focused_panel_widget()` and the tail of
   `_refresh_focused_agent_panel_impl()` in
   `src/sase/ace/tui/actions/agents/_display_panel_layout.py` call `AgentList.focus()`.
   Their only guard is `_hint_input_bar_active()`, which covers hint/accept/rewind bars
   and the Agents filter bar (the same bug class fixed for sase-zf.4) but not the prompt
   input bar. `_focus_focused_panel_widget()` runs at the end of
   `_refresh_panel_widgets_impl`, `_refresh_affected_panel_widgets`, and
   `_refresh_agent_panel_titles`, so it fires on ordinary background work while the
   prompt is open: agent loads (`_refresh_agents_display(list_changed=True)`),
   unread-badge updates on agent completion (`_notification_unread_projection.py`), and
   deck-layout title refreshes (`_agent_detail_deck_layout.py`).
4. `PromptTextArea.on_blur` (`_prompt_text_area_actions.py`) takes focus back, but only
   after several deferred hops: blur -> `call_later(_refocus_if_needed)` -> `focus()` ->
   `app.call_later(set_focus)`. A `Tab` key that reaches the app queue in that window
   sees `app.focused` = `AgentList`, so the `VimTextArea` guard passes,
   `action_next_tab` runs, and then focus snaps back to the prompt. The user sees the
   prompt still focused, but the TUI has moved to Artifacts.

The Services tab already follows the right policy: `_focus_axe_focused_panel` in
`actions/axe_display/_render_panels.py` only moves focus when a `BgCmdList` already owns
it, "so background refreshes never steal focus from the prompt bar". The Agents-tab
equivalent is missing that protection.

Reproduction (full-app `AcePage`): mount the home prompt on the Agents tab, call
`app._refresh_agents_display(list_changed=True)` (or
`app._refresh_agent_panel_titles()`), then immediately
`app.post_message(events.Key("tab", None))` and pause. Result:
`current_tab == "artifacts"`. The control case with no refresh stays on `"agents"`. Each
of the two fixes below, applied alone, keeps it on `"agents"`. Note that
`page.press("tab")` does **not** reproduce it: Textual's pilot waits for idle before
posting the key, which closes the refocus window. Regression tests must post the `Key`
event directly.

A side effect of the same steal: any printable key typed in that window (for example
`j`/`k`) goes to the `AgentList` instead of the prompt. Fix 1 removes that too.

## Changes

### 1. Stop Agents-tab repaints from stealing focus from a mounted prompt (root cause)

In `src/sase/ace/tui/actions/agents/_display_panel_layout.py`:

- Add one small helper on `PanelLayoutMixin`, e.g.
  `_agent_list_focus_suppressed(self) -> bool`. It returns `True` when either
  `_hint_input_bar_active()` or `_prompt_input_active()` returns truthy. Resolve both
  with the existing `getattr(self, name, None)` + `callable(...)` pattern so the
  lightweight test fakes that lack these methods keep working. `_prompt_input_active` is
  defined on `EventHandlersBase` in `actions/_event_base.py` and is `True` whenever a
  `PromptInputBar` is mounted (or the prompt editor is suspended).
- Use the helper in place of the inline hint-bar checks in both focus sites:
  - `_focus_focused_panel_widget()`: return early when the helper is `True`.
  - `_refresh_focused_agent_panel_impl()`: the final `focused_widget.focus()` block
    keeps its `panel_focus is None` condition and swaps its hint-bar check for the
    helper.
- Keep the existing highlight and `-focused-panel` class updates unchanged. Only the
  Textual `focus()` call is suppressed.
- Update the docstrings to say why: background repaints must never steal keyboard focus
  from a mounted prompt input bar, because `tab` / `shift+tab` are priority app bindings
  guarded by focus. Mention the matching Services-tab policy in
  `_focus_axe_focused_panel`.

This is safe for prompt dismissal. `_detach_prompt_bar` in
`actions/agent_workflow/_prompt_bar_mount.py` already moves focus to the active tab's
list explicitly via `_transfer_focus_off_prompt_bar` before it synchronously detaches
the bar. After that, `_prompt_input_active()` is `False` again and later repaints focus
the list as before.

### 2. Defense in depth: `next_tab` / `prev_tab` are unavailable while a prompt owns keys

In `src/sase/ace/tui/_app_action_availability.py`, extend the existing
`if action in ("next_tab", "prev_tab"):` block. It should return `False` when
`isinstance(app.focused, VimTextArea)` **or** `_prompt_input_owns_keys(app)` (the helper
already defined in that module). This follows the module's existing rule that "a mounted
prompt owns ... the rest of the Agents row map". Add a short comment explaining that
focus can transiently leave the prompt's `VimTextArea` (deferred refocus after a blur,
mount-time deferred focus, frontmatter-panel browse mode), so the guard must not depend
on focus alone.

Behavioral consequence (intended): with a prompt open and the frontmatter panel in
browse ("rows") mode, `Tab` no longer switches TUI tabs; it becomes a no-op there. Keys
remapped onto `next_tab`/`prev_tab` follow the same rule, which matches today's behavior
when the prompt `VimTextArea` is focused. With no prompt mounted, nothing changes: the
sase-zf.4 filter-bar behavior and `test_agents_filter_bar.py` keep working.

### 3. Docs

In `docs/ace.md`, under "## Tab System" (the "cycled with `Tab` and `Shift+Tab`"
sentence), add one sentence: while a prompt input bar is open, `Tab`/`Shift+Tab` belong
to the prompt (snippet expansion, tabstops, list indent) and never switch tabs. No
keymap or `src/sase/default_config.yml` changes are needed; no bindings change.

## Tests

1. **Unit (focus sites).** In `tests/ace/tui/test_agent_panel_first_selection.py`, next
   to `test_focused_panel_widget_focus_skips_all_hint_bar_modes`, add a test using the
   `_OptimizedPanelSwitchApp` harness. Set `app._prompt_input_active = lambda: True`,
   then assert `focus_calls == 0` after both `app._focus_focused_panel_widget()` and
   `app._refresh_focused_agent_panel_impl(old_focused_idx=None)`. Also assert that
   highlight and `-focused-panel` updates still happen in the latter. Add a control
   assertion: with `_prompt_input_active` returning `False`, `focus()` is still called.
2. **Unit (availability).** Add a focused `check_app_action` test in the style of
   existing availability tests, for example
   `tests/ace/tui/test_view_agent_metadata_availability.py`. With a fake app whose
   `_prompt_input_active()` is `True` and whose `focused` is a non-`VimTextArea` widget,
   `next_tab` and `prev_tab` are unavailable. With no prompt mounted and the same focus,
   they fall through to the fallback (available).
3. **Full-app regression.** Add a new module such as
   `tests/ace/tui/widgets/test_prompt_tab_focus_steal.py`. Copy the module-scoped
   `AcePageGroup` fixture and the `_mount_home_prompt` pattern from
   `tests/ace/tui/widgets/test_vim_normal_key_containment.py`, patching
   `sase.config.load_merged_config` to `{"ace": {}}`. With the app on the Agents tab and
   the prompt mounted and focused, parametrize over vim mode (INSERT, NORMAL) and
   trigger (`app._refresh_agents_display(list_changed=True)`,
   `app._refresh_agent_panel_titles()`). For each: call the trigger, immediately
   `app.post_message(events.Key("tab", None))` (not `page.press`, see root cause),
   `await page.pause()`, then assert `app.current_tab == "agents"` and `app.focused is`
   the prompt's active text area. Add one more case that exercises Fix 2 on its own:
   synchronously move focus to the Agents `AgentList` with
   `app.screen.set_focus(<first AgentList>)`, post the `tab` key right away, pause, and
   assert the tab is still `"agents"`.

## Verification

Run `just check` (read the `lint_and_test` memory for how). Do not run
`just check-full`. Run `just fmt` first if needed. No Rust-core changes: this is
presentation-only Textual focus/keybinding behavior that belongs in this repo.
