---
tier: tale
title: Scope Agents-tab `L` fold hints to the selected tribe and toggle on one keypress
goal: 'On the Agents tab, `L` hints only the expandable/collapsable fold owners inside
  the currently selected tribe panel (never the tribe panel itself), and one subsequent
  keypress toggles that fold immediately with no input bar and no Enter.

  '
create_time: 2026-07-25 10:37:32
status: done
---

- **PROMPT:** [202607/prompts/tribe_scoped_single_key_fold_hints.md](prompts/tribe_scoped_single_key_fold_hints.md)
- **AGENTS:**
  - [bbugyi200.athena.kq](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.kq/README.md)
  - [bbugyi200.athena.kq--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.kq.md#member-code)
- **COMMITS:**
  - [266eac0](https://github.com/sase-org/sase/commit/266eac076ab2986a6153fd5f2818761d4b11d109) — feat(ace): add tribe-scoped fold hints

# Plan

## Context

`L` on the Agents tab is bound to `expand_all_folds` (`src/sase/ace/tui/bindings.py:176`) and routes to
`action_toggle_selected_agent_panels()` (`src/sase/ace/tui/actions/agents/_folding.py:128`), implemented in
`src/sase/ace/tui/actions/agents/_panel_hint_folding.py`.

Today it:

1. Enumerates fold owners across **every** panel: one `("panel", key)` target per split tribe panel, plus every visible
   group banner and structural/workflow row fold owner inside each expanded panel.
2. Mounts a `HintInputBar(mode="panels")`, waits for the user to type a numeric selection (`1 3 5`, `1-4`), and applies
   the whole set atomically on Enter.

Both behaviors change:

- **Scope**: hint only the fold owners (group banners + agent/clan/family/workflow fold rows) of the panel the user
  currently has focused — `self._panel_group.focused_key`. Drop `("panel", …)` targets entirely; the tribe panel itself
  is excluded, and no other tribe's folds are hinted.
- **Input**: no input bar. Hints become jump-style base-62 labels allocated by `build_jump_hint_maps()`
  (`src/sase/ace/tui/actions/navigation/jump_hints.py:63`), and a single keypress dispatched through `App.on_key`
  toggles exactly one fold, then exits hint mode. This mirrors the existing apostrophe jump mode, which already proves
  that app-level `on_key` sees hint characters while an `AgentList` panel holds focus.

Multi-select and range selection are intentionally removed: one key = one toggle. Panel folding keeps its existing homes
(`l` expands/enters a panel, `h`/`H` collapse), so nothing is lost by dropping panel targets from `L`.

Whole-panel `L` behavior on the **other** tabs (axe, changespecs) and the Tools-detail routing in `_folding.py` are
unchanged.

## Design decisions

- **Hint alphabet**: reuse `JUMP_HINT_CHARS` / `build_jump_hint_maps()` / `match_jump_hint()` from
  `navigation/jump_hints.py` instead of numeric hints. Up to 62 targets get true one-character hints (the first ten are
  `0`–`9`, so the common case still reads as digits); beyond 62 the shared helper falls back to two-character hints with
  a pending prefix, exactly like jump mode. Do **not** invent a second hint allocator.
- **One toggle, then exit.** After a completed hint the fold is applied and hint mode tears down. Re-press `L` to toggle
  another fold. Rationale: applying a fold re-renders rows and invalidates the snapshot, so keeping the mode armed would
  show stale hints.
- **Invalid key exits silently**, matching `_handle_entry_jump_key` (`navigation/_entry_jump_dispatch.py:126`). `escape`
  cancels.
- **Fold hint mode stops piggybacking on `_hint_mode_active` / `_hint_mode_hints_for`.** Those flags mean "a
  `HintInputBar` is mounted" and are consumed by `_hint_input_bar_active()`
  (`src/sase/ace/tui/actions/hints/_types.py:56`), which suppresses panel focus in `_display_panel_layout.py:205,276`.
  With no bar, fold hint mode must leave panel focus alone, so it keeps using its own `_panel_fold_hint_mode_active`
  flag only, and auto-refresh suppression gets an explicit check.
- **Selected tribe = `self._panel_group.focused_key`**, not `_resolve_focused_panel()` (which only returns a value for
  collapsed or explicitly whole-panel-focused panels). In merged (`_agent_panels_grouped`) layout there is a single
  panel and the behavior naturally degrades to "folds of the merged panel", which is already what happens today.

## Implementation

### 1. `src/sase/ace/tui/actions/agents/_panel_hint_folding.py` (core rewrite)

**Types**

- Delete `PanelFoldHintTarget`; `FoldHintTarget = GroupFoldHintTarget | AgentFoldHintTarget`.
- Keep the `PanelKey` element in the group/agent target tuples (still needed for
  `panel_fold_registry(self, panel_key)`), even though it is now constant.
- Class attribute annotations become `_panel_fold_hint_to_target: dict[str, FoldHintTarget]`,
  `_panel_fold_target_to_hint: dict[FoldHintTarget, str]`, plus a new `_panel_fold_hint_pending_prefix: str`.

**`_enumerate_panel_fold_hint_targets()`**

- Keep rebuilding the live `AgentPanelGroup.from_agents(...)` (it is the staleness oracle).
- Resolve `focused_key = getattr(self._panel_group, "focused_key", None)`; return `()` when the live panel key list is
  empty or does not contain `focused_key`.
- Return `()` when `panel_is_collapsed(self, focused_key)` — a collapsed panel has no visible members.
- Walk only that one panel: `panel_fold_registry`, `rendered_panel_slice`, `build_agent_tree` exactly as today, and keep
  the existing dedupe/skip rules (`is_workflow_step_child` rows skipped; `fold_key is None` or missing from
  `_fold_counts` skipped unless `is_clan_container`).
- Emit no `("panel", …)` targets at all.

**`action_toggle_selected_agent_panels()`**

- Drop the `_refocus_existing_hint_bar()` early return. Instead:
  - If another hint bar is open (`_hint_input_bar_active()`, via `getattr` since mixin composition is loose), return
    without arming — the bar owns the keyboard.
  - If fold hint mode is already active (only reachable from the command palette, since `on_key` swallows `L` while
    armed), tear it down first and re-arm with a fresh snapshot.
- Enumerate targets; on empty, notify and return with a message that distinguishes the two causes:
  `"Selected tribe panel is collapsed"` when `panel_is_collapsed(self, focused_key)`, otherwise
  `"No folds in the selected tribe"` (both `severity="warning"`).
- Keep the existing "leave apostrophe jump mode first" guard (`_entry_jump_mode_active` → `_exit_entry_jump_mode()`) so
  the two hint namespaces never coexist.
- Allocate `hint_to_target, target_to_hint = build_jump_hint_maps(list(targets))`; store the snapshot, both maps, and
  `_panel_fold_hint_pending_prefix = ""`; set `_panel_fold_hint_mode_active = True`.
- Do **not** set `_hint_mode_active` / `_hint_mode_hints_for`, and do not mount `HintInputBar`. The mount/`except` path
  and its `"Fold selector is unavailable"` notification are deleted.
- Repaint hints, then set the footer to the new fold-hint mode (see §7).

**`_panel_fold_hint_display_maps()`**

- Return a 2-tuple `(agent_hints, banner_hints)`; the panel channel is gone. Banner keys stay
  `("banner", panel_idx, group_key)` with `panel_idx` looked up from `self._panel_group.panel_keys`.

**`_refresh_panel_fold_hint_display()`**

- Refresh only the focused panel's widget: `self._refresh_affected_panel_widgets({focused_key})` (per the TUI-perf rule
  to prefer selective updates over full rebuilds), falling back to `self._refresh_agents_display(list_changed=True)` as
  today when the selective path is unavailable or returns False.

**`_teardown_panel_fold_hint_mode(*, refresh_titles: bool = True)`**

- Clear `_panel_fold_hint_mode_active`, `_panel_fold_hint_snapshot`, both maps, and the pending prefix.
- Remove the `HintInputBar` query/removal block and the `_hint_mode_active` / `_hint_mode_hints_for` resets.
- On `refresh_titles`, repaint via `_refresh_panel_fold_hint_display()` and restore the normal footer through
  `_refresh_agent_footer_bindings_only()` (guarded with `callable(...)`, as elsewhere in this file).

**New `_handle_panel_fold_hint_key(self, key: str) -> bool`**

```
if not self._panel_fold_hint_mode_active: return False
if key == "escape": teardown; return True
match = match_jump_hint(self._panel_fold_hint_to_target, self._panel_fold_hint_pending_prefix, key)
PENDING  -> store prefix; return True
INVALID  -> teardown; return True
COMPLETE -> clear prefix; self._apply_panel_fold_hint_target(match.target); return True
```

**New `_apply_panel_fold_hint_target(target)`** (replaces `_process_panel_fold_hint_input`)

- Staleness guard first, unchanged in spirit: re-enumerate live targets; if
  `snapshot != live_targets or target not in live_targets`, tear down and notify
  `"Visible folds changed; retry fold selection"`.
- Group branch: `registry = panel_fold_registry(self, panel_key)`; expand when `registry.is_collapsed(group_key)` else
  collapse; on change call `_persist_group_fold_change(group_key, collapsed=…, panel_key=panel_key)` and, when the panel
  is not collapsed, `_snap_focus_after_group_fold_change()`.
- Agent branch: identical to today's `FoldLevel.COLLAPSED` expand / collapse-to-COLLAPSED loop on `_fold_manager`.
- Tear down with `refresh_titles=False`, `_invalidate_agent_panel_cache()`, then the single repaint:
  `_refilter_agents(refresh_content_index=False)` when an agent fold changed, else
  `_refresh_agents_display(list_changed=True)` — same as today. Restore the footer via
  `_refresh_agent_footer_bindings_only()`.
- Notify `"Fold expanded"` / `"Fold collapsed"` (`timeout=1.5`), or `"No fold change"` when the registry/fold manager
  reported no change.

**Deletions**: `_process_panel_fold_hint_input`, `_notify_invalid_panel_hints`, the
`from ....hints import parse_numeric_hint_selection` import, the `set_panel_fold_intent` import, and the
`focused_panel_toggled` / `current_attempt_number` / `_current_group_key` reset block (panel folds are no longer
reachable from this module). `parse_numeric_hint_selection` itself stays — `hints/_processing.py` still uses it. Update
the module and class docstrings to describe tribe-scoped single-key folding.

### 2. `src/sase/ace/tui/actions/_event_keyboard.py`

Add a dispatch branch in `on_key`, immediately after the `self._entry_jump_mode_active` branch and before
`_fold_mode_active`:

```python
elif getattr(self, "_panel_fold_hint_mode_active", False):
    if self._handle_panel_fold_hint_key(key):
        event.prevent_default()
        event.stop()
```

Use the already-computed `key = normalize_jump_key(event.key, event.character)` so shifted hint letters resolve
case-sensitively, exactly like jump mode.

### 3. `src/sase/ace/tui/actions/_state_init.py:540-545`

Retype the initial state to `dict[str, FoldHintTarget]` / `dict[FoldHintTarget, str]`, add
`self._panel_fold_hint_pending_prefix: str = ""`, and rewrite the comment above them (it currently says "Numeric,
set-oriented fold mode … covers panel titles") to describe the tribe-scoped single-key mode.

### 4. Retire the `"panels"` hint-bar mode

- `src/sase/ace/tui/widgets/hint_input_bar.py`: drop `"panels"` from `HintInputMode`, its `compose()` branch (`Folds:`
  label + placeholder) and its `_build_prompt_text()` branch.
- `src/sase/ace/tui/actions/hints/_processing.py`: delete the `event.mode == "panels"` branch in
  `on_hint_input_bar_submitted` (`:105`) and the `_panel_fold_hint_mode_active` branch in `on_hint_input_bar_cancelled`
  (`:132`).

### 5. Display plumbing

- `src/sase/ace/tui/actions/agents/_display.py:516-523` and
  `src/sase/ace/tui/actions/agents/_display_panel_widgets.py:263-270`: unpack the new 2-tuple and set
  `panel_jump_hints = None` while fold hint mode is active.
- `src/sase/ace/tui/actions/agents/_display_panel_collection.py:105-125`: delete the `selection_hint` lookup and stop
  passing `selection_hint=` to `agent_panel_border_title`.
- `src/sase/ace/tui/actions/agents/_display_panel_titles.py:94,101`: remove the now-unused `selection_hint` parameter so
  the title helper takes only `jump_hint` (keeps symvision clean).

### 6. Flag-conflation cleanups

- `src/sase/ace/tui/actions/event_refresh/_auto_refresh.py:186`: add an explicit
  `if getattr(self, "_panel_fold_hint_mode_active", False): return` next to the `_entry_jump_mode_active` guard, so a
  background refresh can no longer invalidate armed fold hints now that `_hint_mode_active` is not set.
- `src/sase/ace/tui/actions/agents/_display_detail_render.py:238-243`: `_should_render_agent_detail_with_hints` can no
  longer see `_hint_mode_hints_for in {"panels", "folds"}` (nothing assigns either value once §1 lands — the only
  remaining assignments are `None`, `"mentors_manage"`, `"hooks_latest_only"`). Reduce the predicate to
  `self.current_tab == "agents" and bool(self._hint_mode_active)` and update its docstring.

### 7. Footer mode for fold hints

- `src/sase/ace/tui/widgets/_keybinding_modes.py`: add `update_fold_hint_bindings()` next to `update_jump_bindings`
  (`:231`), displaying `[("<esc>", "cancel")]` with `mode_label="FOLDS"`.
- Call it from `action_toggle_selected_agent_panels()` (via `query_one("#keybinding-footer", KeybindingFooter)` inside a
  `try/except`, mirroring `_update_jump_footer`), replacing the current `_refresh_agent_footer_bindings_only()` call on
  entry; teardown restores the normal footer as described in §1.

### 8. Keymap docs and labels (all must stay in sync)

- `src/sase/default_config.yml:245-250`: rewrite the `L` sentence in the fold comment block — currently "L selects
  visible folds by numeric hint" → "L hints the selected tribe's folds; one key toggles one fold".
- `src/sase/ace/tui/bindings.py:176`: description → `"Toggle Tribe Fold / Expand All"`.
- `src/sase/ace/tui/keymaps/metadata.py:114`: label → `"Tribe Folds / Expand All"`.
- `src/sase/ace/tui/commands/_app_metadata.py:330-331`: label →
  `"Toggle a fold in the selected tribe / expand all folds on other tabs"`.
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py:148-150`: description → `"Toggle tribe fold by hint key"` (29
  chars, within the 32-char help-modal cap from `src/sase/ace/CLAUDE.md`).

## Tests

- **`tests/ace/tui/test_agent_panel_hint_folding.py`** — the main rewrite. `_StubApp` needs `_panel_fold_hint_to_target`
  keyed by `str`, a `_panel_fold_hint_pending_prefix`, and an `arm_hints()` that allocates through
  `build_jump_hint_maps`. Cover:
  - enumeration contains **no** `("panel", …)` targets and only targets whose panel key equals `focused_key` (agents in
    other tribes contribute nothing);
  - the existing workflow-dedupe and clan/family fold-owner cases, re-asserted under the focused panel;
  - a collapsed focused panel yields `()` and `action_toggle_selected_agent_panels()` notifies
    `"Selected tribe panel is collapsed"` without arming;
  - an expanded focused panel with no fold owners notifies `"No folds in the selected tribe"`;
  - `_handle_panel_fold_hint_key` — one key toggles exactly one group fold (persistence intent recorded, single repaint)
    and exits; `escape` tears down; an unmapped key tears down without mutating fold state;
  - a two-character allocation (>62 targets, or a monkeypatched narrow alphabet) leaves a pending prefix on the first
    key and completes on the second;
  - the stale-snapshot abort still fires and notifies `"Visible folds changed; retry fold selection"`;
  - `_panel_fold_hint_display_maps()` now returns two channels and never hints a panel title. Delete the input-bar tests
    (`test_mixed_panel_group_and_agent_submission_is_atomic`,
    `test_invalid_submission_keeps_mode_open_and_changes_nothing`, the `mode == "panels"` assertion in
    `test_entry_snapshots_all_visible_targets_and_reentry_reuses_bar`) and the `HintInputBar` import/`_MountTarget`
    harness.
- **`tests/ace/tui/test_agent_panel_titles.py:365`** — delete
  `test_panel_title_renders_multi_digit_numeric_selection_hint` (the parameter is gone). The `jump_hint` title tests
  stay.
- **`tests/ace/tui/test_agents_view_hint_survives_refresh.py:201`** — the
  `@pytest.mark.parametrize("hint_namespace", ["panels", "folds"])` case no longer describes reachable state; remove it
  (the `hint_mode_active=False` and `mentors`/`hooks` cases still cover the predicate).
- **`tests/test_command_catalog.py:414-417`** and **`tests/test_keymaps_display_help.py:60-67`** — update to the new
  labels from §8.
- **`tests/ace/tui/test_agent_fold_transitions_tools.py`** — `action_expand_all_folds()` still routes to the selector on
  the agents tab; confirm `fold_selector_calls` bookkeeping still holds and adjust only if the stub touches removed
  attributes.
- **`tests/ace/tui/visual/test_ace_png_snapshots_agents_panels.py:304-360`** — rewrite the fold-selection segment:
  - `@chop` is the focused panel and is _effectively collapsed_ at that point, so press `l` first (and wait for the
    expansion) before `L`; otherwise the new code correctly refuses to arm.
  - Replace the `HintInputBar` / `Input` / `wait_for_svg_contains(page, "Folds:")` assertions with: every panel border
    title is hint-free, `_panel_fold_target_to_hint` holds only `"chop"`-scoped group/agent targets, and the `@chop`
    rows/banners render `[<hint>]` chips.
  - Drive the toggle with a single `await page.press(<hint>)` and assert the group collapsed and
    `_panel_fold_hint_mode_active` is False.
  - The follow-on assertions that depended on `L` toggling `("panel", None)` / `("panel", "chop")`
    (`_collapsed_panel_keys == {None}`, panel ordering, the click-to-focus and `h` checks at `:336-361`) must reach the
    same end state through panel keys (`J`/`K` + `h`/`l`) instead; keep their intent and update the concrete values to
    whatever the new sequence actually produces.
  - Remove the now-unused `HintInputBar` / `Input` imports.
  - Regenerate the `agents_panel_fold_selection_120x40` golden with `just test-visual -- --sase-update-visual-snapshots`
    (keep the snapshot name) and eyeball the artifacts under `.pytest_cache/sase-visual/` before accepting.

## Verification

1. `just install` (ephemeral workspace may have stale deps), then `just check`.
2. `just test-visual` after regenerating the golden; confirm a clean rerun without the update flag.
3. Manual `sase ace` smoke on the Agents tab:
   - split panels, focused tribe expanded → `L` paints hints only inside that tribe, panel titles stay bare, one key
     toggles one fold, footer shows the `FOLDS` mode and returns to normal afterwards;
   - focused tribe collapsed → `L` warns and does not arm;
   - `L` then `escape` restores the previous rendering;
   - `L` then `'` (apostrophe) hands the keyboard cleanly to jump mode;
   - merged (`p`-toggled) layout still hints the merged panel's folds.

## Out of scope

- `L` behavior on the axe and changespecs tabs, and the Tools-detail routing in `_folding.py`.
- Panel (whole-tribe) folding, which stays on `l` / `h` / `H` / `Z`.
- Any change to apostrophe jump mode's target set or its hint rendering.
