---
tier: tale
title: Make Ctrl+Space inert while a prompt input bar is open
goal:
  Pressing Ctrl+Space while a prompt input bar is mounted no longer tears down the bar and wipes the in-progress prompt;
  the binding keeps working everywhere else.
create_time: 2026-07-28 11:56:35
status: done
---

- **PROMPT:** [202607/prompts/ctrl_space_prompt_guard.md](prompts/ctrl_space_prompt_guard.md)
- **AGENTS:**
  - [bbugyi200.athena.n4--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.n4.md#member-code)
- **COMMITS:**
  - [18d0d92](https://github.com/sase-org/sase/commit/18d0d924179e250a32cd14f048fbce01f4acbe6f) — fix(tui): preserve active prompts on Ctrl+Space

# Plan: Make Ctrl+Space inert while a prompt input bar is open

## Problem

`Ctrl+Space` is bound at app level to `start_agent_from_changespec` ("Repeat last +/Ctrl+Space selection"). While the
user is typing a prompt in the prompt input bar, an accidental `Ctrl+Space` destroys that prompt.

Root cause chain (all paths verified by reading the code):

1. `src/sase/ace/tui/bindings.py:157` registers
   `Binding("ctrl+@", "start_agent_from_changespec", "Run Agent (PR)", show=False)`, and
   `src/sase/default_config.yml:282` sets the same default key. `ctrl+@` is Textual's canonical name for Ctrl+Space
   (`Keys.ControlSpace` aliases to `ctrl+@`; the repo's own `src/sase/ace/tui/keymaps/key_validation.py:53-56`
   normalizes `ctrl+space` / `ctrl+at` to `ctrl+@`).
2. The key event first reaches the focused `PromptTextArea`. `PromptTextAreaKeyHandlingMixin._on_key`
   (`src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py`) explicitly consumes `enter`, `ctrl+s`, `ctrl+c`,
   `ctrl+k`, `ctrl+r`, `ctrl+g`, `ctrl+t`, `tab`/`shift+tab`, etc., but has no branch for `ctrl+@`, so it falls through
   to `await super()._on_key(event)`.
3. Textual's `TextArea` only inserts printable characters. A `ctrl+@` key event carries `character="\x00"`, and
   `Key.is_printable` is `False` for it (confirmed against the pinned `textual==8.0.1`), so `TextArea` does not consume
   the event either and it bubbles to the app.
4. `App._on_key` -> `App._check_bindings` matches the app-level `ctrl+@` binding and runs
   `AceApp.action_start_agent_from_changespec` (`src/sase/ace/tui/actions/agent_workflow/_entry_custom.py:35`).
5. That action ends in `_start_custom_agent_from_selection` -> `_show_prompt_input_bar_for_home`
   (`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_mount.py:351`), whose first statement is
   `self._unmount_prompt_bar()`. The in-progress prompt is replaced by a brand-new bar seeded with the replayed VCS
   prefix.

The old text is not lost forever - `_unmount_prompt_bar` routes it through `_save_bar_text_as_cancelled`, so it is
recoverable from prompt history (`Ctrl+K`) - but from the user's point of view the prompt is wiped.

Note why the sibling launch entry points do not have this bug: `+` (`start_custom_agent`) and `space`
(`start_agent_home`) are printable, so the focused `TextArea` swallows them before they can reach the app. `Ctrl+Space`
is the only prompt-clobbering launch action bound to a non-printable key, which is why it is the only one that leaks.

## Chosen fix

Gate the **action**, not the key, in `AceApp.check_action` (`src/sase/ace/tui/app.py:291`): while any prompt-like input
surface is active, `start_agent_from_changespec` is unavailable.

Textual's dispatch makes this sufficient. `App._check_bindings` runs the binding through `App.run_action`, which calls
`check_action` on the action target (the app) and skips dispatch entirely when it returns a falsey value. A skipped
binding does not consume the key, so the event simply dies with nothing else claiming it - a silent no-op, which is
exactly the requested behavior.

### Why this layer and not a widget-local shadow

The obvious alternative is to consume `ctrl+@` inside `PromptTextAreaKeyHandlingMixin._on_key` with `event.stop()` /
`event.prevent_default()`, the way `ctrl+g` is shadowed there. Reject it:

- **It leaves a real hole.** `FrontmatterPanel` sets `can_focus = True`
  (`src/sase/ace/tui/widgets/frontmatter_panel.py:95`) and the prompt bar can hand it focus via
  `focus_frontmatter_panel` (`src/sase/ace/tui/widgets/_prompt_input_bar_frontmatter.py:226`); the panel's cell editors
  are separate `VimTextArea`s. With focus on any of those the `PromptTextArea` handler never runs, `ctrl+@` still
  bubbles to the app, and the whole bar - frontmatter included - is still destroyed. The existing test
  `test_other_main_screen_vim_hosts_contain_normal_space` already exercises exactly that focus position, so it is a
  reachable state, not a theoretical one.
- **It hard-codes the key.** Keys are user-rebindable through the keymap registry; a literal `ctrl+@` branch stops
  protecting anything the moment `start_agent_from_changespec` is rebound to another non-printable key. Gating the
  action name survives rebinding.
- **`check_action` is the established idiom here.** The same method already disables `next_tab` / `prev_tab` when a
  `VimTextArea` has focus and `search_forward` / `search_reverse` when `_prompt_input_active()` is true - the identical
  "do not let an app key fire while the user is typing" concern.

Do not implement both layers. One mechanism, one place to maintain.

### Predicate

Reuse the existing `_prompt_input_active()` helper (`src/sase/ace/tui/actions/_event_base.py:87`). It is the codebase's
single definition of "a prompt-like input surface is mounted": it covers `_prompt_context`, `_approve_prompt_context`,
`_plan_feedback_context`, and any mounted `PromptInputBar` widget. Do not invent a second, narrower predicate.

Mirror the `_screen_stack` guard already used by the `search_forward` branch a few lines above. `_prompt_input_active()`
ends in `self.query(PromptInputBar)`, which raises when no screen is mounted, and `check_action` can be reached before
the app is running.

## Change

Single edit in `AceApp.check_action` in `src/sase/ace/tui/app.py`. Add the new branch immediately after the existing
`{"search_forward", "search_reverse"}` branch (which currently ends with its `return False`, just before the
`if action == "edit_query"` branch):

```python
        # ``Ctrl+Space`` replays the last launch selection by remounting the
        # prompt bar, which tears down whatever the user is currently typing
        # (``_show_prompt_input_bar_for_home`` unmounts first). The printable
        # launch keys (``+``, ``space``) are swallowed by the focused TextArea,
        # so this non-printable one is the only launch entry point that can
        # reach the app mid-prompt. Disable the action instead of the key so
        # the guard survives rebinding and covers every focus position inside
        # the bar (prompt panes, frontmatter panel, frontmatter cell editors).
        if action == "start_agent_from_changespec" and (
            bool(getattr(self, "_screen_stack", ()))
            and self._prompt_input_active()
        ):
            return False
```

Return `False` (disabled and hidden), matching every other gate in this method. Nothing in `src/` reads
`Screen.active_bindings`, and the keybinding footer and help modal build their content from the keymap registry rather
than from Textual's active-binding map, so this has no footer or help-popup side effects.

Deliberately in scope: only `start_agent_from_changespec`. Do not gate `start_custom_agent` / `start_agent_home` /
`restore_prompt_stash` - their keys are printable and cannot leak, so adding them would be dead code.

## Tests

Add to `tests/ace/tui/widgets/test_vim_normal_key_containment.py` ("App-level containment coverage for vim NORMAL/VISUAL
printable keys"). It already has the exact fixtures needed: `_patch_config()`, `_mount_home_prompt()`, and a
frontmatter-focus walkthrough in `test_other_main_screen_vim_hosts_contain_normal_space`.

**Important:** do not assert through the file's `_record_actions` helper for these cases. It patches
`page.app.run_action`, and `check_action` is called _inside_ `run_action` - so the patched recorder would log the
attempted action regardless of the gate and the test would fail for the wrong reason. Assert on the user-visible outcome
instead.

Add these cases:

1. `Ctrl+Space` with the prompt text area focused in INSERT mode leaves the bar intact: mount via `_mount_home_prompt`,
   `await page.press("i")` to return to INSERT (the helper leaves the pane in NORMAL), capture the bar object, press
   `ctrl+@`, then assert `page.query_one_widget("#prompt-input-bar", PromptInputBar)` is still the same object and
   `text_area.text` is unchanged. Also patch `sase.history.prompt.add_or_update_prompt` and assert it was not called,
   the way `test_normal_space_moves_without_remount_or_cancelled_history` does - a remount would have written a
   `cancelled=True` history entry.
2. Same assertions with the pane in NORMAL mode (no extra `press("i")`).
3. Same assertions with focus moved onto the frontmatter panel. Reuse the `as_xprompt_markdown=True` mount plus
   `bar.focus_frontmatter_panel()` sequence from `test_other_main_screen_vim_hosts_contain_normal_space`; this is the
   case the widget-local alternative would have missed.
4. A direct gate assertion: `page.app.check_action("start_agent_from_changespec", ()) is False` while a bar is mounted,
   and `is not False` on a fresh page with no bar mounted (the `is not False` form matches the idiom in
   `tests/ace/tui/test_agents_zoom_panel_action.py`).

Also confirm the existing `test_normal_non_printable_keys_still_reach_app_actions` still passes unchanged - it asserts
`ctrl+l` and `ctrl+o` _do_ still reach app actions from inside the prompt, and is the regression guard proving this
change did not over-gate.

## Non-goals / explicitly not changing

- **Help modal text.** `src/sase/ace/tui/CLAUDE.md` requires updating the `?` popup whenever a `sase ace` option's
  behavior changes. Leave the three entries ("Repeat last +/Ctrl+Space selection" in
  `help_modal/changespecs_bindings.py`, `agents_bindings.py`, `axe_bindings.py`) as they are: the binding's documented
  purpose is unchanged, it is inert only while a prompt bar owns the keyboard - which is already true of every other app
  binding while typing - and the help modal enforces a 32-character description cap that leaves no room for a caveat.
  Note this reasoning in the commit message so the exception is explicit.
- **No docs change.** No file under `docs/` mentions Ctrl+Space (checked).
- **No keymap-registry, `default_config.yml`, or `bindings.py` change.** The binding itself stays exactly as-is; only
  its availability is conditioned.
- **No toast on the suppressed press.** A silent no-op is the requested behavior, and the press is by definition usually
  accidental, so a toast would fire precisely when it is least wanted.
- **No Rust core work.** Per the repo's Rust-core boundary rule, TUI keybindings and availability gating are
  presentation-only and stay in Python.

## Verification

Run from the workspace checkout root:

```bash
just install   # required: ephemeral workspaces may have stale deps
just check
```

Plus the targeted suite while iterating:

```bash
.venv/bin/python -m pytest tests/ace/tui/widgets/test_vim_normal_key_containment.py -q
```

## Manual smoke check

1. `sase ace`, press `+` and launch selection once so a replay target is saved.
2. Press `+` (or `Ctrl+Space`) to open a prompt bar, type some text.
3. Press `Ctrl+Space`: the text and the bar must be untouched, with no toast.
4. Press `Ctrl+C` to cancel the bar, then press `Ctrl+Space` again: the replay must still work normally and mount a
   fresh bar with the VCS prefix.
