---
create_time: 2026-07-13 12:35:22
status: done
prompt: 202607/prompts/group_actions_panel_scope.md
tier: tale
---
# Fix Agents-tab focused-group mark/kill over-targeting agents in other panels

## Problem (user report)

On the Agents tab with `group: by date` active, the user focused folded hour banners (e.g. `10:00`, `09:00`) in one
panel, marked them with `<space>`, and killed the marked set with `x`. The kill took out agents in _other_ tag panels
too — including agents belonging to a running epic (`#sase-5w` panel) that were started in the same hours. Screenshot
evidence (`.sase/home/tmp/screenshots/20260713_122150.png`): after marking `10:00` + `09:00` in the untagged panel (3 +
5 agents), the `#sase-5w` panel's own `10:00` banner also shows a full `[✓]` mark and the footer reads "9 marked" — one
more than the untagged panel contains.

**The suspicion is CONFIRMED.** A standalone repro against the real models shows that with two tag panels each
containing a `Today / 10:00` group, resolving the focused group for the untagged panel's banner returns the agents of
_both_ panels:

```
focused group key: ('Today', '10:00')
members resolved for kill/mark: ['epic_b', 'epic_a', 'untagged_b', 'untagged_a']   # expected only untagged_*
```

## Root cause

Group banners are identified by `GroupRow.group_key`, a tuple with **no panel/tag component** — under `BY_DATE` an hour
subgroup key is just `("Today", "10:00")` (see `src/sase/ace/tui/models/agent_groups/_tree.py`). Panels are tag-buckets
rendered independently (`src/sase/ace/tui/models/agent_panels.py`), and the renderer + navigation correctly build **one
tree per panel**:

- `_panel_navigation_stops()` and `_first_agent_idx_for_focused_group()` in
  `src/sase/ace/tui/actions/agents/_navigation_order.py` / `_panel_navigation.py` slice to the focused panel via
  `rendered_panel_slice(owner, focused_key)`, build the tree from the panel's agents only, and map the tree's local
  indices back to global `self._agents` indices.

But the **action-side resolver** `get_focused_agent_group()` in `src/sase/ace/tui/actions/agents/_group_focus.py` skips
the panel slice entirely: it calls `build_agent_tree(owner._agents, ...)` over the FULL cross-panel agent list and
returns the first group whose `group_key` matches `owner._current_group_key`. In that combined tree, every agent from
every panel that lands in the same bucket (`Today` at `10:00`) is aggregated into one `GroupRow.agent_indices`. All
same-key groups across panels are therefore targeted as one.

### Affected flows (all route through `get_focused_agent_group`)

1. **`<space>` on a folded banner** — `_toggle_mark_focused_group()` in `src/sase/ace/tui/actions/agents/_marking.py`
   marks `top_level_group_agents(group, self._agents)`, i.e. every panel's same-key members. This is exactly what the
   screenshot shows: the `#sase-5w` panel's `10:00` banner renders `[✓]` because _all_ of its members really were marked
   (banner mark state is computed per panel from the marked-identity set in
   `src/sase/ace/tui/widgets/_agent_list_build.py::_banner_mark_state`). The subsequent `x` (`_bulk_kill_marked_agents`)
   then kills the over-marked set — this is how the epic agents died.
2. **`x` with a banner focused and no marks** — `action_kill_agent()` → `_bulk_kill_group_agents()` in
   `src/sase/ace/tui/actions/agents/_kill_action.py` (same over-broad member resolution, single confirm).
3. **Cleanup panel (`X`) "group" action and its `group_count` stat** — `_build_agent_cleanup_panel_state()` /
   `_run_agent_cleanup_panel_action("group")` in `_kill_action.py`.
4. **Mark auto-advance anchor** — `_set_current_idx_to_group_anchor()` in `_marking.py` (cosmetic; uses the first
   covered index).

### Notes on scope of the defect

- Not `BY_DATE`-specific: under `STANDARD` mode the L0 key is `(project,)` and deeper keys are
  `(project, changespec, name_root, ...)` — a project whose agents carry different tags appears in multiple panels with
  identical keys, so group mark/kill over-targets there too. `BY_STATUS` collides on status buckets the same way. The
  by-date hour buckets just make collisions near-certain.
- A second, subtler consequence of building from the full list: layout decisions (`panel_uses_changespec_level`) are
  computed per panel by the renderer but from the full list in `get_focused_agent_group`, so in `STANDARD` mode the
  resolver can disagree with the rendered keys (missed lookups / stale-key fallbacks). Panel-scoping fixes this for
  free.

## Fix

Make `get_focused_agent_group()` panel-scoped and have it return a `GroupRow` whose `agent_indices` are **global**
`self._agents` indices (what every consumer already assumes):

In `src/sase/ace/tui/actions/agents/_group_focus.py`:

1. Resolve the focused panel: `panel_group = getattr(owner, "_panel_group", None)`.
   - When present: `global_indices, panel_agents = rendered_panel_slice(owner, panel_group.focused_key)` (from
     `._navigation_order`; it already handles the merged-panels mode via `_agent_panels_grouped` and uses the memoized
     `_agent_panel_index` when available — no new hot-path cost, and the per-panel tree is strictly smaller than today's
     full-list build).
   - When absent (test fakes, single-list contexts): fall back to the full rendered list as one panel, mirroring
     `_agents_visible_order()`'s no-panel branch (filter by `agent_is_rendered_in_agents_panel`, identity index map).
2. Build the tree from `panel_agents` (same fold registry + grouping mode as today) and find the entry whose
   `group_key == owner._current_group_key`.
3. Return
   `dataclasses.replace(entry.group, agent_indices=tuple(global_indices[i] for i in entry.group.agent_indices if 0 <= i < len(global_indices)))`
   so `top_level_group_agents(group, owner._agents)`, `_set_current_idx_to_group_anchor`, and the cleanup-panel
   `group_count` all keep indexing the global list correctly.
4. `top_level_group_agents()` needs no change.

Because the focused banner is always set from the focused panel's navigation stops (`_focus_panel_navigation_stop`),
`(_panel_group.focused_key, _current_group_key)` uniquely identifies the on-screen group — no `GroupKey` schema change,
no fold-registry or persistence changes, no render changes.

Optional consolidation (small, keeps behavior identical): rewrite `_first_agent_idx_for_focused_group()` in
`_panel_navigation.py` to call the fixed `get_focused_agent_group()` and return the first `agent_indices` entry,
removing the duplicated slice+build+remap logic.

### Explicitly out of scope

- **Fold-state sharing across panels**: `AgentGroupFoldRegistry` is keyed by the same panel-less `GroupKey`, so folding
  `10:00` in one panel collapses every panel's `10:00`. Non-destructive, long-standing, and changing it would alter
  persisted fold semantics — leave as-is (possible follow-up).
- Rust core boundary: this is pure Textual/presentation selection state; no `sase-core` changes.

## Tests

Extend the existing harnesses (they already model this layer well; both fakes currently lack `_panel_group`, which the
fallback branch keeps working — new tests add a minimal `_panel_group`/`AgentPanelGroup` stub plus tagged agents):

1. `tests/ace/tui/test_agent_group_kill.py`
   - Two panels (untagged + tag `epic`), same `Today/HH:00` bucket, `BY_DATE` mode, banner focused in the untagged
     panel: `action_kill_agent()` modal covers ONLY the untagged panel's agents; after confirm, the tagged panel's
     agents survive.
   - Same-key collision under `STANDARD` mode (same project split across two tag panels) to lock in the mode-agnostic
     fix.
   - Cleanup-panel `group_count` counts only the focused panel's members.
2. `tests/ace/tui/test_agent_marking.py`
   - `_toggle_mark_focused_group()` with the two-panel fixture: only the focused panel's identities are marked; the
     other panel's same-key members stay unmarked.
3. New/extended unit coverage for `get_focused_agent_group()` itself: focused-panel scoping, global-index remapping,
   merged-panels mode (`_agent_panels_grouped=True`), stale key → `None`, and the no-`_panel_group` fallback.

## Verification

- Targeted: `pytest tests/ace/tui/test_agent_group_kill.py tests/ace/tui/test_agent_marking.py` plus any new test
  module; also `tests/ace/tui/test_agent_stopped_navigation.py` and panel/navigation suites that exercise
  `_current_group_key`.
- `just check` before finishing (per repo rules). TUI perf rules reviewed (`memory/tui_perf.md`): the change stays on
  the existing keypress path with a smaller tree build and no new I/O; no refresh-path or render changes.
- Manual sanity in `sase ace`: group by date with agents in 2+ tag panels sharing an hour; `<space>` on a folded hour
  banner must mark only that panel's rows (footer count must match the panel), `x` must list only those rows in the
  confirm modal.
