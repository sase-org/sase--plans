---
tier: tale
title: Contain unhandled printable keys inside vim NORMAL/VISUAL mode
goal: Printable keys the vim layer does not consume never reach app-level bindings,
  so NORMAL-mode <space> in the prompt input bar moves the cursor right instead of
  discarding the prompt as cancelled.
create_time: 2026-07-24 22:19:12
status: wip
---

- **PROMPT:** [202607/prompts/vim_normal_mode_key_containment.md](prompts/vim_normal_mode_key_containment.md)

# Plan: Contain unhandled printable keys inside vim NORMAL/VISUAL mode

## 1. Reported symptom

Pressing `<space>` in NORMAL mode while the prompt input bar is focused wipes the entire prompt and records it in prompt
history as a **cancelled** prompt.

## 2. Verified diagnosis

The user's suspicion is **confirmed, and the problem is broader than `<space>`**.

### Confirmed reproduction

With a home-mode prompt bar mounted carrying `hello world`, focused, in NORMAL mode, pressing `space` fires the
app-level `start_agent_home` action. Observed result:

- `action_start_agent_home` is dispatched;
- the mounted `PromptInputBar` is replaced by a brand-new instance (different object identity), whose text is `''` and
  whose vim mode is `insert`;
- the old text is written to prompt history with `cancelled=True`.

### Causal chain

1. `src/sase/ace/tui/bindings.py` binds `space` at app level: `Binding("space", "start_agent_home", ...)` (keymap key
   `app.start_agent_home` in `src/sase/default_config.yml`). That binding is correct and must stay — it is how the user
   opens a home-mode prompt from a list tab.
2. `PromptTextArea._on_key` (`src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py`, NORMAL branch near line 260)
   calls `self._handle_normal_mode_key(event)` and only calls `event.stop()` / `event.prevent_default()` when that
   returns `True`.
3. `VimNormalModeMixin._handle_normal_mode_key` (`src/sase/ace/tui/widgets/_vim_normal.py`) has no `<space>` command:
   `key` resolves to `" "` (a real `space` keypress arrives as `Key(key="space", character=" ")`, so
   `event.character or event.key` is `" "`), no motion or edit branch matches, and it returns `False`.
4. The unconsumed `Key` event therefore bubbles out of the widget to the Screen and App, where the app-level `space`
   binding matches and runs `action_start_agent_home`.
5. `action_start_agent_home` → `_show_prompt_input_bar_for_home()`
   (`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_mount.py`) begins with `self._unmount_prompt_bar()`, whose
   documented "safety net" `_save_bar_text_as_cancelled` → `_save_text_as_cancelled` calls
   `add_or_update_prompt(text, cancelled=True)`. It then mounts a fresh empty bar.

So the prompt is not "cleared" by a bug in the prompt bar: the app faithfully honours a keystroke that should never have
escaped the focused vim editor.

### The bug is a whole class, not one key

`VimTextArea._on_key` documents its contract as _"Only keys the vim layer actually consumes are stopped; everything else
bubbles so host-level bindings (confirm, cancel, etc.) keep working."_ That contract is too loose: it leaks
**printable** keys too, not just structural ones. Probing every app binding key against a focused prompt bar in NORMAL
mode on the ChangeSpecs tab showed **19** app actions firing from inside the editor:

| key     | app action dispatched     |
| ------- | ------------------------- |
| `q`     | `quit`                    |
| `Q`     | `stop_axe_and_quit`       |
| `space` | `start_agent_home`        |
| `+`     | `start_custom_agent`      |
| `@`     | `restore_prompt_stash`    |
| `!`     | `start_bang_mode`         |
| `#`     | `open_config_center`      |
| `*`     | `open_saved_query_picker` |
| `_`     | `next_query`              |
| `'`     | `jump_to_entry`           |
| `` ` `` | `jump_to_all_entries`     |
| `m`     | `toggle_mark`             |
| `M`     | `mail`                    |
| `H`     | `hooks_or_collapse_all`   |
| `L`     | `expand_all_folds`        |
| `U`     | `toggle_agent_unread`     |
| `R`     | `start_rewind`            |
| `z`     | `start_fold_mode`         |
| `Z`     | `zoom_panel`              |

(Other tabs bind different actions to the same keys, so the reachable action set varies by tab.) Each of these is a real
vim NORMAL-mode key that this vim tower does not implement (`q` macro record, `m`/`'`/`` ` `` marks, `z` scroll
commands, `R` replace mode, `H`/`L` screen top/bottom). In vim an unmapped printable NORMAL-mode key is a no-op; here it
runs a global command.

The leak is **not specific to `PromptTextArea`**. It lives in the shared base, so every `VimTextArea` host on the main
screen leaks. Verified: the artifacts filter-bar inputs (`#commit-filter-input`, `#plan-filter-input`,
`#chat-filter-input`) each dispatch `start_agent_home` on NORMAL-mode `space`. The prompt bar's own frontmatter editors
(`#frontmatter-content`, `#frontmatter-raw`, `#frontmatter-inline`) are plain `VimTextArea` / `SingleLineVimTextArea`
instances mounted inside the bar, so they leak the same way — pressing `space` there destroys the very prompt being
edited.

Inside `ModalScreen`s the blast radius is limited to that modal's own bindings, because Textual stops binding
propagation at a modal screen.

### Why not fix this in `check_action`

`AceApp.check_action` (`src/sase/ace/tui/app.py`) already special-cases two focus-sensitive actions (`next_tab` /
`prev_tab` are disabled when `isinstance(self.focused, VimTextArea)`). Extending that pattern would require naming every
present and future app action, per tab — an unbounded maintenance burden that a new binding silently breaks. Containment
belongs where the vim mode is known: in the widget.

## 3. Design

Three parts.

### 3.1 Containment (the root-cause fix)

Tighten the `VimTextArea._on_key` contract to: **in NORMAL / VISUAL / V-LINE mode, a printable key that the vim layer
did not consume is swallowed.** Non-printable keys keep bubbling exactly as today, which preserves every host contract
the current docstring is protecting — those are all structural keys (`enter`, `escape`, `tab`, `ctrl+…`, arrows).
`textual.events.Key.is_printable` is precisely the right discriminator: it is `True` for letters, digits, punctuation
**and `space`**, and `False` for `enter`, `tab`, `escape`, and every ctrl/alt chord.

Verified must-keep behaviours (all non-printable, all unaffected):

- NORMAL-mode `escape` with nothing pending stays unconsumed so a host modal can back out (explicit contract in the
  `_on_key` docstring).
- `enter` reaches the prompt-bar submit path; `ctrl+l` still reaches `dismiss_toasts`; `ctrl+o` still reaches
  `jump_to_entry_fast`.

### 3.2 One deliberate opt-out

Exactly one host intentionally claims a printable NORMAL-mode key, and it is covered by an existing test:
`tests/ace/tui/test_axe_entry_editor_modal.py::test_q_is_insert_text_then_closes_cell_from_normal_mode` asserts that in
the AXE entry editor's focused value cell, INSERT-mode `q` types `q` while NORMAL-mode `q` closes the editor (the
modal's `Binding("q", "quit_editor")`). Preserve that with an overridable host hook rather than a global exception.

This was established empirically, not guessed: running the full suite with the containment guard patched in produced
exactly one failure — that test.

### 3.3 Make `<space>` do what vim does

Containment alone leaves `<space>` inert. In vim, NORMAL-mode `<Space>` is a cursor-right motion. Alias it onto the
existing `l` motion in both NORMAL and VISUAL mode so it inherits counts (`3<space>`), operator composition
(`d<space>`), and dot-repeat for free.

## 4. Implementation

### 4.1 `src/sase/ace/tui/widgets/vim_text_area.py`

Add two members to `VimTextArea` (place the hook with the other "Host hooks" at the bottom of the class, and the swallow
helper next to `_has_pending_normal_state` in the "Lean key dispatch" section):

```python
    def _swallow_unhandled_vim_key(self, event: Key) -> None:
        """Consume a printable NORMAL/VISUAL key the vim layer did not handle.

        In vim an unmapped printable NORMAL-mode key is a no-op; letting it bubble
        instead hands it to whatever app-level binding owns that character -- e.g.
        bare ``space`` would reach ``start_agent_home``, which unmounts the prompt
        bar and rewrites its text to history as cancelled. Only printable keys are
        swallowed, so the structural keys hosts rely on (``enter``, ``escape``,
        ``tab``, every ctrl/alt chord) keep bubbling as before.
        """
        if not event.is_printable:
            return
        if self._host_claims_unhandled_vim_key(event):
            return
        event.stop()
        event.prevent_default()

    def _host_claims_unhandled_vim_key(self, event: Key) -> bool:
        """Return whether the host owns this unhandled printable key. Default: no.

        Overridden by hosts that deliberately bind a printable NORMAL-mode key the
        vim layer leaves free -- the AXE entry editor's value cells let ``q`` reach
        the sheet's quit binding.
        """
        return False
```

Then call it on every path in `_on_key` where a NORMAL/VISUAL key ends up unconsumed — including the
`_vim_tower_dispatched` early return, which is exactly the path a `PromptTextArea` unhandled key takes (the subclass
dispatch runs first and returns without stopping; had it consumed the key it would have called `prevent_default`, ending
the MRO walk before this handler):

```python
        if self._vim_mode in ("visual", "visual_line"):
            if getattr(event, "_vim_tower_dispatched", False):
                self._swallow_unhandled_vim_key(event)
                return
            if self._handle_visual_mode_key(event):
                event.stop()
                event.prevent_default()
                return
            self._swallow_unhandled_vim_key(event)
            return

        if self._vim_mode == "normal":
            if getattr(event, "_vim_tower_dispatched", False):
                self._swallow_unhandled_vim_key(event)
                return
            if event.key == "escape" and not self._has_pending_normal_state():
                return
            if self._handle_normal_mode_key(event):
                event.stop()
                event.prevent_default()
                return
            self._swallow_unhandled_vim_key(event)
            return
```

Update the `_on_key` docstring: the "everything else bubbles" sentence must now say that unconsumed **printable** keys
are swallowed in NORMAL/VISUAL mode and only non-printable keys bubble to host bindings.

### 4.2 `src/sase/ace/tui/widgets/_vim_normal_motions.py`

In `_handle_normal_motion_key`, widen the `l` branch (currently `if key == "l":`) to `if key in ("l", " "):`, and note
in a short comment that `<Space>` is vim's cursor-right motion. Nothing else changes: `<space>` reaches this dispatcher
only as a top-level NORMAL command, because `_handle_normal_mode_key` routes pending sequences to
`_handle_normal_pending_key` first — so `f<space>`, `ds<space>`, `cs(<space>` and `[<space>` / `]<space>` (insert blank
line) keep treating space as literal data.

### 4.3 `src/sase/ace/tui/widgets/_vim_visual_keys.py`

In `_handle_visual_mode_key`, widen the `l` branch the same way so `<space>` extends a visual selection right.

### 4.4 `src/sase/ace/tui/modals/axe_entry_editor_rendering.py`

Both cell-editor classes (`AxeValueInput(SingleLineVimTextArea)` and `AxeValueTextArea(VimTextArea)`) override the hook
so NORMAL-mode `q` still reaches the sheet:

```python
    def _host_claims_unhandled_vim_key(self, event: Key) -> bool:
        """Let NORMAL-mode ``q`` reach the sheet's ``quit_editor`` binding."""
        return event.character == "q"
```

Add the `textual.events.Key` import if the module does not already have it. If both classes would carry an identical
override, factor it into a small shared mixin or module-level helper in that file rather than duplicating the body.

## 5. Tests

### 5.1 New: `tests/ace/tui/widgets/test_vim_normal_key_containment.py`

Use the `AcePage` DSL (`sase.ace.testing`) with `sase.config.load_merged_config` patched to `{"ace": {}}`, mirroring
`tests/test_keymaps_e2e.py`. Mount a home prompt bar via `page.app._show_prompt_input_bar_for_home(initial_text=...)`,
focus the `PromptTextArea`, press `escape` to enter NORMAL mode. Record dispatched actions by replacing
`page.app.run_action` with an async recorder.

1. **The reported bug.** With `initial_text="hello world"`, NORMAL-mode `space` must leave the same `PromptInputBar`
   instance mounted, its text unchanged, no app action dispatched, and no `cancelled=True` history write (patch
   `sase.history.prompt.add_or_update_prompt` — note `_save_text_as_cancelled` imports it inside the function, so patch
   it at its source module — and assert it is never called). Also assert the cursor advanced one column, i.e. `<space>`
   behaved as `l`.
2. **The whole class, parametrized.** For the printable keys above (`q`, `Q`, `space`, `plus`, `at`, `exclamation_mark`,
   `number_sign`, `asterisk`, `underscore`, `apostrophe`, `grave_accent`, `m`, `M`, `H`, `L`, `U`, `R`, `z`, `Z`),
   assert no app action is dispatched from prompt NORMAL mode. Keep this one parametrized test rather than 19 separate
   ones.
3. **VISUAL mode.** Enter visual mode (`v`) and assert the same keys dispatch nothing.
4. **Non-printable keys still bubble.** From prompt NORMAL mode, `ctrl+l` still dispatches `dismiss_toasts` and `ctrl+o`
   still dispatches `jump_to_entry_fast`. This is the regression guard against over-swallowing.
5. **Other vim hosts on the main screen.** Focus an artifacts filter-bar input (`#commit-filter-input`), enter NORMAL
   mode, press `space`, assert nothing dispatched. Do the same for the prompt bar's frontmatter editor: open the panel
   with `PromptInputBar.focus_frontmatter_panel()` on a prompt seeded with frontmatter, focus `#frontmatter-content`,
   and assert NORMAL-mode `space` neither dispatches an action nor unmounts the bar.

### 5.2 `<space>` as a motion

Add to `tests/test_prompt_normal_mode_motions.py` (uses the lighter `PromptPage` harness): `<space>` moves right one
column; `3<space>` moves three; it clamps at end of line; `d<space>` deletes the character under the cursor; `.` repeats
that delete. Add to `tests/test_prompt_visual_mode.py`: `v` then `<space>` extends the selection one column right.

### 5.3 Existing tests that must stay green

`tests/ace/tui/test_axe_entry_editor_modal.py::test_q_is_insert_text_then_closes_cell_from_normal_mode` is the guard for
§4.4 — do not weaken it. A full-suite run with the guard in place broke only this test, so no other fallout is expected;
investigate anything else that fails rather than relaxing the guard.

## 6. Validation

```bash
just install     # ephemeral workspace: dependencies may be stale
just check
```

The visual PNG snapshot suite is unaffected (no rendering change), but `just check` covers it; if a snapshot moves,
treat it as a real regression rather than re-baselining.

## 7. Decisions and non-goals

- **Keep the app-level `space` → `start_agent_home` binding and the keymap entry as they are.** They are correct outside
  a focused vim editor; nothing in `src/sase/default_config.yml` or `src/sase/ace/tui/keymaps/` needs to change.
- **Keep `_unmount_prompt_bar`'s cancelled-history safety net.** It behaved as designed; it was fed a keystroke it
  should never have received.
- **Do not add `check_action` special cases** for the leaking actions (see §2).
- **No help-modal change.** `PROMPT_INPUT_SECTION` in `src/sase/ace/tui/modals/help_modal/binding_common.py` documents
  prompt-specific keys, not the standard vim layer (`h`/`j`/`k`/`l`/`w`/`b` are all absent), and `<space>` is now simply
  an alias of `l`. Adding a row would be inconsistent with that section's scope.
- **Out of scope:** `enter` reaching the priority-bound `activate_bug_link` from a focused prompt bar, and whether
  `check_action`'s `next_tab` / `prev_tab` focus guard actually suppresses `tab` while a `VimTextArea` is focused. Both
  were noticed while probing; both involve non-printable keys and are unrelated to this fix. Do not chase them here.
