---
tier: tale
title: ",J lands on the top of the Main deck's Context card"
goal:
  Every successful ,J jump focuses a Main deck panel and shows the top of the target's
  Context card, so pending gate details are visible.
size: medium
decisions:
  context_scope:
    ask: Should `,J` also make Context the sticky Main card for later selections?
    choices:
      scoped:
        Context holds only while the `,J` target stays selected; other nodes keep your
        chosen card
      sticky:
        Later selections also open on Context until you pick another card; simpler but
        alters j/k
    default: scoped
    why:
      "`,J` is a gate-review jump, so ordinary j/k browsing should keep the reader's
      chosen card"
    answer: sticky
proposed_by: bbugyi200.athena.16a
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 10:42:35
status: wip
---

# `,J` lands on the top of the Main deck's Context card

## Goal

On the Agents tab, `,J` (`leader_mode.keys.jump_to_next_stopped_agent`) selects the next
stopped agent (plan approval, questions, workflow input, pending gates). Gate details
such as tale and epic plan summaries are drawn by the GATE section near the top of the
Main deck's **Context** card, but after `,J` the user often cannot see them:

- the focused deck panel may be showing Files, Tools, FINAL, or Record instead of Main;
- the Main deck re-shows the panel's sticky preferred card. In a spread deck with no
  explicit preference, the "sticky-Reply" landing (`land_document_spread_on_blocks`)
  scrolls to the Reply card's newest block. In a paged deck with Reply chosen, Reply
  stays on screen;
- a Main view pinned to the bottom (`SectionViewMixin.pin_to_bottom`) re-pins on the new
  document.

After this change, every successful `,J` jump:

1. **Focuses a Main deck panel.** If the focused deck panel already shows Main, keep it.
   Otherwise, if another visible panel shows Main, move logical deck focus there (the
   most recently focused one, from `PaneGrid.recent`). Otherwise switch the focused
   panel to Main. While a panel is zoomed only that panel is visible, so it is the one
   that switches. Do not change the split layout, and do not move Textual widget focus.
2. **Shows the top of the target's Context card** in that panel, in both paged and
   spread render modes, after releasing any bottom pin. Nodes without a Context card
   (for example a clan's Summary-only document) land on the top of their first card.
3. **Keeps Context on screen while the target stays selected,** so the deferred detail
   refresh, partial-then-complete Main documents, and later enrichment deliveries do not
   bounce a paged deck back to the sticky card or scroll a spread deck down to Reply.

A `,J` miss keeps today's behavior (the "No stopped agents" toast, no deck changes).
`,j` (next unread done agent) is unchanged.

## Background: how a jump reaches the deck

- `,J` in `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py` calls
  `_jump_to_next_stopped_agent()` in
  `src/sase/ace/tui/actions/agents/_unread_navigation.py`. The command palette reaches
  the same path through `_handle_leader_key`. Every jump path (visible row, collapsed
  clan reveal, off-tab reveal) ends with `self._agents[self.current_idx]` set to the
  target and refreshes with `defer_detail=True`. The detail panel therefore updates
  later through the 150 ms `DetailPanelDebouncer`, and the Main document arrives later
  still, possibly partial first, through the async sink.
- `AgentDetail` (`src/sase/ace/tui/widgets/agent_detail.py`) owns the deck area
  (`#agent-detail-panel`). Main documents for the selected subject go through
  `AgentDetailDeckSourceMixin._deliver_main_document`
  (`src/sase/ace/tui/widgets/_agent_detail_deck_source.py`), which pushes them to every
  Main panel with that panel's sticky preference
  (`area.state.panels[i].preferred_card_for(DeckId.MAIN)`). `show_deck`
  (`_agent_detail_deck_show.py`) and `_push_tab_empty_cause_to_main_deck`
  (`_agent_detail_state.py`) read the same preference.
- `DeckPanel.cycle_card` (`src/sase/ace/tui/widgets/decks/panel_navigation.py`) is how
  Ctrl+J/K show a Main card. `CardDocumentView.show_card` scrolls a paged card to the
  top, and in spread mode it delegates to `scroll_to_card`, which scrolls the card's
  anchor to the top with retries. The first card's anchor is row 0.
  `CONTEXT_CARD_ID = "context"` lives in `src/sase/ace/tui/widgets/decks/card_part.py`.
- `AgentDetailDeckTargetsMixin.ensure_main_deck_shown` (`_agent_detail_deck_targets.py`)
  and `AgentDetailDeckPaneKeysMixin.toggle_deck_focus`
  (`_agent_detail_deck_pane_keys.py`) show the existing patterns for "show Main" and
  "move deck focus": `_apply_deck_area_state(...)` with `pane_grid.focus_pane`, then
  `_notify_deck_state_changed()`.

All of this is presentation-only Textual state (deck focus, card choice, scroll), so it
stays in this repo. No `sase-core` change is needed, and no keymap or
`src/sase/default_config.yml` change is needed, because the key binding itself is
unchanged.

## Implementation

### 1. `DeckPanel`: show a Main card from its top

In `src/sase/ace/tui/widgets/decks/panel_navigation.py`, extract the Main branch of
`cycle_card` that runs after `next_id` is chosen (block-mode lookup,
`main_view.show_card` with the `TypeError` fallback, `_main_active_card`,
`refresh_chrome`, `_sync_block_navigable`, `_sync_block_rail`) into a helper, for
example `_show_main_card(card_id)`. Have `cycle_card` call it, and add a public method:

```python
def land_main_card_top(self, card_id: str) -> str | None:
    """Show ``card_id`` on the Main deck from its top; release any bottom pin."""
```

`land_main_card_top` returns `None` unless the panel's deck is Main and its stored
`_main_document` is complete (not partial) and holds `card_id`. When those hold, it
calls `main_view.release_bottom_pin()` and then `_show_main_card(card_id)`. That lands
at the top of the card in paged mode (`show_card` → `_scroll_document_to_top`) and in
spread mode (`scroll_to_card`). Keep the file's style: defensive `try/except` around
widget calls.

### 2. `AgentDetail`: Context-landing state

Add `src/sase/ace/tui/widgets/_agent_detail_context_landing.py` with
`AgentDetailContextLandingMixin`, and add it to `AgentDetail`'s bases. It holds one
in-memory, never-persisted request. Do **not** put it in `DeckAreaState`, which is
persisted.

```python
@dataclass
class _MainContextLanding:
    identity: tuple[...]   # Agent.identity of the ,J target
    pane_id: int           # the Main panel chosen in step 1 of the Goal
    landed: bool = False   # the one-shot scroll already ran for a complete document
```

Methods:

- `request_main_context_landing(identity) -> None`:
  1. Focus Main using the rules in step 1 of the Goal. Choose a visible Main panel with
     `area.panels_showing(DeckId.MAIN)`, ordered by `area.state.grid.recent`. Move focus
     through
     `self._apply_deck_area_state(dataclasses.replace(state, grid=focus_pane(state.grid, pane)))`.
     Otherwise call `self.show_deck(focused_index, DeckId.MAIN)`. Call
     `_notify_deck_state_changed()` when deck focus or the deck changed.
  2. Store `_MainContextLanding(identity, pane_id)`, replacing any earlier request.
  3. Call `_maybe_apply_main_context_landing()`. This lands immediately when the detail
     already shows the target, for example when the only stopped agent was already
     selected but scrolled down to Reply. In that case no new document will arrive.
- `_main_context_landing_matches() -> bool`: true while a request exists,
  `_current_tribe_identity` is `None`, and
  `_current_agent.identity == request.identity`.
- `_main_preferred_card(pane_id) -> str | None`: returns `CONTEXT_CARD_ID` when the
  request matches and `pane_id == request.pane_id`. Otherwise returns the panel's sticky
  `preferred_card_for(DeckId.MAIN)`. This keeps a paged deck on Context across
  same-subject re-deliveries and keeps a spread deck's new-subject paint at the top
  instead of the sticky-Reply landing.
- `_maybe_apply_main_context_landing() -> None`: no-op unless the request matches and
  has not landed, and pane `pane_id` is visible and shows Main with a complete
  `_main_document` whose `subject == self.metadata_identity`. It then calls
  `panel.land_main_card_top(card)`, where `card` is `CONTEXT_CARD_ID` if the document
  has it and the document's first card otherwise, and sets `landed = True` on success.
- `_clear_main_context_landing() -> None`: drops the request.

> [!decision] context_scope = sticky Skip `_main_preferred_card` and its call-site
> rewiring in step 3. Instead, `request_main_context_landing` calls
> `self.set_deck_preferred_card(pane_id, CONTEXT_CARD_ID, DeckId.MAIN)` after focusing,
> which is the same persisted preference Ctrl+J/K writes. The one-shot request with
> `landed` and `_maybe_apply_main_context_landing` stays, because the pin release and
> the spread/transition landings still need an explicit scroll after the target's
> complete document paints. The card-cycle clear hooks in step 3 still drop a pending
> one-shot.

### 3. Hook points

- **Effective preference** (`context_scope = scoped`): replace the direct
  `preferred_card_for(DeckId.MAIN)` reads with `self._main_preferred_card(index)` in
  `_deliver_main_document`, in the `DeckId.MAIN` branch of `show_deck`, and in
  `_push_tab_empty_cause_to_main_deck`. That last read has subject `None`, so the result
  is the same, but every Main read then goes through one helper. Leave `DeckArea`'s own
  re-shows alone. They run on explicit card choices, which clear the request first (see
  below), or on a persisted-state restore, where no request exists.
- **Apply after paint:** call `_maybe_apply_main_context_landing()` at the end of
  `_deliver_main_document` when `show_panels` is true, and at the end of `show_deck`'s
  Main branch.
- **Clear on subject change:** in `_deck_update_display_impl` (where `_current_agent` is
  reassigned), clear the request when the new agent's identity differs from the
  request's. Do the same in `_deck_show_empty`, `_deck_show_tribe_summary`, and
  `show_record_subject`. Clear only on a subject change, never on a document delivery: a
  stale async delivery for the previous subject can arrive before the deferred refresh
  switches subjects.
- **Clear on explicit card choice:** clear the request at the start of
  `cycle_focused_deck_card`, `cycle_focused_card_block`, and `select_focused_card_block`
  in `_agent_detail_deck_show.py`. Their existing sticky-preference writes and re-shows
  then behave exactly as today.

### 4. App: land after a stopped-agent hit

Add `src/sase/ace/tui/actions/agents/_stopped_jump_landing.py` with
`land_stopped_jump_on_main_context(app) -> None`. It returns early unless
`app.current_tab == "agents"`, `_current_group_key is None`, and `current_idx` indexes
`app._agents`. Otherwise it resolves `app.query_one("#agent-detail-panel", AgentDetail)`
(follow `_deck_layout_detail` in `_deck_layout_actions.py`) and calls
`detail.request_main_context_landing(app._agents[app.current_idx].identity)`, then
`app._refresh_agent_footer_bindings_only()` when it is callable, because deck focus
affects footer hints. Swallow lookup failures: the test fake apps used by the
stopped-navigation tests have no widget tree.

In `AgentUnreadNavigationMixin._jump_to_next_stopped_agent`, call it only when the jump
returns `True`, and still return the jump result. The call is O(panels), does no I/O,
and triggers no detail refresh. The deferred debounced refresh stays the only detail
update (`tui_perf` rules 5 and 7), and the `,J` comments about having no trailing
refresh stay true.

### 5. Docs

Update the Reference page `docs/reference/tui/agents-main-deck.md`, in the paragraph on
the preferred card (Ctrl+J / Ctrl+K). Add that `,J` focuses a Main deck panel (an
already-visible Main panel first, otherwise the focused panel switches to Main) and
opens the target at the top of its Context card so pending gate details are visible.
Under `scoped`, add that this holds while the agent stays selected and leaves the
preferred card unchanged. Under `sticky`, say that it makes Context the preferred card.
Do not edit other pages. The keymap label "Jump to next stopped agent" stays as it is.

## Tests

Add Textual pilot tests in `tests/ace/tui/widgets/decks/test_deck_context_landing.py`,
reusing the `_DetailApp` harness from
`tests/ace/tui/widgets/decks/test_deck_action_targets.py` and the agent and document
helpers used by `test_deck_view_main_pilot.py`. Use agents whose Main document has both
Context and Reply cards. Force render modes with
`detail.set_focused_deck_view(DeckView.PAGE_CARDS / DeckView.SPREAD)`.

1. Focus: in a left-right split with the focused panel on Files and the other on Main, a
   request moves `area.state.focused` to the Main pane and keeps both decks. With no
   visible Main panel, the focused panel switches to Main.
2. Same subject, paged: with sticky preference Reply showing, a request for the shown
   agent makes Context active and puts the scroll at 0 immediately.
3. Same subject, spread, pinned: with the view pinned to the bottom, a request releases
   the pin and lands the scroll at 0 after pilot pauses.
4. New subject: with agent A shown and sticky Reply set, request agent B and then
   `update_display(B)`. Context is active at scroll 0 in both paged and spread modes.
   Re-delivering B's document (same subject) keeps Context.
5. Scope (`scoped`): after step 4, `update_display(A)` lands on Reply again, and the
   stored `preferred_card_for(DeckId.MAIN)` is still Reply. Under `sticky`, assert
   instead that the preference became Context.
6. A card cycle (`cycle_focused_deck_card(1)`) after landing clears the request and
   writes the sticky preference exactly as before.
7. Stale request: request B, then `update_display(C)` and `update_display(B)`. B now
   opens on its sticky card, because the request was cleared when C was shown.

In `tests/ace/tui/test_agent_stopped_navigation.py`, monkeypatch
`land_stopped_jump_on_main_context` and assert that a hit calls it once with an app
whose selected agent is the jump target, and that a miss does not call it. The existing
stopped-jump tests must pass unchanged.

## Verification

Run `sase tool run check` (`just check` through the tool wrapper) and fix all failures.
Do not run `just check-full`. No PNG golden changes are expected. If a visual snapshot
fails, investigate it rather than regenerating goldens blindly.

## Known limits

- A same-subject paged-to-spread mode transition that happens after the landing still
  goes through `_apply_document_transition`'s existing sticky-Reply block landing.
  Stopped agents rarely change mode while selected, so this is out of scope.
- When the sync render is digest-deduped (`_resync_main_sink_after_sync_display` stores
  without pushing), the landing waits for the next delivery for that subject.
