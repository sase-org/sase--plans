---
tier: tale
title: Always show the notification modal's tag tab strip
goal:
  The notification modal's tag tab strip is visible whenever the modal has any
  notifications, including when only one tab exists, so the left pane's header row no
  longer appears and disappears as tabs come and go.
proposed_by: bbugyi200.athena.sase-gz.land.f1
create_time: 2026-08-07 13:34:48
status: done
---

# Plan: Always show the notification modal's tag tab strip

## Which bar this is

`NotificationModal`'s **tag tab strip** — the one-line clickable bar at the top of the
modal's left pane that renders `<icon> <Tab name> <count>` for each top-level tab
(General / Gates / Errors / Beads / Muted / Snoozed / per-tag). It is the widget
`NotificationTagStrip` in `src/sase/ace/tui/modals/notification_modal_tags.py`, mounted
with `id="notification-tag-tabs"`.

This is the only tab bar in the repo whose visibility is gated on tab count. The
top-level ACE `TabBar` (`src/sase/ace/tui/widgets/tab_bar.py`) always renders its three
tabs, and every `PanelTabStrip` consumer (Config Center, Statistics, Plugins Browser,
Projects, Artifacts, Help) is always visible. So no other bar needs to change.

## What happens today

Two places hide the strip when the modal has one tab or fewer:

- `src/sase/ace/tui/modals/notification_modal.py:122` — `compose()` mounts the strip
  with `classes="hidden" if len(tabs) <= 1 else None`.
- `src/sase/ace/tui/modals/notification_modal.py:374-377` — `_refresh_tag_strip()`
  re-applies the same rule on every rebuild (`add_class("hidden")` when
  `len(tabs) <= 1`, `remove_class("hidden")` otherwise).

`_refresh_tag_strip()` runs from `_rebuild_list()`
(`src/sase/ace/tui/modals/notification_modal.py:505`), which every dismiss, mute,
snooze, and reclassification path calls. So the strip currently pops in and out
mid-session: dismiss the second-to-last tab's final row and the bar vanishes while the
list below it jumps two rows taller.

## The rule this plan implements

**The strip is visible whenever there is at least one tab; it is hidden only when there
are zero tabs.**

Zero tabs happens exactly when the modal holds no notifications. In that state
`_rebuild_list()` hides `#notification-list` and mounts the `#notification-empty` "No
unread notifications" message (`src/sase/ace/tui/modals/notification_modal.py:514-527`),
and a zero-tab strip would render as a bare blank line plus its one-row bottom margin
above that message. Keeping it hidden there matches the initial compose path, which does
not yield the strip at all when `self._notifications` is empty
(`src/sase/ace/tui/modals/notification_modal.py:116`).

So this is a one-token change of predicate in both places: `len(tabs) <= 1` becomes
`not tabs`.

## Changes

### 1. `src/sase/ace/tui/modals/notification_modal.py`

In `compose()`, change the strip's class argument (line 122) from

```python
classes="hidden" if len(tabs) <= 1 else None,
```

to

```python
classes=None if tabs else "hidden",
```

This branch already only runs when `self._notifications` is non-empty, so in practice
the strip now always mounts visible; the `else` arm is defensive parity with
`_refresh_tag_strip()` rather than a reachable state.

In `_refresh_tag_strip()`, change lines 374-377 from

```python
if len(tabs) <= 1:
    strip.add_class("hidden")
else:
    strip.remove_class("hidden")
```

to

```python
if tabs:
    strip.remove_class("hidden")
else:
    strip.add_class("hidden")
```

Update the method's docstring to state the rule, and add a short comment recording _why_
zero tabs still hides: an empty strip would put a blank line and its margin above the
"No unread notifications" message.

### 2. Leave the keyboard tab-cycling guards alone

`action_prev_notification_tag_tab()` and `action_next_notification_tag_tab()`
(`src/sase/ace/tui/modals/notification_modal.py:442-464`) return early when
`len(tags) <= 1`. That guard is about _cycling_, not visibility — with one tab, `[` and
`]` correctly do nothing. Do not touch them. Reviewers grepping for `<= 1` in this file
will find these; the comment added in change 1 should make clear that the display rule
and the cycling rule are deliberately different.

### 3. No CSS change

`NotificationModal #notification-tag-tabs` in `src/sase/ace/tui/styles.tcss:4960`
already declares `height: 1; margin-bottom: 1;`, and `#notification-list` is
`height: 1fr`, so the list absorbs the two rows automatically. Nothing to edit.

## Tests

Add to `tests/test_notification_modal_sections.py`, beside the existing
`test_tag_strip_*` cases (which start at line 667).

`_refresh_tag_strip()` is a method on `NotificationModal` itself, not on a mixin, so
build a real modal and stub its `query_one` with a `MagicMock` strip — the same shape
`tests/test_notification_modal_sent_at.py:115-137` uses for the sent-at label. Assigning
over `query_one` on the instance is fine; the existing strip tests already do the
equivalent with `strip.post_message = MagicMock()`.

1. **A single tab keeps the strip visible.** One notification, so `_tag_tabs()` returns
   exactly one tab. Assert `strip.remove_class` was called with `"hidden"` and
   `strip.add_class` was never called. This is the regression test for the reported
   behavior — assert on the one-tab case specifically, not just "not hidden".
2. **Zero tabs hides the strip.** A modal built with no notifications, `_tag_tabs()`
   empty. Assert `strip.add_class("hidden")` and that `remove_class` was not called.
3. **Two tabs still keep it visible.** Cheap guard that the predicate flip did not
   invert the common case.
4. **A dismiss that collapses two tabs to one leaves the strip up.** The behavior the
   user actually hit: build a modal with two notifications in different tabs, dismiss
   the row that empties one tab, and assert the strip was not hidden. Drive this through
   the real path (`_rebuild_list()` with a stubbed option list, or the dismiss action)
   rather than calling `_refresh_tag_strip()` directly, so the test covers the wiring
   and not just the predicate. `tests/test_notification_modal_mark_and_tabs.py:191`
   (`test_dismiss_last_row_in_active_tag_falls_back_to_nearest_tab`) shows the fixture
   shape for a dismiss that empties a tab; reuse its helpers via
   `tests/_notification_modal_helpers.py`.

Also confirm no existing test asserts the old hide-at-one-tab behavior. A search of
`tests/` for `notification-tag-tabs`, `NotificationTagStrip`, and `_refresh_tag_strip`
turns up only the click/reflow tests in `tests/test_notification_modal_sections.py`,
none of which assert visibility — but re-check before assuming nothing needs updating.

## Visual snapshots

Exactly five visual test modules push a `NotificationModal`:
`test_ace_png_snapshots_notification_{sent_at,gates,question,report,beads}.py` under
`tests/ace/tui/visual/`. Every golden they produce from a **single-tab** modal gains one
strip row plus its one-row margin, and its option list shifts down two rows. Expected to
change:

- `notification_sent_at_120x40.png`
- `notification_gate_pending_120x40.png`
- `notification_gate_answered_120x40.png`
- `notification_question_summary_120x40.png`
- `notification_report_pane_120x40.png`

Expected **not** to change:

- `notification_beads_tab_120x40.png` and `notification_filed_by_120x40.png` — their
  fixture builds several tabs, so the strip was already visible.
- `notification_report_modal_120x40.png` — despite the name, that test pushes
  `ReportModal`, not `NotificationModal`
  (`tests/ace/tui/visual/test_ace_png_snapshots_notification_report.py:213`).
- The three `notification_indicator_*` goldens — those capture the main-window
  indicator, not the modal.
- Every `custom_gate_*`, `tale_plan_gate_*`, `epic_plan_gate_*`, and `gate_debug_*`
  golden — those push `CustomGateModal`, `PlanApprovalModal`, or `GateDebugModal`.

Procedure:

1. Run `just test-visual` first, without regenerating, and read the failure list.
2. Compare it against the two lists above. If a golden outside the expected-change set
   fails, or an expected one does not, stop and work out why before accepting anything —
   that is a signal the change reached further than intended.
3. Inspect each diff in `.pytest_cache/sase-visual/` and confirm the only delta is the
   new strip line and the two-row downward shift of the list.
4. Accept with `--sase-update-visual-snapshots`, then re-run `just test-visual` clean.

Note that the visual tests also make `assert_page_svg_contains` assertions; the left
list loses two rows of height, so if any of those assertions target left-pane content
that now falls outside the viewport, fix the assertion rather than the golden.

## Verification

- `just install` first — workspace virtualenvs go stale between sessions.
- `just check` for the lint gates plus the scoped test lane.
- `just test-visual` for the PNG suite (excluded from `just check`).
- `just check-full` before landing, since this touches a widely imported ACE modal.

## Out of scope

- No new config option. The user asked for the bar to be shown, not for a setting.
- No change to `PanelTabStrip`, `TabBar`, or the `ZoomFileRail` `count <= 1` collapse
  (that rail is a file list, not a tab bar).
- No change to the strip's narrow-width reflow, which already sheds inactive labels when
  the full render would overflow. A single-tab strip is far too narrow to trigger it.
