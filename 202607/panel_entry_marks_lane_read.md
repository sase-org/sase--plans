---
tier: tale
title: Mark the landed agent lane read when entering a selected tribe panel
goal:
  Pressing `l` (or `escape`) to descend from a selected Agents-tab tribe panel into an agent lane clears that lane's
  unread marker and dismisses its completion notification, exactly as `j`/`k` already do.
create_time: 2026-07-29 15:36:41
status: done
---

- **PROMPT:** [202607/prompts/panel_entry_marks_lane_read.md](prompts/panel_entry_marks_lane_read.md)
- **AGENTS:**
  - [bbugyi200.athena.ok--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ok.md#member-code)
  - [bbugyi200.athena.ok--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ok.md#member-plan)
- **COMMITS:**
  - [64ffecf](https://github.com/sase-org/sase/commit/64ffecf887426a28d64bb635c6b93ddae709a614) — fix(ace): acknowledge
    unread agents on panel entry

# Entering a Selected Tribe Panel Must Mark the Landed Lane Read

## Problem

On the ACE Agents tab, when a tribe panel is _selected_ (whole-panel focus) and the user presses `l`
(`expand_or_layout`) to descend into the panel, the lane the cursor lands on is **not** marked read even when it is
unread. The user has to nudge the cursor (`j` then `k`) to get the unread marker to clear.

Every other arrival path already acknowledges unread on the destination row:

- `src/sase/ace/tui/actions/navigation/_basic.py:139` — `j`/`k` (`_navigate_agents_panel`)
- `src/sase/ace/tui/actions/agents/_panel_navigation.py:274` — `J`/`K` (`_change_focused_agent_panel`)
- `src/sase/ace/tui/actions/agents/_folding_agent_tree.py:267` — `h` structural parent (`_navigate_agent_left`)
- `src/sase/ace/tui/actions/agents/_marking_navigation.py:299` — mark auto-advance
- `src/sase/ace/tui/actions/navigation/_member_jump.py:344`, `_entry_jump_dispatch.py:269`, `_event_widgets.py:200`,
  `_unread_navigation.py:89` — member jump, entry jump, mouse selection, unread jump

The panel-entry path does not. `AgentSelectionMixin._exit_expanded_panel_focus`
(`src/sase/ace/tui/actions/agents/_selection.py:135`) resolves the destination with `resolve_panel_entry_stop`, calls
`_focus_panel_navigation_stop(destination)`, and then repaints — with no `_acknowledge_agent_unread` call anywhere in
the chain. `_focus_panel_navigation_stop` (`src/sase/ace/tui/actions/agents/_panel_navigation.py:68-89`) sets
`_current_group_key` / `current_idx` and remembers the stop, but never acknowledges.

Why the row is unread in the first place, and why `j`+`k` fixes it: Agents-tab unread state is _projected_ from active
completion notifications by `_reconcile_unread_from_completion_notifications`
(`src/sase/ace/tui/actions/agents/_notification_unread_projection.py:168-181`), which re-adds a terminal agent's
identity to `_unread_completed_agent_ids` on every reconcile while its completion notification is still active — there
is no exemption for the currently selected row. The only thing that clears it is an explicit `_acknowledge_agent_unread`
(`src/sase/ace/tui/actions/agents/_unread_state.py:256`), which discards the identity and dismisses the matching
notification. So an agent that completes while the user sits at whole-panel focus stays unread through `l`, and a
`j`/`k` round trip is the user's workaround because those keys _do_ acknowledge.

`docs/notifications.md:154-157` already states the intended contract ("When a terminal agent is selected after it has
been marked unread ... ACE clears the row's unread marker and dismisses the matching completion notification"), so this
is a straight bug fix and the docs need no change.

## Scope

- Fix the panel-entry arrival so the landed agent lane is acknowledged, exactly like `j`/`k`.
- Cover both keystrokes that reach this path: `l` on a selected expanded panel (`action_expand_or_layout` →
  `_expand_fold` → `_exit_expanded_panel_focus`, `src/sase/ace/tui/actions/agents/_folding_agent_groups.py:203-206`) and
  `escape` (`src/sase/ace/tui/actions/_event_keyboard.py:88`).
- Banner destinations (collapsed grouping banner) must stay unacknowledged — a banner is not an agent row.

Out of scope: keymap changes, help-modal/footer text, docs, notification store behavior, and the manual-unread (`U`)
semantics.

## Implementation

### 1. Acknowledge on arrival in the shared panel-stop funnel

File: `src/sase/ace/tui/actions/agents/_panel_navigation.py`

`_focus_panel_navigation_stop` is the single funnel for "this panel move selected a row". Its three callers are all
user-initiated arrivals:

1. `_selection.py:147` — `_exit_expanded_panel_focus` (`l` / `escape`) — the bug.
2. `_panel_navigation.py:253` — `_change_focused_agent_panel` (`J`/`K`) — already acknowledges afterward.
3. `_folding_panels.py:279` — `_expand_focused_panel`.

Add the acknowledgment in the agent branch of `_focus_panel_navigation_stop`, after `current_idx` is set and after
`_remember_focused_panel_selection`, via a small helper on the same mixin:

```python
    def _acknowledge_panel_stop_arrival(self, agent_idx: int) -> None:
        """Clear unread state for the row a panel move just selected."""
        agents = getattr(self, "_agents", ())
        if not (0 <= agent_idx < len(agents)):
            return
        ack_unread = getattr(self, "_acknowledge_agent_unread", None)
        if callable(ack_unread):
            ack_unread(agents[agent_idx])
```

Use the `getattr(...)` / `callable(...)` guard style the surrounding call sites already use, so lightweight test
harnesses that omit `AgentUnreadMixin` keep working.

Notes for the implementer:

- **Do not change `_change_focused_agent_panel`.** Its existing trailing acknowledgment (lines 269-276) also covers the
  `else` fallback branch that snaps `current_idx` without going through `_focus_panel_navigation_stop`, so it must stay.
  The now-duplicated call on the stop branch is a cheap no-op: `_acknowledge_agent_unread` returns `False` immediately
  when the identity is no longer in `_unread_completed_agent_ids` (`_unread_state.py:245-254` →
  `_clear_agent_unread_and_dismiss_notification`), so nothing repaints twice. Add a short comment on the new helper
  recording that it is idempotent by design.
- **Ordering is already correct** for `J`/`K`: the new ack runs before the existing
  `_arm_manual_unread_after_departure(old_agent)` block, but that block is guarded by
  `destination != ("agent", old_idx)`, so the armed agent is never the agent just acknowledged.
- **Ordering is already correct** for `l`/`escape`: `_exit_expanded_panel_focus` clears `_expanded_panel_focus` _before_
  calling the stop focus (`_selection.py:143`), so the `_refresh_tribe_summary_only()` hop inside the ack's repaint
  (`_notification_unread_projection.py:109-111`) resolves `_focused_tribe_summary()` to `None` and no-ops; the following
  `_refresh_panel_focus_state()` paints the descended panel.
- **Perf:** the ack uses the existing selective `_patch_unread_completed_agent_changes` row patch, not a list rebuild
  (per `sase/memory/tui_perf.md` rule 6). Do not add a new refresh path and do not force `list_changed=True`.
- The banner branch of `_focus_panel_navigation_stop` returns before the new call — leave it untouched.

### 2. Tests

Both layers are required; the recorder-level test alone would not catch a regression in real unread-set mutation.

**a. Panel-entry behavior with real unread state.**

Add a harness to `tests/ace/tui/_agent_panel_collapse_helpers.py` — a subclass of the existing `AgentPanelCollapseApp`
that mixes in the real unread implementation, e.g.:

```python
class AgentPanelUnreadEntryApp(AgentUnreadMixin, AgentPanelCollapseApp):
    ...
```

(`AgentUnreadMixin` from `sase.ace.tui.actions.agents._unread`; put it first in the bases so the real
`_acknowledge_agent_unread` / `_arm_manual_unread_after_departure` win over `AgentPanelCollapseApp`'s recorder stub.) It
needs `_unread_completed_agent_ids`, `_manual_unread_agent_ids`, `_agent_info_metrics_cache = None`, a
`_try_patch_agent_row` recorder, and `_refresh_notification_count`. Model it on `LeaderUnreadJumpApp` in
`tests/ace/tui/_agent_unread_navigation_helpers.py`, which already wires exactly this combination. Note that
`make_agent` in `_agent_panel_collapse_helpers.py` builds a `RUNNING` agent — terminal-status agents (`DONE`,
`PLAN DONE`, with a `stop_time`) are needed so `is_unread_completed_status` holds; either extend that factory with
optional `status` / `stop_time` arguments or build the agents in the test module.

Add tests (new file `tests/ace/tui/test_agent_panel_entry_unread.py`, or alongside the existing entry-indicator tests)
asserting, with the `dismiss_agent_completion_notifications_matching_agents` monkeypatch fixture used by
`tests/ace/tui/test_agent_unread_selection.py`:

1. `l` on a selected expanded panel whose entry stop is an unread terminal lane clears that identity from
   `_unread_completed_agent_ids`, patches that row, and calls the notification dismissal — i.e. drive
   `app.action_expand_or_layout()` with `_expanded_panel_focus = True`, not the private helper.
2. The same holds when the entry stop comes from `_panel_selection_memory` (remembered row), not just the first rendered
   stop.
3. `escape` path parity: `app._exit_expanded_panel_focus()` acknowledges the same way.
4. A banner entry stop (collapsed grouping banner remembered in `_panel_selection_memory`, as in
   `test_collapsed_panel_omits_cursor_until_first_l_expands_it` / the collapsed-group test in
   `tests/ace/tui/test_agent_panel_entry_indicator.py`) acknowledges nothing and leaves `_unread_completed_agent_ids`
   untouched.
5. A lane manually marked unread with `U` (identity in `_manual_unread_agent_ids`) is **not** cleared by panel entry —
   `_clear_agent_unread_and_dismiss_notification` guards it (`_unread_state.py:245-247`).

**b. Existing regression coverage must stay green.**

`tests/ace/tui/test_agent_panel_entry_indicator.py` exercises `action_expand_or_layout()` /
`_exit_expanded_panel_focus()` on `AgentPanelCollapseApp`, which has no `_acknowledge_agent_unread`; the `getattr` guard
must keep those passing unchanged. Also re-run `tests/ace/tui/test_agent_unread_selection.py`,
`test_agent_unread_done_navigation*.py`, `test_agent_panel_collapse_*.py`, `test_agent_panels_display.py`, and
`test_agent_dead_end_panel_navigation.py` for the `J`/`K` ordering and double-ack no-op claims.

## Verification

Run from the workspace directory (workspaces are ephemeral, so install first):

```bash
just install
just check
```

`just check` must pass (lint + mypy + tests). If Symvision flags the new private helper, follow
`sase/memory/symvision.md` rather than widening its visibility.

## Non-goals / explicitly unchanged

- No keymap, `default_config.yml`, footer, or help-modal changes — `l`'s documented meaning ("Expand / Enter Panel") is
  unchanged; only the missing read-acknowledgment side effect is added.
- No `docs/` changes — `docs/notifications.md:154-163` already describes the behavior this fix restores.
- No Rust core change: Agents-tab unread projection and selection are presentation-only Textual state living entirely in
  this repo (`_unread_state.py`, `_notification_unread_projection.py`), with notification dismissal already routed
  through the existing `sase.notifications` facade.
