---
tier: tale
title: Show the selected notification's send time in the notification panel
goal:
  The notification panel's detail pane always shows the absolute date and time the highlighted notification was sent,
  tiered for readability and paired with its relative age.
proposed_by: bbugyi200.athena.ql
create_time: 2026-07-31 13:26:20
status: done
---

- **PROMPT:** [202607/prompts/notification_sent_at.md](prompts/notification_sent_at.md)

# Show when the selected notification was sent in the notification panel

## Problem

The notification panel (`NotificationModal`, opened with `i` from any ACE tab) tells you only how _old_ a notification
is. Every row in the left list ends with a relative age token — `4m ago`, `2h ago`, `3d ago` — produced by
`format_relative_time()`. Nowhere in the panel can you see the actual wall-clock date and time a notification was sent.

That hurts in three concrete ways:

1. **Ambiguity at the coarse end.** `3d ago` covers a 24-hour window; `2h ago` cannot be correlated with a log line, a
   commit, or an agent run whose timestamps are absolute.
2. **Staleness.** The relative token is computed when the list is built. Leave the panel open for twenty minutes and
   every row still claims the age it had on open. There is no authoritative, non-decaying time anywhere on screen.
3. **No burst disambiguation.** When five notifications arrive inside the same minute they all read `0s ago` / `1m ago`
   with no way to see their real order beyond list position.

The data is already there: `Notification.timestamp` is a required ISO-8601 field on every row
(`src/sase/notifications/models.py`), carried on the Rust core wire (`sase_core/src/notifications/wire.rs`). It is
simply never rendered in absolute form.

## Goal

Show the absolute send time of the **currently selected** notification in the panel, always, in a way that is intuitive
at a glance, reliable across every pane type and edge case, and visually at home in the modal.

## Design

### Placement: a dedicated meta line in the detail pane

Add one new single-line `Label` (`#notification-sent-at`) to the right-hand detail pane of `NotificationModal`, mounted
directly **below** the existing pane title (`#notification-file-title`) and **above** the scrollable content
(`#notification-file-scroll`).

```
┌ Notifications ─────────────────────────────────────────────────────────────┐
│ [HITL] [Errors] [General]                                                  │
│ ┌────────────────────────────────┐ │ 📝 fix-hook · Agent Question          │
│ │ * 📝 [fix-hook] Protected me…  │ │ sent today 13:18:42 · 4m ago          │
│ │   🤖 [axe] lint run finished…  │ │                                       │
│ │ ~ 📊 [release] readiness re…   │ │ Question from codex--fix-hook         │
│ └────────────────────────────────┘ │ ChangeSpec memory-protection · sess…  │
└────────────────────────────────────────────────────────────────────────────┘
```

Why here:

- **It follows the selection.** The detail pane is already the "everything about the highlighted row" surface. The eye
  travels from the highlighted row to the pane once and finds identity (title line) then time (new line).
- **It cannot be truncated away.** The obvious cheap alternative — appending the stamp to `#notification-file-title` —
  puts it behind a variable-length, home-shortened file path (`File 2/3: ~/very/long/path…`) inside a `text-overflow`
  container. On a narrow terminal the timestamp would silently disappear. A dedicated line always renders.
- **It is stable while the pane churns.** `Ctrl+N`/`Ctrl+P` cycle attachments and rewrite the title line; the send time
  belongs to the notification, not the attachment, so it gets its own row and stays put while files cycle.
- **It costs one row.** The modal is 90% of terminal height; the detail pane is `1fr`. One line is affordable and keeps
  the top of the pane dense rather than padded.

Rejected alternatives:

| Alternative                                                             | Why not                                                                                                                                                                                                                                                  |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Right-align the stamp on the modal title row (`Notifications … sent …`) | Far from both the selected row and the detail content; the eye has to cross the whole modal, and the title row does not otherwise track selection.                                                                                                       |
| Absolute stamp on every list row                                        | The left pane is 40% wide with `text-overflow: clip`; rows already carry icon, sender, note, age, badge, file count, snooze, and up to three tags. Adding absolute stamps to all rows is noise and clips the note text that actually identifies the row. |
| Replace the row's relative token with an absolute one                   | Relative age is the right scanning primitive for a list. Absolute time is the right precision primitive for a single selected item. Keep both, each where it works.                                                                                      |
| A keybinding that toggles relative/absolute                             | Hidden state the user must discover and remember. Showing both, always, costs one line and no learning.                                                                                                                                                  |

### Format: tiered absolute time, with the familiar relative token beside it

The line renders as:

```
sent <absolute> · <relative>
```

`<absolute>` comes from a new `format_absolute_time()` helper with four tiers, all rendered in the configured SASE
timezone:

| Tier                  | Example            | Rationale                                                                                                                                              |
| --------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Same calendar day     | `today 13:18:42`   | Seconds included — today is exactly when bursts need disambiguating. The explicit `today` word removes any doubt that a bare clock time is from today. |
| Previous calendar day | `yesterday 21:04`  | Reads the way a person would say it; seconds no longer carry information.                                                                              |
| Same year             | `Jul 12 09:31`     | Month/day is the natural handle at this range.                                                                                                         |
| Any other year        | `Jul 12 '25 09:31` | Compact year disambiguation for archived rows.                                                                                                         |

This tiering deliberately matches `_format_finish_timestamp()` in `src/sase/ace/tui/models/agent_time.py`, which the
Agents tab already uses for finish times (same-day `%H:%M:%S`, prior-day `%b %-d %H:%M`, other-year `%b %-d '%y`). The
TUI should speak about time in one voice; a user reading an agent finish time and a notification send time should not
have to re-learn the format.

`<relative>` reuses the existing `format_relative_time()` — the _same function_ the list rows call, so the detail line
and the highlighted row can never disagree about a notification's age. It stays because relative age is genuinely the
faster read for "is this fresh?", and because it anchors the new absolute value to something the user already knows.

Styling gives three steps of hierarchy with no new hue (the line inherits `$text-muted` from CSS):

- `sent ` — inherited muted, unstyled
- absolute value — `bold` (the payload)
- `·` and relative token — `dim` (context)

That mirrors the pane title above it (`$text-muted` + bold) so the two lines read as one header block.

### Reliability

1. **One choke point.** `_display_file()` in `notification_modal_attachments.py` is the single method that repaints the
   detail pane for _every_ case: question pane, report pane, image attachment, text attachment, video placeholder,
   unreadable file, no files, and no selection. The new update call goes at the very top of that method, before any of
   its early returns, so no pane type can miss it and no future pane type can forget it.
2. **No stale value on empty state.** With no selection (`notification is None`, e.g. the empty-inbox path in
   `_rebuild_list()`), the line is cleared and hidden via the global `.hidden` class (`display: none`), collapsing the
   row instead of showing a leftover stamp from a dismissed row.
3. **Never raises.** Unparsable timestamps fall back to the raw string, exactly as `format_relative_time()` already
   does. The widget lookup uses the same defensive `try/except` the modal uses elsewhere, so a missing widget degrades
   to "no line" rather than a broken pane.
4. **Correct timezone, always.** Formatting resolves through `sase.core.time.get_timezone()`; aware timestamps are
   converted into it and naive timestamps are interpreted in it — the same contract `format_relative_time()` uses. Never
   call bare `datetime.now()` / `.astimezone()` (see the module docstring in `src/sase/core/time.py`).
5. **Deliberate non-goal: no refresh timer.** The relative token can drift while the panel sits open. That is acceptable
   _because the absolute value is the authoritative one and never decays_ — which is precisely the problem this feature
   solves. List rows already behave this way, so adding a ticker only for the detail line would make the panel
   internally inconsistent, and periodic pump callbacks are exactly what the TUI perf rules tell us not to add casually.
   Both surfaces recompute on every rebuild and every highlight change.

### De-duplication in the question pane

`_identity_header()` in `notification_modal_question.py` currently appends `asked <relative>` to its metadata line. Once
the sent line renders one row above it, that token states the same fact, less precisely, two lines apart. Remove the
`asked …` token from the question metadata line and leave the rest (`ChangeSpec …`, `session …`) untouched. Single
source of truth for "when".

The report pane's `live · updated <relative>` / `snapshot · captured <relative>` line stays: it describes the report
document's `updated_at`, which is a genuinely different timestamp from the notification's send time. Having the send
time directly above it actually clarifies the distinction.

### Rust core boundary

No change in `../sase-core`. This is display formatting of a field that already exists on the notification wire; there
is no new domain state, no store change, and no wire change. The Python `sase/notifications/` package is where
notification display formatting already lives (`format_relative_time`, `format_relative_until`), and
`sase_core/src/notifications/` carries no formatting at all today. Adding only the absolute half of the pair to Rust
would split one formatting concern across two languages, which is worse than the status quo. If both formatters should
migrate to core later, that is its own change.

## Scope

In scope:

- New `format_absolute_time()` in `src/sase/notifications/models.py`, exported from `sase/notifications/__init__.py`.
- New `src/sase/ace/tui/modals/notification_modal_sent_at.py` (mixin + text builder).
- `NotificationModal` composes the new label and mixes in the new behavior.
- `_display_file()` calls the update at its top.
- Styles for `#notification-sent-at`.
- Drop the redundant `asked <relative>` token from the question pane metadata line.
- Unit, modal, and PNG snapshot tests; docs update.

Out of scope (do **not** do these):

- Any change to `sase notify` CLI output.
- Any change under `sase/repos/linked/sase-core/`.
- A refresh timer / live-ticking relative token.
- Absolute stamps on list rows.
- New keybindings (and therefore no help-modal or footer changes).

## Implementation

### Step 1 — `format_absolute_time()` in `src/sase/notifications/models.py`

Add beside `format_relative_time()`:

```python
def format_absolute_time(iso_timestamp: str, now: datetime | None = None) -> str:
    """Format an ISO-8601 timestamp as an absolute wall-clock string.

    Rendered in the configured timezone, in four tiers:

    - same calendar day: ``"today 13:18:42"``
    - previous calendar day: ``"yesterday 21:04"``
    - same year: ``"Jul 12 09:31"``
    - other year: ``"Jul 12 '25 09:31"``

    Unparsable input is returned unchanged, matching
    :func:`format_relative_time`.
    """
```

Implementation notes:

- `from sase.core.time import get_timezone` inside the function body (match the existing lazy-import style in this
  module).
- Parse with `datetime.fromisoformat`; on `ValueError` return `iso_timestamp` unchanged.
- Naive parse result → `.replace(tzinfo=tz)`; then `.astimezone(tz)` so aware timestamps from other zones display in the
  configured zone.
- Reference time: `now` when given (normalize it into `tz` the same way — naive `now` gets `tz` attached, aware `now`
  gets converted), otherwise `datetime.now(tz)`. The `now` parameter exists for deterministic tests and mirrors
  `format_wait_until()` in `src/sase/ace/tui/models/agent_time.py`.
- Tier selection on calendar dates, not elapsed seconds: `days = (reference.date() - local.date()).days`; `0` → today,
  `1` → yesterday, else year comparison. A future timestamp (clock skew) yields a negative `days` and falls through to
  the month/day tier, which renders honestly rather than lying with "today".
- `%-d` (no zero padding) is already used by `agent_time.py`; keep it for visual consistency.

Export `format_absolute_time` from `src/sase/notifications/__init__.py` (import list **and** `__all__`).

### Step 2 — new module `src/sase/ace/tui/modals/notification_modal_sent_at.py`

```python
"""Sent-at line for the notification modal detail pane."""

from __future__ import annotations

from typing import Any

from rich.text import Text
from textual.widgets import Label

from sase.notifications import (
    Notification,
    format_absolute_time,
    format_relative_time,
)

SENT_AT_ID = "notification-sent-at"


def build_sent_at_text(notification: Notification) -> Text:
    """Build the styled 'sent <absolute> · <relative>' line."""
    text = Text(no_wrap=True, overflow="ellipsis")
    text.append("sent ")
    text.append(format_absolute_time(notification.timestamp), style="bold")
    text.append(" · ", style="dim")
    text.append(format_relative_time(notification.timestamp), style="dim")
    return text


class NotificationSentAtMixin:
    """Keep the detail pane's send-time line in sync with the selection."""

    def _update_sent_at(self: Any, notification: Notification | None) -> None:
        """Render, or hide, the send-time line for the highlighted row."""
        try:
            label = self.query_one(f"#{SENT_AT_ID}", Label)
        except Exception:
            return
        if notification is None:
            label.update(Text(""))
            label.add_class("hidden")
            return
        label.update(build_sent_at_text(notification))
        label.remove_class("hidden")
```

Keeping `format_absolute_time` / `format_relative_time` as module-level names here (rather than calling them through
`sase.notifications`) is what lets visual tests monkeypatch them for determinism, matching how
`notification_modal_options.format_relative_time` is already patched.

### Step 3 — wire it into `NotificationModal`

In `src/sase/ace/tui/modals/notification_modal.py`:

- Import `NotificationSentAtMixin` and add it to the class bases (place it alongside the other pane mixins, before
  `OptionListNavigationMixin`).
- In `compose()`, inside `Vertical(id="notification-right")`, between the title label and the `VerticalScroll`:

```python
yield Label("No files attached", id="notification-file-title")
yield Label("", id="notification-sent-at", classes="hidden")
with VerticalScroll(id="notification-file-scroll"):
```

Starting hidden means no empty reserved row before the first `_display_file()` call.

In `src/sase/ace/tui/modals/notification_modal_attachments.py`, make `_display_file()` start with:

```python
def _display_file(self: Any, notification: Notification | None) -> None:
    """Render file content with syntax highlighting in the right pane."""
    self._update_sent_at(notification)
    title = self.query_one("#notification-file-title", Label)
    ...
```

Nothing else in that method changes.

### Step 4 — styles

In `src/sase/ace/tui/styles.tcss`, in the `/* ===== Notification Modal Styling ===== */` block, immediately after the
`#notification-file-title` rule:

```css
NotificationModal #notification-sent-at {
  height: 1;
  color: $text-muted;
  text-wrap: nowrap;
  text-overflow: ellipsis;
}
```

No margin — the title line, the sent line, and the content should read as a tight header block. The global `.hidden`
rule (`display: none`, top of the file) collapses the row when there is no selection.

### Step 5 — remove the duplicated `asked` token

In `src/sase/ace/tui/modals/notification_modal_question.py`, `_identity_header()`: delete the
`metadata.append(f"asked {format_relative_time(notification.timestamp)}")` line. Remove the now-unused
`format_relative_time` import if nothing else in that module uses it (check before deleting). Everything else in the
header is unchanged.

## Testing

### Unit — `tests/test_notification_absolute_time.py` (new)

Covers `format_absolute_time` with an injected `now` so results are deterministic. This repo splits test files by
concern (see recent `test: split …` commits), so keep it out of `tests/test_notification_models.py`. Use
`inline_snapshot`'s `snapshot()` for expected strings, matching the style of `tests/test_notification_models.py`.

Cases:

- same day → `today HH:MM:SS`
- previous day → `yesterday HH:MM`
- earlier this year (multi-day gap) → `Mon D HH:MM`
- previous year → `Mon D 'YY HH:MM`
- day boundary: a timestamp 30 minutes old that lands on the previous calendar date (e.g. `23:50` vs a `00:20`
  reference) → `yesterday 23:50`, proving tiering is calendar-based, not elapsed-seconds-based
- naive timestamp → interpreted in the configured timezone, not UTC
- aware timestamp from another offset → converted into the configured timezone
- unparsable string → returned unchanged
- future timestamp (skew) → month/day tier, never `today`

Pin the timezone the way the existing suite does (the `_pin_configured_timezone` conftest fixture / patching
`sase.core.time`); do not depend on the host zone.

### Modal — `tests/test_notification_modal_sent_at.py` (new)

Build on `tests/_notification_modal_helpers.py` (`_make_notification`, `_FakeOptionList`, the `MagicMock` `query_one`
pattern used by the existing modal tests).

- `build_sent_at_text()` produces `sent <absolute> · <relative>` with the expected plain text and with the absolute
  segment bold / the relative segment dim.
- `_update_sent_at(None)` clears the label and adds `hidden`.
- `_update_sent_at(notification)` sets the text and removes `hidden`.
- `_display_file()` updates the sent line for: a notification with no files, one with a text attachment, a question
  notification (question pane path), and a report notification (report pane path) — i.e. the line survives every early
  return in `_display_file()`.
- Cycling attachments with `action_next_file()` leaves the sent line unchanged while the title line changes.
- A notification whose `timestamp` is garbage still renders a line and does not raise.

### Question pane — `tests/test_notification_modal_question_pane.py`

Update the existing assertion at line ~111 (`"ChangeSpec memory-protection · asked 4m ago · session 3f2a1234…"`) to the
new metadata line without the `asked …` token. Grep for other `asked ` assertions before finishing.

### Visual — PNG snapshots

The new line lands inside the detail pane, so two existing goldens change and must be regenerated:

- `tests/ace/tui/visual/test_ace_png_snapshots_notification_question.py` → `notification_question_summary_120x40`
- `tests/ace/tui/visual/test_ace_png_snapshots_notification_report.py` → `notification_report_pane_120x40`

(`notification_report_modal_120x40` is the full-screen report modal and is unaffected — confirm it does not change.)

In both tests, add determinism patches next to the existing `format_relative_time` patches:

```python
monkeypatch.setattr(
    "sase.ace.tui.modals.notification_modal_sent_at.format_absolute_time",
    lambda _timestamp, now=None: "today 08:00:00",
)
monkeypatch.setattr(
    "sase.ace.tui.modals.notification_modal_sent_at.format_relative_time",
    lambda _timestamp: "4m ago",
)
```

(Use the age string each test already patches elsewhere so the pane stays self-consistent.) Add an
`assert_page_svg_contains(page, "sent today 08:00:00")` assertion to each.

Also add one new snapshot, `notification_sent_at_120x40`, in a new
`tests/ace/tui/visual/test_ace_png_snapshots_notification_sent_at.py`, covering the ordinary path the other two do not:
a plain notification with a small text attachment, so the golden captures the title/sent/content header block and its
styling as the user normally meets it.

Regenerate goldens with:

```bash
just test-visual -- --sase-update-visual-snapshots
```

then re-run `just test-visual` clean, and eyeball the three PNGs (they are the deliverable for "beautiful" — check the
sent line sits tight under the title, the bold/dim hierarchy reads, and nothing wrapped).

### Docs

`docs/notifications.md`, "Viewing Notifications" (line ~16). Replace the sentence

> Notifications display relative timestamps (e.g., "2m ago", "1h ago") and can be marked as read or dismissed.

with wording that keeps the row behavior and adds the detail line: rows show relative timestamps; the detail pane shows
the selected notification's absolute send time alongside its relative age (`sent today 13:18:42 · 4m ago`), tiered as
`today HH:MM:SS` / `yesterday HH:MM` / `Mon D HH:MM` / `Mon D 'YY HH:MM` in the configured timezone. Keep it to two or
three sentences.

## Verification

```bash
just install     # ephemeral workspaces need this before anything else
just check       # required before reporting done
just test-visual # PNG snapshot suite, after regenerating goldens
```

Manual smoke: open ACE, press `i`, and confirm the sent line appears under the pane title, updates as you `j`/`k`
through rows, stays put while `Ctrl+N` cycles attachments, and disappears cleanly when the last notification is
dismissed.

## Risks

- **Golden churn.** Two existing PNG goldens change. That is expected and intended; regenerate them in the same commit
  so the diff shows exactly one visual delta each.
- **`%-d` is glibc-specific.** Already relied on by `agent_time.py`, so this adds no new portability exposure.
- **One row of vertical space.** The detail pane loses a line. At the modal's 90%-height sizing this is not a squeeze;
  it is what buys truncation-proof placement.
