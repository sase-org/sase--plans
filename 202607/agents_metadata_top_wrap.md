---
tier: tale
title: Agents metadata navigation wraps through document top
goal: "Ctrl+J and Ctrl+K keep cycling through rendered Agents metadata sections, but
  crossing either section boundary visits the true top of the metadata document before
  navigation continues from the opposite end.

  "
create_time: 2026-09-09 19:52:57
status: wip
---

# Plan: Wrap Agents Metadata Navigation Through Document Top

## Context

The Agents tab's `Ctrl+J` and `Ctrl+K` actions currently navigate a cached, width-aware
list of rendered section-title anchors in `AgentPromptPanel`. A fresh document has no
active section: forward navigation selects the first title and reverse navigation
selects the last. Once a title is active, `resolve_section_target()` uses modular
arithmetic, so `Ctrl+J` on the final title immediately selects the first title and
`Ctrl+K` on the first immediately selects the final title. The mounted navigation test
locks in that direct section-to-section wrap.

The beginning of the metadata document is distinct from its first marked section.
Regular agent views can render unmarked header fields before `AGENT XPROMPT` /
`AGENT PROMPT`, and a true top jump must scroll `#agent-prompt-scroll` to its origin
rather than align the first section anchor. The requested behavior therefore needs an
explicit document-top navigation state, not a synthetic Rich heading or a row-zero
section anchor.

This remains presentation-only Textual behavior. No Rust core, key assignment,
configuration schema, rendered-section marker, or backend change is needed. The existing
TUI performance contract still applies: every keypress must resolve from the published
in-memory anchor cache, perform no I/O or document rebuild, and keep any after-refresh
retry thin and synchronous.

## Product Behavior

- Treat the metadata document as a circular sequence with an implicit top waypoint
  before all rendered section titles:
  `top -> first section -> ... -> last section -> top` for `Ctrl+J`, and the reverse
  order for `Ctrl+K`.
- Preserve initial behavior because a newly selected document starts at the top
  waypoint: its first `Ctrl+J` selects the first rendered section, while its first
  `Ctrl+K` selects the last.
- From the last active section, `Ctrl+J` clears the active section and scrolls to the
  true document origin. From the first active section, `Ctrl+K` does the same. A
  subsequent press continues around the cycle: forward from top selects the first
  section, and reverse from top selects the last.
- A top transition must set the metadata scroll container to vertical offset zero
  immediately and without animation. It must not substitute the first title's rendered
  row, which may occur below ordinary agent header fields or container padding.
- Keep one shared navigation cursor for both directions. Manual scrolling, `g` / `G`,
  same-document enrichment, live reply updates, and reflow do not reinterpret that
  cursor; document changes and pinned-attempt changes retain their existing reset
  behavior. The top waypoint survives a same-document rerender just as the current
  fresh/no-active state does.
- Documents with no rendered section markers remain a safe no-op rather than turning
  either shortcut into an alternate scroll-to-top command.
- Preserve all other behavior: every real section title can still align at the viewport
  top, the short-final-section layout reserve remains non-copyable, `G` still stops at
  the end of real metadata, invalidated layouts get at most one cached retry, and the
  actions remain Agents-only without stealing focused-widget or other-tab key handling.

## Implementation

### 1. Represent document top explicitly in section-target resolution

- Refine the `AgentPromptPanel` navigation result contract so callers can distinguish
  three valid outcomes: a rendered section anchor, the document-top waypoint, and a
  no-op for a ready document with no sections. Keep readiness for the brief
  generation/width invalidation window explicit as it is today; do not overload one
  `None` value with top, empty-document, and not-ready meanings.
- Continue using `_active_section_identity` for real titles and its no-active state for
  document top. When moving forward from the final cached anchor or backward from the
  first, transition to top instead of applying modulo directly to the opposite anchor.
  From top, retain the existing direction-dependent first/last selection. Only mutate
  navigation state after a ready, valid outcome has been resolved.
- Preserve stable semantic identities across same-document rerenders, the cheap-paint
  preservation path, width changes, disappearing sections, and document resets. Do not
  add a fake section marker, reserved identity, extra render pass, scroll-position
  inspection, or per-key filesystem work.

### 2. Scroll top and real-section targets through their correct paths

- Update `BasicNavigationMixin._cycle_agent_metadata_section()` to handle the explicit
  top outcome by scrolling `#agent-prompt-scroll` to vertical offset zero immediately,
  vertically only, and without animation. Continue using the existing virtual-region
  `scroll_to_region(..., top=True)` path for real title anchors so panel padding and
  nested offsets remain correct.
- Leave lazy layout-reserve enablement and the single `call_after_refresh` retry flow
  intact. A retried boundary key must resolve against the newly published anchors and
  may produce either top or a real section without scheduling additional work on
  Textual's serial message pump.
- Keep zero-section documents unchanged, and do not trigger the detail debouncer, a
  metadata refresh, agent selection, panel switching, or entry-jump history from either
  outcome.

### 3. Lock down the new navigation state machine

- Extend the prompt-panel resolver tests to cover forward and reverse boundary
  transitions as explicit top outcomes, then verify that another press selects the first
  or last real anchor respectively. Adjust the same-document reconciliation assertion
  that currently depends on last-to-first modular wrapping while retaining its coverage
  of identity preservation after anchors are inserted or reflowed.
- Update the mounted Agents navigation test to include unmarked preamble/header content
  before the first marked section, proving that the top waypoint is observably different
  from the first anchor. Exercise complete forward and reverse sequences, including
  `last -> top -> first` and `first -> top -> last`; at top assert scroll offset zero
  and no active section, and at each real section continue asserting exact
  title-to-viewport-top alignment.
- Retain coverage for the short final title and `G`'s real-content endpoint,
  zero-section no-op behavior, reflow, same-document state preservation, new-document
  reset, and the one-shot invalidation retry. Include a one-section case if the
  full-cycle test does not already prove that the sole title transitions to top in both
  directions.

### 4. Update user-facing navigation documentation

- Revise the Agents navigation table and metadata-panel explanation in `docs/ace.md` to
  describe wrapping through the top of the document rather than directly between the
  first and last titles. Document the follow-on behavior from top so the cycle is
  unambiguous.
- Update the Agents `?` help entry to make the top-wrap behavior discoverable while
  respecting the help modal's 32-character description limit, and update its
  exact-string test. The action names, command-palette metadata, configurable keymap
  fields, default `ctrl+j` / `ctrl+k` assignments, and tab-gating tests should remain
  unchanged unless focused verification exposes wording that falsely promises direct
  section-only wrapping.

## Verification

1. Run the focused prompt-panel navigation tests, including the mounted Textual pilot
   cases for forward/reverse cycles, reflow, zero-section content, short final sections,
   and invalidation retry.
2. Run the help/document-adjacent and Agents action-gating/keymap tests to confirm the
   wording change and that the intentional `Ctrl+K` overlap still dispatches only within
   its existing tab/focus contexts.
3. Run `just test-visual` and inspect any Agents help-modal snapshot difference; only
   the intentional help description should change visually.
4. Because implementation changes touch repository files, run `just install` first as
   required for an ephemeral workspace, then finish with `just check` before handoff.

## Risks and Guardrails

- Reusing a nullable anchor without an explicit outcome would make a top transition
  indistinguishable from a valid zero-section no-op or a not-yet-rendered cache, causing
  missed jumps or retry loops. Keep the result states distinct.
- Treating the first title's row as document top would hide unmarked header metadata and
  can be off by container padding; the mounted test must include pre-title content and
  assert the scroll origin directly.
- Deriving the next target from the live scroll offset would break the established
  shared semantic cursor after manual scrolls and asynchronous reflow. Continue deriving
  navigation solely from cached anchors and the active identity.
- Do not remove or fold the trailing layout reserve into the Rich document. It is still
  required for exact alignment of the last short section and must remain excluded from
  ordinary bottom scrolling and copied content.
- Avoid new async work, rendering, or I/O in the key action or after-refresh callback.
  The change should remain a small in-memory state transition plus one immediate scroll
  operation.
