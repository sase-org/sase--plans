---
tier: tale
title: ',H leader chord: collapse any fold by hint across the selected tribe or every
  tribe'
goal: Agents-tab users can press ,H from any row to hint-collapse one expanded fold
  in the selected tribe panel, or from a selected tribe panel to hint-collapse any
  expanded fold or panel across every tribe panel, with stable focus, clear chips,
  and a scoped footer.
size: medium
proposed_by: bbugyi200.apollo.u
status: done
---

# Plan: `,H` — Collapse Any Fold By Hint (Selected Tribe, or Every Tribe)

## Goal

Add an Agents-tab leader chord `,H` that opens the same "collapse one fold by hint"
picker that whole-panel `H` opens today, reachable from any selection, and widen its
scope by one level when a tribe panel itself is selected:

| Selection when `,H` is pressed                                     | Hint scope                         | Targets hinted                                                                                                                                  |
| ------------------------------------------------------------------ | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Any row or banner in a panel                                       | **Selected tribe** (focused panel) | Exactly what whole-panel `H` hints: every expanded agent-node / clan / workflow / family fold owner and top-level grouping banner in that panel |
| A tribe panel (whole-panel `❖`/`▸` focus, expanded _or_ collapsed) | **All tribes**                     | The above for **every** effectively expanded panel, in render order, **plus** each expanded panel's title chip (collapses that whole panel)     |

Mental model the design is built around: **`,H` is always one scope wider than your
selection.** A row selects a tribe's contents; a selected tribe selects all tribes. This
mirrors the existing `-` (one panel) / `_` (all panels) sweep pair, so it reads as a
natural extension rather than a new concept.

Picking a hint fully collapses that one entry and exits. It never expands anything.
`Esc` cancels; an unmapped key exits silently; a stale snapshot aborts with the existing
`Visible folds changed; retry fold selection` warning.

Why title chips appear in the all-tribes scope: the user explicitly selected the tribe
level, and "every chip you see collapses the thing it sits on" is the most intuitive
rule for that level. It is also the only one-keystroke way to collapse a _different_
panel without navigating to it. The selected-tribe scope never offers the title chip,
matching whole-panel `H` (panel collapse from inside stays lowercase `h`'s job).

## Background (current code, verified)

- Hint picker: `src/sase/ace/tui/actions/agents/_panel_hint_folding.py`
  (`AgentPanelHintFoldingMixin`). `_arm_panel_fold_hint_mode(intent=...)` enumerates via
  `_enumerate_panel_fold_hint_targets(collapsible_only=...)` (focused panel only),
  allocates with `build_jump_hint_maps`, sets `_panel_fold_hint_*` state, repaints via
  `_refresh_panel_fold_hint_display()` (selective, `{focused_key}` only), and calls
  `footer.update_fold_hint_bindings(collapse_only=...)`. `_apply_panel_fold_hint_target`
  re-enumerates to verify the snapshot, then mutates.
- Whole-panel `H`: `action_hooks_or_collapse_all` in
  `src/sase/ace/tui/actions/agents/_folding.py` calls
  `_arm_panel_fold_hint_mode(intent="collapse")` when `_resolve_focused_panel()` is
  non-`None` and not collapsed. `_resolve_focused_panel()` (in `_selection.py`) returns
  `None` in merged layout and for ordinary row focus.
- Chip rendering: agent rows / banners get hints through
  `_panel_fold_hint_display_maps()`, consumed in two places that currently force
  `panel_jump_hints = None` while fold-hint mode is active:
  `_refresh_affected_panel_widgets` in `_display_panel_widgets.py` and the
  `list_changed` path in `_display.py`. Panel titles already support a `[x] `
  bold-yellow chip via `panel_jump_hints[("panel", key)]` →
  `agent_panel_border_title(jump_hint=...)` (used by `'` jump mode).
- Panel collapse: `_collapse_focused_panel()` and the generic
  `_apply_panel_fold_layout(live_keys, desired_collapsed)` in `_folding_panels.py` (sets
  intent, notes the change so isolation disarms, invalidates cache, repaints).
- Leader dispatch: `_dispatch_leader_key` in
  `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`. `,H` was previously the
  leader id `toggle_selected_agent_panels` (moved to `L` in commit `415704d977`); that
  id is in `_RETIRED_LEADER_KEYS` (`keymaps/registry.py`) and must stay retired — the
  new command gets a **new** id. `H` is currently unbound in leader mode.
- Row re-anchoring helpers: `_reanchor_to_fold_owner(fold_key)` and the
  `_remember_focused_panel_selection()` follow-up used by
  `_collapse_agent_structural_fold` (`_folding_agent_tree.py`);
  `selected_enclosing_clan_fold_key` (`_folding_clans.py`); `tree_parent_lookup` /
  `agent_parent_fold_key` / `agent_fold_key` (`models/_agent_tree.py`).
- Footer: `update_fold_hint_bindings` in `widgets/_keybinding_modes.py`;
  `_apply_agent_footer_update` in `actions/agents/_display_detail_footer.py` has mode
  branches (member-jump, fold, leader, bang, copy, ...) but **no** fold-hint branch, so
  an incidental footer refresh during hint mode currently replaces the `COLLAPSE` /
  `FOLDS` footer. Fix this as part of the work (it matters more once the picker spans
  all panels and stays open while auto-refresh ticks land).

## Design

### 1. Command identity and keymap plumbing

- New leader action id **`collapse_fold_by_hint`**, default key `H`.
  - `src/sase/default_config.yml` → `keymaps.modes.leader_mode.keys`: add
    `collapse_fold_by_hint: "H"` (place it next to `toggle_agent_panel_grouping`). Also
    extend the fold/collapse comment block above `hooks_or_collapse` /
    `hooks_or_collapse_all` with one sentence: `,H` opens the same collapse hints from
    any row (selected tribe) and, from a selected panel, across every tribe panel.
  - `src/sase/ace/tui/keymaps/mode_keymaps.py` → `LeaderModeKeymaps` defaults: same key.
  - `src/sase/ace/tui/commands/_mode_commands.py`:
    `_LEADER_LABELS["collapse_fold_by_hint"] = "Collapse a fold by hint"`,
    `_LEADER_TABS[...] = AGENTS_ONLY`, and an `_LEADER_ALIASES` entry such as
    `("fold", "hint", "collapse", "tribe", "panels")`.
  - Do **not** touch `_RETIRED_LEADER_KEYS`; `toggle_selected_agent_panels` stays
    retired.
- Leader dispatch branch in `_dispatch_leader_key` (near `toggle_agent_panel_grouping`):
  remember the key (so `,,` repeats it and re-resolves scope against the _current_
  selection); on non-Agents tabs call `_refresh_current_tab()` and return. On Agents,
  call `self.action_collapse_fold_by_hint()`; then, **only if hint mode did not arm**
  (`not self._panel_fold_hint_mode_active`), call
  `self._refresh_agent_footer_bindings_only()` to replace the stale LEADER footer. Do
  **not** call `_refresh_current_tab()` after a successful arm — the arm path already
  painted chips and the `COLLAPSE` footer.
- Leader footer (`update_leader_bindings` in `widgets/_keybinding_modes.py`): on the
  Agents tab append `(k("collapse_fold_by_hint"), "collapse by hint")` right after the
  `group panels` entry (the leader footer is a mode menu; like `g`, it is shown
  unconditionally on Agents).

### 2. Action and scope resolution

In `AgentPanelHintFoldingMixin` add:

```python
def action_collapse_fold_by_hint(self) -> None:
    """Hint-collapse one fold: focused tribe from a row, every tribe from a panel."""
    if self.current_tab != "agents":
        return
    panel_focus = self._resolve_focused_panel()
    scope = "all" if panel_focus is not None else "tribe"
    self._arm_panel_fold_hint_mode(intent="collapse", scope=scope)
```

- Unlike `H`, `,H` does **not** route to Tools compaction first; it is an explicit
  hint-picker entry. (`H` behavior is unchanged.)
- Merged layout never has whole-panel focus, so `,H` there is always selected-tribe
  scope over the single merged panel — which already covers every agent.
- Also register the action in the command-metadata/availability surfaces only if the
  existing leader-command catalog requires it; the palette already executes leader
  commands through `_handle_leader_key`, so no app-level binding is added.

### 3. State

- New attribute `_panel_fold_hint_scope: Literal["tribe", "all"]`, initialized to
  `"tribe"` in `src/sase/ace/tui/actions/_state_init_agents.py` beside
  `_panel_fold_hint_intent`, declared on the mixin, set in `_arm_panel_fold_hint_mode`,
  reset to `"tribe"` in `_teardown_panel_fold_hint_mode`.
- `_arm_panel_fold_hint_mode(self, *, intent, scope="tribe")`. Guard: `scope == "all"`
  is only meaningful with `intent == "collapse"`; coerce/assert so `L` stays tribe-only.
- Extend the target type union:
  `type PanelFoldHintTarget = tuple[Literal["panel"], "PanelKey"]` and
  `FoldHintTarget = GroupFoldHintTarget | AgentFoldHintTarget | PanelFoldHintTarget`.

### 4. Enumeration (render order = hint order)

Refactor `_enumerate_panel_fold_hint_targets(*, collapsible_only=False, scope="tribe")`:

- Build `live_panel_group` once (as today). Extract the existing per-panel tree walk
  into a helper, e.g.
  `_panel_fold_hint_targets_for(panel_key, *, collapsible_only, seen_actions) -> list[FoldHintTarget]`,
  which keeps every current rule (workflow step rows skipped, `fold_counts` gating, clan
  containers, collapsed owners skipped when `collapsible_only`, `seen_actions`
  de-duplication) but uses the given `panel_key` instead of the focused key.
- `scope == "tribe"`: identical to today — focused key only, `()` when the focused panel
  is missing or collapsed. Existing tests must pass unchanged.
- `scope == "all"`: iterate `live_panel_group.panel_keys` in order; skip effectively
  collapsed panels (`effective_panel_collapses`). For each expanded panel emit, in this
  order:
  1. `("panel", key)` — only when not merged **and** there are at least two live panel
     keys (a lone panel cannot collapse; mirrors the documented "collapsing requires
     multiple panels" rule — confirm against the lowercase `h` path and match it).
  2. that panel's banner / fold-owner targets from the helper. Share one `seen_actions`
     set across panels.
- Hints come from `build_jump_hint_maps(list(targets))` exactly as today, so chips read
  top-to-bottom `0 1 2 … a b …` down the screen, and sessions over 62 targets use the
  same two-character alphabet as `'` jump mode (docs already describe it).
- Perf note (see TUI perf memory): enumeration is keypress-triggered, in-memory, and
  bounded by visible rows — same class of work as `_`. It must not be called from any
  render path. The footer's existing `panel_hint_collapse_available` probe keeps calling
  the tribe-scope default only.

### 5. Messages when nothing can be hinted

In `_arm_panel_fold_hint_mode`, when enumeration is empty:

- tribe scope: keep today's messages byte-for-byte.
- all scope: if every live panel is effectively collapsed →
  `All tribe panels are collapsed` (same wording as `_`); otherwise →
  `No expanded folds in any tribe panel`. Severity `warning`, no arming, no repaint.

### 6. Rendering: chips on titles, repaint the right panels

- Add `_panel_fold_hint_title_map() -> dict[PanelJumpTarget, str]` returning
  `{("panel", key): hint}` for `panel` targets (empty in tribe scope). Keep
  `_panel_fold_hint_display_maps()`'s 2-tuple return as-is and make it ignore `panel`
  targets explicitly.
- In `_refresh_affected_panel_widgets` (`_display_panel_widgets.py`) and the
  `list_changed` path in `_display.py`, replace `panel_jump_hints = None` in the
  fold-hint branch with `self._panel_fold_hint_title_map() or None`. The existing `[x] `
  bold-yellow title chip then renders with no widget changes, e.g. `[0] ❖ 🔧 @chop · 3`
  on the selected panel and `[4] @review · 2` on another. If a title-only repaint path
  (`_refresh_agent_panel_titles`) also reads entry-jump panel hints, give it the same
  fold-hint fallback so chips never flicker away.
- `_refresh_panel_fold_hint_display()`: affected keys are `{focused_key}` in tribe scope
  and **every** live panel key in all scope (rows, banners, and titles in other panels
  must gain/lose chips). Call it with the scope still set during teardown (capture scope
  before clearing state, or clear state after computing the key set). Keep the
  full-refresh fallback.

### 7. Footer while picking

- `update_fold_hint_bindings(self, *, collapse_only=False, all_tribes=False)`: mode
  label `COLLAPSE · ALL TRIBES` when `all_tribes`, else unchanged (`COLLAPSE` /
  `FOLDS`); bindings stay `[("<esc>", "cancel")]`. The arm path passes
  `all_tribes=(scope == "all")`.
- Add a fold-hint branch to the mode chain in `_apply_agent_footer_update`
  (`_display_detail_footer.py`), placed with the other transient-mode branches, that
  re-renders
  `update_fold_hint_bindings(collapse_only=intent == "collapse", all_tribes=scope == "all")`
  while `_panel_fold_hint_mode_active`. This keeps the picker footer stable across
  auto-refresh ticks for `L`, `H`, and `,H`.
- Normal Agents footer / help: no new conditional footer chip is needed for `,H` (it is
  a leader command; conditional chips belong to bare keys per `src/sase/ace/CLAUDE.md`).

### 8. Applying a pick (reliability rules)

Extend `_apply_panel_fold_hint_target` (snapshot check re-enumerates with the stored
`intent` **and** `scope`):

- **Group target** `("group", panel_key, group_key)`: unchanged mutation/persistence,
  but call `_snap_focus_after_group_fold_change()` only when `panel_key == focused_key`
  and the panel is not collapsed — the snap helper operates on the focused panel's
  context and must not rewrite `_current_group_key` because a banner closed in some
  other panel.
- **Agent target** `("agent", panel_key, global_idx, fold_key)`: before mutating, if the
  user has **row focus** (`_resolve_focused_panel() is None`) and the selected row would
  be hidden by this fold, call `_reanchor_to_fold_owner(fold_key)`; after the
  `_refilter_agents(refresh_content_index=False)` repaint, call
  `_remember_focused_panel_selection()` when a re-anchor happened. "Would be hidden" =
  the selected row is not itself the owner and either
  `selected_enclosing_clan_fold_key(self._agents, self.current_idx) == fold_key` or
  walking tree parents from the selected row (via `tree_parent_lookup` /
  `agent_parent_fold_key`; use `agent_gating_fold_key` for monitor/gate rows as
  `_resolve_agent_structural_collapse_target` does) reaches a row whose `agent_fold_key`
  equals `fold_key`. Put this in a small private helper. Because the apply path is
  shared, this also fixes the same latent jump for `L` picks made from row focus — cover
  it with a test.
- **Panel target** `("panel", panel_key)` (all scope only; whole-panel focus is
  guaranteed): tear down chips first (`refresh_titles=False`), then
  - if `panel_key == focused_key`: `_collapse_focused_panel()` — focus becomes the
    collapsed whole-panel `▸` selection, exactly like `h`;
  - else: `_apply_panel_fold_layout(live_keys, effective_collapses | {panel_key})` with
    `live_keys` from the live panel group — focus stays on the selected panel (row
    indices are flat, so collapsing a sibling cannot invalidate selection). Refresh
    footer bindings, toast `Panel collapsed` (timeout 1.5).
- After any fold/banner pick in all scope, whole-panel focus must remain on the same
  panel (`_resolve_focused_panel()` unchanged) — the user can immediately press `,,` to
  pick the next thing to close.
- Toasts: keep `Fold collapsed` / `No fold change`.

### 9. Help and docs

- Help modal `src/sase/ace/tui/modals/help_modal/agents_bindings.py` (respect the
  57-char box / ≤32-char description rules):
  - Leader section:
    `key_sequence_display(lm.prefix, sk(lm.keys, "collapse_fold_by_hint"))` →
    `Collapse fold by hint`.
  - Fold/collapse section, right after the `Panel: collapse fold by hint key` row: the
    same `,H` sequence → `Row: tribe hints; panel: all` (or equivalent ≤32 chars).
- `docs/ace.md`:
  - Agents-tab leader table (the second `### Leader Mode (`,` prefix)` section): add a
    `,H` row — "Collapse one fold by hint: the selected tribe's expanded folds from a
    row; every tribe's expanded folds and panel titles from a selected panel".
  - Agents fold key table (the `l`/`h`/`L`/`H`/`=`/`-` table): add a `,H` row.
  - Whole-panel `H` prose ("Whole-panel focus gives `H` a hinted collapse…"): add a
    short paragraph describing `,H`'s two scopes, title chips, focus retention,
    `COLLAPSE · ALL TRIBES` footer, and `,,` repeat.
- `docs/agent_families.md`: the paragraph beginning "The `,H` leader chord numbers every
  currently toggleable visible fold owner…" is stale (describes the retired numeric
  selector). Replace it with an accurate description of the new `,H`, and add one
  sentence to the uppercase-`H` paragraph pointing at `,H` for any-selection /
  all-tribes collapse.
- No CHANGELOG hand edits unless the repo's changelog lint requires them for new keys
  (check `just lint` output).

## Tests

Keep new test modules small (split rather than grow a file past the size gates).

1. **New** `tests/ace/tui/test_agent_panel_hint_collapse_scopes.py` — reuse `_agent`,
   `_EntryApp`, `_FooterStub` from `test_agent_panel_hint_folding.py` and the
   `_PanelFocusEntryApp` pattern from `test_agent_panel_hint_collapse.py` (extend the
   stubs with `_panel_fold_hint_scope`, `all_tribes` capture on the footer stub, and
   stubs for `_collapse_focused_panel` / `_apply_panel_fold_layout` /
   `_reanchor_to_fold_owner` / `_remember_focused_panel_selection` as needed):
   - row focus → `scope == "tribe"`, snapshot equals whole-panel `H`'s collapsible set
     for the focused panel, footer `all_tribes is False`.
   - whole-panel focus (expanded) → `scope == "all"`; targets span alpha and beta; order
     is `("panel","alpha")`, alpha folds…, `("panel","beta")`, beta folds…; collapsed
     panels contribute nothing; footer `all_tribes is True`.
   - whole-panel focus on a **collapsed** panel still arms all scope over the other
     expanded panels.
   - single live panel → no `panel` targets; merged layout → tribe scope, no `panel`
     targets.
   - all panels collapsed → `All tribe panels are collapsed`, not armed; expanded panels
     with nothing open and only one panel → `No expanded folds in any tribe panel`.
   - picking a banner in a non-focused panel collapses it in that panel's registry,
     persists with the right `panel_key`, does **not** call the focus snap, keeps
     whole-panel focus.
   - picking an agent fold in a non-focused panel lands `COLLAPSED`.
   - picking the focused panel's title calls the focused-panel collapse path; picking
     another panel's title applies a layout collapsing only that key; toast
     `Panel collapsed`.
   - stale snapshot in all scope (add an agent to beta) aborts with the retry warning.
   - `_panel_fold_hint_title_map()` has entries only for `panel` targets;
     `_panel_fold_hint_display_maps()` ignores them.
   - teardown repaints every live panel key in all scope and `{focused}` in tribe scope,
     and resets scope to `"tribe"`.
   - row-focus re-anchor: selected workflow child / clan member hidden by the picked
     fold → selection moves to the owner and is remembered; selected row not inside the
     fold → no re-anchor. Include one `L` (toggle intent) row-focus case.
2. `tests/ace/tui/test_leader_keymap_dispatch.py` + `_leader_keymap_helpers.py`: replace
   `test_leader_h_uppercase_no_longer_dispatches_selected_panel_toggle` /
   `..._noops_on_non_agents_tabs` with: `,H` on Agents invokes
   `action_collapse_fold_by_hint` once, records `H` so `,,` invokes it again, and
   refreshes the footer only when hint mode did not arm; `,H` on other tabs no-ops with
   one `_refresh_current_tab`. Keep an assertion that the retired
   `toggle_selected_agent_panels` action is never called.
3. `tests/test_keymaps_defaults.py`: default
   `leader_mode.keys["collapse_fold_by_hint"] == "H"` in both the loaded registry and
   `LeaderModeKeymaps()`; a stale user override of `toggle_selected_agent_panels` is
   still dropped (extend the existing retired-key test if one exists).
4. `tests/ace/tui/test_leader_keybinding_footer.py`: `collapse by hint` present on the
   Agents leader footer, absent on other tabs.
5. `tests/test_command_catalog_build.py` / `tests/test_command_catalog.py`:
   `leader.collapse_fold_by_hint` exists, label as above, Agents-only.
6. Help: a `tests/test_keymaps_display_help.py`-style assertion that both `,H` rows
   render with the configured prefix/key (e.g. with `leader_mode.prefix: semicolon`).
7. Footer widget: `update_fold_hint_bindings(collapse_only=True, all_tribes=True)`
   yields mode label `COLLAPSE · ALL TRIBES`; and `_apply_agent_footer_update` keeps the
   fold-hint footer while `_panel_fold_hint_mode_active`.
8. **PNG visual snapshot** in
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_panels.py` (or a sibling module
   if that file is near the size gate), modeled on the existing
   `agents_panel_fold_selection_120x40` flow: with ≥2 expanded tribe panels containing
   folds, climb to whole-panel focus, press `,` then `H`, wait for
   `_panel_fold_hint_mode_active`, assert chips appear on rows/banners in more than one
   panel and on expanded panel titles (`[x] ` prefix), assert footer
   `([("<esc>", "cancel")], "COLLAPSE · ALL TRIBES")`, capture
   `agents_panel_fold_collapse_all_tribes_120x40`, then press a title chip for a
   non-focused panel and assert it collapsed while whole-panel focus stayed put.
   Generate the golden with `--sase-update-visual-snapshots` and inspect the PNG for
   legibility (chip alignment on titles, no clipped borders) before accepting.

## Verification

- `just install` if the workspace venv is stale.
- `just fmt`.
- `just check` (whole-repo lint gates incl. mypy/symvision/toobig + scoped tests); fix
  every failure.
- `just test-visual -k agents_panel` (or the full `just test-visual`) for the new and
  existing panel hint snapshots.
- Manual smoke (optional but recommended): `sase ace -t agents` with multiple tribes;
  `,H` from a row (tribe scope), `h` to a panel then `,H` (all scope, title chips,
  footer label), pick a chip in another panel, `,,` to repeat, `Esc` to cancel.

## Out of Scope

- Changing bare `H`, `L`, `-`, or `_` semantics (beyond the shared footer-stability and
  row-focus re-anchor fixes in the apply/footer paths).
- Moving any of this into `sase-core`: this is presentation-only Textual selection and
  fold UI state.
- Resurrecting the retired `toggle_selected_agent_panels` leader id.
