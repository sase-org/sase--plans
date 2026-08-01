---
tier: tale
title: Agents-tab J/K skip collapsed tribe panels
goal: Uppercase J/K on the Agents tab land only on expanded tribe panels, skipping
  collapsed ones entirely and doing nothing when no other panel is expanded.
create_time: 2026-07-25 11:05:32
status: done
---

- **PROMPT:** [prompts/202607/jk_skips_collapsed_tribe_panels.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/jk_skips_collapsed_tribe_panels.md)
- **AGENTS:**
  - [bbugyi200.athena.ks](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.ks/README.md)
  - [bbugyi200.athena.ks--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ks.md#member-code)
- **COMMITS:**
  - [178f5d5](https://github.com/sase-org/sase/commit/178f5d53fd7228e8363cfd1ce7adf594bb6127ac) — fix(ace): skip collapsed tribe panels during jumps

# `J` / `K` Skip Collapsed Tribe Panels on the Agents Tab

## Goal

Make the Agents-tab `J` (`focus_next_agent_panel`) and `K` (`focus_prev_agent_panel`) keymaps land only on **expanded**
tribe panels. These keymaps mean "jump into the first / last selectable row of the next / previous tribe panel", so a
collapsed panel — which has no selectable rows — is never a sensible destination. When no other panel is expanded, the
keymaps become a pure no-op.

## Current Behavior

`src/sase/ace/tui/actions/agents/_panel_navigation.py::AgentPanelNavigationMixin._change_focused_agent_panel` walks
`self._panel_group` one panel at a time with `AgentPanelGroup.focus_next()` / `focus_prev()`
(`src/sase/ace/tui/models/agent_panels.py:229-257`), which advance `focused_idx` by exactly ±1 with wrap and no
awareness of whole-panel collapse state.

When the destination is a collapsed panel:

- `_panel_navigation_stops()` (`src/sase/ace/tui/actions/agents/_navigation_order.py:110-207`) returns `[]`, because
  `panel_is_collapsed(...)` forces `whole_panel_focused` / `suppress_panel_rows`.
- `_change_focused_agent_panel` therefore falls into its `else` branch and calls
  `_snap_current_idx_to_focused_panel(...)`, leaving the user on a whole-panel (container) focus of a collapsed panel
  instead of on a row.

That behavior is asserted today by
`tests/ace/tui/test_agent_panel_collapse_isolation.py::test_panel_switch_lands_on_collapsed_panel_and_l_reanchors`, and
documented in `docs/agent_families.md:342` ("`J` / `K` always move to the first / last selectable row of the next /
previous panel").

## Desired Behavior

1. `J` moves focus forward (with wrap) to the nearest panel that is **not** effectively collapsed, skipping any number
   of collapsed panels in between, and selects the first selectable row of that panel (a collapsed grouping banner still
   counts as a selectable row).
2. `K` does the same in reverse and selects the last selectable row.
3. This holds regardless of where focus starts, including when the currently focused panel is itself collapsed (reached
   via lowercase `j` / `k` whole-panel cycling, `'` jump hints, or `h`).
4. "Collapsed" means **effectively** collapsed — the result of `effective_panel_collapses(self, panel_keys)` in
   `src/sase/ace/tui/actions/agents/_panel_fold_intent.py`, which folds together explicit `_collapsed_panel_keys` /
   `_expanded_panel_keys` intents _and_ config-driven `ace.tribes.<name>.initially_expanded: false` defaults. Do not
   read `_collapsed_panel_keys` directly.
5. When there is no expanded panel other than the currently focused one (every sibling collapsed, e.g. right after `Z`
   isolation; or a lone panel), `J` / `K` are a **pure no-op**: no focus change, no `current_idx` change, no
   `_expanded_panel_focus` change, no jump-anchor push, no refresh, no notification. This mirrors the established
   precedent for `h` from a collapsed panel
   (`tests/ace/tui/test_agent_panel_collapse_navigation.py::test_h_from_collapsed_panel_with_no_expanded_target_is_a_pure_noop`).

## Design Decisions

**Skip logic lives in the model, collapse resolution stays in the mixin.** `AgentPanelGroup` is a pure, easily unit
tested model, but it cannot know about fold intents or merged config. So the mixin computes the collapsed key set and
passes it down as a `skip` argument. This keeps `focus_next` / `focus_prev` as the single mutation path for
`focused_idx` (important: do not bypass them and assign `focused_idx` directly from the mixin, or the model methods
become dead code that Symvision will flag).

**Lowercase `j` / `k` whole-panel cycling is unchanged.** `_change_whole_panel_focus`
(`src/sase/ace/tui/actions/agents/_panel_navigation.py:140-166`, driven by
`src/sase/ace/tui/actions/navigation/_basic.py:85`) intentionally cycles through _every_ panel including collapsed ones
— that is how a user selects and then expands a collapsed panel. It must keep doing so; it does not use `focus_next` /
`focus_prev`, so leaving it alone is automatic. Same for `h` → `_focus_last_expanded_panel`, `'` jump hints (which still
list collapsed titles), and `Z` isolation.

**Destination check happens before any mutation.** Today `_change_focused_agent_panel` clears `_expanded_panel_focus`
and calls `_save_agents_jump_anchor()` _before_ asking the panel group to move, which is safe only because the move can
never fail after the `len(panel_keys) <= 1` guard. Once the move can fail (everything else collapsed), that ordering
would push a bogus `Ctrl+O` jump anchor and silently drop whole-panel focus without a repaint (leaving a stale `❖` title
marker). So the new code must resolve the destination first and return early before touching any state.

**The single-panel case becomes a pure no-op too.** As a consequence of the reordering above, pressing `J` in a
single-panel layout while a whole panel is selected no longer clears `_expanded_panel_focus`. That is a small,
intentional behavior refinement: the old clear happened without a repaint, so it desynced the rendered `❖` marker from
`_expanded_panel_focus`. No existing test asserts the old clearing
(`test_focus_next_agent_panel_single_panel_does_not_overwrite_anchor` only checks the anchor stack and `current_idx`).
`l` and `Esc` remain the documented ways to leave whole-panel focus.

**No footer entry.** `src/sase/ace/CLAUDE.md` says the footer surfaces keymaps whose availability is state-conditional.
`J` / `K` already carry a state condition today (they need ≥2 panels) and are deliberately absent from the footer
(`Binding(..., show=False)` in `src/sase/ace/tui/bindings.py:185-186`); adding "no expanded sibling" to that condition
does not change the tradeoff. Do not add a footer hint.

**Rust core boundary.** This is Textual focus/keybinding state — presentation only under the
`rust_core_backend_boundary` memory's litmus test (no other frontend needs to match Agents-tab panel focus). Everything
stays in this repo; no `../sase-core` change.

## Implementation

### 1. `src/sase/ace/tui/models/agent_panels.py`

Add opt-in skipping to `AgentPanelGroup.focus_next` / `focus_prev` via a shared private stepper. Keep the existing
default behavior (`skip=()` ⇒ plain ±1 step) so no other caller changes meaning.

```python
def focus_next(self, *, skip: Collection[PanelKey] = ()) -> bool:
    """Advance focus to the next panel not in *skip*, with wrap.

    Returns ``True`` when the focused index changed; ``False`` when the
    panel set is empty, holds a single panel, or every other panel is
    skipped.
    """
    return self._focus_step(1, skip)

def focus_prev(self, *, skip: Collection[PanelKey] = ()) -> bool:
    """Retreat focus to the previous panel not in *skip*, with wrap."""
    return self._focus_step(-1, skip)

def _focus_step(self, delta: int, skip: Collection[PanelKey]) -> bool:
    """Step focus by *delta* until a non-skipped panel is found."""
    n = len(self.panel_keys)
    if n <= 1:
        return False
    skipped = {normalize_panel_key(key) for key in skip}
    for step in range(1, n):
        idx = (self.focused_idx + delta * step) % n
        if idx == self.focused_idx:
            break
        if self.panel_keys[idx] in skipped:
            continue
        self.focused_idx = idx
        return True
    return False
```

Notes:

- `Collection` is already imported in this module (used by `from_agents`); `normalize_panel_key` is defined at
  `agent_panels.py:42` and is idempotent, matching how `from_agents` normalizes `collapsed_panel_keys`.
- Update the `focus_next` / `focus_prev` docstrings to document the skip semantics and the new `False` case.

### 2. `src/sase/ace/tui/actions/agents/_panel_navigation.py`

Rework the head of `_change_focused_agent_panel` (lines 168-202) so the destination is resolved before any mutation:

```python
if self.current_tab != "agents":
    return
if self._guard_agent_navigation_for_artifact_file_viewer():
    return
panel_keys = self._panel_group.panel_keys
if len(panel_keys) <= 1:
    return
# ``J`` / ``K`` mean "enter the first / last row of an adjacent panel", so a
# collapsed panel — which renders no selectable rows — is never a
# destination. With no expanded sibling left the keymaps are a pure no-op:
# lowercase j/k whole-panel cycling remains the way to reach collapsed panels.
collapsed_keys = effective_panel_collapses(self, panel_keys)
focused_idx = self._panel_group.focused_idx
if all(
    key in collapsed_keys
    for idx, key in enumerate(panel_keys)
    if idx != focused_idx
):
    return
# Uppercase J/K retain their row-jump meaning even when the origin is a
# selected panel, so clear explicit expanded focus before choosing the
# destination row.
if getattr(self, "_expanded_panel_focus", False):
    self._expanded_panel_focus = False
old_focused_idx = focused_idx
old_idx = self.current_idx
old_group_key = self._current_group_key
old_agent = (...)  # unchanged
save_jump_anchor = getattr(self, "_save_agents_jump_anchor", None)
if callable(save_jump_anchor):
    save_jump_anchor()
if forward:
    changed = self._panel_group.focus_next(skip=collapsed_keys)
else:
    changed = self._panel_group.focus_prev(skip=collapsed_keys)
if not changed:
    return
```

Everything from `self.current_attempt_number = None` onward stays exactly as it is: the `stops[0]` / `stops[-1]`
selection, the empty-`stops` fallback through `_snap_current_idx_to_focused_panel` (keep it — an expanded panel can
still render zero rows), the manual-unread arming, the unread acknowledgement, and both refresh paths.

Ordering constraints to preserve:

- `_expanded_panel_focus = False` must happen **before** `_panel_navigation_stops()` is called, because
  `_resolve_focused_panel()` feeds that method's `suppress_panel_rows` computation.
- `effective_panel_collapses` is already imported in this module (used by `_resolve_last_expanded_panel_target`).
- Keep the `if not changed: return` guard as defense in depth even though the pre-check makes it unreachable.

Also refresh the stale comment "Collapsed destinations still imply panel focus." — collapsed destinations no longer
happen on this path.

### 3. Help modal

`src/sase/ace/tui/modals/help_modal/agents_bindings.py:37-40`: change the `J` / `K` description from
`"Jump into next / previous panel"` to `"Jump into next / prev open panel"` (32 chars, at the documented 32-char maximum
for keybinding descriptions in `src/sase/ace/CLAUDE.md`). Leave the short `Binding` labels in
`src/sase/ace/tui/bindings.py` and `src/sase/ace/tui/keymaps/metadata.py` ("Next Panel" / "Prev Panel") alone.

### 4. Docs

- `docs/ace.md:392` (keymap table row for `J` / `K`): note that collapsed panels are skipped, e.g. "Cycle focus across
  expanded tribe side panels (forward / reverse)".
- `docs/ace.md:711`: extend the "Use `J` / `K` to move across panels..." sentence — collapsed panels are skipped
  entirely, and the keymaps do nothing when no other panel is expanded.
- `docs/agent_families.md:342`: replace "`J` / `K` always move to the first / last selectable row of the next / previous
  panel." with wording covering the skip and the no-op, and contrast it with lowercase `j` / `k`, which still cycle
  through collapsed panels.

## Tests

### `tests/ace/tui/models/test_agent_panels.py` (model unit tests)

- `focus_next(skip=...)` hops over one or more skipped panels and wraps.
- `focus_prev(skip=...)` does the same in reverse.
- Stepping from a panel that is itself in `skip` still finds the nearest non-skipped panel.
- Every other panel skipped ⇒ returns `False` and leaves `focused_idx` untouched.
- No-arg `focus_next()` / `focus_prev()` still behave exactly as before (single-step + wrap, `False` for `n <= 1`).

### `tests/ace/tui/test_agent_panel_collapse_isolation.py` (or a new sibling module)

Use the existing `AgentPanelCollapseApp` harness from `tests/ace/tui/_agent_panel_collapse_helpers.py`
(`make_multi_panel_agents` / `make_four_panel_agents`; set `_collapsed_panel_keys`, then call `_sync_panel_group()`).

- **Update the existing conflicting test.** `test_panel_switch_lands_on_collapsed_panel_and_l_reanchors` currently
  asserts that a second `action_focus_next_agent_panel()` lands on collapsed `"alpha"` with `current_idx == 1`. Rewrite
  it (rename if the name no longer fits) so `J` skips `"alpha"` and wraps to the next expanded panel instead, and move
  the `l`-reanchor coverage it provides onto a panel reached by lowercase `j` / `k` whole-panel cycling so that path
  stays covered.
- `J` from an expanded panel with a collapsed panel in between lands on the first selectable row of the next expanded
  panel (`_current_group_key is None`, `_expanded_panel_focus is False`).
- `K` symmetrically lands on the **last** selectable row of the previous expanded panel, including the case where that
  last stop is a collapsed grouping banner (`_current_group_key` set) — mirror the existing
  `test_prev_panel_switch_can_land_on_last_collapsed_banner` in `tests/ace/tui/test_agent_panel_first_selection.py`.
- `J` and `K` starting from a **collapsed** focused panel land on an expanded panel and descend into a row.
- Config-driven collapse is skipped: monkeypatch `sase.ace.tui.models.tribe_display.tribe_display_for` to return
  `initially_expanded=False` for one tribe (same pattern as `tests/ace/tui/test_agent_fold_persistence.py:203-211`) with
  no explicit intent set, and assert `J` skips it.
- Pure no-op when every sibling panel is collapsed: snapshot
  `(focused_idx, focused_key, current_idx, current_attempt_number, _current_group_key, _expanded_panel_focus, anchor stacks, refresh_calls, notifications)`
  before and after both `action_focus_next_agent_panel()` and `action_focus_prev_agent_panel()` and assert equality —
  model it on `test_h_from_collapsed_panel_with_no_expanded_target_is_a_pure_noop`.
- Merged layout (`AgentPanelCollapseApp(..., merged=True)`) stays a no-op, as today.
- Lowercase `j` / `k` regression guard: `BasicNavigationMixin._navigate_agents_panel(app, ±1)` from whole-panel focus
  still cycles onto collapsed panels.

Also re-run the neighboring suites that exercise these actions, since the reordering touches shared state:
`tests/ace/tui/test_agent_panel_first_selection.py`, `tests/ace/tui/test_agent_panel_collapse_navigation.py`,
`tests/ace/tui/test_agent_panel_isolation_revert.py`, `tests/ace/tui/test_agents_zoom_panel_action.py`.

## Validation

```bash
just install            # ephemeral workspace may have stale deps
just test               # or targeted: pytest tests/ace/tui -k "panel"
just check              # required before finishing (lint + mypy + tests)
```

## Out of Scope

- Lowercase `j` / `k` whole-panel cycling, `h` / `l` / `L` panel fold navigation, `Z` isolation, and `'` jump hints all
  keep their current collapsed-panel behavior.
- No new keymap, config option, footer hint, or notification.
- No change to how panel collapse state itself is computed, persisted, or ordered (collapsed panels keep rendering last
  via `AgentPanelGroup.from_agents`).
