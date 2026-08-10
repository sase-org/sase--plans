---
tier: epic
title: Split Agents-tab `Z` into panel isolation (`=`) and tribe-aware zoom
goal: "On the Agents tab, `=` isolates or restores tribe panels from any selection
  (whole-panel focus or a row inside a panel), and `Z` opens the zoom modal for agent
  rows, clan containers, agent lanes, and selected tribe panels alike.

  "
phases:
  - id: isolate
    title: Move panel isolation onto a new `=` keymap
    depends_on: []
    size: medium
    description: "isolate: add the configurable `isolate_panels` action bound to `=`,
      stop `action_zoom_panel` from owning whole-panel isolation, broaden isolate and
      restore so they work from an in-panel row selection, and resync the footer, help
      modal, command palette, and docs.

      "
  - id: tribezoom
    title: Zoom the tribe metadata document from whole-panel focus
    depends_on:
      - isolate
    size: medium
    description:
      "tribezoom: teach ZoomPanelModal a metadata-only tribe mode fed by a tribe
      snapshot provider, route `action_zoom_panel` to it when a tribe panel is selected,
      keep the zoomed document live through tribe enrichment, and resync the affected
      help, command palette, and docs copy."
proposed_by: bbugyi200.athena.xh
create_time: 2026-08-10 14:07:55
status: wip
---

- **PROMPT:**
  [prompts/202608/tribe_zoom_and_panel_isolation_keymap.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/tribe_zoom_and_panel_isolation_keymap.md)

# Plan: Split Agents-tab `Z` into panel isolation (`=`) and tribe-aware zoom

## Problem

On the Agents tab the `Z` keymap (`zoom_panel`) currently does two unrelated things,
chosen by whichever selection is active:

1. With an agent row selected (a plain agent, a clan container, or an agent lane
   container), it opens `ZoomPanelModal` on the largest detail panel.
2. With whole-panel focus on a tribe panel, `AgentPanelDetailMixin.action_zoom_panel`
   calls `_isolate_focused_panel()` first; that helper takes ownership of the key and
   either collapses every sibling tribe panel or restores the remembered pre-isolation
   layout.

Two problems follow from that overload:

- Isolation is only reachable from whole-panel focus, because `_isolate_focused_panel`
  bails out when `_resolve_focused_panel()` returns `None`. Isolating the panel you are
  already working inside costs an extra `h`/`'` round trip out to the panel title and
  back.
- Zoom is unreachable from whole-panel focus, so the tribe metadata document — the
  richest document ACE renders for a tribe — is the one document that cannot be zoomed,
  searched full-screen, copied, or opened in `$EDITOR` from the zoom modal.

## Desired behavior

- `=` (new `isolate_panels` action) isolates the tribe panel that currently holds focus,
  or restores the remembered layout when a restore is armed. It works from whole-panel
  focus **and** from an ordinary row/banner selection inside a panel.
- `Z` (`zoom_panel`) always zooms: agent rows and clan/lane containers behave exactly as
  they do today, and whole-panel focus on a tribe panel (expanded or collapsed) zooms
  that tribe's metadata document.

## Relevant code map

Isolation:

- `src/sase/ace/tui/actions/agents/_folding_panels.py` — `AgentPanelFoldingMixin` owns
  `_isolate_focused_panel`, `_apply_panel_fold_layout`,
  `_focus_whole_panel_after_layout`, and the `PanelIsolationRevert` bookkeeping
  (`_panel_isolation_revert_record`, `_marked_keys_for_panel_isolation`,
  `_disarm_panel_isolation_revert`, `_refresh_panel_isolation_ui`).
- `src/sase/ace/tui/actions/agents/_panel_detail.py` — `action_zoom_panel` calls
  `_isolate_focused_panel()` before the modal path.
- `src/sase/ace/tui/actions/agents/_selection.py` — `_resolve_focused_panel`,
  `_focused_tribe_summary`, `_get_selected_agent`.
- `src/sase/ace/tui/actions/agents/_panel_fold_intent.py` — effective collapse state.

Keymaps / discoverability:

- `src/sase/default_config.yml` (`ace.keymaps.app`), `src/sase/ace/tui/bindings.py`,
  `src/sase/ace/tui/keymaps/metadata.py` (`_BINDING_META`),
  `src/sase/ace/tui/keymaps/app_keymaps.py` (`AppKeymaps`).
- `src/sase/ace/tui/_app_action_availability.py` (tab gating).
- `src/sase/ace/tui/commands/_app_metadata.py`, `commands/availability.py`,
  `commands/types.py`, `commands/context.py` (command palette).
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py` (help modal).
- `src/sase/ace/tui/widgets/_keybinding_bindings.py`, `widgets/_keybinding_modes.py`,
  `src/sase/ace/tui/actions/agents/_display_detail_footer.py` (conditional footer).

Zoom modal:

- `src/sase/ace/tui/modals/zoom_panel_modal.py` (screen + bindings),
  `zoom_panel_types.py` (`ZoomPanelTarget`, `ZoomPanelSeed`), `zoom_panel_content.py`
  (seed + refresh + copy/edit), `zoom_panel_navigation.py` (targets, file reveal,
  scrolling), `zoom_panel_rendering.py` (`agent_label`, `status_text`),
  `zoom_panel_search.py` (search overlay; no agent coupling).
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display.py` —
  `AgentPromptPanel.update_tribe_display`.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py` —
  `build_tribe_detail_text`.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_async_groups.py` —
  `start_tribe_section_enrichment`, `_cancel_tribe_section_worker_for_agent_selection`,
  and the `TribeSectionSnapshotLoaded` message.
- `src/sase/ace/tui/models/agent_tribe_summary.py` — `AgentTribeSummarySnapshot`
  (`label`, `status`, `container_identity`, `panel_collapsed`, …).

Docs: `docs/ace.md` (Agents keymap table, whole-panel focus prose, three-layer folding
table, zoom-modal section) and `docs/agent_families.md` (fold-prefix prose and the
whole-panel focus section).

---

## Phase `isolate`: Move panel isolation onto a new `=` keymap

### Goal

Introduce a configurable `isolate_panels` action, bind it to `=` by default, remove
isolation from `action_zoom_panel`, and make isolate/restore work from an in-panel row
or banner selection as well as from whole-panel focus.

### Keymap plumbing

Add one new configurable app action named `isolate_panels`, placed immediately after
`zoom_panel` in every ordered list so the runtime bindings and the fallback bindings
stay aligned:

1. `src/sase/default_config.yml`, under `ace.keymaps.app`, next to `zoom_panel: "Z"`:

   ```yaml
   zoom_panel: "Z"
   isolate_panels: "="
   ```

   `=` is currently unbound at the app level and `equals_sign` is already a known key
   name in `src/sase/ace/tui/keymaps/key_validation.py` (`_KEY_DISPLAY`), so
   `canonicalize_key_binding` and `footer_key_display` need no change.

2. `src/sase/ace/tui/keymaps/app_keymaps.py`: add `isolate_panels: str` after
   `zoom_panel: str`. `AppKeymaps` has no defaults on purpose — a field without a
   `default_config.yml` entry fails at startup, so both edits must land together.

3. `src/sase/ace/tui/keymaps/metadata.py`: add
   `("isolate_panels", "Only/Restore Panels", False)` after the `zoom_panel` entry, and
   retitle `zoom_panel` from `"Zoom Detail / Only/Restore Panels"` to `"Zoom Detail"`.

4. `src/sase/ace/tui/bindings.py`: mirror both changes in `DEFAULT_BINDINGS` — retitle
   the `Z` binding to `"Zoom Detail"` and add
   `Binding("=", "isolate_panels", "Only/Restore Panels", show=False)` right after it.
   Textual accepts the literal `=` glyph here, matching how `<`, `>`, and `~` are
   already spelled in this list.

5. `src/sase/ace/tui/_app_action_availability.py`: extend the existing `zoom_panel` tab
   guard to cover the new action, e.g.

   ```python
   if action in {"zoom_panel", "isolate_panels"} and app.current_tab != "agents":
       return False
   ```

### Behavior: `action_isolate_panels`

Rename `AgentPanelFoldingMixin._isolate_focused_panel` to `_toggle_panel_isolation` (it
no longer requires "focused panel" in the whole-panel sense) and add a thin action
wrapper on the same mixin:

```python
def action_isolate_panels(self) -> None:
    """Isolate the focused tribe panel, or restore the remembered layout."""
```

`AgentPanelFoldingMixin` already reaches the app through `AgentStructuralFoldingMixin` →
`AgentFoldingMixin`, so no new mixin wiring is needed; the same is true for the
`AgentPanelCollapseApp` test harness in
`tests/ace/tui/_agent_panel_collapse_helpers.py`.

`_toggle_panel_isolation` keeps the current control flow but changes how it resolves its
target and how it lands focus afterwards:

**Target resolution.** Replace the early `return False` when `_resolve_focused_panel()`
is `None` with:

- `panel_focus = self._resolve_focused_panel()`
- `target_key = panel_focus.panel_key if panel_focus is not None else panel_group.focused_key`
- `whole_panel_focus = panel_focus is not None`

Keep the existing bail-outs for a missing `_panel_group` and for the merged layout
(`_agent_panels_grouped`). Note that a row selection always implies an expanded panel:
`_resolve_focused_panel` returns collapsed focus whenever the focused panel is
collapsed, so `target_key` is guaranteed expanded in the row-focus case.

**Restore branch** (an armed `PanelIsolationRevert` exists) — unchanged persistence and
notification (`"Restored N panels"`); only focus landing becomes conditional:

- `whole_panel_focus` →
  `_focus_whole_panel_after_layout(target_key, collapsed=target_key in desired_collapsed)`
  exactly as today.
- row focus and `target_key in desired_collapsed` → the restore is about to collapse the
  panel the cursor lives in, so call
  `_focus_whole_panel_after_layout(target_key, collapsed=True)`. That helper already
  clears `_expanded_panel_focus`, `_current_group_key`, and `current_attempt_number`,
  and snaps `current_idx` to the panel's first rendered global index, which is the
  collapsed-panel focus state.
- row focus and the panel stays expanded → leave `current_idx`, `_current_group_key`,
  and `_expanded_panel_focus` untouched so the user keeps the row they were on.

**Isolate branch** — unchanged `collapsed_before` capture, `desired_collapsed`
computation, `_apply_panel_fold_layout` call, idempotent early return, and
`PanelIsolationRevert` arming. Focus landing:

- `whole_panel_focus` → `_focus_whole_panel_after_layout(target_key, collapsed=False)`
  as today.
- row focus → do not touch selection state at all. Collapsing sibling panels cannot
  invalidate `current_idx`, because `current_idx` indexes the flat `self._agents` list
  and `rendered_panel_slice` / `_panel_navigation_stops` are computed from
  `panel_group.focused_key`, which does not move.

**Availability feedback.** `action_isolate_panels` should not fail silently when the
layout cannot be isolated. Warn (`severity="warning"`) and return when the Agents tab is
inactive is unnecessary (the binding is already tab-gated), but do warn with
`"No tribe panels to isolate"` when `_agent_panels_grouped` is set (merged layout) or
when the split layout has fewer than two live panel keys. Keep the existing silence for
an isolation that changes nothing (already-isolated panel), which must also keep _not_
arming a restore.

### `action_zoom_panel` cleanup

In `src/sase/ace/tui/actions/agents/_panel_detail.py`, delete the
`if self._isolate_focused_panel(): return` branch and retitle the docstring to
`"""Zoom the active agent detail panel."""`. Until the `tribezoom` phase lands, `Z` on
whole-panel focus falls through to `_get_selected_agent()`, which returns `None` for
whole-panel focus, so it emits the existing `"No agent selected"` warning. That
intermediate state is intentional and is replaced in the next phase.

### Conditional footer

`src/sase/ace/tui/actions/agents/_display_detail_footer.py`:

- Compute a new `panel_isolation_available` flag: split layout (`_panel_group` is not
  `None` and not `_agent_panels_grouped`) with at least two panel keys.
- Drop the `panel_focused and` conjunct from the existing `panel_restore_armed`
  computation so an armed restore is advertised from row focus too.
- Pass `panel_isolation_available=` through `footer_widget.update_agent_bindings(...)`.

`src/sase/ace/tui/widgets/_keybinding_modes.py`: thread the new keyword through the
`update_agent_bindings` signature, its `TYPE_CHECKING` protocol stub, and the forwarding
call into `_compute_agent_bindings`.

`src/sase/ace/tui/widgets/_keybinding_bindings.py`:

- Accept `panel_isolation_available: bool = False` in `_compute_agent_bindings`.
- Remove the `zoom_panel` entry from the `panel_focused` block — zoom is now
  unconditional on the Agents tab, and per `src/sase/ace/CLAUDE.md` unconditional
  actions live in the help modal only.
- Append, gated on `panel_isolation_available` (independent of `panel_focused`):

  ```python
  bindings.append(
      (
          self._kd("isolate_panels"),
          "restore panels" if panel_restore_armed else "only panel",
      )
  )
  ```

The footer's alphabetical sort with symbol keys first (see
`src/sase/ace/tui/widgets/keybinding_footer.py`) then places `=` ahead of the letter
keys with no extra work.

### Help modal and command palette

`src/sase/ace/tui/modals/help_modal/agents_bindings.py`: replace the current
`(d(a.zoom_panel), "Zoom detail / only panel ⇄ restore panels")` pair with two pairs,
keeping every description at or under the 32-character limit documented in
`src/sase/ace/CLAUDE.md`:

- `(d(a.zoom_panel), "Zoom detail panel")`
- `(d(a.isolate_panels), "Only panel ⇄ restore panels")`

`src/sase/ace/tui/commands/_app_metadata.py`: retitle the `zoom_panel` command to
`"Zoom detail panel"` with aliases `("zoom",)`, and add an adjacent `isolate_panels`
entry: `"Isolate or restore tribe panels"`, group `"Display"`, `AGENTS_ONLY`, aliases
`("only panel", "restore panels", "isolate")`.

`src/sase/ace/tui/commands/types.py` / `commands/context.py`: add
`split_panel_count: int = 0` to `CommandContext` and populate it on the Agents tab from
`len(app._panel_group.panel_keys)` when `_agent_panels_grouped` is falsey (`0`
otherwise, and `0` on other tabs).

`src/sase/ace/tui/commands/availability.py`: leave `app.zoom_panel` as
`panel_focused or agent is not None` (still correct, and exactly right once tribe zoom
lands) and add `if spec.id == "app.isolate_panels": return ctx.split_panel_count >= 2`.

### Docs

- `docs/ace.md`
  - Agents keymap table row for `Z` (≈ line 917): change to "Zoom the active detail
    panel" and add a `=` row: "Isolate the focused tribe panel, or restore the
    remembered pre-isolation layout".
  - Whole-panel focus prose (≈ line 1393, the paragraph starting "With a whole panel
    selected, uppercase `Z` …"): rewrite for `=`, and state that `=` also works from a
    row selection inside a panel, isolating the panel that holds the cursor without
    changing the selected row.
  - Three-layer folding table (≈ line 1475): replace the `Z` row with a `=` row.
- `docs/agent_families.md`
  - Fold-prefix paragraph (≈ line 226): `Z` zooms; `=` isolates/restores.
  - Whole-panel focus section (≈ line 506): update `Z restore panels` to
    `= restore panels` and note the row-focus entry point.

### Tests

Update existing coverage:

- `tests/ace/tui/test_agent_panel_isolation_revert.py` and
  `tests/ace/tui/test_agent_panel_collapse_isolation.py`: swap every
  `app.action_zoom_panel()` for `app.action_isolate_panels()`.
- `tests/ace/tui/_agents_zoom_panel_helpers.py`: drop the `_isolate_focused_panel` stub
  and the `isolation_owned` / `isolation_calls` plumbing from `_FakeZoomApp`.
- `tests/ace/tui/test_agents_zoom_panel_action.py`: delete
  `test_action_zoom_panel_routes_focused_panel_without_modal_or_warning` (its behavior
  now belongs to isolation) and extend
  `test_default_zoom_migrates_to_uppercase_z_and_fold_keeps_lowercase` — or add a
  sibling test — asserting `registry.app.isolate_panels == "="` and that
  `build_app_bindings` emits an `isolate_panels` binding for `=`.
- `tests/test_keymaps_defaults.py`, `tests/test_keymaps_display_help.py`,
  `tests/test_keymaps_app_bindings.py`: update the zoom description/help pairs and
  assert the new `=` pair.
- `tests/test_command_catalog.py`, `tests/test_command_availability_agents.py`: update
  the `app.zoom_panel` description/alias assertions and add coverage for
  `app.isolate_panels` (available with ≥2 split panels, unavailable in the merged layout
  and on non-Agents tabs).
- `tests/ace/tui/widgets/test_keybinding_footer_tools_detail.py`: the
  `("Z", "only panel")` assertions become `("=", "only panel")`.

Add new coverage (extend `tests/ace/tui/test_agent_panel_isolation_revert.py` or add a
focused module beside it, using `AgentPanelCollapseApp`):

- Isolating from a row selection collapses the siblings, keeps `_expanded_panel_focus`
  `False`, and leaves `current_idx` / `_current_group_key` untouched.
- The restore armed by a row-focus isolation round-trips the previous layout from a row
  selection.
- A restore that would collapse the panel under the cursor lands on collapsed
  whole-panel focus with `current_idx` snapped into that panel.
- The merged layout and a single-panel layout warn instead of mutating folds.
- Footer: with two or more panels and no whole-panel focus, the conditional bindings
  include `("=", "only panel")`, and `("=", "restore panels")` once a restore is armed.

---

## Phase `tribezoom`: Zoom the tribe metadata document from whole-panel focus

### Goal

Pressing `Z` while a tribe panel is selected (expanded or collapsed) opens
`ZoomPanelModal` on that tribe's metadata document, with the modal's scrolling, search,
copy, edit, and refresh behavior intact.

### Modal: metadata-only tribe mode

`src/sase/ace/tui/modals/zoom_panel_types.py`

- No new seed dataclass. Tribe zoom reuses `ZoomPanelSeed` with only
  `metadata_renderable` / `metadata_subtitle` populated and `has_file_content=False`,
  `has_tools_content=False`.

`src/sase/ace/tui/modals/zoom_panel_modal.py`

- Widen `__init__`: `agent_provider: Callable[[], Agent | None] | None = None`,
  `initial_agent: Agent | None = None`, and a new
  `tribe_provider: Callable[[], AgentTribeSummarySnapshot | None] | None = None` plus
  `initial_tribe: AgentTribeSummarySnapshot | None = None`. Exactly one of
  `agent_provider` / `tribe_provider` is expected; raise `ValueError` otherwise so a
  miswired caller fails loudly.
- Store `self._tribe_provider`, `self._last_tribe = initial_tribe`, and expose
  `self._is_tribe_zoom = tribe_provider is not None`. In tribe mode force
  `self._target = ZoomPanelTarget.METADATA`, `_has_file_content = False`, and
  `_has_tools_content = False`.
- `_update_header`: in tribe mode render the tribe identity instead of the agent label —
  `Text.assemble((snapshot.label, "bold"), "  ", status_text(snapshot.status))` using
  `_last_tribe` (the snapshot's `label` already carries the tribe icon and `status` is
  the aggregate tribe status). Keep the single `METADATA` tab chip.
- `_update_hints`: in tribe mode drop the panel-cycling and file hints, e.g.
  `"j/k g/G ^D/^U scroll  /? search  n/N match  E edit  y copy  r refresh  q close"`.
- Add `on_tribe_section_snapshot_loaded(self, message: TribeSectionSnapshotLoaded)`:
  when `_is_tribe_zoom` and
  `message.panel_identity == self._last_tribe.container_identity`, re-render the tribe
  document; call `message.stop()` in tribe mode either way so the modal's own enrichment
  never triggers a redundant base-panel repaint through
  `AgentDetailRenderMixin.on_tribe_section_snapshot_loaded`.
- `on_unmount`: in tribe mode also call
  `self.query_one("#zoom-metadata-panel", AgentPromptPanel)._cancel_tribe_section_worker_for_agent_selection()`
  so a closing modal leaves no running enrichment worker.

`src/sase/ace/tui/modals/zoom_panel_navigation.py`

- `available_targets`: return `[ZoomPanelTarget.METADATA]` in tribe mode. `]` / `[` are
  then already inert because `cycle_target` returns early for a single target, and `h` /
  `l` / `H` / `L` are inert because the target is never `TOOLS`.
- `reveal_file_panel`: return `False` early in tribe mode with a
  `"No files for a tribe panel"` warning instead of calling `agent_provider`.

`src/sase/ace/tui/modals/zoom_panel_content.py`

- `seed_panels`: guard the file-panel seeding on `modal._last_agent is not None` so
  tribe mode does not stash a `None` agent on `ZoomFilePanel`.
- `refresh_active_panel`: branch to a new `refresh_tribe_metadata(modal)` when
  `modal._is_tribe_zoom`, before the `agent_provider()` lookup.
- New `refresh_tribe_metadata(modal)`: resolve `snapshot = modal._tribe_provider()`;
  when it is `None`, just `modal._update_header()` and return (the seeded document stays
  on screen). Otherwise store `modal._last_tribe = snapshot`, call
  `panel.update_tribe_display(snapshot)` on `#zoom-metadata-panel`, and refresh the
  header.
- `zoom_text` / `editor_info` need no change: both fall through to the metadata panel's
  rendered `content`, which is exactly the tribe document, so `y`, `E`, and the search
  overlay work unmodified.

`src/sase/ace/tui/widgets/prompt_panel/_agent_display.py`

- Add a keyword-only `publish_member_jump_map: bool = True` to `update_tribe_display`
  and pass `member_jump_map_publisher=None` when it is `False`. The zoom modal passes
  `False` so a modal render never overwrites the app-level `_member_jump_maps` registry
  that drives the base panel's `0-9` member jumps.

Everything else in `update_tribe_display` already works from inside the modal:
`panel_fold_state_from_widget` reads `app.panel_fold_level` and
`app._panel_fold_overrides`, so the zoomed document honors the same fold level as the
base panel, and `prepare_tribe_section_snapshot` resolves members through
`app._agents_in_focused_panel()`, which still points at the focused tribe panel while
the modal is open.

### App: route whole-panel focus to tribe zoom

`src/sase/ace/tui/actions/agents/_panel_detail.py`, in `action_zoom_panel`, after the
`current_tab` guard and before `_get_selected_agent()`:

- Resolve `resolver = getattr(self, "_focused_tribe_summary", None)` and
  `snapshot = resolver(with_entry_target=False)` when callable. `_focused_tribe_summary`
  returns `None` unless whole-panel focus is active, so this branch is self-gating;
  `with_entry_target=False` suppresses the "press `l` to enter" roster hint, which is
  meaningless inside the modal.
- When `snapshot` is not `None`, build the seed with a new
  `_zoom_seed_for_tribe(agent_detail)` helper — `metadata_renderable` from
  `#agent-prompt-panel`'s `content`, `metadata_subtitle` from the existing
  `_zoom_border_subtitle(agent_detail, "#agent-prompt-scroll")`, and both content flags
  `False` — then push:

  ```python
  ZoomPanelModal(
      tribe_provider=tribe_provider,
      initial_tribe=snapshot,
      initial_target=ZoomPanelTarget.METADATA,
      seed=seed,
      refresh_interval=getattr(self, "refresh_interval", 10),
  )
  ```

- `tribe_provider` re-resolves `_focused_tribe_summary(with_entry_target=False)` on each
  refresh tick and returns it only when its `container_identity` matches the identity
  captured at push time; otherwise it returns `None` so the modal keeps showing the
  tribe it was opened for. This mirrors the identity guard the existing `agent_provider`
  closure uses.
- Leave the agent path below untouched; clan containers and lane containers keep
  reaching the modal through `_get_selected_agent()` exactly as they do today.

### Discoverability and docs

- `src/sase/ace/tui/commands/_app_metadata.py`: retitle the `zoom_panel` command to
  `"Zoom agent or tribe detail panel"` and add a `"tribe"` alias.
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py`: retitle the zoom pair to
  `"Zoom agent/tribe detail"` (within the 32-character limit).
- `docs/ace.md`: update the Agents keymap `Z` row and the whole-panel focus prose to say
  `Z` zooms the selected tribe's metadata document; in the zoom-modal section (≈
  line 2961) note that a tribe zoom exposes only the metadata target, so panel cycling
  and file paging are inert while search, copy, edit, and refresh work as usual.
- `docs/agent_families.md`: update the fold-prefix paragraph to say `Z` zooms an agent
  row's detail panel or the selected tribe's metadata document.

### Tests

Add `tests/ace/tui/test_agents_zoom_panel_tribe.py` (extend
`tests/ace/tui/_agents_zoom_panel_helpers.py` with a tribe-focused fake app and an
`AgentTribeSummarySnapshot` factory built through `build_agent_tribe_summary_snapshot`):

- Whole-panel focus pushes a `ZoomPanelModal` in tribe mode with
  `initial_target == ZoomPanelTarget.METADATA` and a seed carrying the base prompt
  panel's renderable and subtitle.
- `_available_targets()` is `[METADATA]` in tribe mode, so `]`/`[` cannot leave the
  metadata document.
- The tribe provider returns a fresh snapshot for the same `container_identity` and
  `None` once the identity changes.
- `Ctrl+N` in tribe mode warns instead of revealing the file panel.
- `on_tribe_section_snapshot_loaded` re-renders for the current identity, is a no-op for
  a stale identity, and stops the message in both cases.
- A row selection still pushes the agent-mode modal (regression guard for the existing
  behavior).
- `ZoomPanelModal(...)` with neither provider — or both — raises `ValueError`.

Add a mounted/pilot test in the style of `tests/ace/tui/test_agent_tribe_modal_pilot.py`
or `tests/ace/tui/test_agents_zoom_panel_modal.py` that mounts the modal in tribe mode
and asserts the metadata scroll is visible, the header shows the tribe label, and the
file/tools views stay hidden.

Update `tests/test_command_catalog.py` and `tests/test_keymaps_display_help.py` for the
retitled zoom command and help description.

---

## Verification

Each phase must leave the tree green on its own:

```bash
just install
just check
```

Run `just check-full` before landing the epic's combined tree. The Agents-tab detail
panel and footer are covered by the PNG snapshot suite as well, so run
`just test-visual` when the footer or tribe document rendering changes and accept
intentional diffs with `--sase-update-visual-snapshots`.

## Risks and notes

- **Keymap collision.** `=` is unbound at the app level today. `grep -rn 'equals_sign'`
  finds only `key_validation.py` and the zoom modal's own search-overlay key list
  (`zoom_panel_search.py`), which handles keys inside the modal and is unaffected by an
  app binding.
- **Intermediate `Z` behavior.** Between the two phases, `Z` on whole-panel focus emits
  `"No agent selected"`. This is deliberate and is the reason `tribezoom` declares
  `depends_on: [isolate]`; do not run the phases in parallel.
- **Duplicate enrichment work.** Tribe zoom runs its own tribe-section enrichment on the
  modal's prompt panel rather than borrowing the base panel's cache. That keeps the
  modal self-contained at the cost of one extra worker while it is open; the
  `on_unmount` cancellation and the `message.stop()` in the modal's handler keep it from
  leaking work or repaints back into the base panel.
- **Rust core boundary.** Everything here is presentation-only Textual state,
  keybindings, and widget rendering, so it stays in this repo per `CLAUDE.md`'s Rust
  core backend boundary rule.
