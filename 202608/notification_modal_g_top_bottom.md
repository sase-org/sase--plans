---
tier: tale
title: Add g/G top/bottom jumps to the notifications panel detail pane
goal:
  In the notifications panel, `g` jumps the right-hand detail pane to the top of its
  contents and `G` jumps it to the bottom, without moving the notification list
  highlight, advertised in the panel footer and documented in docs/notifications.md.
size: medium
proposed_by: bbugyi200.athena.04i
create_time: 2026-08-17 06:59:50
status: wip
---

# Add `g` / `G` top/bottom jumps to the notifications panel detail pane

## Goal

In the notifications panel (`NotificationModal`, opened with `i` or the `,n` leader
chord), add two keymaps:

- `g` — jump the right-hand detail pane to the **top** of its contents.
- `G` — jump the right-hand detail pane to the **bottom** of its contents.

These complete the existing `Ctrl+D` / `Ctrl+U` half-page scrolling on the same pane,
and match the `g` / `G` convention already used by every other ACE surface that pairs a
focused list on the left with a scrollable detail pane on the right.

Neither key moves the notification list highlight. The left list keeps `j` / `k` /
`Ctrl+N` / `Ctrl+P` for row movement and Textual's built-in `home` / `end` for
first/last row; `g` / `G` are detail-pane-only, exactly as in `LogsPane` and
`PluginsBrowserPane`.

## Background: what already exists

The notifications panel is `NotificationModal`, split across
`src/sase/ace/tui/modals/notification_modal*.py`:

- `notification_modal.py` holds the class, its `compose()`, and its literal `BINDINGS`
  list.
- `compose()` builds `#notification-panels` as a `Horizontal` of `#notification-left`
  (tag strip + `OptionList#notification-list`) and `#notification-right`. The right pane
  contains a title `Label`, a sent-at `Label`, a snooze-status `Label`, and
  `VerticalScroll#notification-file-scroll` wrapping `Static#notification-file-content`.
  **`#notification-file-scroll` is the only scrollable widget in the right pane, and it
  is always composed** — even when there are zero notifications.
- `notification_modal_attachments.py` (`NotificationAttachmentMixin`) owns the right
  pane's rendering plus the existing scroll actions `action_scroll_file_down` /
  `action_scroll_file_up` (half-page, via
  `scroll_relative(y=±height // 2, animate=False)`), and `_reset_file_scroll`, which
  already calls `scroll_home(animate=False)` on `#notification-file-scroll`.
- `notification_modal_constants.py` holds the three footer hint strings
  (`DEFAULT_HINT_TEXT`, `QUESTION_HINT_TEXT`, `GATE_HINT_TEXT`) rendered into
  `Label#notification-hints` by `_update_hint_footer` in
  `notification_modal_options.py`.

Facts established while researching, which the implementation depends on:

1. **Screen-level `BINDINGS` are the right place; no `OptionList` subclass is needed.**
   Textual delivers a key to the focused widget and bubbles it up the DOM running every
   node's `on_key` handler, and only checks `BINDINGS` afterwards at the `App` level
   (`App._on_key` → `_check_bindings`, walking `screen._modal_binding_chain`). The
   focused widget here is a plain `textual.widgets.OptionList`, whose `BINDINGS` are
   `down`/`end`/`enter`/`home`/`pagedown`/`pageup`/`up` — **no `g` or `G`**. So a `g` /
   `G` entry on `NotificationModal.BINDINGS` will fire, the same way the modal's
   existing `x`, `m`, `M`, `s`, `R`, `Ctrl+D`, `Ctrl+U` bindings already do while that
   list has focus.

   This is why the sibling `PluginsBrowserList` / `_LogSourceList` widget subclasses are
   _not_ the pattern to copy: those panes live under `ConfigCenterModal`, whose screen
   `on_key` grabs `g` / `G` during bubbling and therefore beats any pane-level binding.
   `NotificationModal` has no such competing ancestor.

2. **Jump mode keeps working for free.** `notification_modal_options.py` defines
   `on_key` on the modal screen, which consumes keys while `jump_mode_active` and
   returns immediately otherwise. Because that `on_key` runs during bubbling — before
   any binding is checked — `g` and `G` remain ordinary jump-hint characters while `'`
   hints are painted (both are in the base-62 `JUMP_HINT_CHARS` alphabet), and they only
   reach the new scroll bindings when jump mode is off. No guard code is required; this
   is a behavior to _test_, not to implement.

3. **`("G", ...)` matches.** Textual's `_character_to_key` returns the character itself
   for alphanumerics, so a shifted `G` arrives as `event.key == "G"`. The repo-wide
   convention is to bind `("G", ...)` **and** `("shift+g", ...)` for the bottom jump
   (`report_modal`, `commit_view_modal`, `glossary_preview_modal`,
   `preview_panel_modal`, `word_definition_modal`, `logs_pane`, `procs_pane`,
   `plugins_browser_pane`), covering terminals/keyboard protocols that report the
   modifier separately. Follow that.

4. **No configurable-keymap plumbing is involved.** `src/sase/default_config.yml` only
   configures `show_notifications: "i"` (the key that _opens_ the panel). Every in-modal
   key is a literal in `NotificationModal.BINDINGS`, so there is nothing to add to
   `default_config.yml`, `keymaps/app_keymaps.py`, `keymaps/metadata.py`, or the
   command-palette metadata in `commands/_app_metadata.py`.

## Design decisions

**Where the actions live.** Add them to `NotificationAttachmentMixin` in
`notification_modal_attachments.py`, immediately after `action_scroll_file_up`. That
mixin already owns every other behavior of the right pane, already imports
`VerticalScroll`, and already resolves `#notification-file-scroll` in three places.

**Implementation shape.** `scroll_home(animate=False)` and `scroll_end(animate=False)`,
matching `_reset_file_scroll` in the same file and the dominant repo convention
(`report_modal.action_scroll_top` / `action_scroll_bottom`,
`launch_approval_modal.action_scroll_to_top` / `action_scroll_to_bottom`). Do **not**
copy `LogsPane`'s `_force_scroll_detail_to(max_scroll_y)` variant — that exists only
because `LogsPane` reconciles scroll position against worker-driven content reloads,
which the notification pane does not do.

Mirror the existing siblings' error handling exactly: `action_scroll_file_down` /
`action_scroll_file_up` call `query_one` unguarded, so the new actions should too. (The
`try`/`except` in `_reset_file_scroll` exists because it runs from `_display_file`,
which is called before mount in unit tests.)

**Naming.** `action_scroll_file_top` / `action_scroll_file_bottom`, matching the
`scroll_file_*` prefix of the two existing scroll actions on this modal rather than the
`scroll_to_*` / `scroll_*` spellings other modals use. Local consistency wins here
because these four actions sit next to each other and act on the same widget.

**Footer discoverability, and its cost.** The three hint constants are the panel's only
in-modal help — the `?` help modal documents the key that _opens_ the panel, not the
panel's internal keys — so a new keymap that is advertised nowhere is only half shipped.
Advertise it.

Be aware of the trade-off, which is real but acceptable: `Label#notification-hints` is
`height: 1` in `styles.tcss`, and the modal container is `width: 95%`. At the 120x40
size used by the PNG snapshot fixtures the label is about 108 columns wide, while
`DEFAULT_HINT_TEXT` is already 171 characters and `GATE_HINT_TEXT` 116 — **both are
already clipped there today**. At the widths ACE is actually used at (95% of a 180–200
column terminal is 184+ columns) the full string does fit, including the addition.
`QUESTION_HINT_TEXT` (101 chars) is the one that currently fits at 120 columns and will
begin clipping its trailing `q: close`, matching the other two.

Reflowing or shortening the footer to fit 120 columns is deliberately **out of scope** —
it would churn the same goldens for an unrelated reason and belongs in its own change.

## Implementation

### 1. `src/sase/ace/tui/modals/notification_modal.py`

In `NotificationModal.BINDINGS`, keep the scroll keys grouped: insert three entries
directly after the existing `("ctrl+u", "scroll_file_up", "Scroll up")` line, which is
currently last in the list.

```python
        ("ctrl+d", "scroll_file_down", "Scroll down"),
        ("ctrl+u", "scroll_file_up", "Scroll up"),
        ("g", "scroll_file_top", "Top"),
        ("G", "scroll_file_bottom", "Bottom"),
        ("shift+g", "scroll_file_bottom", "Bottom"),
```

### 2. `src/sase/ace/tui/modals/notification_modal_attachments.py`

Add two actions immediately after `action_scroll_file_up`. `VerticalScroll` is already
imported at the top of the file; no new imports.

```python
    def action_scroll_file_top(self: Any) -> None:
        """Jump the file content pane to the top."""
        scroll = self.query_one("#notification-file-scroll", VerticalScroll)
        scroll.scroll_home(animate=False)

    def action_scroll_file_bottom(self: Any) -> None:
        """Jump the file content pane to the bottom."""
        scroll = self.query_one("#notification-file-scroll", VerticalScroll)
        scroll.scroll_end(animate=False)
```

### 3. `src/sase/ace/tui/modals/notification_modal_constants.py`

Insert `  g/G: top/bot` directly after the existing `C-d/C-u: scroll` segment in all
three hint constants, so the scroll affordances stay adjacent and the segment style
(`key: meaning`, double-space separated) is preserved:

- `DEFAULT_HINT_TEXT`:
  `... C-n/C-p: next/prev file  C-d/C-u: scroll  g/G: top/bot  R: read tab ...`
- `QUESTION_HINT_TEXT`:
  `Enter: answer  d: debug  C-d/C-u: scroll  g/G: top/bot  m: mark ...`
- `GATE_HINT_TEXT`:
  `Enter: review  d: debug  C-n/C-p: file  C-d/C-u: scroll  g/G: top/bot  m: mark ...`

Introduce no square brackets or other markup characters — `DEFAULT_HINT_TEXT` is
asserted to round-trip through `Content.from_markup(...).plain` unchanged, and
`g/G: top/bot` is markup-free. Let `just fmt` settle the implicit string concatenation
line breaks.

### 4. `docs/notifications.md`

In the `### Modal Keybindings` table under `## Viewing Notifications`, add one row
directly after the ``| `Ctrl+D` / `Ctrl+U` | Scroll file content down / up |`` row:

```markdown
| `g` / `G` | Jump the detail pane to the top / bottom of its contents |
```

Markdown is formatted by prettier via `just fmt-md`, so write the row without
hand-padding the columns and let the formatter realign the table.

No change is needed in `docs/ace.md`: its `## Notifications Modal` section explicitly
defers to `docs/notifications.md` for "the full keybinding reference".

### 5. Tests

Extend `tests/test_notification_modal_action_bindings.py` (it already owns
binding-presence and footer-hint assertions for this modal, and imports the hint
constants). Add:

- `test_notification_modal_binds_g_and_capital_g_to_detail_scroll`: assert
  `("g", "scroll_file_top", "Top")`, `("G", "scroll_file_bottom", "Bottom")`, and
  `("shift+g", "scroll_file_bottom", "Bottom")` are all in `NotificationModal.BINDINGS`.
- `test_notification_modal_g_scrolls_detail_pane_home`: construct
  `NotificationModal([_make_notification("n1")])`, patch `query_one` with a `MagicMock`
  side effect that returns a mock for `#notification-file-scroll`, call
  `modal.action_scroll_file_top()`, and assert `scroll_home` was called once with
  `animate=False` — mirroring how
  `tests/ace/tui/modals/test_notification_image_files.py` fakes `query_one` and asserts
  `scroll.scroll_home.assert_called_once_with(animate=False)`.
- `test_notification_modal_capital_g_scrolls_detail_pane_end`: same shape, calling
  `action_scroll_file_bottom()` and asserting `scroll_end(animate=False)`.
- Footer-hint assertion alongside the existing ones: `"g/G: top/bot"` appears in all
  three of `DEFAULT_HINT_TEXT`, `QUESTION_HINT_TEXT`, and `GATE_HINT_TEXT`, and
  `Content.from_markup(DEFAULT_HINT_TEXT).plain == DEFAULT_HINT_TEXT` still holds.
- Regression test for the jump-mode interaction, which is the one non-obvious behavior
  here — put it in `tests/test_notification_modal_jump.py`, which already imports
  `_KeyEvent`, `_make_notification`, and `_wire_fake_option_list`: with several
  notifications, call `modal.action_jump_to_entry()` to paint hints, then
  `modal.on_key(_KeyEvent("g", "g"))`, and assert the event was stopped (i.e. jump mode
  consumed it) so the scroll binding can never fire while hints are up. Add the `G`
  counterpart in the same test or a sibling.

### 6. PNG visual goldens

Changing the hint constants changes rendered pixels in every 120x40 golden that shows
the `NotificationModal` footer. Run `just test-visual` to find the exact set — expect
the notification-modal goldens under `tests/ace/tui/visual/snapshots/png/` (e.g.
`notification_beads_tab_120x40.png`, `notification_gate_pending_120x40.png`,
`notification_gate_answered_120x40.png`, `notification_question_summary_120x40.png`,
`notification_sent_at_120x40.png`, `notification_filed_by_120x40.png`,
`notification_selected_snooze_status_120x40.png`, `notification_report_pane_120x40.png`)
and _not_ the `notification_indicator_*` goldens, which render the top-bar chip rather
than the modal.

Before accepting anything, inspect the diff artifacts under `.pytest_cache/sase-visual/`
and confirm the **only** changed region is the single footer hint line. If any other
region moved, stop and diagnose — that means the change did something unintended. Then
rebaseline with:

```bash
just test-visual --sase-update-visual-snapshots
```

and re-run `just test-visual` clean.

## Verification

```bash
just install        # ephemeral workspaces may have stale deps; required first
just fmt
just check          # whole-repo lint gates + diff-scoped tests
just test-visual    # excluded from `just check`; must be run explicitly here
```

Then, before landing, run the exhaustive gate through `/sase_monitor` rather than inline
(it routinely outruns a single agent turn), handing it a `--next` action so the
follow-up agent acts on the result:

```bash
sase monitor start --command 'just check-full' ...
```

Manual smoke check in a real terminal, since the interesting behavior is interactive:

1. Open ACE, press `i` to open the notifications panel.
2. Highlight a notification whose detail pane overflows — a gate-backed row (plan, epic,
   launch, question) or one with a long text attachment.
3. `Ctrl+D` a few times, then `g` — the pane returns to the top and the highlighted row
   in the left list does not change.
4. `G` — the pane jumps to the bottom; the left-list highlight still does not change.
5. Press `'` to paint jump hints, then press `g` — it selects the `g`-hinted
   notification (or exits hint mode) and does **not** scroll the detail pane.
6. Confirm the footer hint line shows `g/G: top/bot` for a plain row, a question row,
   and a gate-backed row.

## Out of scope / explicitly not doing

- **No Rust core change.** Per the `rust_core_backend_boundary` memory, this is
  presentation-only Textual keybinding and scroll behavior with no backend or domain
  semantics; nothing here would need to match in a web app or CLI frontend. `sase-core`
  is untouched.
- **No feature flag.** Flags are for user-reaching behavior that is _not ready_ — a
  disabled beta, an early landed path, or a deprecation whose old branch must stay
  reachable. This keymap is complete and correct on land and is not something users
  choose forever, so it is neither a flag nor a config field.
- **No `default_config.yml` / keymap-registry change.** In-modal keys are literals in
  `BINDINGS` (see Background fact 4).
- **No `?` help-modal change.** It documents `show_notifications` (the opener), not the
  panel's internal keys; the footer hint is this panel's in-modal help and is updated
  above.
- **No `OptionList` subclass** for the notification list (see Background fact 1).
- **No footer reflow.** The pre-existing over-length of `DEFAULT_HINT_TEXT` and
  `GATE_HINT_TEXT` at narrow widths is documented above as a known condition. If it is
  worth fixing, file it separately with `/sase_new_task`; do not fold it into this
  change.
- **No change to what `g` / `G` mean elsewhere.** Only `NotificationModal` gains
  bindings.
