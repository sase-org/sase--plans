---
tier: tale
title: Browse-first XPrompts filtering
goal: "The Admin Center Config/XPrompts child opens as a clean, list-focused browser and
  reveals its live filter only when the user presses slash, with predictable focus,
  dismissal, navigation, and visual behavior.

  "
size: small
proposed_by: bbugyi200.athena.sase-ri.land.w2.f2.w3
create_time: 2026-08-21 09:52:58
status: wip
---

# Plan: Browse-first XPrompts filtering

## Context and intended experience

The Config/XPrompts child currently composes its `BrowserFilterInput` visibly and
focuses it on every activation. That makes the pane filter-first, forces the input to
emulate row-navigation and Admin Center numeric dispatch, and permanently costs one line
of vertical space. Neighboring Config children such as Glossary and Snippets instead
open on their row lists and reveal a focused inline filter from `/`.

Bring XPrompts onto that browse-first interaction model:

- On first mount, show the full XPrompt list and preview with the filter removed from
  layout and the highlighted row list focused.
- Add a pane-local `slash -> focus_filter` binding alongside the XPrompt browser's
  existing local bindings. Pressing `/` reveals the existing filter widget, keeps any
  already-applied query, places the cursor for immediate editing, and focuses the input
  without reloading catalog data.
- Keep filtering live and retain the existing identity-based selection restoration,
  preview updates, jump invalidation, and empty-result handling while text changes.
- Treat Enter and Escape in the filter as “done”: keep the applied query/results, hide
  the filter completely, and return focus to the row list. Escape remains layered: it
  cancels active jump mode first, ends an active filter edit second, and only closes the
  Admin Center when the list owns focus.
- Preserve cached-pane intent across tab/sub-tab navigation. A closed filter stays
  hidden with its query applied; if the user navigates away while the editor is still
  visibly open, returning to XPrompts restores focus to that editor. Normal browse
  activation focuses the list.
- Once filtering is explicitly opened, printable characters are filter text. In
  particular, a leading digit or apostrophe must no longer be redirected to Admin Center
  tab selection or entry-jump mode; those shortcuts remain available from the
  list-focused browse state. Bracket keys continue to cycle Config sub-tabs, and the
  Admin Center's priority Tab/Shift+Tab behavior remains intact.

Use the XPrompt pane's existing binding table rather than introducing a new configurable
keymap scope solely for this action; consequently no `default_config.yml` keymap entry
or schema change is needed.

## Implementation

1. Update `XPromptBrowserPane` to own the new slash action and the filter session's
   visibility/focus transitions. Initialize the input hidden after composition, make
   `focus_default()` choose the visible editor or the browser list as appropriate, and
   add small helpers/actions for opening and closing the editor. Reuse the current
   synchronous, in-memory filtering/repaint path so `/` adds no disk I/O, workers,
   timers, or event-pump work.

2. Simplify `BrowserFilterInput` around its new role as an explicitly opened text
   editor. Route Escape to the pane's close-filter helper after giving active jump
   cancellation first refusal, let Enter submit to the pane instead of editing the
   highlighted XPrompt, and remove the empty-input digit/apostrophe forwarding that
   existed only because the input previously owned focus by default. Keep the
   readline/control actions that are useful during filtering and the embedded Config
   bracket forwarding; row-only actions continue to work from the list.

3. Refresh `browser_hint_text` and the pane's hint synchronization so the resting view
   clearly advertises `/: filter`, while an open filter describes its own Enter/Escape
   completion behavior rather than claiming Escape will close the Admin Center. Keep the
   wording compact enough for the standard Admin Center width and continue to
   conditionally advertise inline load only for eligible rows.

4. Rework the XPrompt browser interaction tests to assert the new state machine instead
   of the old filter-first workaround:
   - initial filter hidden and list focused;
   - `/` reveals/focuses it without changing the selection;
   - live filtering still updates rows, preview, selection bookmark, and jump state;
   - Enter/Escape retain the query, hide the bar, and restore list focus;
   - a subsequent list-focused Escape closes the Admin Center;
   - leading digits and apostrophes type normally in an explicitly opened filter;
   - list-focused apostrophe jump, numeric Admin Center navigation, row actions,
     brackets, Tab/Shift+Tab, and cached reactivation retain their intended owners;
   - empty catalogs and no-match filters remain safe no-ops with a cleared preview.

5. Update the Config/XPrompts PNG coverage so the baseline golden shows the clean,
   browse-first layout with the extra vertical room and list focus. Add a focused
   filter-state snapshot (with representative text/results) so the revealed bar, focus
   border, spacing, preview, and state-specific hint line are reviewed rather than
   leaving the new visual state unguarded.

## Verification

Run `just install` first in the implementation workspace. Then run the focused XPrompt
browser interaction suites, including the jump and inline-load/keymap tests, and the
Config hub/navigation tests touched by focus ownership. Regenerate only the intentional
Config/XPrompts PNG golden(s), inspect the actual images, and rerun their visual test
without snapshot-update mode. Finally run `just check`; run `just test-visual` as the
dedicated full PNG snapshot gate because this change intentionally alters the Admin
Center layout.

The completed behavior is acceptable when XPrompts always opens browse-first with no
filter row, `/` reveals a polished live editor, leaving that editor never loses the
applied query or selection semantics, printable filter text cannot trigger unrelated
navigation, all three Escape layers are deterministic, and both resting and active
filter states are protected by behavioral and pixel-level tests.
