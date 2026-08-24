---
tier: tale
title: Select the next folded agent group after marking
goal:
  The Agents-tab mark action advances to the immediately following selectable row,
  including an intervening folded group banner.
size: small
proposed_by: bbugyi200.athena.0cw
---

# Select the next folded agent group after marking

## Goal

Make the Agents-tab `m` action advance to the immediately following selectable row in
render order. When a folded group banner lies between the marked Agent Node and the next
visible Agent Node, focus the folded banner so a subsequent `m` can act on that group.
Keep ordinary agent-to-agent advancement, wraparound, mark ordering, unread transitions,
and the fast selective-repaint path unchanged.

## Current behavior and root cause

- Agents-tab `j`/`k` navigation and folded-group mark advancement already use
  `_panel_navigation_stops()`, whose cached tree projection contains visible agent rows
  and folded group banners in their actual rendered order.
- Single-agent mark advancement in
  `src/sase/ace/tui/actions/agents/_marking_navigation.py` instead uses
  `_agents_visible_order()`. That agent-only projection intentionally omits folded
  banners, so `m` jumps over a banner such as the folded `09:00` group and selects the
  following agent.
- `tests/ace/tui/test_agent_marking.py` currently codifies that obsolete policy in
  `test_toggle_mark_skips_collapsed_banner_rows`, even though folded banners are now
  selectable mark targets and group-mark advancement already visits them.

## Implementation

1. Update single-agent post-mark advancement in
   `src/sase/ace/tui/actions/agents/_marking_navigation.py` to resolve the selected
   agent's position in `_panel_navigation_stops()` and advance by one selectable stop,
   with the same wraparound order as ordinary Agents-tab navigation.
   - Reuse `_panel_navigation_stop_maps()` when available instead of rebuilding or
     linearly re-deriving navigation state on the keystroke path.
   - For an agent destination, retain the existing behavior: clear banner focus, update
     `current_idx`, acknowledge unread arrival state, and return the selected agent so
     selective row patching can update its styling.
   - For a banner destination, set `_current_group_key`, anchor `current_idx` to a
     backing member using the existing group-anchor helper, and report that focus moved
     without treating the hidden backing member as an unread arrival.
   - Arm manual-unread departure state when leaving the marked agent for either kind of
     distinct destination.
   - Retain the current visible-agent/raw-index fallback only for partial harnesses or
     stale states where the selectable-stop list cannot locate the current agent.
   - Keep `_toggle_mark_agent()` on its selective repaint path: patch the toggled agent
     row and refresh panel highlights so a banner destination is painted, without a full
     agent-list rebuild when patching succeeds.

2. Replace the obsolete skip-banner regression in `tests/ace/tui/test_agent_marking.py`
   with a mixed-order case containing a visible agent, an intervening folded group, and
   a later visible agent.
   - Assert that `m` marks only the original agent.
   - Assert that focus lands on the folded group's key and that `current_idx` is
     anchored to one of that group's backing agents rather than jumping to the later
     visible agent.
   - Assert the highlight refresh/selective patch behavior when the row patch succeeds,
     so the state change is guaranteed to become visible without forcing a rebuild.
   - Preserve the existing agent-to-agent rendered-order, wraparound, unread-arrival,
     group-mark, and group-to-banner tests as regression coverage for the unaffected
     paths. Adjust the shared marking harness only if needed to expose the production
     grouping mode or selectable-stop behavior accurately.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral SASE
   workspace.
2. Run `pytest tests/ace/tui/test_agent_marking.py` to exercise the focused mark and
   folded-banner regressions.
3. Run `pytest tests/ace/tui/test_agent_jk_navigation.py` to confirm the shared
   selectable-stop ordering and banner-focus semantics remain aligned with ordinary
   navigation.
4. Run `just check` for whole-repository lint gates and diff-scoped tests.

No visual snapshot update is expected because the rendered rows and banner styling do
not change; only the post-keypress focus target changes.
