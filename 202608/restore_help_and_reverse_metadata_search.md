---
tier: tale
title: Restore global help and scope reverse metadata search
goal:
  Bare question mark opens the tab-aware Ace help panel from every top-level tab, while Ctrl+R reverses an
  already-active Agents metadata search without losing the query, matches, or reverse-search functionality.
proposed_by: bbugyi200.athena.s5
create_time: 2026-08-02 11:58:38
status: wip
---

- **PROMPT:**
  [prompts/202608/restore_help_and_reverse_metadata_search.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/restore_help_and_reverse_metadata_search.md)

# Plan: Restore global help and scope reverse metadata search

## Context and intended behavior

Ace currently binds `/` to forward metadata search on the Agents tab, `?` to starting that search in reverse, and `,?`
to the tab-aware help panel. Restore the earlier global-help ownership of `?`, but preserve reverse metadata search as a
transient operation inside the search that `/` starts.

The resulting behavior must be:

- On the normal ChangeSpecs, Agents, and Axe surfaces, `?` opens the help panel for the current tab. The old `,?` leader
  chord no longer opens help.
- On the Agents tab, `/` continues to start a forward metadata search. Outside an active metadata search, `Ctrl+R` has
  no metadata-search action and remains available to focused widgets that already own it.
- While the metadata search is active, its configurable reverse action defaults to `Ctrl+R`. Toggling direction keeps
  the frozen corpus and current query. In typing mode it recomputes the nearest preview match from the original search
  origin; in committed mode it changes the recorded direction so subsequent `n` / `N` navigation follows the new /
  opposite order without clearing the current match.
- `?` is no longer a reverse-search starter or committed-search restart for the metadata panel. When the committed
  overlay is no longer consuming text, allow the restored app-level help binding to receive it. Do not change the
  separate Vim search behavior inside prompt editors or locally scoped help bindings such as the Admin Center Statistics
  pane.
- User overrides remain coherent: `app.show_help` is configurable again, `app.search_reverse` configures the transient
  reversal key, and stale `leader_mode.keys.show_help` overrides are ignored instead of resurrecting the removed chord.

## Implementation

1. Reassign the configurable keymap ownership across the default configuration, app keymap dataclass, binding
   metadata/fallback bindings, mode defaults, and registry migration logic. Set `app.show_help` to `question_mark`,
   change `app.search_reverse` to `ctrl+r`, remove `show_help` from the built-in leader mode, unretire the app-level
   help action, and retire the former leader action for backward-compatible config loading. Keep the intentional `/`
   collision between Agents metadata search and non-Agents query editing unchanged.

2. Rewire dispatch and availability around the new scopes. Remove the leader help dispatch/footer entry, catalog
   `show_help` as an all-tabs app command, and keep it available on every Artifacts sub-tab. Make `search_reverse`
   describe reversing search order and make both Textual action availability and command-context availability require an
   active Agents metadata search; extend command-context extraction with the transient search state so tests and palette
   filtering use the same rule. Retain `search_forward` as the Agents-only action that can start a search.

3. Add an explicit direction-toggle operation to the shared Vim search controller. It should invert the direction
   without rebuilding the frozen corpus or dropping the query, refresh the preview/command sigil and selected match in
   typing mode, and update committed `last_search` state so repeat navigation honors the new order. Give the
   controller/host call a way to disable the hard-coded committed `?` reverse restart for metadata search only, leaving
   prompt-editor search semantics unchanged.

4. In the Agents metadata-search host, recognize the configured `app.search_reverse` key before generic search
   text/navigation handling and invoke the controller toggle only while the overlay is active. Keep the app action
   itself guarded as well, allow a committed `?` to leave the overlay and reach the restored help action, and render the
   configured reversal key in the inline search instructions rather than hard-coding it.

5. Update every user-facing key reference. The three tab-specific help binding builders should show the app-level help
   key in their general/miscellaneous sections; the Agents metadata-search section should document `/` plus the
   configured reversal key instead of `/ / ?`; onboarding and empty-tab quick starts should derive help hints from
   `app.show_help`; and leader-mode hints should no longer list help. Regenerate only the visual snapshots whose help,
   onboarding, command-palette, or metadata-search text changes.

## Test coverage and validation

- Update keymap default, registry-loading, binding-construction, duplicate-key, display-help, command-catalog,
  command-context, leader-dispatch/footer, onboarding, quick-start, and end-to-end tests to assert the new defaults,
  scope, custom overrides, and retirement behavior.
- Add focused controller tests for toggling before and after query commit, preserving query/corpus/matches, selecting in
  the opposite direction from the same origin, updating `last_search`, and leaving the prompt-search `?` behavior
  intact.
- Extend the Agents metadata-search integration test to prove that `/` starts forward search, `Ctrl+R` (and an override
  of `app.search_reverse`) reverses it only while active, repeated toggles are stable, `n` / `N` follow the chosen order
  after commit, and `?` opens help instead of starting/restarting reverse metadata search. Verify bare `?` opens the
  correct help panel across all three top-level tabs and `,?` is retired.
- Run `just install` first as required for an ephemeral workspace. Run the focused unit/integration suites for keymaps,
  commands, help/onboarding, the shared search controller, and Agents metadata search; inspect and accept only
  intentional PNG changes with `just test-visual -- --sase-update-visual-snapshots`, then rerun `just test-visual`
  without the update flag. Finish with the mandatory repository-wide `just check`.
