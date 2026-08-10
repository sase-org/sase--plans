---
tier: epic
title: Split Agents-tab `H` into a panel fold sweep (`-`) and a hinted fold collapse
goal: "On the Agents tab, `-` collapses every open fold in the tribe panel that holds
  focus — from whole-panel focus or from a row inside it — and reverses itself when that
  panel has nothing left to collapse, while `H` on a selected tribe panel hints every
  collapsible fold and collapses exactly the one you pick.

  "
phases:
  - id: sweep
    title: Add the `-` panel fold sweep with a per-panel reverse
    depends_on: []
    size: medium
    description: "sweep: add a configurable `collapse_panel_folds` action bound to `-`
      that saturates every open lane, clan, and top-level grouping banner in the focused
      tribe panel in one press, remembers exactly what it closed in a session-local
      per-panel record, and re-expands that record when the panel has nothing left to
      collapse. Works from whole-panel focus, from a row selection, and in merged
      layout. Resync the footer, command palette, help modal, and docs.

      "
  - id: hint
    title: Give `H` a hinted fold collapse on a selected tribe panel
    depends_on:
      - sweep
    size: medium
    description:
      "hint: replace the whole-panel `H` collapse ladder with a collapse-intent fold
      hint mode — the `L` hint affordance restricted to folds that are currently
      expanded, collapsing the picked fold instead of toggling it. Retire the now-dead
      panel-ladder resolvers, and resync the footer, help modal, keymap labels, and
      docs."
proposed_by: bbugyi200.athena.xo
create_time: 2026-08-10 17:20:10
status: wip
---

- **PROMPT:**
  [prompts/202608/agents_panel_fold_sweep.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agents_panel_fold_sweep.md)

# Plan: Split Agents-tab `H` into a panel fold sweep (`-`) and a hinted fold collapse

## Problem

On the Agents tab the `H` keymap (`hooks_or_collapse_all`) currently means two different
things depending on the selection, and the whole-panel meaning is both hard to reach and
irreversible.

With whole-panel focus on a tribe panel,
`AgentFoldingMixin.action_hooks_or_collapse_all`
(`src/sase/ace/tui/actions/agents/_folding.py:90`) walks a panel-wide collapse ladder,
one rung per press:

1. fully collapse every open canonical lane in the panel,
2. collapse every open canonical clan in the panel,
3. collapse the last expanded level-0 grouping banner, one per press,
4. collapse the panel itself through lowercase `h`'s path.

Three problems follow:

- **It is unreachable from a row.** Every rung bails out unless
  `_resolve_focused_panel()` returns a focus, so tidying the panel you are already
  working inside costs an `h`/`'` round trip out to the panel title and back — the same
  friction the `sase-j2` epic just removed for panel isolation.
- **It is one-way.** Four-plus presses fold a panel down and nothing brings it back. The
  structural fold levels that were open before the sweep are simply lost, so the key is
  risky to press exploratorily.
- **It occupies `H` for a bulk action, leaving no precise one.** `L` (`expand_all_folds`
  → `action_toggle_selected_agent_panels`) already hints every visible fold in the
  selected tribe and toggles the one you pick. There is no collapse-direction
  counterpart, so `H` is all-or-nothing where `L` is surgical.

## Desired behavior

- `-` (new `collapse_panel_folds` action) collapses **every** open fold — lanes, clans,
  and expanded level-0 grouping banners — in the tribe panel that holds focus, in one
  press. It works from whole-panel focus, from an ordinary row/banner selection inside a
  panel, and in merged layout (which is one scope). It never collapses the panel itself.
- When that panel has nothing left to collapse, `-` reverses itself: it re-expands
  exactly the folds its own last sweep closed in that panel, restoring each structural
  fold to the level it held before. Press `-` twice and you are back where you started.
- `H` on a selected tribe panel hints every fold that is currently expanded (the `L`
  affordance, restricted to collapsible targets) and fully collapses the single fold
  whose hint you type.
- `H` on a row, `h`, `l`, `L`, and `=` keep their current behavior exactly.

## Design decisions

**`-` saturates in one press instead of porting the ladder.** The ladder existed because
`H` had to serve both bulk and incremental intent. After this split it does not:
incremental, precise collapsing is `H`'s new hinted mode, and group-scoped laddering
still lives on row-focus `H`. A one-press sweep also makes the reverse discoverable —
the "nothing left to collapse" state is one press away, so `-` `-` reads as a toggle
instead of requiring four presses to find the bounce. The invariant is easy to state and
to test: **`-` reverses exactly when pressing it again would change nothing.**

**The reverse is per-panel and validated at restore time.** Each panel gets at most one
record. The record is replaced by the next sweep in that panel, popped when consumed,
and dropped when the panel stops being live. It is deliberately _not_ eagerly disarmed
by unrelated fold edits (unlike `PanelIsolationRevert`, whose `↺` title markers must be
exact): a fold-granularity marker would be visual noise, so instead the restore filters
to folds that are still live in that panel and still collapsed. That keeps the reverse
forgiving without hooking every fold-mutation path in the app.

**`-` never collapses the panel.** Leaving the panel expanded keeps the sweep's result
visible and reversible, and it keeps the reverse condition unambiguous. Collapsing a
whole panel stays `h`'s job, which already handles selection, isolation, and persistence
for that transition.

**`H`'s hint set is filtered to collapsible folds.** Hinting an already-collapsed fold
would offer a keystroke that does nothing. Filtering also keeps the hint alphabet short
in a panel that is mostly folded already. `L` keeps hinting every fold and toggling,
because expansion is its own inverse there.

**Rust core boundary.** Fold and panel state here is presentation-only Textual state
(the litmus test in `CLAUDE.md`: no other frontend needs to match ACE's transient fold
cursor), which is the same conclusion the `sase-j2` epic reached for panel isolation. No
`sase-core` changes.

**Performance.** Both phases follow `sase/memory/tui_perf.md`: every sweep or restore
applies all fold mutations first and then repaints exactly once through the existing
`_refilter_focused_panel_inner_fold` / `_refilter_agents(refresh_content_index=False)`
paths — no new refresh path. The footer's new availability probe is computed once per
footer update and short-circuits on the first collapsible fold it finds.

## Non-goals

- No change to `L`, to `h`, to `l`, to `=`, or to row-focus `H`'s group-scoped ladder.
- No disk persistence of sweep records; they are session-local, like
  `PanelIsolationRevert`.
- No new panel-title markers.

## Relevant code map

Fold actions and resolvers:

- `src/sase/ace/tui/actions/agents/_folding.py` — `AgentFoldingMixin`;
  `action_hooks_or_collapse_all` holds the whole-panel ladder (lines 90–139) and the
  Tools-detail short circuit.
- `src/sase/ace/tui/actions/agents/_folding_agents.py` — `AgentTreeFoldingMixin`, the
  single-line facade over `AgentGroupFoldingMixin` (the mixin insertion point).
- `src/sase/ace/tui/actions/agents/_folding_agent_groups.py` — `AgentGroupFoldingMixin`:
  `_collapse_agent_lane_folds`, `_refilter_focused_panel_inner_fold`,
  `_resolve_focused_panel_group_collapse_target`, `_collapse_focused_panel_group_fold`,
  `_persist_group_fold_change`, `_snap_focus_after_group_fold_change`,
  `_active_grouping_mode`.
- `src/sase/ace/tui/actions/agents/_folding_lanes.py` —
  `resolve_panel_lane_collapse_target(owner, panel_key)` and
  `AgentPanelLaneCollapseTarget` (panel-keyed, already independent of panel focus).
- `src/sase/ace/tui/actions/agents/_folding_clans.py` —
  `_resolve_panel_clan_collapse_target(owner, panel_key)`,
  `_selected_enclosing_clan_fold_key`, `_collapse_focused_panel_clan_folds`.
- `src/sase/ace/tui/actions/agents/_folding_agent_tree.py` —
  `AgentStructuralFoldingMixin`, `_reanchor_to_fold_owner`.
- `src/sase/ace/tui/actions/agents/_folding_panels.py` — `AgentPanelFoldingMixin` and
  the `PanelIsolationRevert` bookkeeping to mirror.
- `src/sase/ace/tui/actions/agents/_panel_hint_folding.py` —
  `AgentPanelHintFoldingMixin`: `action_toggle_selected_agent_panels`,
  `_enumerate_panel_fold_hint_targets`, `_panel_fold_hint_display_maps`,
  `_handle_panel_fold_hint_key`, `_apply_panel_fold_hint_target`,
  `_teardown_panel_fold_hint_mode`.
- `src/sase/ace/tui/actions/agents/_panel_fold_intent.py` — `effective_panel_collapses`,
  `panel_is_collapsed`, `retire_panel_fold_intents`.
- `src/sase/ace/tui/actions/agents/_selection.py` — `_resolve_focused_panel`.
- `src/sase/ace/tui/actions/agents/_navigation_order.py` — `rendered_panel_slice`.
- `src/sase/ace/tui/actions/agents/_fold_scope.py` — `panel_fold_registry`.

Models:

- `src/sase/ace/tui/models/agent_panels.py` — `PanelIsolationRevert`, `AgentPanelGroup`
  (merged layout is `panel_keys=[None]`).
- `src/sase/ace/tui/models/fold_state.py` — `FoldLevel`, `FoldStateManager` (`get`,
  `expand`, `collapse`, `collapse_fully_all`, `snapshot`).

Hint-mode integration points (Phase 2 must keep all of these working):

- `src/sase/ace/tui/actions/_state_init.py:579-583` — hint-mode state fields.
- `src/sase/ace/tui/actions/_event_keyboard.py:46` — key routing while armed.
- `src/sase/ace/tui/_app_watchers.py:68` — teardown on tab switch.
- `src/sase/ace/tui/actions/agents/_display.py:517` and `_display_panel_widgets.py:264`
  — hint chips replace jump hints.
- `src/sase/ace/tui/actions/agents/_panel_navigation.py:341`,
  `src/sase/ace/tui/actions/navigation/_entry_jump_mode.py:24` — teardown on competing
  modes.
- `src/sase/ace/tui/widgets/_keybinding_modes.py:262` — `update_fold_hint_bindings`.

Keymaps / discoverability (the `sase-j2` `=` migration in commit `5f6d8ea64` is the
exact template for the plumbing set):

- `src/sase/default_config.yml` (`ace.keymaps.app`, lines 380–460),
  `src/sase/ace/tui/bindings.py`, `src/sase/ace/tui/keymaps/metadata.py`
  (`_BINDING_META`), `src/sase/ace/tui/keymaps/app_keymaps.py` (`AppKeymaps`),
  `src/sase/ace/tui/keymaps/key_validation.py` (`_KEY_ALIASES`).
- `src/sase/ace/tui/_app_action_availability.py:184` (tab gating).
- `src/sase/ace/tui/commands/_app_metadata.py`, `commands/availability.py`.
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py`.
- `src/sase/ace/tui/widgets/_keybinding_bindings.py`, `widgets/_keybinding_modes.py`,
  `src/sase/ace/tui/actions/agents/_display_detail_footer.py`.

Docs: `docs/ace.md` (Agents keymap table ~line 918; whole-panel prose ~1355–1400;
three-layer folding table and key table ~1460–1480; `H` ladder prose ~1521–1546) and
`docs/agent_families.md` (~line 179; ~227; ~497–528).

Key-name facts confirmed against the pinned Textual 8.0.1 in this workspace:
`textual.keys._character_to_key("-") == "minus"`, and `Binding.parse_key` runs printable
binding keys through `_character_to_key`, so a raw `-` glyph in a `Binding` is
normalized correctly — exactly like the existing `=` binding. SASE's own
`is_valid_key()` only validates _user overrides_, and it rejects the raw `-` glyph today
(`_KEY_ALIASES` maps only `+`), so Phase 1 adds that alias.

---

## Phase `sweep`: add the `-` panel fold sweep with a per-panel reverse

Purely additive. `H` keeps its current whole-panel ladder throughout this phase, so the
app stays coherent between phases; Phase 2 removes the redundancy.

### 1. Models

`src/sase/ace/tui/models/agent_panels.py`:

- Add, next to `PanelIsolationRevert`:

  ```python
  @dataclass(frozen=True, slots=True)
  class PanelFoldSweepRecord:
      """Folds one ``-`` sweep closed in a panel, remembered for its reverse."""

      panel_key: PanelKey
      agent_levels: tuple[tuple[str, FoldLevel], ...]
      group_keys: tuple[GroupKey, ...]
  ```

  `agent_levels` pairs each structural fold key with the `FoldLevel` it held before the
  sweep (so a `FULLY_EXPANDED` lane comes back fully expanded, not merely expanded).
  `group_keys` are the level-0 banner keys that were expanded. Import `FoldLevel` and
  `GroupKey` under `TYPE_CHECKING`; the module already has
  `from __future__ import annotations`.

- Fix the stale docstring on `PanelIsolationRevert` (line 69): it still says "remembered
  by the `H` solo toggle" — isolation moved to `=` in `sase-j2`.

`src/sase/ace/tui/models/fold_state.py`:

- Add `FoldStateManager.restore_levels(levels: Mapping[str, FoldLevel]) -> bool`,
  writing each exact level and returning whether anything changed. `expand()` cannot
  express this (it steps one level and cannot reach `EXHAUSTIVE`), and the restore must
  land on the remembered level exactly.

### 2. Sweep action

New module `src/sase/ace/tui/actions/agents/_folding_panel_sweep.py` holding
`AgentPanelFoldSweepMixin(AgentGroupFoldingMixin)`. Wire it in by changing
`_folding_agents.py` to `class AgentTreeFoldingMixin(AgentPanelFoldSweepMixin)`, which
keeps the existing linear mixin chain and gives every direct-mixin test harness the new
action for free.

Add to `_folding_agent_groups.py` a module-level
`expanded_panel_level_zero_group_keys(owner, panel_key) -> tuple[GroupKey, ...]`
returning every expanded level-0 banner key in that panel's rendered order, built the
way `_resolve_focused_panel_group_collapse_target` builds its list today
(`rendered_panel_slice` + `build_agent_tree` with an empty `GroupFoldRegistry` +
`panel_fold_registry(...).is_collapsed`). Rewrite that existing resolver to consume the
helper (it just takes the last entry) so there is one enumeration source. The sweep
module imports the helper; the dependency stays one-directional.

`action_collapse_panel_folds()`:

1. Return unless `current_tab == "agents"`.
2. If `_panel_fold_hint_mode_active` is set (reachable via the command palette), tear it
   down first.
3. Resolve the scope. `panel_focus = self._resolve_focused_panel()`; when it is not
   `None` use `panel_focus.panel_key` with `whole_panel_focus=True`; otherwise fall back
   to `self._panel_group.focused_key` with `whole_panel_focus=False`. In merged layout
   `_resolve_focused_panel()` returns `None` and `focused_key` is `None`, which
   `rendered_panel_slice` and `panel_fold_registry` already treat as the single merged
   scope. With no `_panel_group` at all, notify `"No tribe panel to fold"`
   (`severity="warning"`) and return.
4. If `whole_panel_focus` and the panel is collapsed, notify `"Panel is collapsed"`
   (`severity="warning"`) and return — its inner folds are invisible, so neither
   direction should mutate them.
5. Collect collapsible targets for that panel key:
   - lanes: `resolve_panel_lane_collapse_target(self, panel_key)` → `.fold_keys`
   - clans: `_resolve_panel_clan_collapse_target(self, panel_key)` → `.fold_keys`
   - banners: `expanded_panel_level_zero_group_keys(self, panel_key)`

   Lanes and clans are disjoint by construction (a canonical lane owner is never a clan
   container and vice versa), and both resolvers already include owners hidden behind
   collapsed banners and already skip ambiguous or malformed owners per candidate.

6. **Sweep** when anything is collapsible:
   - Build `PanelFoldSweepRecord` from `self._fold_manager.get(key)` for every lane and
     clan key plus the banner keys, and store it under `panel_key`, replacing any prior
     record for that panel.
   - Row focus only: compute a re-anchor index before mutating, mirroring
     `_collapse_agent_lane_folds` — resolve the outermost enclosing structural owner of
     the current selection that is about to close (clan container via
     `_selected_enclosing_clan_fold_key`, else the lane owner via
     `agent_parent_fold_key`), restricted to rows in this panel, and assign
     `self.current_idx` to that owner's global index.
   - Apply every mutation before any repaint:
     `self._fold_manager.collapse_fully_all([*lane_keys, *clan_keys])`, then for each
     banner key `panel_fold_registry(self, panel_key).collapse(key)` followed by
     `self._persist_group_fold_change(key, collapsed=True, panel_key=panel_key)`.
   - Repaint once: `self._refilter_focused_panel_inner_fold(panel_key)` under
     whole-panel focus (it preserves whole-panel focus and the panel's dormant row
     memory); otherwise `self._refilter_agents(refresh_content_index=False)` followed by
     `self._snap_focus_after_group_fold_change()` and
     `self._remember_focused_panel_selection()`.
   - Refresh the footer via `_refresh_agent_footer_bindings_only`, then notify
     `f"Collapsed {n} folds"` (singular `fold`), `timeout=1.5`.
7. **Reverse** when nothing is collapsible:
   - Look up the record for `panel_key`. With no record, notify
     `"No folds to collapse or restore"` (`severity="warning"`) and return.
   - Filter the record: keep `agent_levels` whose key still owns a fold in that panel's
     current `rendered_panel_slice` **and** is currently `COLLAPSED`; keep `group_keys`
     still enumerable as level-0 keys in that panel **and** currently collapsed.
   - Apply `self._fold_manager.restore_levels(...)` and `registry.expand(key)` +
     `self._persist_group_fold_change(key, collapsed=False, panel_key=panel_key)` per
     banner, then pop the record and take the same single-repaint path as the sweep.
   - Notify `f"Restored {n} folds"` (`timeout=1.5`), or, when the filter left nothing,
     pop the record and notify `"No folds to restore"` (`severity="warning"`).

Also in the sweep module:

- `_panel_fold_sweep_records` accessed through a small helper that lazily initializes
  the dict via `getattr(self, "_panel_fold_sweep_records", None)`, matching
  `set_panel_fold_intent`'s defensive pattern, so the many direct-mixin test harnesses
  keep working without edits.
- `_panel_has_collapsible_folds(panel_key) -> bool`, short-circuiting lanes → clans →
  banners, for the footer.
- `_panel_fold_sweep_restore_available(panel_key) -> bool`, applying the same liveness
  filter the reverse uses.
- `retire_panel_fold_sweep_records(owner, live_keys)`, called from
  `_display_panel_collection.py:50` right next to the existing
  `retire_panel_fold_intents(self, live_keys)` so records for panels that stopped
  existing are dropped.

`src/sase/ace/tui/actions/_state_init.py`: initialize
`self._panel_fold_sweep_records: dict[PanelKey, PanelFoldSweepRecord] = {}` beside
`_panel_isolation_revert` (line 567).

Note the sweep must **not** touch whole-panel fold intent, so it must not call
`_note_panel_fold_change` — an armed `=` restore stays armed across a `-` sweep.

### 3. Keymap plumbing

- `src/sase/default_config.yml`: add `collapse_panel_folds: "-"` next to
  `isolate_panels: "="` (this block is not under a keep-sorted directive).
- `src/sase/ace/tui/bindings.py`:
  `Binding("-", "collapse_panel_folds", "Collapse/Restore Panel Folds", show=False)`.
- `src/sase/ace/tui/keymaps/app_keymaps.py`: add the `collapse_panel_folds: str` field.
- `src/sase/ace/tui/keymaps/metadata.py`: add
  `("collapse_panel_folds", "Collapse/Restore Panel Folds", False)` next to
  `isolate_panels`.
- `src/sase/ace/tui/keymaps/key_validation.py`: add `"-": "minus"` to `_KEY_ALIASES` so
  a user rebind spelled with the glyph validates and canonicalizes to Textual's key
  name. (`"="` has the same latent gap; leave it alone here and record a
  `PROPOSED FOLLOW-UP:` note on this phase's bead instead of widening scope.)
- `src/sase/ace/tui/_app_action_availability.py:184`: extend the agents-only set to
  `{"zoom_panel", "isolate_panels", "collapse_panel_folds"}`.
- `src/sase/ace/tui/commands/_app_metadata.py`: add a `Display` entry
  `("collapse_panel_folds", "Collapse or restore tribe panel folds", "Display", AGENTS_ONLY, ("collapse folds", "restore folds", "fold panel"))`.
- `src/sase/ace/tui/commands/availability.py`: in `_agents_available`, add
  `if spec.id == "app.collapse_panel_folds": return panel_focused or agent is not None`
  (mirrors `app.zoom_panel`; no new `CommandContext` field is needed).
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py`: in the "Panel / Group / Clan
  / Workflow Folding" section, add
  `(d(a.collapse_panel_folds), "Collapse panel folds ⇄ restore")`.

### 4. Footer

Per `src/sase/ace/CLAUDE.md`, both states are genuinely conditional, so they belong in
the footer.

- `_display_detail_footer.py`: resolve the target panel key exactly as the action does
  (whole-panel focus, else `panel_group.focused_key`), then compute **once**
  `panel_fold_sweep_available = self._panel_has_collapsible_folds(panel_key)` and, only
  when that is `False`,
  `panel_fold_restore_armed = self._panel_fold_sweep_restore_available(panel_key)`. Skip
  both when the panel is collapsed or no panel group exists. Pass them to
  `update_agent_bindings`.
- `widgets/_keybinding_bindings.py` and `widgets/_keybinding_modes.py`: add the two
  keyword parameters and emit one chip —
  `(self._kd("collapse_panel_folds"), "collapse folds")` when a sweep is available, else
  `(self._kd("collapse_panel_folds"), "restore folds")` when a restore is armed, else
  nothing.

### 5. Docs

- `docs/ace.md`: add `-` to the Agents keymap table (~918) and the folding key table
  (~1477); add a paragraph after the `=` isolation prose (~1392) describing the sweep,
  its scope resolution (whole-panel focus, row selection, merged layout), the fact that
  it never collapses the panel, the reverse and its liveness filter, and the two footer
  chips; note in the three-layer folding table that `-` collapses all three inner layers
  at once.
- `docs/agent_families.md`: extend the fold-prefix prose (~179) and the whole-panel
  focus section (~497–528) with `-`.

### 6. Tests

New `tests/ace/tui/test_agent_panel_fold_sweep.py`, built on the existing
`tests/ace/tui/_agent_panel_collapse_helpers.py` harness:

- Sweeps lanes, clans, and expanded level-0 banners in a single press from whole-panel
  focus, including owners hidden behind collapsed banners (port the batching assertions
  from `test_agent_fold_transitions_panel_clans.py`).
- Same sweep from a row selection inside the panel, with the selection re-anchored to a
  visible owner and the panel's remembered selection updated.
- Merged layout sweeps the merged roster as one scope.
- Sibling panels are untouched; equal group names in another panel keep their state.
- Ambiguous/malformed lane and clan owners are skipped per candidate without blocking
  valid siblings (port from `test_agent_fold_transitions_panel_clans.py`).
- The panel itself is never collapsed by `-`.
- A second press restores every swept fold to its exact prior `FoldLevel`, including a
  `FULLY_EXPANDED` lane, and re-expands the swept banners.
- Restore skips folds whose owners have disappeared and folds the user re-expanded by
  hand; when nothing is restorable the record is dropped with the warning.
- Records are per panel and independent; a fresh sweep replaces that panel's record; a
  panel that stops being live drops its record.
- A collapsed selected panel warns without mutating anything.
- A `-` sweep leaves an armed `=` isolation restore armed.
- Notification text and single-repaint behavior (assert one `_refilter_*` call per
  action).

Updates: `tests/test_keymaps_defaults.py`, `tests/test_keymaps_app_bindings.py`,
`tests/test_keymaps_display_help.py`, `tests/test_keymaps_validation.py` (glyph alias),
`tests/test_command_catalog.py`, `tests/test_command_availability_agents.py`,
`tests/ace/tui/test_agents_panel_fold_footer.py` and/or
`tests/ace/tui/widgets/test_keybinding_footer_tools_detail.py` for the new chip.

### 7. Verification

```bash
just install
just check
just test-visual            # regenerate footer-driven goldens with
                            # --sase-update-visual-snapshots after reviewing each diff
```

Confirm every regenerated PNG diff is confined to the footer/reflow region.

---

## Phase `hint`: give `H` a hinted fold collapse on a selected tribe panel

Depends on `sweep`: the panel-wide collapse must already live on `-` before `H`'s panel
branch is replaced.

### 1. Collapse-intent hint mode

`src/sase/ace/tui/actions/agents/_panel_hint_folding.py`:

- Extract the body of `action_toggle_selected_agent_panels` into
  `_arm_panel_fold_hint_mode(*, intent: Literal["toggle", "collapse"])`;
  `action_toggle_selected_agent_panels` becomes
  `self._arm_panel_fold_hint_mode(intent="toggle")`.
- Store `self._panel_fold_hint_intent` alongside the other hint fields; initialize it to
  `"toggle"` in `_state_init.py:579-583` and reset it in
  `_teardown_panel_fold_hint_mode`.
- `_enumerate_panel_fold_hint_targets(*, collapsible_only: bool = False)` filters out
  targets that are already collapsed: `registry.is_collapsed(group_key)` for banners and
  `self._fold_manager.get(fold_key) == FoldLevel.COLLAPSED` for agent folds. The
  stale-snapshot guard in `_apply_panel_fold_hint_target` must re-enumerate with the
  same flag so the comparison stays apples-to-apples.
- Empty-target messages: keep `"Panel is already collapsed"` for a collapsed panel under
  collapse intent (`H`'s existing wording), and use
  `"No expanded folds in the selected tribe"` for an expanded panel with nothing to
  collapse.
- `_apply_panel_fold_hint_target` under collapse intent never expands: banners take the
  `registry.collapse(...)` branch with its existing persistence and focus snap; agent
  folds take the existing saturating `while` collapse loop. Notify `"Fold collapsed"`.
- `widgets/_keybinding_modes.py`:
  `update_fold_hint_bindings(*, collapse_only: bool = False)` renders
  `mode_label="COLLAPSE"` instead of `"FOLDS"` under collapse intent.

Every existing hint-mode integration point stays untouched and must keep working: key
routing (`_event_keyboard.py:46`), teardown on tab switch (`_app_watchers.py:68`),
teardown from competing modes (`_panel_navigation.py:341`,
`navigation/_entry_jump_mode.py:24`), chip rendering (`_display.py:517`,
`_display_panel_widgets.py:264`), and the auto-refresh guard
(`event_refresh/_auto_refresh.py:190`).

### 2. Retire the whole-panel `H` ladder

`src/sase/ace/tui/actions/agents/_folding.py`, `action_hooks_or_collapse_all`: keep the
Tools-detail short circuit and the collapsed-panel notification, then replace the four
ladder rungs with `self._arm_panel_fold_hint_mode(intent="collapse")`. The row-focus
branch below it is unchanged.

Then delete what that leaves dead, using `just _lint-symvision` to find it — expected:
`_resolve_focused_panel_lane_collapse_target`,
`_resolve_focused_panel_clan_collapse_target`, `_collapse_focused_panel_clan_folds`,
`_resolve_focused_panel_group_collapse_target`, `_collapse_focused_panel_group_fold`,
and the `AgentPanelLaneCollapseTarget` branch inside `_collapse_agent_lane_folds`. Keep
`resolve_panel_lane_collapse_target`, `_resolve_panel_clan_collapse_target`,
`expanded_panel_level_zero_group_keys`, and `_refilter_focused_panel_inner_fold` — the
sweep owns them now.

### 3. Footer, help, and keymap labels

- `_display_detail_footer.py` / `widgets/_keybinding_bindings.py`: under whole-panel
  focus, the `hooks_or_collapse_all` chip becomes a single `collapse fold` label, shown
  when the panel has any collapsible fold (reuse the sweep's
  `_panel_has_collapsible_folds` probe rather than adding a second one). The row-focus
  labels (`collapse lanes` / `collapse clan` / `collapse clans` /
  `collapse <structural>` / `collapse group`) are unchanged, as is the Tools
  `compact tools` precedence.
- `src/sase/ace/tui/keymaps/metadata.py`: retitle `hooks_or_collapse_all` to reflect the
  new split (e.g.
  `"Collapse Scoped Lanes/Clans/Groups / Hint Panel Fold / Compact Tools / All"`), and
  update the matching description string in `bindings.py`.
- `modals/help_modal/agents_bindings.py`: replace the `"Panel: clans / groups / panel"`
  line with `"Panel: collapse fold by hint key"`, keeping it adjacent to `L`'s
  `"Toggle tribe fold by hint key"` so the pair reads together.

### 4. Docs

- `docs/ace.md`: rewrite the whole-panel `H` ordering paragraph (~1537–1546) and the `H`
  rows in the keymap tables (~918, ~1478) to describe the hinted collapse; adjust the
  folding prose at ~1375–1390 that narrates the panel ladder; keep the row-focus ladder
  prose (~1521–1536) intact.
- `docs/agent_families.md`: update the `H` prose at ~179 and ~516–528.

### 5. Tests

- New coverage next to `tests/ace/tui/test_agent_panel_hint_folding.py` (a new
  `test_agent_panel_hint_collapse.py`, or a clearly separated section in the existing
  module if it stays within the file-size budget):
  - `H` on a selected expanded panel arms hint mode with collapse intent and hints only
    folds that are currently expanded.
  - The picked banner collapses (never toggles open) and persists its fold change; the
    picked agent fold lands fully collapsed from `EXPANDED` and from `FULLY_EXPANDED`.
  - `H` on a selected panel with no expanded folds warns without arming; on a collapsed
    panel it keeps the `"Panel is already collapsed"` warning.
  - Escape, unmapped keys, two-character prefixes, and the stale-snapshot guard behave
    as they do for `L`.
  - `L` still hints every fold and still toggles, and arming one mode after the other
    re-enumerates rather than reusing the previous intent.
  - The footer shows mode label `COLLAPSE` while collapse intent is armed.
- Retarget or delete `tests/ace/tui/test_agent_fold_transitions_panel_clans.py` — its
  four tests cover the removed panel ladder, and Phase 1 already ported their
  substantive assertions onto `-`.
- Sweep `tests/` for other assertions on whole-panel `H`
  (`test_agent_fold_transitions_*`, `test_agents_panel_fold_footer.py`,
  `tests/test_keymaps_display_help.py`, `tests/test_keymaps_defaults.py`) and update
  them.

### 6. Verification

```bash
just install
just check
just test-visual            # regenerate footer/mode-label goldens as in Phase 1
just check-full             # before landing the epic's combined tree
```

---

## Risks and mitigations

- **Selection loss on a row-focus sweep.** Collapsing every fold in a panel can hide the
  cursor's row. Mitigated by reusing the existing re-anchor pattern (owner index before
  the refilter, `_snap_focus_after_group_fold_change` after it) and by explicit tests
  for a selection inside a lane, inside a clan, and under a level-0 banner.
- **Footer probe cost.** The sweep availability probe walks the panel slice and builds
  one grouping tree. Mitigated by computing it once per footer update, short-circuiting
  on the first hit, and skipping it entirely for a collapsed panel. If `SASE_TUI_PERF=1`
  j/k p95 regresses past 16 ms on a large roster, memoize the probe on
  `(id(self._agents), panel_key, panel_fold_version_signature(...))`.
- **Stale restore records.** Bounded by the liveness filter at restore time, per-panel
  replacement, and retirement when a panel stops being live; a stale record can never
  resurrect a fold that no longer exists or force-open one the user just opened.
- **PNG golden churn.** Both phases change footer chips, so expect visual-snapshot churn
  like the 23 goldens `sase-j2` regenerated. Each phase regenerates only the goldens its
  own change explains and reviews every diff.
- **Merged layout.** `_resolve_focused_panel()` returns `None` there, so the sweep must
  reach merged mode through `panel_group.focused_key is None`; covered by a dedicated
  test.
