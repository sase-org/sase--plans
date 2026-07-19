---
tier: tale
title: Reveal neighbor jump target containers
goal: 'Jumping to an Agents-tab neighbor with ~ always reveals and focuses the exact
  target inside its expanded tribe panel and clan ancestry without disturbing unrelated
  folds or the existing fast neighbor lookup.

  '
create_time: 2026-07-18 21:09:20
status: wip
prompt: 202607/prompts/neighbor_jump_container_expansion.md
---

# Plan: Reveal neighbor jump target containers

## Context

Agents-tab `~` navigation already finds visible ancestors, descendants, and hood neighbors through a cached render-order
index. Its final jump path, however, assigns the stored global row and panel indices directly and then performs a
selective highlight refresh. A target in another collapsed tribe panel can therefore become the logical selection while
its panel remains collapsed. The same index-based handoff is also brittle if a modal selection outlives a structural
projection change.

The numbered clan/family member jump has the stronger reveal semantics needed here: it resolves a stable agent identity
against the complete in-memory projection, expands only the target's tree ancestors, rebuilds only when a structural
fold changes, expands the target's enclosing grouping banners and tribe panel through the existing persistence-aware
helpers, and then resolves and selects the target again. Neighbor navigation should use the same reveal contract while
preserving its own unread acknowledgement, departure arming, back-jump anchor, detail refresh, and modal behavior.

## Target behavior and boundaries

- Both the single-result fast jump and a choice made from the neighbor modal identify the target by stable agent
  identity before any fold mutation.
- Reveal only the structural path containing the chosen target: its outer clan ancestor chain, enclosing grouping
  banners, and tribe panel. Do not expand a target agent's own workflow/family descendants, sibling clans, other groups,
  or other tribe panels.
- Persist automatic tribe-panel and grouping-banner expansions through the same fold-intent hooks as explicit and
  numbered-member expansion. Automatic expansion remains in effect after an apostrophe back-jump; history restores
  focus, not prior fold state.
- Keep neighbor discovery semantics unchanged. Agents hidden when the cached visible-neighbor index is built do not
  become new choices merely because the reveal path can expand containers. Dismissed-descendant revival remains a
  separate path.
- Preserve the artifact-viewer guard and fail safely if a modal target becomes stale or filtered out: never jump to
  whichever agent later occupies the old numeric index.
- Preserve the keystroke hot path. A neighbor whose ancestors, group, and panel are already expanded continues to use
  the cached lookup and selective highlight/detail updates; a full list rebuild occurs only when revealing the target
  changes structural visibility or panel layout.

## Implementation

### Share the target-reveal contract

Refactor the bounded, in-memory target reveal operations currently used by numbered member navigation into a reusable
navigation helper. Keep stable identity resolution, ancestor-cycle/missing-parent guards, enclosing-group enumeration,
panel lookup, visibility verification, and persistence callbacks in one path so member and neighbor jumps cannot drift.
Have the helper return enough structured state for each caller to choose the appropriate refresh strategy without
coupling their acknowledgement or detail-panel policies.

Retain the existing cheap path when no tree fold changes: avoid disk access, subprocesses, async work, or an
unconditional `_refilter_agents()` call from `~`. When a clan ancestor must be expanded, invalidate the relevant cached
panel/index state, refilter from `_agents_with_children` with content-index refresh disabled, and re-resolve the
identity before touching selection.

### Route neighbor selection through reveal-before-focus

Update the neighbor action payload and direct-jump handoff in `src/sase/ace/tui/actions/agents/_neighbors.py` to carry
the target identity alongside any display-time index. Before changing focus, invoke the shared reveal helper, expand a
collapsed destination tribe panel, and resolve the destination panel/index from current state. Save one back-jump anchor
before the first structural or focus mutation, then retain the existing departure arming and target unread
acknowledgement behavior.

Teach the neighbor refresh path to distinguish a pure row/panel focus change from a structural reveal. Pure focus
changes should keep selective panel, highlight, info, immediate-detail, and debounced-detail updates. A clan, grouping,
or tribe expansion should rebuild the list once, preserve focus by stable panel key and agent identity across any panel
reorder, and then update detail state without redundant rebuilds.

### Preserve numbered-member behavior

Move numbered-member navigation onto the shared reveal result without changing its digit buffering, roster validation,
notifications, fold levels for hidden steps, or full-refresh decisions. This regression check is important because the
member path is the reference implementation for expanding clan ancestry, grouping banners, and tribe panels.

## Verification

Extend `tests/ace/tui/test_agent_neighbor_navigation.py` with production-shaped fold and panel state covering:

- a direct single-neighbor jump into a collapsed tribe panel, including panel reorder/focus, persisted expansion intent,
  exact target selection, unread and departure behavior, and one structural refresh;
- modal selection into a collapsed destination panel and stable-identity handling when numeric indices become stale;
- a target nested beneath a clan/family ancestor chain, proving only its containing ancestors are expanded and unrelated
  clan/workflow folds remain unchanged;
- an already-visible same-panel neighbor, proving no refilter/full rebuild is introduced and existing selective detail
  refreshes remain intact;
- guarded, filtered-out, or otherwise stale targets, proving no fold mutation, wrong-row selection, acknowledgement, or
  jump-history corruption occurs;
- apostrophe back-jump compatibility after an automatic expansion, with focus restored while the revealed container
  remains expanded.

Keep the numbered-member tests in `tests/ace/tui/test_member_jump_navigation.py` green and add focused assertions for
the shared helper where extraction changes observable boundaries. Add an Agents interaction PNG scenario in
`tests/ace/tui/visual/test_ace_png_snapshots_agents_interactions.py` that collapses a target tribe panel, invokes `~`,
and captures the expanded panel with the exact neighbor highlighted; update only that intentional golden.

Run the focused neighbor, member-jump, fold-persistence, and interaction visual tests first. Then run `just install`
followed by the repository-required `just check`, and run `just test-visual` for the complete ACE PNG suite. Inspect any
changed visual artifact before accepting a golden and report unrelated repository-baseline failures separately rather
than widening this change.
