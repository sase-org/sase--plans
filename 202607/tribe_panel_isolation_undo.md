---
tier: tale
title: Tribe panel isolation undo for the Agents-tab H keymap
goal: 'Pressing H a second time restores the tribe-panel expansion layout that existed
  before the last H isolation, with clear visual marking of the panels a restore would
  change, until the user changes any other panel''s expansion state.

  '
create_time: 2026-07-19 16:20:17
status: done
prompt: 202607/prompts/tribe_panel_isolation_undo.md
---

# Plan: Tribe panel isolation undo (`H` solo toggle with revert)

## Product context

On the `sase ace` Agents tab, `H` (`action_hooks_or_collapse_all` → `_isolate_focused_panel` in
`src/sase/ace/tui/actions/agents/_folding.py`) isolates the selected tribe panel: it expands that panel and collapses
every other one. Today this is destructive — the user's carefully arranged mix of expanded and collapsed tribe panels is
lost, and rebuilding it takes many individual panel toggles.

This plan turns `H` into a **solo toggle**: the first press isolates as today but remembers the prior whole-panel
expansion layout; while that memory is "armed", the next press on any tribe panel restores the remembered layout instead
of isolating. The memory is dropped the moment the user changes the expansion state of any panel _other than_ the one
that was selected when `H` was last pressed, so `H` never surprisingly time-travels past newer intent. While armed,
every panel whose current expansion state differs from the remembered one carries a small restore marker, so the user
can always see both _that_ `H` will restore and _what_ it will change.

## Current behavior (grounding)

- Whole-panel expansion state is the in-memory set `self._collapsed_panel_keys: set[PanelKey]`
  (`src/sase/ace/tui/actions/_state_init.py:513`), where `PanelKey = str | None` is the tribe tag, `None` for the
  untagged panel (`src/sase/ace/tui/models/agent_panels.py`). Panels are derived live from agents, so panels
  appear/disappear as tribes come and go, and expanded panels render before collapsed ones.
- `_isolate_focused_panel` (`_folding.py:466`) only acts in split layout with whole-panel focus
  (`_resolve_focused_panel` non-`None`, `_agent_panels_grouped` false). It computes
  `desired_collapsed = live keys − selected`, applies the diff to `_collapsed_panel_keys`, records each change via
  `_persist_panel_fold_change`, then `_invalidate_agent_panel_cache()` + `_refresh_agents_display(list_changed=True)`.
  It is idempotent (returns `True` with no fold change when already isolated).
- Every user-driven whole-panel expansion change funnels through `_persist_panel_fold_change` (`_folding.py:58`):
  - `_isolate_focused_panel` (`_folding.py:513`),
  - `_collapse_focused_panel` (`h` on panel focus, `_folding.py:552`),
  - `_expand_agent_panel` (`l`/activation, `_folding.py:568`),
  - numeric fold-hint toggles (`L`, `_panel_hint_folding.py:278`). The two non-funnel mutations are the merged-layout
    toggle, which clears the whole set (`_panel_navigation.py`, `action_toggle_agent_panel_grouping`), and the startup
    fold-state install/replay (`_fold_persistence.py:_install_loaded_agents_fold_state`).
- `_fold_persistence.py` is cross-session durability (intent journal + coalesced snapshot saves to
  `ace_agents_fold_state.json`), not an undo system. Restored `H` changes must keep flowing through
  `_persist_panel_fold_change` so the resulting layout persists as usual; the undo memory itself is deliberately
  session-transient. Its snapshot shape (`collapsed_panels=frozenset(...)`, `_fold_persistence.py:186`; restore at
  `:246`) is the primitive to mirror in-memory.
- Whole-panel focus is resolved by `_resolve_focused_panel` (`src/sase/ace/tui/actions/agents/_selection.py:35`), which
  returns an `AgentPanelFocus(panel_key, collapsed)` only when the focused panel is collapsed or `_expanded_panel_focus`
  is set — the same gate the revert branch reuses, so "press `H` while any tribe panel is selected" keeps one meaning.
- Each split panel is its own `AgentList` widget; its title bar is composed by `agent_panel_border_title`
  (`src/sase/ace/tui/actions/agents/_display_panel_titles.py:119`), fed per panel by `_agent_panel_title`
  (`src/sase/ace/tui/actions/agents/_display_panel_collection.py:89`). The title already carries transient chips —
  numeric fold hint `[n]`, selected marker `❖` (bold `#FFD75F`), collapsed caret `▸` — driven by transient app flags and
  repainted selectively via `_refresh_affected_panel_widgets`
  (`src/sase/ace/tui/actions/agents/_display_panel_widgets.py:215`). Panel titles are rebuilt on every refresh and are
  NOT in the row/banner LRU (`_agent_list_render_cache.py`), so a title-level marker needs no cache-key change.
- Footer shows `H` = "only panel" when a panel is focused (`src/sase/ace/tui/widgets/_keybinding_bindings.py:165`); the
  help modal documents `H` as "Only panel / collapse group / compact Tools"
  (`src/sase/ace/tui/modals/help_modal/agents_bindings.py:135`).
- No panel fold logic touches the Rust core (`sase_core_rs`) — it is all Python/Textual presentation state under
  `src/sase/ace/tui/`.

## Design

### Semantics

- **Arm on isolate.** When `H` isolates and actually changes at least one panel fold, capture a revert record _before_
  mutating: the snapshot `collapsed_before: frozenset[PanelKey]` and the `target_key: PanelKey` (the panel selected at
  press time). An idempotent isolation (no fold changed) does not arm — there is nothing to undo.
- **Revert on the next armed press.** While armed, `H` with whole-panel focus on _any_ tribe panel restores
  `_collapsed_panel_keys` to `collapsed_before ∩ live panel keys`, then disarms. Restoring from any panel (not just the
  original target) is what makes the marker semantics well-defined and gives `H` clean toggle behavior: isolate ⇄
  restore. Vanished panels are silently dropped; panels that appeared after arming keep their current state (they are
  absent from the snapshot, and any user collapse of them would have disarmed first — see invalidation). The revert
  applies its diff exactly like `_isolate_focused_panel` does today: mutate the set, `_persist_panel_fold_change` per
  changed key, `_invalidate_agent_panel_cache()`, one `_refresh_agents_display(list_changed=True)`. Keep whole-panel
  focus on the currently focused panel, mirroring the existing focus bookkeeping (`_expanded_panel_focus`,
  `_current_group_key`, `current_attempt_number`, and the collapsed-panel `current_idx` snap used by
  `_collapse_focused_panel`). Finish with a short toast (`Restored N panels`, existing 1.5 s style).
- **`H` elsewhere is untouched.** With the selection inside a panel (no whole-panel focus), `H` still falls through to
  `_collapse_group_fold`; in-panel group/agent folds never arm, disarm, or get restored. The feature scope is
  whole-panel expansion only, matching the user-visible notion of "tribe panel expansion state".

### Invalidation (disarm) rules

The armed record survives exactly as long as it is still an honest undo:

1. **Target exemption.** Fold changes to `target_key` (collapse it, re-expand it) never disarm — the user may keep
   working the isolated panel freely.
2. **Any other panel's expansion change disarms.** Hook the check into the `_persist_panel_fold_change` funnel: when
   armed and `panel_key != target_key`, drop the record. Because revert disarms _before_ applying its own diff, and
   arming happens _after_ isolation's own persist calls, `H`'s own transitions never trip the hook — no guard flag
   needed. The mutation inventory is verified complete: the only user-driven paths that can touch a non-target panel are
   the `L` numeric fold selector, another isolate (which the revert branch replaces while armed), and the wholesale
   cases in rule 3; `h`/`l` only ever touch the focused panel, and mouse focus changes, zoom, and kill flows never
   mutate panel folds. All per-key paths call the funnel. Future mutation paths inherit correct behavior by using it.
3. **Wholesale layout changes disarm.** The merged-layout toggle (`action_toggle_agent_panel_grouping`) clears all panel
   folds without per-key persistence — disarm there explicitly. The startup fold-state install
   (`_install_loaded_agents_fold_state`) replaces the set — disarm there too (defensive; arming normally postdates first
   load).
4. **Vanished target disarms lazily.** If `target_key` is no longer a live panel key, treat the record as expired:
   checked when `H` is pressed and when computing markers, so agent lifecycle churn cannot leave a stale marker pointing
   at a tribe that no longer exists. Non-target panels vanishing or appearing do not disarm (they are handled by the
   ∩/absent rules above).

### State

A small frozen dataclass (e.g. `PanelIsolationRevert(target_key, collapsed_before)`) stored as
`self._panel_isolation_revert: PanelIsolationRevert | None`, initialized in `_state_init.py`. Session- transient by
design: it is never persisted, and a fresh session starts disarmed. All logic stays in the existing folding mixin family
(presentation-only Textual state — per the Rust core boundary litmus test this does not touch `sase-core`).

### Visual marking

While armed, compute `marked = (current collapsed △ collapsed_before) ∩ live keys` — precisely the panels a restore
would change. Surface it as a helper (e.g. `_panel_isolation_marked_keys()`) consumed by the panel title render path:

- **Marker.** A single `↺` glyph rendered in each marked panel's title bar, right of the tribe name, in the theme's
  warning/accent tone (visually quiet next to the focused-panel highlight, identical in light/dark). One glyph for both
  directions (will-expand and will-collapse) keeps the message simple: "H returns this panel to how it was". Implement
  it as a new flag threaded through `_agent_panel_title` → `agent_panel_border_title`, mirroring the existing transient
  title-chip pattern (`[n]` fold hints, `❖`, `▸`) — the marker must live in the title, not in rows/banners, so the row
  LRU cache keys stay untouched. Arm/disarm repaints go through `_refresh_affected_panel_widgets` with only the affected
  keys.
- **Footer.** While armed and a panel is focused, the `H` footer label flips from "only panel" to "restore panels"
  (`_keybinding_bindings.py:165`), satisfying the footer conditional-keymap convention (the condition is sometimes true,
  sometimes false).
- **Help modal.** Update `H`'s Agents entry to document the toggle ("Only panel ⇄ restore panels / collapse group /
  compact Tools"), keeping the ≤32-char description limit of `help_modal` boxes in mind.
- No new keymap and no `default_config.yml` change — `H` is reused.

### Performance constraints (per `tui_perf.md`)

- Marker math is pure in-memory set arithmetic, O(#panels), computed on the render path with no disk I/O, stat, or
  subprocess.
- Arm/disarm/marker changes repaint through the selective panel-widget path (`_refresh_affected_panel_widgets`) rather
  than full list rebuilds, except where the fold change itself already triggers the standard
  `_refresh_agents_display(list_changed=True)` repaint.
- Panel titles are rebuilt every refresh and bypass the row/banner LRU, so a title-level marker cannot go stale; keeping
  the marker out of rows/banners is what preserves that property (`agent_render_key`/`banner_render_key` stay
  unchanged).
- Disarm from the persistence funnel is a cheap field reset; it must not add work to keystroke paths.

## Implementation outline

1. **State + record type.** Add the dataclass and `_panel_isolation_revert` field (`_state_init.py`, folding models or a
   small module next to `_folding.py`).
2. **Arm/revert in `_isolate_focused_panel`.** Branch at the top: armed and valid (target still live) → run the revert
   routine; otherwise capture, isolate, and arm only when `changed_keys` is non-empty. Factor the shared
   apply-diff/refresh logic so isolate and revert cannot drift.
3. **Disarm hooks.** Funnel check in `_persist_panel_fold_change`; explicit disarms in
   `action_toggle_agent_panel_grouping` and `_install_loaded_agents_fold_state`; lazy target-liveness check.
4. **Marker rendering.** `_panel_isolation_marked_keys()` + a marker flag threaded through `_agent_panel_title`
   (`_display_panel_collection.py:89`) into `agent_panel_border_title` (`_display_panel_titles.py:119`), with a style
   constant beside the existing `❖`/`▸` chip styles.
5. **Footer + help modal.** Conditional label flip and help text update.
6. **Docs/UX text.** Revert toast; comment in the keymap block of `default_config.yml` (`_folding.py` docstrings)
   updated to describe the toggle.

## Testing

- **Behavioral tests** (new `tests/ace/tui/test_agent_panel_isolation_revert.py`, alongside
  `test_agent_panel_collapse.py` patterns):
  - isolate → revert round-trips an arbitrary expanded/collapsed mix;
  - idempotent isolate does not arm;
  - target-panel collapse/expand keeps the record; other-panel `h`/`l` and `L`-hint toggles disarm;
  - merged-layout toggle disarms; fold-state install disarms;
  - vanished target expires the record; vanished non-target panels are dropped from the restore; panels born after
    arming keep their state;
  - revert records per-key intents through `_record_agents_panel_fold_change` (persistence journal, cf.
    `test_agent_fold_persistence.py`);
  - marker set equals the symmetric difference while armed and empties on disarm.
- **Visual**: PNG snapshot of an armed layout with marked panels
  (`tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py` or a sibling), run via `just test-visual`.
- **Footer/help**: extend the existing keybinding-footer and help-modal coverage for the label flip and new text.
- `just check` before completion.

## Risks and edge cases

- **Focus after revert.** Restoring may collapse the focused panel; reuse the `_collapse_focused_panel` bookkeeping (idx
  snap to first rendered row) so selection never lands on a hidden row.
- **Panel ordering shifts.** Expanded panels render before collapsed ones, so revert reorders panels; the one coalesced
  repaint keeps this atomic.
- **Marker legibility.** The glyph must not collide with existing title adornments (fold hints, counts, unread badges);
  the coding agent should match the existing title composition and PNG-verify both light/dark.
- **Merged layout.** Isolation already refuses merged mode; revert must too (armed state cannot survive the merge
  toggle, rule 3).

## Out of scope

- Multi-level undo history (only the last isolation is remembered).
- Restoring in-panel group/agent folds.
- Persisting the armed record across sessions.
