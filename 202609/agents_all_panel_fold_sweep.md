---
tier: tale
title: Add an all-panel fold sweep keymap (`_`) to the Agents tab
goal:
  On the Agents tab, `_` collapses every open agent-node and clan fold across every
  visible tribe panel in one press, and re-expands exactly what the sweep records closed
  once nothing is left to collapse anywhere — working identically from row,
  group-banner, whole-panel, and collapsed-panel focus.
size: medium
proposed_by: bbugyi200.athena.043
create_time: 2026-09-07 14:43:00
status: wip
---

# Plan: Agents-tab all-panel fold sweep (`_`)

## 1. Outcome

Add a new configurable app action `collapse_all_panel_folds`, bound by default to `_`
(Textual `underscore`), available only on the Agents tab. It is the all-panels sibling
of the existing `-` / `collapse_panel_folds` sweep, in the same way `H` is the
saturating sibling of `h`.

Behavior, in one press:

- **Collapse phase** — if _any_ eligible tribe panel still has an open canonical
  agent-node (lane) or clan fold, collapse every such fold in **every** eligible panel,
  recording each panel's prior fold levels so the sweep can be reversed.
- **Restore phase** — otherwise, re-expand every fold that the recorded sweeps closed,
  in every eligible panel, restoring each fold to the exact `FoldLevel` it held before
  (a `FULLY_EXPANDED` node comes back fully expanded, not merely expanded).
- It must behave identically regardless of which tribe panel or node is selected: row
  focus, group-banner focus, whole-panel focus on an expanded panel, whole-panel focus
  on a _collapsed_ panel, and merged (`_agent_panels_grouped`) layout.

"Eligible panel" = a live panel key in `_panel_group.panel_keys` that is not
_effectively collapsed_ (see §3.2). Grouping banners (`Done`, `Running`, …) are never
touched, exactly as with `-`; `H` remains the key for those. `_` never collapses a panel
itself — that stays lowercase `h`'s job.

This is presentation-only Textual state (fold levels, keybindings, footer, panel
titles). It does not cross the Rust-core backend boundary: `FoldStateManager`,
`AgentPanelGroup`, and `PanelFoldSweepRecord` are all local TUI models with no
`sase_core_rs` involvement, so nothing belongs in `../sase-core` for this change.

## 2. Key choice and why `_` is free

`_` canonicalizes to Textual's `underscore`, which is currently the default for the app
action `next_query` (`src/sase/default_config.yml:610`,
`src/sase/ace/tui/bindings.py:239`). That is not a real conflict:

- `action_next_query` (`src/sase/ace/tui/actions/patch/_query.py:358`) returns
  immediately unless `current_tab == "artifacts"` and the pane is `patches`.
- `check_app_action` already hard-gates it: `_ARTIFACT_QUERY_HISTORY_ACTIONS` returns
  `False` when `app.current_tab != ARTIFACTS_TAB`
  (`src/sase/ace/tui/_app_action_availability.py`).

So `underscore` is dead on the Agents tab today. Textual resolves a key to the first
binding whose `check_action` is enabled, and this repo already relies on that for `x`
(`kill_agent` / `toggle_hide_submitted`) and `.` (`toggle_relation_panel` /
`toggle_hide_reverted`). The new action must therefore be registered as an intentional,
tab-disjoint duplicate rather than rebinding anything.

**Do not move or rebind `next_query`.**

## 3. Design

### 3.1 Where the code goes

Extend the existing `AgentPanelFoldSweepMixin` in
`src/sase/ace/tui/actions/agents/_folding_panel_sweep.py` (currently ~285 lines; the
`toobig` gate allows far more). Keeping it in the same module lets the new action reuse
the module-private helpers `_panel_fold_sweep_records()`, `_live_sweep_record_entries()`
and `_global_index_for_fold_owner()` **without** cross-file private imports, which
Symvision rejects (`sase/memory/symvision.md`). Do not create a sibling module for this.

Reuse — do not duplicate — these existing pieces:

- `resolve_panel_lane_collapse_target(self, panel_key)` (`_folding_sase_agents.py`) and
  `resolve_panel_clan_collapse_target(self, panel_key)` (`_folding_clans.py`). Both
  already take an explicit panel key, already validate candidate owners against the full
  `_agents_with_children` projection, and already reject fold keys that are ambiguous
  across panels. Nothing about them needs to change.
- `self._fold_manager.collapse_fully_all(keys)` and
  `self._fold_manager.restore_levels(mapping)` — call each **once** for the whole
  cross-panel key set, so one fold mutation drives one repaint.
- `PanelFoldSweepRecord` and the `_panel_fold_sweep_records` store. **The new action
  writes into the same per-panel record store that `-` uses.** That is what makes the
  two keys compose (§3.5) and it makes the existing `▿` / `▿N` restore markers and the
  `retire_panel_fold_sweep_records()` lifecycle work for free.
- `self._panel_sweep_reanchor_index(...)` for selection re-anchoring.
- `self._repaint_after_panel_fold_sweep(...)` for the single repaint + footer refresh.

### 3.2 Eligible panels

```python
panel_group = getattr(self, "_panel_group", None)          # None -> warn, return
collapsed = effective_panel_collapses(self, panel_group.panel_keys)
target_keys = [k for k in panel_group.panel_keys if k not in collapsed]
```

`effective_panel_collapses` is already imported into this package via
`._panel_fold_intent`. In merged layout it returns an empty set and `panel_keys` is
`[None]`, so `_` degenerates to exactly `-`'s merged-roster behavior.

Skipping effectively-collapsed panels is deliberate and must be preserved in **both**
phases:

- A collapsed panel renders no rows, so sweeping its folds would report work the user
  cannot see — and the very next `_` would silently reverse it. `-` already refuses on a
  collapsed panel ("Panel is collapsed"), and `_panel_fold_restore_marked_keys()`
  already skips collapsed panels for the same reason.
- A collapsed panel's sweep record is _not_ discarded. It stays live in
  `_panel_fold_sweep_records` and becomes reachable again by `-` or `_` as soon as the
  panel is expanded.

If `target_keys` is empty, warn `"All tribe panels are collapsed"` and return.

### 3.3 `action_collapse_all_panel_folds`

```
if self.current_tab != "agents": return
if self._panel_fold_hint_mode_active: self._teardown_panel_fold_hint_mode()
resolve panel_group / target_keys per §3.2 (warn + return on the two empty cases)

panel_focus      = self._resolve_focused_panel()
whole_panel_focus = panel_focus is not None and not panel_focus.collapsed
focused_key      = panel_group.focused_key

# --- resolve every panel's targets up front, before mutating anything ---
per_panel: list[(panel_key, lane_keys, clan_keys)] for panel_key in target_keys
           where lane_keys or clan_keys is non-empty

if per_panel:      -> collapse phase
else:              -> restore phase
```

Crucially, `panel_focus` is resolved **once**, and the collapsed-focused-panel case must
_not_ take `-`'s early `"Panel is collapsed"` return — `_` still has work to do in the
other panels. This is the single most important difference from
`action_collapse_panel_folds` and is the crux of the "works regardless of selection"
requirement.

**Collapse phase**

1. For each `(panel_key, lane_keys, clan_keys)`, write
   `PanelFoldSweepRecord(panel_key, agent_levels=tuple((k, self._fold_manager.get(k)) for k in (*lane_keys, *clan_keys)))`
   into `_panel_fold_sweep_records(self)`, **replacing** any prior record for that
   panel. Panels with nothing to collapse keep whatever record they already had.
2. Re-anchor the selection _before_ any fold mutates, and only when
   `not whole_panel_focus`: call
   `self._panel_sweep_reanchor_index(focused_key, lane_keys=…, clan_keys=…)` using the
   focused panel's own resolved key tuples (empty tuples when the focused panel
   contributed nothing), and assign `self.current_idx` when it returns an index. Row
   focus always implies the focused panel is expanded and eligible — see
   `_resolve_focused_panel` in `_selection.py`, which returns whole-panel focus whenever
   the panel is collapsed or `_expanded_panel_focus` is set — so the focused panel is
   the only panel whose rows can hold the cursor.
3. `self._fold_manager.collapse_fully_all([...every lane and clan key from every panel...])`
   — one call.
4. Repaint (§3.4).
5. `self.notify(f"Collapsed {total} {fold_noun} in {count} {panel_noun}", timeout=1.5)`
   where `total` is the fold count, `count` is the number of panels swept, and the nouns
   singularize (`fold`/`folds`, `panel`/`panels`).

**Restore phase**

1. For each eligible panel with a record, compute
   `_live_sweep_record_entries(self, panel_key, record)`.
2. Drop records for eligible panels whose live entries are empty (they are stale — the
   owners disappeared or the user re-expanded them by hand). This mirrors what `-` does
   in `_restore_panel_fold_sweep`.
3. If no eligible panel yields live entries, warn `"No folds to collapse or restore"`
   and return without repainting.
4. `self._fold_manager.restore_levels(dict(all_entries_across_panels))` — one call —
   then delete every restored panel's record.
5. Repaint (§3.4).
6. `self.notify(f"Restored {total} {fold_noun} in {count} {panel_noun}", timeout=1.5)`.

### 3.4 Repaint and focus settling

Call the existing helper with the focused panel key:

```python
self._repaint_after_panel_fold_sweep(focused_key, whole_panel_focus=whole_panel_focus)
```

with `whole_panel_focus` computed as in §3.3
(`panel_focus is not None and not panel_focus.collapsed`). That gives the three correct
behaviors:

- **Expanded whole-panel focus** → `_refilter_focused_panel_inner_fold(focused_key)`,
  which preserves whole-panel focus and the panel's remembered row.
- **Row or banner focus** → `_refilter_agents(refresh_content_index=False)` +
  `_snap_focus_after_group_fold_change()` + `_remember_focused_panel_selection()`.
- **Collapsed whole-panel focus** → deliberately routed down the same branch as row
  focus. `_remember_focused_panel_selection()` self-guards (it returns early when
  `_resolve_focused_panel()` is not `None`), and `_snap_focus_after_group_fold_change()`
  is a no-op-or-clear for a selection that is not in the focused panel's slice. Do
  **not** send this case through `_refilter_focused_panel_inner_fold`, because
  `_restore_focused_panel_inner_fold_state` force-sets `_expanded_panel_focus = True`,
  which would leak expanded-panel focus onto a panel the user has collapsed.

`_refilter_agents` re-renders every panel widget, so one call covers all panels; the
helper already refreshes the footer via `_refresh_agent_footer_bindings_only`.

### 3.5 How `-` and `_` compose

Because both write the same per-panel records:

- `-` on panel A, then `_`: panel B still has open folds, so `_` collapses B and writes
  B's record; A keeps its record. A second `_` restores A **and** B.
- `_`, then `-` on the focused panel: `-` restores just that panel; a following `_`
  restores the remaining panels.
- After `_`'s collapse phase, every swept panel satisfies
  `_panel_fold_restore_marked_keys()`'s conditions, so every swept panel shows its gold
  `▿N` title marker and per-row `▿` markers with no extra work.

No changes are needed to `_panel_fold_restore_marked_keys`,
`_panel_has_collapsible_folds`, `_panel_fold_sweep_restore_available`, or
`retire_panel_fold_sweep_records`.

## 4. Wiring checklist

Every item below is required; the keymap registry raises at startup if
`default_config.yml` and `AppKeymaps` disagree.

1. **`src/sase/ace/tui/keymaps/key_validation.py`** — add `"_": "underscore"` to
   `_KEY_ALIASES`, with a comment mirroring the existing `-`/`+`/`$` entries. Without
   this, a config value of `"_"` fails `is_valid_key` (it is not alphanumeric and is not
   a Textual key _name_).
2. **`src/sase/default_config.yml`** — add `collapse_all_panel_folds: "_"` immediately
   below `collapse_panel_folds: "-"` (line 638), inside `ace.keymaps.app`. (Required by
   the core `gotchas` memory: keymap changes must update `default_config.yml`.)
3. **`src/sase/ace/tui/keymaps/app_keymaps.py`** — add `collapse_all_panel_folds: str`
   to `AppKeymaps`, directly after `collapse_panel_folds`.
4. **`src/sase/ace/tui/keymaps/metadata.py`** — add
   `("collapse_all_panel_folds", "Collapse/Restore All Panel Folds", False)` to
   `_BINDING_META`, directly after the `collapse_panel_folds` entry.
5. **`src/sase/ace/tui/bindings.py`** — add
   `Binding("_", "collapse_all_panel_folds", "Collapse/Restore All Panel Folds", show=False)`
   to `DEFAULT_BINDINGS`, directly after the `-` binding (line 43). Description text
   must match item 4 exactly; `tests/test_keymaps_app_bindings.py` asserts the runtime
   and fallback descriptions agree.
6. **`src/sase/ace/tui/keymaps/registry.py`** — add
   `frozenset({"next_query", "collapse_all_panel_folds"})` to
   `_CONTEXTUAL_APP_DUPLICATES`, with a comment explaining the disjointness
   (`next_query` is Artifacts-only, `collapse_all_panel_folds` is Agents-only). Without
   this, any user override that lands both actions on one key would be reverted with a
   spurious duplicate-key warning.
7. **`src/sase/ace/tui/_app_action_availability.py`** — add `"collapse_all_panel_folds"`
   to the existing `{"zoom_panel", "isolate_panels", "collapse_panel_folds"}` set that
   returns `False` off the Agents tab. Add **no** focus requirement: unlike
   `collapse_panel_folds`, this action must stay enabled with no selected agent and no
   panel focus. That is what lets the `underscore` key resolve to this action on Agents
   and to `next_query` on Artifacts.
8. **`src/sase/ace/tui/commands/_app_metadata_actions.py`** — register the palette
   command after `collapse_panel_folds`:
   `("collapse_all_panel_folds", "Collapse or restore folds in every tribe panel", "Display", AGENTS_ONLY, ("collapse all folds", "restore all folds", "fold all panels", "sweep all panels"))`.
9. **`src/sase/ace/tui/commands/_availability_agents.py`** — no rule needed; the
   catalog's `AGENTS_ONLY` tab list plus the trailing `return True` are correct, because
   the command is runnable from any selection. Confirm no earlier generic rule
   (`_REQUIRES_AGENT`, `_COLLAPSED_PANEL_HIDDEN_AGENT_COMMANDS`, …) accidentally catches
   the new id.
10. **`src/sase/ace/tui/modals/help_modal/agents_bindings.py`** — add
    `(d(a.collapse_all_panel_folds), "All panels: collapse folds ⇄ restore ▿")`
    immediately after the existing `collapse_panel_folds` row (line ~207).

### 4.1 Footer chip

Surface the key the same way `-` and `=` are surfaced, but only when it is not merely a
duplicate of `-` — i.e. only when at least two panels are eligible under §3.2.

- **`src/sase/ace/tui/actions/agents/_display_detail_footer.py`** — next to the existing
  `panel_fold_sweep_available` / `panel_fold_restore_armed` computation (~lines
  138-156), compute `all_panel_fold_sweep_available` and `all_panel_fold_restore_armed`
  by folding `_panel_has_collapsible_folds` / `_panel_fold_sweep_restore_available` over
  the eligible panel keys, and set both to `False` when fewer than two panels are
  eligible. Pass them through the `update_agent_bindings(...)` call (~line 275).
- **`src/sase/ace/tui/widgets/_keybinding_modes.py`** — thread the two new keyword
  arguments through the `update_agent_bindings` protocol stub (~line 52) and
  implementation (~lines 133, 165).
- **`src/sase/ace/tui/widgets/_keybinding_bindings.py`** — in `_compute_agent_bindings`
  (~line 224), after the existing `-` chip, emit
  `(self._kd("collapse_all_panel_folds"), "collapse all folds")` when
  `all_panel_fold_sweep_available`, else `(…, "restore all folds")` when
  `all_panel_fold_restore_armed`. Add the two parameters to the signature (~line 118).

## 5. Documentation

- **`docs/ace.md`** — after the `-` paragraph (ends ~line 1808), add a paragraph for
  `_`: the two phases, that it skips collapsed panels and never collapses a panel
  itself, that it shares the per-panel sweep records with `-` so the two compose, that
  it works from any selection including a collapsed focused panel, and the
  `_ collapse all folds` / `_ restore all folds` footer labels.
- **`docs/agent_families.md`** — add the matching shorter paragraph after the `-`
  paragraph (ends ~line 577).
- No `CHANGELOG.md` edit: `tools/validate_changelog` requires the file to contain only
  release-please sections.

## 6. Tests

Add a new file `tests/ace/tui/test_agent_panel_fold_sweep_all.py` (keeps
`test_agent_panel_fold_sweep.py`, already 583 lines, clear of the `toobig` thresholds).
Build it on the existing harness: `AgentPanelCollapseApp` from
`tests/ace/tui/_agent_panel_collapse_helpers.py`, plus `_named_workflow_lane`-style lane
builders, `_clan_member` / `_clan_container`, and `_populate_fold_counts` — copy or lift
those helpers from `test_agent_panel_fold_sweep.py`. Give every lane a unique
`raw_suffix` so its fold key is unique across panels; a fold key that owns rows in two
panels is correctly rejected by the resolvers and would silently produce an empty sweep.

Cover at minimum:

1. One press collapses open lane **and** clan folds in **every** eligible panel.
2. A second press restores every panel's record, and a fold that was `FULLY_EXPANDED`
   comes back `FULLY_EXPANDED`, not `EXPANDED`.
3. Works from row focus inside one panel, and re-anchors `current_idx` onto the
   enclosing fold owner's row when the selected row is folded away.
4. Works from group-banner focus (`_current_group_key` set).
5. Works from expanded whole-panel focus (`_expanded_panel_focus = True`) without
   dropping whole-panel focus.
6. **Works when the focused panel is collapsed** (`_collapsed_panel_keys = {key}`): the
   other panels still sweep, no `"Panel is collapsed"` warning is emitted, and
   `_expanded_panel_focus` is not flipped to `True` by the repaint.
7. Effectively-collapsed panels are skipped in both phases: their folds stay open, and
   an existing record of theirs survives a restore press aimed at the visible panels.
8. Composition with `-`: sweep panel A with `-`, then `_` collapses only panel B; a
   further `_` restores both. And: `_`, then `-` restores only the focused panel, then
   `_` restores the rest.
9. Merged layout (`AgentPanelCollapseApp(..., merged=True)`) treats the merged roster as
   one scope.
10. `_panel_group is None` warns and does not raise; every panel collapsed warns
    `"All tribe panels are collapsed"`; nothing to do anywhere warns
    `"No folds to collapse or restore"`.
11. Notification strings and their singular/plural forms.
12. After a global sweep, `_panel_fold_restore_marked_keys()` names every swept panel.
13. `collapse_fully_all` / `restore_levels` are each invoked once per press (assert via
    the harness's recorded refilter/refresh calls, so the "one mutation, one repaint"
    property is locked in).

Extend the existing suites:

- `tests/test_keymaps_defaults.py` — assert
  `reg.app.collapse_all_panel_folds == "underscore"` in
  `test_zoom_and_agents_fold_defaults_are_in_sync_with_help`, and add the help-modal
  pair assertion for the new `agents_bindings` row.
- `tests/test_keymaps_app_bindings.py` — extend
  `test_h_binding_metadata_describes_navigation_and_contextual_collapse` with the new
  action's description in both `DEFAULT_BINDINGS` and `build_app_bindings`.
- `tests/test_keymaps_validation.py` — a default registry load emits no duplicate-key
  warning for `underscore`, and a user override that puts `collapse_all_panel_folds` on
  `next_query`'s key is accepted rather than reverted.
- `tests/test_command_catalog.py` — a `test_collapse_all_panel_folds_command_…`
  mirroring `test_collapse_panel_folds_command_is_agents_only_display_command`,
  asserting `key_sequence == ("underscore",)`, `key_display == "_"`,
  `tabs == ("agents",)`, category `Display`, and the aliases.
- `tests/test_command_availability_agents.py` — available for
  `CommandContext(tab="agents")` with no agent, no panel focus, and with
  `group_focused=True`; unavailable on other tabs.
- `tests/ace/tui/test_agents_zoom_panel_action.py` (or a sibling using the same
  `check_action` pattern) — the action is enabled on Agents and disabled on Artifacts,
  and `underscore` still reaches `next_query` on the Artifacts/Patches pane.
- `tests/ace/tui/widgets/test_keybinding_footer_tools_detail.py` — chip labels and
  precedence for the new kwargs, and that no chip is emitted when fewer than two panels
  are eligible.
- `tests/ace/tui/test_agent_display_defer_detail.py` — the footer probe passes the two
  new kwargs (mirror
  `test_footer_refresh_uses_panel_fold_sweep_probe_during_whole_panel_focus`).

### 6.1 PNG snapshot

`tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py::test_tribe_panel_fold_sweep_armed_png_snapshot`
renders the footer with two panels present and a `-` restore armed, so the new `_` chip
will change `agents_panel_fold_sweep_armed_120x40`. This is an intentional visual
change: run `just test-visual` to confirm that is the only golden that moves, then
accept it with `--sase-update-visual-snapshots` and commit the refreshed PNG. Inspect
`.pytest_cache/sase-visual/` if anything else drifts.

## 7. Verification

Per `sase/memory/lint_and_test.md`:

```bash
just install                      # ephemeral sase_<N> clone may have drifted deps
just check                        # every lint gate + diff-scoped tests
just test-visual                  # PNG suite; then re-run with
                                  # --sase-update-visual-snapshots to accept §6.1
```

Run `just check-full` through the `/sase_monitor` skill (never inline) before landing,
since this change touches the keymap registry, the command catalog, the footer, and the
visual suite. Expect `symvision` to be quiet: every new symbol is either a mixin method
reached by Textual's action dispatch (like the existing `action_collapse_panel_folds`)
or a module-private helper used in its own file.

## 8. Explicitly out of scope

- Rebinding, moving, or changing `next_query`, `collapse_panel_folds`, `isolate_panels`,
  `hooks_or_collapse_all`, or any fold-mode (`z…`) key.
- Touching grouping-banner folds — `_` deliberately leaves those to `H`, matching `-`.
- Collapsing or expanding whole panels — that stays `h` / `l` / `=` / `L`.
- Persisting sweep records across sessions; like `-`'s records, they stay session-local.
- Any change in `../sase-core` (see §1).
