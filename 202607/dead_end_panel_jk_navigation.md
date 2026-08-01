---
tier: tale
title: Make j/k escape a dead-end tribe panel by selecting the adjacent panel
goal:
  On the Agents tab, pressing `j` / `k` while a row-focused tribe panel has no other selectable row selects the next /
  previous whole tribe panel instead of doing nothing.
create_time: 2026-07-28 15:52:11
status: done
---

- **PROMPT:** [prompts/202607/dead_end_panel_jk_navigation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/dead_end_panel_jk_navigation.md)
- **AGENTS:**
  - [bbugyi200.athena.nc--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.nc.md#member-code)
  - [bbugyi200.athena.nc--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.nc.md#member-plan)
- **COMMITS:**
  - [9cdfb37](https://github.com/sase-org/sase/commit/9cdfb378a43753436abac5a9c673f4efa8c60e51) — fix(ace): escape dead-end panel row focus

# Plan: j/k escapes a dead-end tribe panel

## Problem

On the Agents tab, a tribe panel whose entire content is a single selectable row (one agent lane, or one clan container
rendered as one collapsed row) is a navigation dead end. The user presses `j` / `k`, the keystroke is consumed, and
absolutely nothing moves. This is doubly confusing because a lone highlighted row in a one-row panel reads as "nothing
is selected" — the highlight has no siblings to contrast against — so the user's natural reaction is to press `j` again
and conclude the keymap is broken.

Reproduction: an Agents tab with several tribe panels where one panel (e.g. `@research`) contains a single clan row.
Focus that panel's clan row, press `j` or `k`, observe no change to the selection, the panel chrome, or the detail pane.

The intent behind the keystroke in that situation is unambiguous — there is nothing else to move to inside the panel, so
the user wants to leave it. This plan makes that keystroke move to the adjacent tribe panel.

## Current behavior

`BasicNavigationMixin._navigate_agents_panel` (`src/sase/ace/tui/actions/navigation/_basic.py:73`) is the single entry
point for `j` / `k` on the Agents tab (`action_next_changespec` / `action_prev_changespec`, same file, dispatch into it
when `current_tab == "agents"`). It resolves in three layers:

1. `src/sase/ace/tui/actions/navigation/_basic.py:85-87` — when a _whole panel_ is selected (collapsed panel, or an
   expanded panel with `_expanded_panel_focus`), `_change_whole_panel_focus`
   (`src/sase/ace/tui/actions/agents/_panel_navigation.py:140`) cycles whole-panel focus across every panel key
   (including collapsed panels, which sort last) and returns `True`, ending the keystroke.
2. `src/sase/ace/tui/actions/navigation/_basic.py:88-93` — otherwise it collects the focused panel's selectable stops
   (`_panel_navigation_stops`, `src/sase/ace/tui/actions/agents/_navigation_order.py:107`), returning early on an empty
   list, then applies the artifact-file-viewer guard.
3. `src/sase/ace/tui/actions/navigation/_basic.py:94-147` — it walks `stops` by `(pos + direction) % len(stops)`.

Layer 3 is where the dead end lives: with `len(stops) == 1` the modulo lands back on the same stop, so focus, chrome,
and detail all stay put. With `len(stops) == 0` layer 2 returns before the guard even runs.

Uppercase `J` / `K` (`action_focus_next_agent_panel` / `action_focus_prev_agent_panel`,
`src/sase/ace/tui/actions/agents/_panel_navigation.py:269-293`) already move across panels, but they deliberately _skip
collapsed panels_ and land on a row inside the destination, so they are not a substitute for panel-level selection and
cannot reach a collapsed neighbor.

## Design decision

**When the focused panel offers at most one selectable stop, `j` / `k` select the adjacent tribe panel as a whole
panel** — the same target `_change_whole_panel_focus` would pick: `(focused_idx ± 1) % len(panel_keys)` over the full
panel-key order, wrapping, and including collapsed panels.

Rationale, and why this beats the alternative of reusing the `J` / `K` row-hop:

- It matches what the keystroke is being asked to do: select the next/previous _panel_, not a row inside it.
- Whole-panel selection is the most legible state ACE has — `❖` title marker, focus-accent title chrome, and a
  fold-aware `TRIBE` summary in the metadata pane. That directly answers the reported complaint that it is hard to see
  what is selected in a one-row panel.
- It composes into one rule. After the hop, whole-panel focus is active, so the _existing_ layer-1 branch handles every
  subsequent `j` / `k` and keeps cycling panels. The result is symmetric and reversible: `j` then `k` returns to the
  panel you left. A `J` / `K` style row-hop would instead drop the user into a multi-row neighbor where `k` immediately
  reverts to intra-panel wrapping, stranding them.
- It can reach collapsed neighbors, which `J` / `K` intentionally cannot. Whole-panel cycling is already documented as
  the way to reach collapsed panels.
- `l` or `Esc` descends into the destination panel's remembered row, and `Ctrl+O` returns to the departed row, both via
  existing machinery.

Scope guards on the new behavior:

- Split layout only. In the merged layout (`_agent_panels_grouped`) there is a single panel key and no whole-panel
  focus; behavior is unchanged.
- Only when more than one panel exists. A lone panel with a lone row stays a no-op.
- Panels with two or more selectable stops are untouched: `j` / `k` keep wrapping inside the panel exactly as today.
  This change adds an escape hatch for a dead end; it does not turn `j` / `k` into a general panel-spilling motion.

## Implementation

### 1. Extract the shared whole-panel step (`src/sase/ace/tui/actions/agents/_panel_navigation.py`)

Add a private helper to `AgentPanelNavigationMixin` that performs one wrapping whole-panel hop and owns the key-to-paint
instrumentation, then rewrite `_change_whole_panel_focus` (line 140) to use it:

```python
def _step_whole_panel_focus(self, *, forward: bool) -> bool:
    """Select the adjacent whole panel, wrapping over every panel key."""
    panel_keys = self._panel_group.panel_keys
    current_idx = self._panel_group.focused_idx
    target_idx = (
        (current_idx + 1) % len(panel_keys)
        if forward
        else (current_idx - 1) % len(panel_keys)
    )
    if not self._focus_whole_panel_target((target_idx, panel_keys[target_idx])):
        return False
    # Whole-panel focus is reactive state outside ``current_idx``. Its
    # remembered row may be the same across a hop, so the ordinary index
    # watcher is not a reliable paint-sample terminator for lower-case
    # j/k instrumentation.
    jk_perf = getattr(self, "_jk_perf", None)
    if jk_perf is not None:
        self.call_after_refresh(jk_perf.mark_painted)
    return True
```

`_change_whole_panel_focus` keeps its existing contract exactly — return `False` when no whole panel is selected, `True`
(consuming the keystroke) when one is, including the single-panel short circuit at line 146 — and delegates the index
arithmetic, `_focus_whole_panel_target` call, and `mark_painted` scheduling to `_step_whole_panel_focus`. Move the
existing comment about paint-sample termination with the code. No behavior change in this step.

### 2. Add the dead-end escape (`src/sase/ace/tui/actions/agents/_panel_navigation.py`)

Add to the same mixin:

```python
def _escape_dead_end_panel_focus(self, *, forward: bool) -> bool:
    """Select the adjacent whole panel from a panel with nowhere left to move."""
```

Behavior, in order:

1. Return `False` unless `current_tab == "agents"`, the layout is split (`not _agent_panels_grouped`), `_panel_group` is
   not `None`, and `len(_panel_group.panel_keys) > 1`.
2. Call `_remember_focused_panel_selection()` with no argument so the departing panel keeps its row anchor for the later
   `l` / `Esc` / return hop. (It is a no-op when whole-panel focus is already active, which cannot be the case here, and
   it validates the stop itself.)
3. Capture the departing agent — `self._agents[self.current_idx]` when `_current_group_key is None` and the index is in
   range, else `None`.
4. Call `_step_whole_panel_focus(forward=forward)`; return `False` if it returns `False`.
5. On success call `_arm_manual_unread_after_departure(<departing agent>)` through `getattr`, mirroring the row →
   whole-panel transition in `_navigate_agent_left` (`src/sase/ace/tui/actions/agents/_folding_agent_tree.py:244-250`),
   so a manually-unread row can clear normally the next time it is selected. Return `True`.

Note `_focus_whole_panel_target` (line 91) already saves the `Ctrl+O` jump anchor, clears `_current_group_key` and
`current_attempt_number`, restores the destination's remembered row, sets `_expanded_panel_focus`, and refreshes panel
chrome plus detail — so the escape adds nothing beyond selection memory, unread arming, and the guards.

### 3. Wire it into `j` / `k` (`src/sase/ace/tui/actions/navigation/_basic.py`)

Rewrite `src/sase/ace/tui/actions/navigation/_basic.py:88-93` as:

```python
stops = self._panel_navigation_stops()  # type: ignore[attr-defined]
guard = getattr(self, "_guard_agent_navigation_for_artifact_file_viewer", None)
if callable(guard) and guard():
    return
if len(stops) <= 1:
    escape = getattr(self, "_escape_dead_end_panel_focus", None)
    if callable(escape) and escape(forward=direction > 0):
        return
if not stops:
    return
```

Points that matter here:

- The guard must run **before** the escape, otherwise `_change_focused_agent_panel`-style re-entry into the guard would
  emit the "Close the artifact viewer before switching agents" warning twice. Hoisting it above `if not stops: return`
  is the only behavioral side effect of the reorder: a focused panel that renders zero selectable rows now surfaces the
  existing warning instead of silently swallowing the keystroke. That is consistent with every other row-navigation path
  and is intended.
- The escape is attempted for `len(stops) <= 1`, not just `== 1`, so a panel that renders no selectable rows at all is
  no longer a silent dead end either.
- When the escape declines (single panel, merged layout, no `_panel_group`), control falls through to the untouched
  legacy path, so the existing single-stop re-focus — which still clears a stale `_current_group_key` — is preserved
  verbatim.
- `_panel_navigation_stops()` is memoized per refresh cycle
  (`src/sase/ace/tui/actions/agents/_navigation_order.py:141-159`), so computing it before the guard costs nothing new;
  it is already computed there today.

Update the `_navigate_agents_panel` docstring to state the dead-end rule.

### 4. Documentation

- `docs/ace.md:462` — extend the `j` / `k` Navigation row to cover the dead-end fall-through, e.g. "Move to the next /
  previous visible row; while a whole panel is selected, or when the focused panel has no other selectable row, cycle
  whole panels instead".
- `docs/ace.md` (the `J` / `K` paragraph beginning "Use `J` / `K` to move across expanded panels", around line 835) —
  add a sentence: when the focused panel's only selectable row is already selected — or it renders none — lowercase `j`
  / `k` select the adjacent whole panel instead, wrapping across every panel including collapsed ones, and `l` or `Esc`
  descends into the newly selected panel's remembered row. Contrast it explicitly with `J` / `K`, which skip collapsed
  panels and land on a row.
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py` — the Agents "Navigation" section currently maps `j` / `k` to
  `"Move row / selected panel"` (line 33-36). Keep that entry and add a second entry for the same key pair reading
  `"Lone row: cycle whole panels"` (28 chars, within the 32-char description budget). Repeated-key entries are the
  established idiom in this section (see the four `hooks_or_collapse_all` rows). Required by the help-popup maintenance
  rule in `src/sase/ace/CLAUDE.md`.
- No footer change. `src/sase/ace/CLAUDE.md` scopes the footer to conditional keymaps driven by the selected entry; `j`
  / `k` is global navigation and belongs to the help modal only.
- No keymap-config change: `j` / `k` are already bound (`next_changespec` / `prev_changespec` in
  `src/sase/default_config.yml`), and this plan changes only what the existing action does.

### 5. Boundary check

This is presentation-only Textual focus state — `AgentPanelGroup`, `_expanded_panel_focus`, and
`_panel_selection_memory` all live in `src/sase/ace/tui/models/` and `src/sase/ace/tui/actions/` with no `sase_core_rs`
involvement. Per the Rust-core boundary rule, nothing here belongs in `../sase-core`; do not add a Rust wire change for
it.

## Tests

Add `tests/ace/tui/test_agent_dead_end_panel_navigation.py`. Build its harness by subclassing `AgentPanelCollapseApp`
from `tests/ace/tui/_agent_panel_collapse_helpers.py` (it already composes `AgentPanelNavigationMixin`,
`AgentSelectionMixin`, `AgentNavigationOrderMixin`, and `PanelRefreshMixin`, and records `armed_departures`,
`_panel_selection_memory`, `refresh_calls`, and `notifications`) and mixing in `BasicNavigationMixin` plus a
`_refresh_agents_display_debounced` stub. Drive `_navigate_agents_panel(1)` / `_navigate_agents_panel(-1)` directly, as
`tests/ace/tui/test_agent_jk_navigation.py` does, so no `_jk_perf` / `call_after_refresh` stubs are needed. Use
`make_agent` to build fixtures where one tribe panel holds exactly one agent and another holds several.

Cases to cover:

1. Single-row focused panel, several panels: `j` selects the next panel as a whole — `_panel_group.focused_idx` advances
   by one, `_resolve_focused_panel()` is not `None`, and `current_idx` lands on the destination's first rendered row.
   `k` from the same start selects the previous panel.
2. Wrap-around in both directions from the first and last panel keys.
3. A collapsed neighbor is a valid destination (unlike `J` / `K`): the hop lands on it with
   `_resolve_focused_panel().collapsed is True`.
4. The departing panel's row is remembered: `_panel_selection_memory[<departed key>] == ("agent", <old idx>)`.
5. The departing agent is passed to `_arm_manual_unread_after_departure` (assert via `armed_departures`); departing from
   a banner stop arms nothing.
6. A focused panel whose only selectable stop is a _collapsed grouping banner_ also escapes.
7. Multi-row focused panel: `j` moves within the panel and leaves `_panel_group.focused_idx` unchanged — the regression
   guard for the intra-panel path.
8. Exactly one panel that has one row: no exception, `focused_idx`, `current_idx`, and `_resolve_focused_panel()` all
   unchanged.
9. Merged layout (`merged=True`): a one-row merged panel stays a no-op.
10. Whole-panel focus already active: layer 1 still wins, i.e. exactly one panel hop per keystroke and no double
    advance.
11. Artifact-file-viewer guard active: no panel hop, `focused_idx` unchanged, and exactly one warning notification
    recorded.
12. A zero-stop focused panel now emits the guard warning when the viewer is open (the intended reorder side effect).

Also extend `tests/ace/tui/test_agent_jk_navigation.py` with a case asserting that its existing panel-less stub — which
does not implement `_escape_dead_end_panel_focus` — still behaves exactly as before on a single-stop panel, proving the
`getattr` fall-through path.

Re-run these existing suites, which exercise the touched code paths:

```bash
just test tests/ace/tui/test_agent_jk_navigation.py \
          tests/ace/tui/test_agent_panel_first_selection.py \
          tests/ace/tui/test_agent_panel_collapse_navigation.py \
          tests/ace/tui/test_agent_panel_collapse_isolation.py \
          tests/ace/tui/test_agent_collapsed_panel_selection.py \
          tests/ace/tui/test_agent_panels_display.py \
          tests/ace/tui/test_jk_reliability.py \
          tests/ace/tui/test_agent_stopped_navigation.py \
          tests/ace/tui/test_member_jump_navigation.py
```

## Validation

Workspace directories are ephemeral, so install first, then run the full gate:

```bash
just install
just check
```

`just check` covers ruff, mypy, symvision, keep-sorted, and the parallel pytest run. If symvision flags the new private
helpers, consult the symvision long-term memory note before adjusting.

## Out of scope

- Changing `J` / `K`, which keep their "jump into the first/last row of the next/previous _expanded_ panel" meaning.
- Making `j` / `k` spill out of a panel that still has other rows to visit.
- Any footer, keymap-config, or Rust-core change.
