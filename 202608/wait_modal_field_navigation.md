---
tier: tale
title: Wait modal ctrl+j/ctrl+k field navigation
goal:
  The Wait panel supports <ctrl+j> / <ctrl+k> to jump to the next / previous form input,
  wrapping at both ends, without Textual's Input bindings eating field text.
size: small
proposed_by: bbugyi200.athena.zs
create_time: 2026-08-13 13:45:53
status: wip
---

# Plan: Wait Modal `<ctrl+j>` / `<ctrl+k>` Field Navigation

## Problem

The Wait panel (`WaitModal`), opened with `w` on the Agents tab, is a five-field form:

| Order | Widget id        | Label    |
| ----- | ---------------- | -------- |
| 1     | `agents-input`   | Agents   |
| 2     | `beads-input`    | Beads    |
| 3     | `time-input`     | Time     |
| 4     | `runners-input`  | Runners  |
| 5     | `priority-input` | Priority |

Today the only way to move between those fields is `tab`, and `tab` is overloaded:
`action_accept_completion` first tries to accept the highlighted agent/bead completion
and only falls through to `self.focus_next()` when nothing is highlighted. Because the
agents/beads completion `OptionList` is focusable while displayed (see
`_apply_active_completion_visibility` in
`src/sase/ace/tui/modals/wait_modal_completion.py`), `focus_next()` also stops on the
completion list, so plain focus cycling is neither direct nor predictable.

There is no dedicated keymap for "jump to the next / previous form input".

## Goal

Add modal-local `<ctrl+j>` (next form input) and `<ctrl+k>` (previous form input)
keymaps to `WaitModal`. Navigation moves strictly between the five inputs above,
skipping completion lists and preview `Static`s, and wraps at both ends.

## Non-Goals

- Do **not** change `tab` / `action_accept_completion` behavior. The existing test
  `test_modal_tab_falls_through_to_focus_next_without_highlight` in
  `tests/ace/tui/test_wait_modal.py` pins that behavior and must keep passing.
- Do **not** change `ctrl+n` / `ctrl+p`. Those stay bound to `action_next_candidate` /
  `action_prev_candidate`, which move the highlight _within_ the active completion list.
  The two pairs are complementary: `^n`/`^p` move inside a list, `^j`/`^k` move between
  fields.
- Do **not** add these keys to `src/sase/default_config.yml`. See "Why no keymap
  registry change" below.
- Do not touch the Rust core (`../sase-core`). This is presentation-only Textual focus
  handling and stays in this repo per the `rust_core_backend_boundary` memory.

## Research Findings the Implementer Must Not Rediscover

These were verified against the pinned Textual version (`8.0.1`, per `uv.lock`) in a
workspace venv. They drive the design; do not simplify past them.

1. **`ctrl+k` is already taken by the focused widget.** Textual's `Input` ships
   `Binding(key='ctrl+k', action='delete_right_all', ...)`. A non-priority modal binding
   therefore loses, and pressing `<ctrl+k>` inside a Wait field would silently **delete
   the rest of the field's text** instead of navigating. Both new bindings must be
   declared with `priority=True`.

2. **`priority=True` on the `ModalScreen` is sufficient — no `on_key` interception is
   needed.** `App._check_bindings(key, priority=True)` iterates
   `reversed(self.screen._binding_chain)`, i.e. `[App, ModalScreen, ..., focused]`, and
   fires only bindings whose `binding.priority` is `True`. `Input`'s `ctrl+k` binding is
   non-priority, so it is skipped in that pass. `Screen._binding_chain` does drop
   ancestor bindings that a descendant would capture, but only via `check_consume_key`,
   and `Input.check_consume_key` returns `True` only for printable characters — `ctrl+j`
   / `ctrl+k` are not printable and survive the filter.

   This was confirmed empirically with a throwaway Textual harness: a `ModalScreen` with
   `priority=True` bindings for `ctrl+j` / `ctrl+k` and a focused `Input` containing
   `"hello"` with the cursor at position 2 fired both modal actions and left the value
   as `"hello"` (untruncated).

   Note for the implementer: `src/sase/ace/tui/modals/revive_agent_modal.py` and
   `src/sase/ace/tui/modals/prompt_history_modal.py` pair a `priority=True` `ctrl+k`
   binding with an extra `on_key` interception. **Do not copy that pattern here.** It is
   redundant given the dispatch order above, and `WaitModal.on_key` already has a narrow
   `enter`-only contract that should stay narrow.

3. **App-level `ctrl+j` / `ctrl+k` cannot leak into the modal.** `DEFAULT_BINDINGS` in
   `src/sase/ace/tui/bindings.py` binds `ctrl+j` → `next_agent_metadata_section` and
   `ctrl+k` → `prev_agent_metadata_section`. Those are non-priority, so they lose the
   priority pass; and the ordinary pass uses `_modal_binding_chain`, which truncates at
   the modal. The same throwaway harness confirmed app-level actions did not fire.

4. **`ctrl+j` is a distinct key from `enter` in Textual.** `enter` is aliased from
   `ctrl+m`; `ctrl+j` maps from `\x0a` and aliases to `newline`. Binding `ctrl+j` will
   not shadow `enter` / submit.

5. **`focus_next()` / `focus_previous()` are not usable.** The displayed completion
   `OptionList` toggles `can_focus` in `_apply_active_completion_visibility`, so it sits
   in the focus chain. An explicit ordered ring of input ids is required.

6. **File-size lint headroom is fine.** `just _lint-toobig` uses thresholds
   `1000 850 700`; `wait_modal.py` is 334 lines and `tests/ace/tui/test_wait_modal.py`
   is 358 lines. Both stay well under 700 after this change.

## Design

### Field ring and focus resolution

Add to `WaitModal` (in `src/sase/ace/tui/modals/wait_modal.py`), alongside the existing
`_TIME_CLASSES` class constant:

```python
_FIELD_INPUT_IDS = (
    "agents-input",
    "beads-input",
    "time-input",
    "runners-input",
    "priority-input",
)
# A focused completion list navigates as though its owning input were focused.
_COMPLETION_OWNER_IDS = {
    "agent-completion": "agents-input",
    "bead-completion": "beads-input",
}
```

Resolution rules for "which field am I on?":

- Focused widget id is one of `_FIELD_INPUT_IDS` → that index.
- Focused widget id is `agent-completion` / `bead-completion` → the index of the owning
  input (`agents-input` / `beads-input` respectively). This matters because `tab` can
  land focus on the completion list, and `<ctrl+j>` from there must move on to the next
  _field_ rather than dead-end.
- Anything else (or nothing focused) → `<ctrl+j>` focuses the first field and `<ctrl+k>`
  focuses the last field.

Otherwise move by `(index + offset) % len(_FIELD_INPUT_IDS)`, so `<ctrl+j>` on
`priority-input` wraps to `agents-input` and `<ctrl+k>` on `agents-input` wraps to
`priority-input`. Wrapping is deliberate: the form is short, and it matches the existing
"wraps" idiom already documented for `Ctrl-N / Ctrl-P` in the Zoom Modal help section.

### Cursor placement

After focusing the target field, set `cursor_position = len(value)`. This matches the
two existing places that programmatically focus a Wait field — `WaitModal.on_mount` and
`_accept_candidate_index` / `_accept_bead_candidate_index` in `wait_modal_completion.py`
— so navigating into a prefilled field leaves the caret ready to append rather than at
column 0.

### Type-safety detail

Read the focused id as `focused.id if focused is not None else None` using
`self.focused` (typed `Widget | None`, and `Widget.id` is `str | None`). Do **not** use
`getattr(self.focused, "id", None)`, which returns `Any` and is noisier under the repo's
mypy gate.

### Interactions that must be preserved (no extra code needed, but verify)

- **Completion-list visibility.** Focusing `agents-input` / `beads-input` fires
  `on_descendant_focus` → `_set_active_completion`, which swaps which completion list is
  displayed. Focusing `time-input` / `runners-input` / `priority-input` intentionally
  leaves the last active list unchanged. That is existing, tested behavior
  (`test_modal_focus_swaps_completion_list_but_time_leaves_it_unchanged`) and navigating
  by `<ctrl+j>` / `<ctrl+k>` inherits it for free.
- **Bead guard.** `_bead_guard_armed` is only disarmed by `Input.Changed` on
  `beads-input`. Navigating away from an armed guard must not disarm it or rewrite the
  footer.
- **Scrolling.** The modal `Container` is `overflow-y: auto` with `max-height: 100%`;
  `Input.focus()` scrolls the target into view by default, so tall-completion-list
  layouts still reveal the destination field.

## Files to Change

### 1. `src/sase/ace/tui/modals/wait_modal.py`

- Import `Binding` from `textual.binding` (the current `BINDINGS` list uses plain
  tuples; mixing tuples and `Binding` objects is supported by Textual).
- Append to `BINDINGS`:
  ```python
  Binding("ctrl+j", "next_field", "Next Field", priority=True),
  Binding("ctrl+k", "prev_field", "Prev Field", priority=True),
  ```
- Add the `_FIELD_INPUT_IDS` / `_COMPLETION_OWNER_IDS` class constants.
- Add `action_next_field` / `action_prev_field` public actions plus two private helpers
  (`_focus_field_by_offset`, `_focused_field_index`). Place them next to
  `action_next_candidate` / `action_prev_candidate` so the "within a list" and "between
  fields" pairs read together. Every method needs a docstring (repo lint requires them).
- Update `_footer_text` so the new keys are discoverable in the modal itself:
  ```
  enter apply | tab complete | ^j/^k field | ^r run now | esc cancel
  ```
  That is 66 characters. The modal `Container` is `width: 76` with a thick border and
  `padding: 1 2`, leaving ~70 content columns, so it fits on one centered line. Leave
  the armed-bead-guard footer (`enter again to wait on unverified beads | esc cancel`)
  untouched.

### 2. `src/sase/ace/tui/modals/help_modal/agents_bindings.py`

`src/sase/ace/CLAUDE.md` requires the `?` help popup to stay in sync when `sase ace`
behavior changes. There is direct precedent for per-modal sections here —
`"Artifact Files Modal"` and `"Zoom Modal"`. Add a `"Wait Modal"` section immediately
after `"Zoom Modal"` and before `"Agent Query Syntax"`:

```python
(
    "Wait Modal",
    [
        ("Ctrl-J / Ctrl-K", "Next / prev field, wraps"),
        ("Ctrl-N / Ctrl-P", "Next / prev completion row"),
        ("Tab", "Accept completion / next field"),
        ("Ctrl-R", "Run now"),
        ("Enter", "Apply wait spec"),
    ],
),
```

Key-style (`Ctrl-N / Ctrl-P`) matches the Zoom Modal entries. All descriptions are
within the 32-character cap from the help-modal formatting rule (longest is
`"Accept completion / next field"`, 30). This section also finally documents the Wait
modal's pre-existing local keys, which were previously only in the modal footer.

### 3. `tests/ace/tui/test_wait_modal.py`

Add async pilot tests using the existing `_TestApp` / `_candidate` helpers from
`tests/ace/tui/_wait_modal_helpers.py`. Await with `pilot.pause()` (the repo runs a
`lint (test waits)` gate — do not introduce sleeps):

1. `<ctrl+j>` from `agents-input` focuses `beads-input`.
2. `<ctrl+j>` from `priority-input` wraps to `agents-input`.
3. `<ctrl+k>` from `agents-input` wraps to `priority-input`.
4. **Regression guard for finding 1**: focus a field prefilled with a value, set
   `cursor_position` to the middle, press `<ctrl+k>`, and assert both that focus moved
   _and_ that the field's `value` is unchanged (i.e. Textual's `delete_right_all` did
   not fire). This is the single most important test in the set.
5. `<ctrl+j>` while the agent completion `OptionList` has focus moves to `beads-input`.
6. Navigating into a prefilled field leaves `cursor_position == len(value)`.
7. `<ctrl+j>` away from an armed bead guard leaves `_bead_guard_armed` set and the
   footer still reading `"enter again"`. (Reuse the guard-arming setup from
   `tests/ace/tui/test_wait_modal_beads.py`; put this test in whichever of the two
   modules keeps the helper imports simplest — the beads module is the natural home.)

### 4. `tests/ace/tui/visual/snapshots/png/wait_modal_100x32.png` and `wait_modal_beads_focused_100x32.png`

The footer text change alters both Wait modal PNG goldens. Regenerate them intentionally
(see Verification) rather than editing them by hand.

## Why No Keymap Registry Change

`src/sase/default_config.yml` only exposes `ace.keymaps.app`, `ace.keymaps.gate`, and
`ace.keymaps.statistics`. `WaitModal` is not registered in the keymap registry — all of
its local keys (`escape`, `ctrl+r`, `tab`, `down`/`up`, `ctrl+n`/`ctrl+p`) are hardcoded
in its `BINDINGS`. Adding hardcoded `ctrl+j` / `ctrl+k` is therefore the consistent
choice, and the "update `default_config.yml` when changing keymaps" gotcha in
`CLAUDE.md` does not apply. Making the whole Wait modal user-configurable would be a
separate, larger change; if the project owner wants that, file it as its own task bead
via `/sase_new_task`.

## Verification

Run from the implementing agent's own `sase_<N>` workspace directory:

```bash
just install                    # workspaces are ephemeral; deps may be stale
just check                      # whole-repo lint gates + diff-scoped tests
just update-visual-snapshots    # regenerate the two Wait modal PNG goldens
just test-visual                # confirm the regenerated goldens now pass clean
```

Then, before landing, run the exhaustive gate through `/sase_monitor` (never inline):

```bash
sase monitor start --command 'just check-full' --next '<follow-up action>'
```

Manual smoke check (optional but cheap): open `sase ace`, press `shift+tab` to the
Agents tab, select a `WAITING` / `QUEUED` / `RUNNING` agent, press `w`, then confirm
`<ctrl+j>` walks Agents → Beads → Time → Runners → Priority → Agents, `<ctrl+k>` walks
the reverse, and that `<ctrl+k>` in a field with text does not truncate the text.

## Acceptance Criteria

- `<ctrl+j>` in the Wait panel focuses the next form input, wrapping from Priority back
  to Agents.
- `<ctrl+k>` focuses the previous form input, wrapping from Agents back to Priority.
- Neither key modifies any field's text — in particular `<ctrl+k>` never triggers
  Textual's `Input.delete_right_all`.
- Both keys work when focus is on an agent/bead completion list, resolving relative to
  that list's owning input.
- Focusing a field via either key places the caret at the end of the existing value.
- `tab`, `ctrl+n`, `ctrl+p`, `ctrl+r`, `enter`, and `escape` behave exactly as before;
  the whole existing `tests/ace/tui/test_wait_modal.py` and
  `tests/ace/tui/test_wait_modal_beads.py` suites still pass unmodified.
- The modal footer and the `?` help popup both document the new keys.
- `just check` passes, and `just test-visual` passes against the regenerated goldens.
