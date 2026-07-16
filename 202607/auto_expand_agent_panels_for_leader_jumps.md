---
tier: tale
title: Auto-expand agent panels for leader jumps
goal: 'The Agents-tab `,j` and `,J` commands find eligible rows inside collapsed whole
  panels, expand the destination panel, and focus the exact row while preserving recency,
  unread, fold-persistence, navigation-history, and TUI performance behavior.

  '
create_time: 2026-07-16 16:39:14
status: wip
prompt: 202607/prompts/auto_expand_agent_panels_for_leader_jumps.md
---

# Plan: Auto-expand collapsed agent panels for `,j` and `,J`

## Product context

The split Agents tab can collapse a whole tag panel into a title-only strip. The leader commands `,j` (next unread
completed agent) and `,J` (next stopped agent) already search across expanded panels by event recency, move focus to the
target panel and row, and participate in the Agents-tab apostrophe back/forward history. Their shared candidate
discovery currently calls `_visible_agent_panel_indices()`, which deliberately excludes every row in a collapsed whole
panel. This leaves the leader commands available when matching agents exist in `_agents`, but they can report “No unread
completed agents” or “No stopped agents” merely because the matching panel is collapsed.

Teach both commands to treat a whole-panel collapse as a navigable presentation state: if the next eligible row is in
such a panel, expand that panel and land on the exact row. This is Python/Textual navigation and fold-state behavior; it
does not introduce shared backend semantics, so the Rust core does not need to change.

This is a single cohesive frontend change, so the plan is a `tale` rather than an `epic`.

## Target behavior

- `,j` and `,J` consider eligible rows in expanded panels and in collapsed whole panels. Candidate sorting, timestamp
  fallbacks, wrap behavior, leader-repeat behavior, and status predicates remain unchanged.
- Whole-panel collapse is the only visibility barrier these commands bypass. Rows suppressed by an in-panel group
  banner, workflow/render filtering, or the existing STARTING-row exclusion remain ineligible until those independent
  conditions make the row visible. Apostrophe jump hints also retain their current contract: a collapsed whole panel
  contributes one header target, not hidden row targets.
- When the chosen row belongs to a collapsed panel, the command removes that panel from `_collapsed_panel_keys`, focuses
  it by stable `PanelKey`, clears banner and attempt focus, selects the exact target row, and performs the one
  structural panel rebuild required to render and reorder the newly expanded panel. Existing in-panel fold registries
  are preserved.
- Auto-expansion is a real fold-state change, not a transient peek. Record it through the established panel-fold
  persistence hook so it survives refreshes and sessions and wins correctly if the asynchronous persisted-state load is
  still in flight. The keystroke handler itself performs no disk I/O.
- `,j` retains its acknowledgment contract after expansion: clear the target’s unread completion state unless manually
  guarded, dismiss the matching completion notification, refresh the notification count, and render the expanded row and
  panel counts in their new state. A manually guarded unread row still causes its panel to expand and receive focus
  without being acknowledged.
- `,J` retains its non-acknowledging contract. Expanding and selecting a stopped row must not change unread sets,
  manual-unread guards, or notification state and must not issue an opportunistic row patch before the structural
  rebuild.
- A collapsed panel header is not an agent-row selection even though it carries a backing `current_idx` for detail
  context. Starting `,j` or `,J` from a collapsed header begins at the newest eligible candidate instead of accidentally
  skipping the backing row as though it were currently selected.
- Existing apostrophe history stays correct when expansion moves a panel from the collapsed partition to the expanded
  partition. Row and banner anchors resolve their panels by stable key, so back/forward restores the intended panel even
  if its rendered index changed. Restoring history never implicitly re-collapses an auto-expanded panel; a header anchor
  that ceased to exist because that same panel expanded remains stale under the established validation rule.
- If no eligible row exists across expanded and collapsed whole panels, preserve the current no-match notification and
  leave focus, fold state, persistence, and history untouched. Merged/single-panel layouts and ordinary `j`/`k`,
  `J`/`K`, `H`/`L`, and apostrophe behavior remain unchanged.

## Design

### 1. Discover jumpable rows without weakening normal visibility

Extend the cross-panel navigation-order seam in `src/sase/ace/tui/actions/agents/_navigation_order.py` with an explicit
way to enumerate rows that would render if a whole panel were expanded. Reuse the existing per-panel index, fold
registry, grouping mode, and `build_agent_tree()` walk so the result continues to exclude hidden group members, filtered
rows, and non-rendered agents. Keep `_visible_agent_panel_indices()` and its default semantics intact for apostrophe
allocation and other consumers; the new behavior should be opt-in from the leader-jump path only.

Update `_jump_to_next_matching_agent_by_time()` in `src/sase/ace/tui/actions/agents/_unread.py` to use that opt-in
candidate view and carry the destination panel’s stable key through selection. Detect whether the current focus
represents a real visible agent row before using `current_idx` to advance within the sorted candidates; collapsed panel
headers, like group banners, start from the newest match.

### 2. Make panel expansion part of one selection transaction

Factor or extend the whole-panel fold helper in `src/sase/ace/tui/actions/agents/_folding.py` so explicit `L` expansion
and leader auto-expansion share the state mutation, cache invalidation, and persistence hook without forcing the helper
to select the panel’s first row or refresh immediately. The leader jump can then:

1. capture the logical origin for history;
2. expand the target key if necessary and journal `collapsed=False`;
3. focus that key, select the exact candidate, and clear group/attempt focus; and
4. apply acknowledgment effects before issuing one list-changed display refresh.

Distinguish a structural `panel_expanded` result from an ordinary cross-panel focus change. Expansion requires
`_refresh_agents_display(list_changed=True, defer_detail=True)` so the stable expanded/collapsed partition, widget
contents, panel heights, title metrics, highlight, and detail context agree. Existing jumps between already-expanded
panels should retain their selective highlight/row-patch path. Avoid a speculative patch against the title-only widget
and avoid adding a second full rebuild in the leader dispatcher.

For `,j`, separate the unread/notification state mutation from its usual visible-row patch so the expanded panel is
rebuilt exactly once with the post-acknowledgment state. For `,J`, rebuild without calling any unread acknowledgment
helper.

### 3. Keep jump history stable across panel repartitioning

Upgrade Agents-tab agent and banner history anchors in `src/sase/ace/tui/actions/navigation/jump_hints.py` and
`src/sase/ace/tui/actions/navigation/_entry_jump_agents.py` to scope themselves by `PanelKey` rather than rendered panel
index. Panel-header anchors already use this stable identity. Capture the focused key, resolve it to the current index
only when validating or restoring, and verify that an agent anchor’s row still belongs to its recorded panel and that a
banner remains selectable in that panel.

This protects the complete back and forward stacks—not only the newest origin—from the deterministic index changes
caused by moving an expanded destination ahead of collapsed panels. Transient apostrophe banner targets may continue to
use the render-time panel index for hint routing; only durable history needs stable keys. Update affected neighbor,
panel-switch, apostrophe, unread, and stopped navigation expectations to use keyed anchors.

### 4. Preserve the TUI fast paths and fold lifecycle

Keep candidate discovery and fold mutation entirely in memory. Reuse the established non-blocking fold-state recorder,
navigation caches, list-changed refresh, detail debouncer, and leader footer/notification flow. Do not add widget
queries to candidate discovery, synchronous persistence, a new refresh pipeline, or Rust-side behavior.

An auto-expansion is one of the rare cases where a full Agents-list rebuild is necessary because panel order and
title-only/full contents both change. Record and test that structural refresh explicitly; all no-match and
already-expanded paths must remain on their current zero-rebuild or selective-update behavior, in line with the audited
TUI performance guidance.

## Testing strategy

Extend the focused helpers and suites in `tests/ace/tui/_agent_unread_navigation_helpers.py`,
`tests/ace/tui/test_agent_unread_done_navigation.py`, `tests/ace/tui/test_agent_stopped_navigation.py`, and
`tests/ace/tui/test_agent_panel_collapse.py` to cover:

- candidate discovery across one and multiple collapsed whole panels while rows hidden by in-panel folds remain
  excluded;
- `,j` choosing the newest collapsed-panel candidate, expanding and reordering the correct stable key, focusing the
  exact row rather than the first row, persisting the expansion, acknowledging it once, and performing one structural
  rebuild;
- the manually-unread `,j` case, which still expands and focuses but preserves the guard and unread state;
- `,J` expanding a stopped target while preserving unread and notification state, including PLAN/QUESTION-specific
  timestamps and repeat/wrap behavior;
- all matches being inside collapsed panels, ensuring neither command emits a false no-match notification;
- invocation from a collapsed panel header, proving its backing index is not treated as the current agent row;
- no-match, merged/single-panel, already-expanded, and collapsed in-panel-group regressions with no unintended fold
  mutation or full rebuild; and
- fold-persistence journaling when auto-expansion races the initial asynchronous state load.

Update the Agents jump-history suites, primarily `tests/ace/tui/test_jump_hints_for_folded_banners.py`,
`tests/ace/tui/test_agent_panel_first_selection.py`, and `tests/ace/tui/test_agent_neighbor_navigation.py`, for
stable-key agent/banner anchors. Add a regression in which expanding the destination changes several panel indices, then
verify apostrophe back and forward restore the original keyed row or banner. Also verify that a same-panel header anchor
becomes stale after expansion without re-collapsing the panel.

Add a visual interaction state in `tests/ace/tui/visual/test_ace_png_snapshots_agents_interactions.py` with an eligible
row inside a collapsed panel. Drive the actual leader key sequence, assert the collapsed key is removed, the panel moves
into the expanded partition, and the target row owns the highlight; for `,j`, assert the unread title/row styling is
cleared. Capture and inspect a dedicated PNG golden for the resulting expanded layout while preserving the existing
resting and apostrophe-hint collapse goldens.

## Validation and hand-off

The implementing agent must run `just install` first in its ephemeral workspace, then iterate with the focused
navigation, folding, persistence, and history suites. Run `just test-visual` and inspect `.pytest_cache/sase-visual/`
artifacts before accepting the intentional new PNG. Finish with `just check`, including static types, Symvision, the
complete unit suite, and visual snapshots. Do not edit memory files or generated provider instruction shims.
