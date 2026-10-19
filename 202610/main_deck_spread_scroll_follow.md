---
tier: tale
size: small
title: Stop the Main deck spread from yanking a parked reader to the newest Reply block
goal:
  A reader who scrolls away from the newest Reply block in a deck-spread Main panel
  keeps their scroll position across same-agent refreshes, while a reader still at the
  follow landing keeps following new turns.
decisions:
  bottom_reader:
    ask:
      Should a reader parked at the real bottom of a tall newest Reply block still
      auto-land on newly arrived turns?
    choices:
      land_new_turns:
        Lands on each newly arrived turn, but stays put on in-place refreshes (chat-log
        behavior)
      stay_put: Never moves; new turns only announce through block-rail arrival dots
    default: land_new_turns
    why:
      Keeps today's new-turn landing at the bottom; only the per-refresh yank to the
      header goes
    answer: land_new_turns
proposed_by: bbugyi200.athena.11w
proposed_with: claude/opus
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.11w](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.11w.md)
- **COMMITS:**
  - [45b0f45](https://github.com/sase-org/sase/commit/45b0f45bd66d80215c267468d54a4575795003d5)
    — fix(ace): preserve parked Main deck scroll positions

# Stop the Main deck spread from yanking a parked reader to the newest Reply block

## Problem

In the Agents tab, a Main deck panel in **spread** mode (`MAIN spread · auto`) keeps
scrolling down by itself while the user reads near the top. Example: agent `11u` (a
stopped TALE waiting on its plan gate). The user scrolls to the top to read the Context
card, and within one refresh tick the panel jumps back down to the newest Reply block
(the `GATE TURN`). This repeats on every tick, so the top of the deck can't be read.

## Root cause (diagnosed and reproduced)

1. **Every refresh re-delivers a changed Main document.** The gate turn block renders
   `Deadline: … (23h54m42s left)` (`_deadline_text()` in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_gate_section.py`). The countdown text
   changes on every re-render, so the prompt panel's content digest changes. Its Main
   sink (`PromptPanel.update()` → `AgentDetail._on_main_document()` →
   `_deliver_main_document()`) then calls `DeckPanel.show_main_document()` with a
   same-subject document. Any live agent with growing output does the same. The detail
   refresh paths in `_agent_detail_state.py` and `_agent_detail_deck_show.py` also
   re-push the document.
2. **Same-subject spread refreshes re-land any reader whose block cursor says
   `following`.** In `show_main_document()`
   (`src/sase/ace/tui/widgets/decks/panel_navigation.py`, the `# Same-subject streaming`
   branch), when the deck is spread and the Reply card's `BlockCursor.following` is
   true, it calls `_land_deck_spread_on_blocks()`. That scrolls to
   `min(newest block header, real bottom)`. The comment's promise, "a parked reader
   stays put", relies on the cursor's `following` flag being accurate.
3. **The `following` flag is never cleared for most "parked" positions.** The cursor
   becomes `following=True` on the first landing, from `reconcile_cursor()` /
   `land_cursor()` in `block_model.py`. The only thing that clears it is the scroll
   watcher (`_on_main_scroll_y` → `sync_document_spread_block_cursor` →
   `CardDocumentViewBlocksMixin.sync_spread_cursor_from_scroll`), and only when
   scrolling derives a _different, older_ block. These positions all leave it `True`:
   - Scrolling above the first Reply block header (the Context card, the top of the
     document, or the Reply card intro). There `derive_spread_block()` returns `None`,
     which is treated as "no change". This is the user's case.
   - `Ctrl+K` to Context (`cycle_card` → `scroll_to_card`). `_on_main_scroll_y` returns
     early while `_spread_pending_card` is set, so it never syncs the cursor.
   - Scrolling within the newest block, or to the real bottom below a tall newest
     block's header. The derived block is still the newest, so nothing changes, and
     every refresh yanks the reader back up to the header.

The screenshot's header shows `Context` as the active card (`1/2`), so a stuck
`_spread_pending_card` is not involved. The cause is the stale `following` cursor alone.

**Reproduction.** These are Textual pilot scenarios using the
`test_deck_block_spread_pilot.py` helpers, with spread pinned via
`AgentDecksSettings(spread_max_screens=10, block_spread_max_screens=1000)`, a 12-line
Context card, and a 3-block Reply. After landing, scroll to `y=0` and call
`show_main_document()` again with the same subject and blocks but a new digest. Today
the scroll jumps `0 → 25` (the landing target), and the cursor still reports
`following=True` after the user scrolled to the top.

**Prototype.** The design below was prototyped against a scratch copy of `src/`. Five
pilot scenarios that fail on `master` all passed, and the full `tests/ace/tui` suite
passed with it (18,110 tests). Those scenarios are: scroll-to-top, parked arrival dot,
`Ctrl+K` to Context, mid-newest-block, and re-follow after returning.

## Design

Whether a reader "follows" in a deck-spread render should come from **where the reader
actually is**. A sticky flag that most user scrolls never clear is not reliable. Before
a same-subject spread refresh applies new content, classify the reader against the
**currently shown (pre-refresh) layout**. This must happen before the new content is
applied, because `SectionViewMixin._apply_section_content()` clears the published
anchors (`_section_anchors = ()`).

- `LANDED`: `round(scroll_y) == spread_landing_target(card)`, the exact
  `min(newest header, real bottom)` spot a landing puts a follower. **Re-land.** This
  keeps the current behavior for real followers. When content above shifts, or a short
  newest block grows, the landing re-applies correctly, because the classification uses
  the old layout, which matches the reader's old scroll position.
- `BOTTOM`: `round(scroll_y) >= real_bottom` but not `LANDED`. This is the tail of a
  tall newest block whose header is above the real bottom. **Re-land only if the newest
  block id changed** (a new turn arrived). Otherwise leave the scroll alone. See
  decision `bottom_reader`: with `stay_put`, `BOTTOM` never re-lands, behaving like
  `PARKED` for scrolling, but the cursor is not parked.
- `PARKED`: anywhere else (above the newest header, mid-block, or the Context card),
  plus any in-flight explicit card navigation (`view._spread_pending_card is not None`).
  **Never scroll.** Park the cursor (`following=False`, same `block_id` and `known_ids`)
  **before** the refresh reconciles cursors. A block that arrives in this refresh then
  shows as an arrival dot instead of being marked seen.
- `None` (unmeasurable): anchors or real bottom not published yet, or a block landing or
  jump in flight (`view._block_spread_pending is not None`). Fall back to today's rule:
  re-land iff `cursor.following`.

A reader who returns to the landing spot is `LANDED` on the next refresh, and re-landing
there re-follows them (`land_block_spread` → `land_block_cursor` sets `following=True`).
Existing re-follow paths still apply: the scroll watcher selecting the newest block from
an older one, and `(`/`)` stepping onto the newest block. The bottom pin
(`is_pinned_to_bottom`) keeps its existing early skip.

This is presentation-only TUI scroll state (Textual widget and scroll positions), so it
stays in this repo. Nothing crosses the `sase_core` Rust boundary.

## Implementation

### 1. Pure helpers: `src/sase/ace/tui/widgets/decks/block_model.py`

Add these next to the other cursor helpers, export them in `__all__`, and keep the
module pure (no Textual):

```python
from dataclasses import dataclass, replace
from enum import StrEnum

def park_cursor(cursor: BlockCursor) -> BlockCursor:
    """Return ``cursor`` without following (block and known ids kept)."""
    if not cursor.following:
        return cursor
    return replace(cursor, following=False)

class SpreadFollowState(StrEnum):
    """Where a deck-spread reader sits relative to the follow landing."""
    LANDED = "landed"
    BOTTOM = "bottom"
    PARKED = "parked"

def classify_spread_follow(
    *, scroll_y: float, landing_target: int | None, real_bottom: int | None
) -> SpreadFollowState | None:
    if landing_target is None or real_bottom is None:
        return None
    row = int(round(scroll_y))
    if row == int(landing_target):
        return SpreadFollowState.LANDED
    if row >= int(real_bottom):
        return SpreadFollowState.BOTTOM
    return SpreadFollowState.PARKED

def spread_follow_relands(
    state: SpreadFollowState | None, *, following: bool, newest_changed: bool
) -> bool:
    if state is None:
        return following
    if state is SpreadFollowState.LANDED:
        return True
    if state is SpreadFollowState.BOTTOM:
        return newest_changed  # decision bottom_reader=stay_put: return False
    return False
```

Give each helper a docstring that matches the module's style and states the semantics
above.

### 2. View helpers: `src/sase/ace/tui/widgets/decks/main_view_blocks.py`

On `CardDocumentViewBlocksMixin`, import the new names and add:

- `spread_follow_state(self, card) -> SpreadFollowState | None`:
  - return `PARKED` if `getattr(self, "_spread_pending_card", None) is not None`. That
    attribute lives on `CardDocumentView`, so use `getattr`.
  - return `None` if `self._block_spread_pending is not None`, or if the parent is not a
    `VerticalScroll`.
  - otherwise return
    `classify_spread_follow(scroll_y=float(scroll.scroll_y), landing_target=self.spread_landing_target(card), real_bottom=self._real_bottom())`.
  - wrap it in `try/except` → `None`, like its neighbors.
  - its docstring must say it has to run before new content is applied.
- `park_block_cursor(self, card_id) -> BlockCursor | None`: replace the stored cursor
  with `park_cursor(cursor)` and return it. Return `None` when there is no cursor.

### 3. Capture helper: `src/sase/ace/tui/widgets/decks/panel_blocks.py`

On `DeckPanelBlocksMixin`, next to `_spread_block_card` / `_land_deck_spread_on_blocks`,
add the Main-only wrapper below. Put it here, not in `panel_navigation.py`, which is
already 642 lines; the `toobig` gate's first threshold is 700.

```python
def _capture_main_spread_follow(
    self, previous_document: Any, preferred_card: str | None
) -> tuple[SpreadFollowState | None, str | None]:
    """Classify the reader before a same-subject spread refresh.

    Runs before the new content lands (which clears the published anchors).
    A parked reader's cursor stops following here, before the refresh
    reconciles cursors, so a newly arrived block is announced instead of
    marked seen.
    """
    try:
        if bool(getattr(self.main_view, "is_pinned_to_bottom", False)):
            return (None, None)
        if previous_document is None or previous_document.partial:
            return (None, None)
        card = self._spread_block_card(previous_document, preferred_card)
        if card is None:
            return (None, None)
        state = self.main_view.spread_follow_state(card)
        if state is SpreadFollowState.PARKED:
            self.main_view.park_block_cursor(card.card_id)
        return (state, card.newest_block_id)
    except Exception:
        return (None, None)
```

### 4. Decision site: `src/sase/ace/tui/widgets/decks/panel_navigation.py`

In `show_main_document()`, in the `else:` branch (`new_mode is old_mode`), make two
changes:

- **Before** the `try:` that calls `_show_main_paged` / `self.main_view.show_document`,
  capture the reader:

  ```python
  follow_state = None
  previous_newest = None
  if new_mode is RenderMode.SPREAD and not is_new_subject:
      follow_state, previous_newest = self._capture_main_spread_follow(
          previous_document, preferred_card
      )
  ```

- In the `# Same-subject streaming` branch, replace the
  `if cursor is not None and bool(getattr(cursor, "following", False)):` gate with:

  ```python
  following = cursor is not None and bool(getattr(cursor, "following", False))
  newest_changed = (
      previous_newest is not None and card.newest_block_id != previous_newest
  )
  if spread_follow_relands(
      follow_state, following=following, newest_changed=newest_changed
  ):
  ```

  The body (the `_land_deck_spread_on_blocks(document, card.card_id)` call) stays as it
  is.

Import `spread_follow_relands` from `.block_model`; also import `SpreadFollowState` in
`panel_blocks.py`. Update the branch comment to say the reader's pre-refresh position
decides, not the cursor flag alone. Leave the new-subject branch and the mode-transition
paths (`document_transitions_apply.py`) unchanged.

### 5. Tests

**Pure tests** in `tests/ace/tui/widgets/decks/test_block_model.py`:

- `park_cursor` clears `following`, keeps `block_id` and `known_ids`, and returns a
  parked cursor unchanged.
- `classify_spread_follow`: `None` inputs → `None`; exact target → `LANDED` (including
  target == bottom); `>= bottom` → `BOTTOM`; above target → `PARKED`; mid-block
  (`target < y < bottom`) → `PARKED`; fractional `scroll_y` rounds.
- `spread_follow_relands`: the full truth table, including `None` → `following`.

**Pilot regressions** in `tests/ace/tui/widgets/decks/test_deck_block_spread_pilot.py`.
That file is 319 lines; if it nears 700, put these in a new sibling module such as
`test_deck_spread_follow_pilot.py` that reuses its helpers. Use
`_pin(app, deck=10, blocks=1000)` with a multi-line Context card, for example
`context_card(Text("\n".join(f"context-{i}" for i in range(12))))`, so the top of the
document is above the first block header. Land first with
`show_main_document(..., "reply")` plus
`wait_for_anchor_scroll(... view.spread_landing_target(card))`. For the "refresh",
re-call `panel.show_main_document()` with the same subject, a **new digest**, and
`preferred_card=None`, then let a few `pilot.pause()` calls settle. Each scenario below
fails on today's code; the last two are controls and must keep passing:

1. Scroll to `y=0`, then refresh with the same 3 blocks: `scroll_y` stays `0` and the
   Reply cursor's `following` is `False`.
2. Scroll to `y=0`, then refresh with 4 blocks: `scroll_y` stays `0` and
   `view.arrived_block_ids("reply") == ("b3",)`.
3. `panel.cycle_card(-1) == "context"` (lands at `0`), then refresh: `scroll_y` stays
   `0`.
4. Use 40-line blocks so the newest header (≈99) is above the real bottom (≈115). Scroll
   to `landing + 5`, then refresh with the same blocks: `scroll_y` stays `landing + 5`.
5. Same tall document, scroll to `view._real_bottom()`. A refresh with the same blocks
   keeps `scroll_y` at the bottom. A refresh with a 4th block lands on `b3`
   (`wait_for_anchor_scroll` to `spread_landing_target`) under the default
   `land_new_turns`; under `stay_put`, assert it stays at the bottom with an arrival dot
   instead.
6. Re-follow: park at `0`, refresh (parked), scroll back to the original landing row,
   then refresh with a 4th block: it lands on `b3`.
7. Controls that must keep passing: a never-scrolled follower refreshed with a 4th block
   lands on `b3` with `following=True`; the existing
   `test_scroll_derived_cursor_and_streaming_stays` still passes.

Use `wait_for` / `wait_for_anchor_scroll` instead of fixed sleeps. The test-waits lint
gate checks this.

## Verification

- `sase tool run check`. This runs every lint gate except `toobig`: ruff, mypy,
  symvision (the new public helpers must all be used), test-waits, and the rest, plus
  the diff-scoped tests. Do not run `check-full`.
- Also run the focused suites directly:
  `pytest tests/ace/tui/widgets/decks tests/ace/tui/test_tool_runs_live_blocks.py tests/ace/tui/test_main_deck_reply_steal_recovery.py tests/ace/tui/test_agent_deck_persistence.py`.
- Manual check, optional, in a fresh `sase ace` (an already-running TUI keeps the code
  it started with):
  1. Select a stopped agent with a pending gate (live deadline countdown) whose Main
     deck shows `spread`.
  2. Scroll to the top; it must stay there across several refresh ticks.
  3. Scroll back to the newest block's landing; a new turn must land again.

## Out of scope

- Same-subject **mode transitions** into spread (`_apply_document_transition` and
  `_refresh_document_mode_for_shown` in `document_transitions_apply.py`) still use the
  sticky landing. They only fire when auto mode crosses its hysteresis band, not on
  every refresh.
- The FINAL deck's own live-tail follow (`decks/final/live.py`).
- Making the gate deadline countdown stop changing the document digest. The Main deck
  must tolerate live content anyway; running agents stream constantly.
- No memory, docs, or glossary changes are needed. The glossary's "landing on the newest
  block" still describes the first landing.
