---
tier: tale
title: Sort collapsed Agents-tab panels last
goal: 'Whole agent panels that are collapsed are rendered after every expanded panel,
  while panel focus, deterministic ordering, persistence, navigation, and fast refresh
  behavior remain correct.

  '
create_time: 2026-07-16 10:57:46
status: done
prompt: 202607/prompts/collapsed_agent_panels_last.md
---

# Plan: Sort Collapsed Agents-Tab Panels Last

## Context and tier choice

The Agents tab splits agents into an optional untagged panel followed by case-insensitively sorted tag panels. Whole
panels can be collapsed independently of the group banners inside a panel, and that collapse state persists across
sessions. Today collapse changes only a panel's rendering and height; it does not change the ordered
`AgentPanelGroup.panel_keys`, so a collapsed panel such as `#chop` can remain between expanded panels.

This is a `tale`: the behavior is a focused Python/Textual presentation change that one coding agent can implement and
verify cohesively. It does not require a Rust core wire/API change because panel layout, focus, navigation, and
rendering are frontend-only concerns.

## Product behavior

- In split-panel mode, render every expanded panel before every collapsed panel.
- Preserve the existing canonical order within both partitions: the untagged panel precedes tag panels in the base
  order, and tag panels remain case-insensitively alphabetical. If the untagged panel is collapsed, the "collapsed
  panels last" rule takes precedence and moves it below all expanded tag panels.
- When multiple panels are collapsed, order them by that same canonical base order rather than by collapse time. This
  keeps layout deterministic across refreshes and persisted-state restoration.
- Keep focus attached to the same panel key when collapsing, expanding, refreshing, or restoring state even though its
  numeric index changes. A newly collapsed focused panel should remain focused at its new bottom position; expanding it
  should return it to its canonical expanded position without changing the selected panel identity.
- Make keyboard panel traversal, jump targets, neighbor/unread navigation, widget positions, separators, height
  allocation, and visual order all consume the same reordered panel-key sequence.
- Leave in-panel group/banner folds and their ordering unchanged. Merged-panel mode still renders its single panel and
  continues to disallow whole-panel collapse.

## Design and implementation

1. Make collapsed-last ordering part of the panel collection contract in `src/sase/ace/tui/models/agent_panels.py`.
   - Extend panel-group construction (or a small model-level ordering helper used by it) to accept the currently
     collapsed panel keys.
   - First derive the existing canonical panel list, then apply a stable partition into expanded and collapsed keys. Do
     not sort by mutable set iteration order and do not rescan agents separately for each panel.
   - Resolve `focused_idx` only after applying the partition so `focused_key` remains the source of truth.
   - Update the module/class documentation to distinguish canonical base order from rendered collapsed-last order.

2. Feed current collapse state into every place that reconstructs or predicts rendered panel order.
   - In `src/sase/ace/tui/actions/agents/_display_panel_refresh.py`, have `_sync_panel_group()` rebuild with the active
     collapsed-key set while retaining its existing stale-key pruning and focused-key preservation.
   - Ensure both explicit collapse/expand full refreshes and fold-state restoration reach this same synchronization
     path; avoid a second ad hoc ordering mutation in the action handlers.
   - In `src/sase/ace/tui/actions/agents/_display_diff.py` and its callers in `_display.py`, make incremental-refresh
     panel-order comparisons use the same collapse-aware ordering contract. Otherwise a valid collapsed-last layout
     would look different from the alphabetical-only prediction and force unnecessary full list rebuilds on every agent
     refresh.
   - Keep `AgentPanelIndex` and per-agent panel membership keyed by panel key; collapse changes presentation order, not
     membership, so it should not create a new indexing or disk-I/O path.

3. Preserve focus and navigation correctness across the index move.
   - Verify the existing collapse flow invalidates panel-derived navigation caches before the full refresh and that
     `_sync_panel_group()` relocates `focused_idx` by the preserved key.
   - Verify expand computes its first selectable row for the still-focused panel and then returns that panel to its
     canonical expanded position on refresh.
   - Keep jump-hint panel indices and Textual widget slot IDs derived from the final ordered `panel_keys`, so rendered
     order and all index-based consumers cannot diverge.

4. Update regression and visual coverage.
   - Extend `tests/ace/tui/models/test_agent_panels.py` with stable-partition cases covering one collapsed middle panel,
     multiple collapsed panels, a collapsed untagged panel, focus preservation after reordering, and merged mode.
   - Extend `tests/ace/tui/test_agent_panel_collapse.py` (and the smallest relevant display-diff test) to prove collapse
     moves the focused panel to the last slot, expansion restores canonical order, navigation follows the reordered
     sequence, persisted/restored collapse state sorts on refresh, and ordinary agent updates can still use the
     incremental refresh path.
   - Adjust panel sizing/display tests whose setup assumes a collapsed panel remains in its old slot, asserting instead
     that the collapsed widget is the final rendered slot while expanded panels retain their height/separator behavior.
   - Update `tests/ace/tui/visual/test_ace_png_snapshots_agents_interactions.py` so the collapsed-panel interaction
     explicitly asserts `#chop` is the last panel, then inspect and intentionally regenerate the existing
     `agents_collapsed_panel_120x40` PNG golden to capture the new visual order.

## Validation

1. Run focused model and behavior tests:
   `pytest tests/ace/tui/models/test_agent_panels.py tests/ace/tui/test_agent_panel_collapse.py tests/ace/tui/test_agent_display_diff.py tests/ace/tui/test_agent_panels_display.py`.
2. Run the collapsed-panel visual test, inspect the generated expected/actual/diff artifacts, and update the PNG golden
   only after confirming the sole intentional change is the collapsed panel moving below expanded panels.
3. Run `just install` before project checks as required for an ephemeral workspace, then run `just check`. This includes
   the broader test and PNG snapshot coverage required by the repository.
4. If ordering logic adds measurable work to a hot refresh path, use the existing TUI trace/performance tests to confirm
   it remains an in-memory linear pass and does not introduce synchronous I/O, per `memory/tui_perf.md`.

## Risks and safeguards

- Reordering by widget index can accidentally leave stale content, focus classes, or jump hints in reused Textual slots.
  Full-render tests should assert title/content and focus by panel key after both collapse and expansion.
- Incremental display checks currently compare ordered panel-key tuples. They must use the same collapse-aware helper as
  full rebuilds so optimization behavior remains correct rather than silently degrading to full rebuilds.
- `None` is both the untagged panel key and a common default focused key. Tests must explicitly cover a collapsed
  untagged panel so membership and focus checks do not confuse it with a missing key.
- Persisted collapse state can arrive after the first paint. Restoration coverage should confirm the next established
  refilter/full-refresh path performs the partition and prunes vanished keys without adding event-loop work.
