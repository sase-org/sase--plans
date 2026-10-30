---
tier: tale
title: Keep Ctrl+J card choices across background deck repaints
goal:
  A card chosen with Ctrl+J/Ctrl+K on FINAL, RECORD, or Tools stays put through live
  ticks, worker completions, and Runs reloads.
size: medium
proposed_by: bbugyi200.athena.16f
proposed_with: claude/opus
create_time: 2026-10-30 12:55:39
status: wip
---

# Keep Ctrl+J card choices across background deck repaints

## Problem

Pressing `Ctrl+J` (`next_deck_card`) on the Agents tab's FINAL deck moves to the next
card (for example the `commit` instance card), and then about a second later the deck
snaps back to the card it was on before. The user sees the same "switches back" problem
on other decks too.

## Diagnosis (root cause, reproduced)

Each deck panel tracks which card is active in two places:

1. **The sticky choice (the source of truth).** This is
   `DeckPanelState.preferred_cards[deck]` in the deck area state
   (`src/sase/ace/tui/widgets/decks/model.py`). `Ctrl+J`/`Ctrl+K` write it through
   `AgentDetail.cycle_focused_deck_card` → `set_deck_preferred_card` /
   `_remember_deck_preferred_card`
   (`src/sase/ace/tui/widgets/_agent_detail_deck_show.py`). It is also persisted.
2. **Cached copies that background repaints read.** These copies are captured when a
   load is requested and never updated by `Ctrl+J`:
   - `FinalDeckView._current_preferred` and `RecordDeckView._current_preferred` are set
     only in `update_display()`. FINAL's 1 Hz live tick (`_live_tick_run`) and the
     worker-completion paths (`on_worker_state_changed` in
     `src/sase/ace/tui/widgets/decks/final/view.py` and
     `src/sase/ace/tui/widgets/decks/record/view.py`) repaint with that stale copy, so
     `_paint_final_result` / `_paint_record_result` fall back to the stale or default
     card. That default is the attention instance or the first instance.
   - For Tools, every `⚒ Runs` paint posts `ToolRunsDeckLoaded(active_card="runs")`,
     because the Runs document has exactly one card, `runs`.
     `DeckPanelNavigationMixin.handle_tool_runs_deck_loaded`
     (`src/sase/ace/tui/widgets/decks/panel_navigation.py`) then unconditionally calls
     `_store_document_active(DeckId.TOOLS, "runs")` and `_sync_tools_hosts()`, which
     overwrites a `Ctrl+J` choice of `llm-calls` and flips the visible card back to
     Runs. Runs paints happen on worker completion, on the live tick's drift reload, and
     on every refresh whose render digest changed. On a node with live tool runs, that
     is nearly every refresh.

This breaks `tui_perf` rule 4: UI state captured before an await is applied after the
await without being re-read.

**Evidence (headless pilot, scratch only).** The repro reused the FINAL fixtures in
`tests/ace/tui/visual/_ace_agents_final_png_snapshot_shared.py`. The node had the cards
`overview`, `instance:check` (attention), and `instance:commit`. The deck was shown, the
test pressed `ctrl+j`, and it sampled the state every 100 ms:

- **Summary phase `settled`:** the card stays on `instance:commit`, because no tick
  runs.
- **Summary phase `executing`:** the card is `instance:commit` for about 0.5 s, then
  reverts to `instance:check` and stays there. The area sticky still says
  `instance:commit`, but `final_view._current_preferred` is still `None`.
- **Control run:** setting `final_view._current_preferred` to the sticky value right
  after `ctrl+j` removes the revert completely. This confirms the stale copy is the only
  cause in the FINAL case.
- **Tools:** a headless `DeckPanel(0)` with stored Tools card `llm-calls` reads `runs`
  after one `handle_tool_runs_deck_loaded(ToolRunsDeckLoaded(active_card="runs"))`.

FINAL is most interesting to watch while finalizers run, and that is exactly when its 1
Hz tick is active (summary phase `declaring`/`executing`). That is why the user hits it
"every time." Worker completions that land after a `Ctrl+J` cause the same revert when
the node is not finalizing, but less often.

## Fix

The rule: **background repaints re-read the sticky choice at paint time, and a
loaded-notification handler never overwrites an explicit card choice.** The sticky state
stays the single source of truth. Do not add a second mirror that card-switch paths must
remember to update.

### 1. Panel accessor for the live sticky card

Add a public method to `DeckPanelCardDocumentsMixin` in
`src/sase/ace/tui/widgets/decks/card_documents.py`:
`sticky_card_for(self, deck: DeckId) -> str | None`. It returns this panel's sticky card
for `deck` from the hosting deck area's state:
`self.parent.state.panels[self._panel_index].preferred_card_for(deck)`. `DeckPanel` is
composed directly inside `DeckArea`. Use duck typing (`getattr`) on the parent:
`area.py` already imports `panel.py`, so importing `DeckArea` back would create an
import cycle.

- Raise `LookupError` when there is no hosting area or no session for this pane. This
  covers headless `DeckPanel(0)` in unit tests. Callers can then tell "no host" apart
  from "host has no sticky card" (`None`).
- Keep it pure in-memory and O(1): no I/O, safe on pump and tick paths.

### 2. View helper: `CardDocumentView.host_preferred_card(fallback)`

Add a public method in `src/sase/ace/tui/widgets/decks/document_view.py`. It finds the
host panel through `self.parent.parent` (VerticalScroll → DeckPanel). This is the same
hop `_spread_accent` and `_publish_section_layout` already use. Then it returns
`panel.sticky_card_for(self._deck)`. If the host or method is missing, or the call
raises (including `LookupError`), it returns `fallback`. A reachable host's `None` is
authoritative and means "no explicit choice." With no sticky choice, FINAL's documented
default rule still applies unchanged (`final_default_card`: sticky, then attention, then
`overview` on trouble, then the first instance).

### 3. FINAL view: re-read on async paints

In `src/sase/ace/tui/widgets/decks/final/view.py`, change two paths:

- `_live_tick_run`
- the `WorkerState.SUCCESS` branch of `on_worker_state_changed`

Each one should compute `preferred = self.host_preferred_card(self._current_preferred)`,
store it back into `self._current_preferred`, and pass it to `_paint_final_result`.

Leave the synchronous cached-hit paint inside `update_display` alone: its caller already
passes the fresh sticky value.

### 4. RECORD view: same for the worker path

In `src/sase/ace/tui/widgets/decks/record/view.py`, apply the same re-read in the
`WorkerState.SUCCESS` branch of `on_worker_state_changed`. RECORD has no live tick, so
only the race where a worker lands after a `Ctrl+J` is affected.

### 5. Tools: stop the Runs-loaded handler from clobbering the card choice

In `handle_tool_runs_deck_loaded`
(`src/sase/ace/tui/widgets/decks/panel_navigation.py`), store `message.active_card` as
the Tools active card **only when no Tools card is recorded yet**, that is, when
`_stored_document_active(DeckId.TOOLS)` is `None`. Keep a comment explaining why: the
message's `active_card` is the Runs document's internal card (always `runs`), not a
Tools-deck choice. The deck-level choice belongs to two places:

- `_show_tools_card`, used by `Ctrl+J`/`Ctrl+K` and tool-run reveal
- `_refresh_tools_panel` in `src/sase/ace/tui/widgets/_agent_detail_deck_refresh.py`,
  which re-applies the sticky choice on every refresh

`active_tools_card()` already validates a stored id against the available cards, so a
kept `llm-calls` that has disappeared still falls back correctly.

Do not change the Tool Runs view's own `_current_preferred` plumbing. Its document has
only one card, so that preference never picks a different card.

### Constraints

- Keep every new helper fail-open (`try/except` → fallback), like the surrounding code.
  No new I/O on keystroke, tick, or pump paths (`tui_perf` rules 1, 2, and 11).
- New public names must have real callers (symvision). Do not import `_`-prefixed
  symbols across modules.
- Keep the touched files under the `toobig` line-count gates. The current sizes are
  roughly 565–660 lines against a 700-line soft limit. If one would cross it, put the
  helper in a smaller sibling module rather than growing `panel.py`.

## Tests

Add `tests/ace/tui/widgets/decks/test_deck_card_choice_survives_repaint.py`. It must not
be `visual`-marked, so it runs in the default lane. It may import the FINAL fixture
helpers from `tests/ace/tui/visual/_ace_agents_final_png_snapshot_shared.py`
(`make_agent`, `deck_summary`, `patch_final_view`, `goto_agents`, `show_final`,
`node_failed_check`) and `patches` / `patch_startup_loaders` from
`tests/ace/tui/visual/_ace_png_snapshot_helpers.py`.

1. **FINAL live tick keeps the `Ctrl+J` card.**
   - **Setup:** an `AcePage` pilot with one agent whose summary is
     `deck_summary("check", "commit", status="failed", phase="executing")`. The node
     view is `node_failed_check()` with a second `commit` instance appended to both
     `instances` lists, giving the cards `overview`, `instance:check` (attention), and
     `instance:commit`.
   - **Steps:** call `show_final`, assert the active card is `instance:check`, press
     `ctrl+j`, and assert `instance:commit`. Then drive one tick deterministically by
     awaiting `panel.final_view._live_tick_run()` directly (no sleeps; see the
     `check_test_wait_helpers` lint). Let the posted `FinalDeckLoaded` message settle
     through the page's existing wait helpers.
   - **Assert:** the view's active card, the panel's stored FINAL active card, and the
     area sticky are all still `instance:commit`.
   - **Must fail before the fix.** The scratch repro showed the revert to
     `instance:check`.
2. **FINAL worker completion keeps the `Ctrl+J` card.**
   - **Setup:** same fixture. After the `ctrl+j`, deliver a fake SUCCESS event to
     `final_view.on_worker_state_changed`. The event's `.worker` is the view's
     `_current_worker`, which the test sets to a stub whose `.result` is a
     `FinalDeckLoadResult` for the current subject and generation; build it via
     `load_final_deck` with the view's current targets, or reuse a cached result. Leave
     `_current_preferred` stale (`None` or `instance:check`).
   - **Assert:** the active card stays `instance:commit`.
3. **RECORD worker completion keeps the `Ctrl+J` card.** Use the same fake-event
   approach on `record_view.on_worker_state_changed` after cycling RECORD with `ctrl+j`.
   Use a pilot if the fixture agent offers a multi-card RECORD deck. If that is
   impractical, write a focused test showing the RECORD SUCCESS path resolves its
   preference through `host_preferred_card`.
4. **Tools Runs-loaded message does not clobber `llm-calls`** (headless, no pilot):
   - With `DeckPanel(0)`, `_deck = DeckId.TOOLS`, and stored Tools card `llm-calls`,
     call
     `handle_tool_runs_deck_loaded(ToolRunsDeckLoaded(<one-card runs document>, active_card="runs", ...))`.
     The stored card must stay `llm-calls`.
   - With nothing stored, the same message stores `runs`.
5. **Unit tests for the helpers:**
   - A headless `DeckPanel(0)` raises `LookupError` from `sticky_card_for`.
   - An unmounted `CardDocumentView` returns `fallback` from `host_preferred_card`.

## Verification

- Run the new test module and the neighbors most likely to be affected:
  - `tests/ace/tui/widgets/decks/test_final_deck_shell.py`
  - `tests/ace/tui/widgets/decks/test_record_deck_shell.py`
  - `tests/ace/tui/widgets/decks/test_record_deck_archive_rows.py`
  - `tests/ace/tui/test_tool_runs_live_blocks.py`
  - `tests/ace/tui/test_tool_runs_deck_cards_nav.py`
  - `tests/ace/tui/widgets/decks/test_deck_panels.py`
- Then run `sase tool run check`. The FINAL and Tools PNG goldens should not change:
  they set the sticky choice before showing the deck, so the paint-time value matches.
  Do not run `check-full` unless instructed.

## Out of scope: file these as task beads with `/sase_new_task`

1. **Tool-run reveal bypasses the sticky choice.** Reveal activates Runs through
   `_show_tools_card("runs")` (`src/sase/ace/tui/tool_runs/reveal.py`) without recording
   the sticky Tools card. If the sticky choice is `llm-calls`, the next refresh
   (`_refresh_tools_panel`) flips the deck back to LLM Calls. This is the same "switches
   back" family, in the other direction.
2. **`Ctrl+J` on FINAL or RECORD re-renders the hidden Main document.**
   `cycle_focused_deck_card` records the sticky choice through
   `DeckArea.set_preferred_card`, which also re-runs `show_main_document` on the
   not-shown Main document for every FINAL or RECORD `Ctrl+J`. That is wasted work on a
   keystroke path. The fix is to use `set_preferred_card_state` for non-Main decks.
   First check whether FINAL's block rail or navigability currently depends on that
   incidental Main re-show: the FINAL branch of `cycle_card` does not call
   `_sync_block_navigable()` / `_sync_block_rail()` itself.
