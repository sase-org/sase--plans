---
tier: tale
title: Collapse the Artifacts relation panel with `.`
goal:
  Every Artifacts pane that declares relations can collapse its relation panel to a
  one-line count rail with `.`, reclaiming list rows while ancestor/child/sibling counts
  and navigation keys stay live.
size: medium
proposed_by: bbugyi200.athena.04h
create_time: 2026-08-17 06:54:58
status: wip
---

- **PROMPT:**
  [prompts/202608/artifacts_relation_panel_collapse.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifacts_relation_panel_collapse.md)

# Plan: Collapse the Artifacts relation panel with `.`

## 1. Outcome

Pressing `.` on any Artifacts pane whose contract enables `PaneCapability.RELATIONS`
collapses the host-owned relation panel at the bottom of the list column into a single
compact **relations rail** that still reports the ancestor and child (and sibling /
link) counts. Pressing `.` again restores the full panel. The toggle is one shared
app-level state, so it behaves identically on Patches, Stitches, Beads, Files, Plans,
and any document-provider pane that declares relations — present or future — with no
per-pane code.

Collapsing is **purely presentational**: the relation view is still built, the relation
keymap is still published, and `<` / `>` / `~` navigation keeps working unchanged while
collapsed. No data reload is triggered.

## 2. Current behavior (what exists today)

- `RelationPanel` (`src/sase/ace/tui/widgets/artifacts/relation_panel.py`) is a `Static`
  mounted at the bottom of each pane's list column, with id `<pane>-relation-panel` and
  the shared class `artifacts-relation-panel`. Five panes mount one today (`panes.py`,
  `commits_pane.py`, `plans_pane.py`, `beads_pane.py`, `files_pane.py`).
- `RelationPanelHostMixin.refresh_relation_panel()` is the single generic seam every
  pane calls on selection change. It builds the view through
  `sase.core.artifact_relation_layout.build_relation_view`, renders it, publishes the
  keymap to `app._relation_keymap`, and refreshes the conditional footer entries.
- `RelationPanel.clear()` sets `display = False` when a selection has no relations.
- Styling lives in `src/sase/ace/tui/styles.tcss` under `.artifacts-relation-panel`
  (`height: auto; max-height: 24; border: solid $secondary`).
- `PaneCapability.RELATIONS` is derived from declared relation edges (`_rule_relations`
  in `src/sase/ace/tui/_artifact_tab_contract_rules.py`), and
  `CAPABILITY_HOST_ACTIONS[PaneCapability.RELATIONS]`
  (`src/sase/ace/tui/_artifact_tab_actions.py`) is the registry of host actions that
  capability owns. The contract conformance harness
  (`tests/ace/tui/artifacts_contract/harness.py`) asserts each declared action is bound,
  reachable, and that its key resolves to **exactly one available action** per pane.
- Declared relation labels are contract-owned and read naturally in lowercase: Patches →
  `Ancestors` / `Children` / `Siblings`; Beads → `Parent` / `Children` / `Plans` /
  `Dependencies`; Plans → `Parents` / `Children` / `Patches`.

## 3. The `.` collision, and how this plan resolves it

`.` (`full_stop`) is already bound, app-wide, to `toggle_hide_reverted`
(`src/sase/default_config.yml`, `src/sase/ace/tui/bindings.py`,
`src/sase/ace/tui/keymaps/metadata.py`). That single action dispatches by tab
(`action_toggle_hide_reverted` in `src/sase/ace/tui/actions/patch/_core.py`):

| Tab       | Today's `.` behavior                                           |
| --------- | -------------------------------------------------------------- |
| Agents    | Show/hide non-run agents                                       |
| Axe       | Show/hide axe background commands                              |
| Artifacts | Show/hide reverted Patches (**Patches pane only** in practice) |

On non-Patches Artifacts panes `.` is already inert: `check_app_action`
(`src/sase/ace/tui/_app_action_availability.py`) returns `False` for any artifacts
action outside `NON_PRS_ARTIFACT_ACTIONS`, and `toggle_hide_reverted` is not in that
set. So the only real conflict is the **Patches pane**.

**Design decision — relocate Patch's reverted toggle to `X`, give `.` to relations.**

- `.` in the Artifacts tab → `toggle_relation_panel` (new action), every relations pane
  including Patches.
- `.` on Agents / Axe → unchanged (`toggle_hide_reverted` keeps those two branches).
- Patch's reverted-visibility toggle moves to a new, Patches-scoped action
  `patches_toggle_reverted` bound to `X`.

Why `X`: `x` in the Patches pane already toggles _submitted_ visibility
(`action_kill_agent` dispatches to `action_toggle_hide_submitted` on the Artifacts tab).
Putting _reverted_ on `X` produces a memorable `x`/`X` pair for the two terminal Patch
statuses, matching the app's existing shift-is-the-sibling idiom (`h`/`H`, `l`/`L`,
`B`/`I`). `X` is presently a no-op in the Patches pane —
`action_open_agent_cleanup_panel` returns immediately when the tab is neither `agents`
nor `axe` — so nothing is displaced.

This is the one user-visible behavior change beyond the new feature; it is called out
here deliberately so it can be rejected at plan approval. If it is rejected, the
fallback is to keep `toggle_hide_reverted` on `.` in the Patches pane only and scope the
new toggle to non-Patches panes — but that forfeits the "generic across all artifact
tabs" requirement and will fail `check_declared_keys_resolve_to_named_actions`, so it is
not the recommended path.

No SASE feature flag is warranted: the feature lands complete and enabled, the key is an
ordinary config field users can rebind, and the relocation is a rebinding rather than a
deprecated branch that must stay reachable (see `sase/memory/sase_flags.md`).

## 4. Visual design — the relations rail

Collapsed, the panel becomes exactly one line, borderless, sitting directly under the
bordered list:

```
▸  < 2 ancestors  ·  > 5 children  ·  ~ 3 siblings
▸  < 1 parent  ·  > 4 children  ·  2 plans  (1 hidden)
```

Rules:

1. **Glyph** — `▸` collapsed, exactly as `widgets/artifacts/group_banner.py` renders a
   collapsed banner (`▸` / `▾`), styled `bold {accent}` from the pane contract accent.
   The expanded rendering is **not** changed (no new header row) — that keeps every
   existing PNG golden of the expanded panel byte-identical.
2. **Segments** — one per relation section that has visible rows or a hidden count, in
   declared section order, so ancestors and children always come first and links last.
   Each segment is `{key} {count} {label}`:
   - `{key}` is the configured relation-mode key display
     (`key_display_name(km.app.start_ancestor_mode)` → `<`, `start_child_mode` → `>`,
     `start_sibling_mode` → `~`), styled `bold #FFAF00` — the same amber the expanded
     panel already uses for row keys, so the rail teaches the navigation keys. Link
     sections have no key mode and render `{count} {label}` only.
   - `{count}` styled `bold {accent}`; `{label}` is the **contract-declared section
     label**, lowercased, dim. No invented vocabulary and no pluralization gymnastics —
     the labels already read correctly.
3. **Separator** — `  ·  `, the shared `_SEPARATOR` from `widgets/artifacts/shell.py`.
4. **Hidden** — a trailing `({N} hidden)` in `dim #808080` when any section reported a
   hidden count, matching the expanded panel's existing vocabulary.
5. **One line, always** — the rail `Text` is built with
   `no_wrap=True, overflow="ellipsis"`. The list column can be as narrow as 43 cells, so
   segment order guarantees that ancestors and children survive truncation.
6. **Nothing to say, nothing shown** — when the selection has no relations at all the
   panel stays hidden (`clear()`), collapsed or not, exactly as today.

## 5. Implementation

### 5.1 Core summary model (Textual-free)

`src/sase/core/artifact_relation_layout.py`:

- Add `RelationSummaryEntry` (frozen, slots): `role: RelationRole`, `relation: str`,
  `label: str`, `count: int`, `hidden: int`.
- Add `RelationSummary` (frozen, slots): `entries: tuple[RelationSummaryEntry, ...]`
  plus `__bool__` (true when any entry exists) and a `hidden_total` property.
- Add `build_relation_summary(view: RelationView) -> RelationSummary`: one entry per
  section with `rows or hidden_count`, preserving section order. `count` is the total
  number of rendered rows in that section **including nested descendant children**
  (recursive over `RelationRow.children`), because that is what the expanded panel
  displays.
- Export the three new names from `__all__`.

This is presentation-shaped state derived from an already-built view, so per
`sase/memory/` boundary guidance it stays in this repo alongside `build_relation_view`;
no `../sase-core` change is needed.

### 5.2 App state

`src/sase/ace/tui/app.py`: add, next to `hide_reverted`, a public reactive

```python
artifacts_relations_collapsed: reactive[bool] = reactive(False, recompose=False)
```

Session-scoped and shared by every pane. No watcher — the toggle action drives the
refresh explicitly so the refresh path stays a single, traceable call.

### 5.3 Panel rendering

`src/sase/ace/tui/widgets/artifacts/relation_panel.py`:

- `RelationPanel.update_relations(...)` gains keyword-only `collapsed: bool = False` and
  `mode_keys: Mapping[RelationRole, str] | None = None`. It must return `view.keymap` on
  both branches — this is the reliability guarantee that `<` / `>` / `~` keep working
  while collapsed.
- Add `_refresh_collapsed(view, *, accent, mode_keys)` implementing §4, driven by
  `build_relation_summary(view)`. Falls back to `clear()` when the summary is empty.
- Add/remove the `-collapsed` class on the widget (`set_class`) so CSS switches with
  state; `clear()` must also drop the class.
- `RelationPanelHostMixin.refresh_relation_panel`:
  - read `collapsed = bool(getattr(self.app, "artifacts_relations_collapsed", False))`;
  - resolve `mode_keys` from the app keymap registry (`app._keymap_registry.app`) via
    `key_display_name`, falling back to `<` / `>` / `~` when unavailable;
  - pass both into `update_relations`.
- `RelationPanelHostMixin.relation_footer_entries` appends, **last**, when `keymap` is
  truthy:
  `("toggle_relation_panel", "expand relations" if collapsed else "collapse relations")`.
  The existing `_relation_footer_signature` guard already keeps this from repainting the
  footer on unchanged state.

`src/sase/ace/tui/styles.tcss`, beneath the existing `.artifacts-relation-panel` rule:

```
.artifacts-relation-panel.-collapsed {
    height: 1;
    max-height: 1;
    min-height: 1;
    padding: 0 1;
    border: none;
}
```

### 5.4 The action

Add `action_toggle_relation_panel` beside the relation-mode actions in
`src/sase/ace/tui/actions/navigation/_tree.py`:

```python
def action_toggle_relation_panel(self) -> None:
    """Collapse or expand the Artifacts relation panel."""
```

- Return early unless `self.current_tab == ARTIFACTS_TAB`.
- Flip `self.artifacts_relations_collapsed`.
- Refresh the active pane's relation panel through the existing display-only path:
  `self._artifacts_entry_navigator()` → `refresh_relation_panel()`, storing the returned
  keymap in `self._relation_keymap`; then `self._sync_active_artifacts_entry_state()` so
  the footer follows (on Patches that routes to `_refresh_display()`).
- **Never** call `_reload_and_reposition()` or any loader — per
  `sase/memory/tui_perf.md` rules 5 and 6 this is a re-render, not a refresh, and must
  not touch disk or rebuild rows.

`src/sase/ace/tui/actions/patch/_core.py`:

- `action_toggle_hide_reverted` drops its `artifacts` branch (keeps `agents` and `axe`);
  add `action_patches_toggle_reverted` performing the old artifacts branch
  (`self.hide_reverted = not self.hide_reverted; self._reload_and_reposition()`), gated
  to `current_tab == ARTIFACTS_TAB and current_artifacts_pane_key == "patches"`.

### 5.5 Availability

`src/sase/ace/tui/_app_action_availability.py`:

- Rename `_ARTIFACT_RELATION_NAV_ACTIONS` → `_ARTIFACT_RELATION_ACTIONS` and add
  `toggle_relation_panel`, so it is gated on `contract.has(PaneCapability.RELATIONS)`
  through the existing `_artifact_contract_action_available` path. Update the two
  references (`_CONTRACT_GATED_ARTIFACT_ACTIONS` and
  `_artifact_contract_action_available`).
- `toggle_relation_panel` → `False` when `current_tab != ARTIFACTS_TAB`.
- `toggle_hide_reverted` → `False` when `current_tab == ARTIFACTS_TAB`.
- `open_agent_cleanup_panel` → `False` when `current_tab` is not `agents` or `axe`
  (already true behaviorally; making it explicit is what lets `X` resolve on Patches).
- `patches_toggle_reverted` → `False` unless
  `current_tab == ARTIFACTS_TAB and current_artifacts_pane_key == "patches"` (fold into
  the existing `change_status`/`mark_pr_origin` block).

`src/sase/ace/tui/actions/artifacts.py`: add `toggle_relation_panel` to
`NON_PRS_ARTIFACT_ACTIONS`.

`src/sase/ace/tui/_artifact_tab_actions.py`: add `toggle_relation_panel` to
`CAPABILITY_HOST_ACTIONS[PaneCapability.RELATIONS]`. This is what makes the feature
generic: any pane — including a third-party document provider — that declares relation
edges gets the capability, the binding, the help row, and the conformance guarantees for
free.

### 5.6 Keymap plumbing

All four sites must move together (`tests/test_keymaps_app_bindings.py` asserts parity
between `DEFAULT_BINDINGS` and `_BINDING_META`):

1. `src/sase/default_config.yml`, under `ace.keymaps.app`:
   - add `toggle_relation_panel: "full_stop"` near the relation-mode prefixes, with a
     comment noting it shares `.` with `toggle_hide_reverted` and is disambiguated by
     tab availability (the same arrangement `x` already uses for
     `kill_agent`/`toggle_hide_submitted`);
   - add `patches_toggle_reverted: "X"`;
   - keep `toggle_hide_reverted: "full_stop"` and update its comment to say Agents/Axe
     only.
2. `src/sase/ace/tui/keymaps/app_keymaps.py`: add `toggle_relation_panel: str` and
   `patches_toggle_reverted: str`.
3. `src/sase/ace/tui/keymaps/metadata.py`: add
   `("toggle_relation_panel", "Relations Panel", False)` and
   `("patches_toggle_reverted", "Toggle Reverted", False)`.
4. `src/sase/ace/tui/bindings.py`: add the matching fallback `Binding` entries
   (`full_stop` → `toggle_relation_panel`, `X` → `patches_toggle_reverted`) in the same
   relative order as `_BINDING_META`.

### 5.7 Footer, help, palette

- **Footer (non-Patches)** — covered by §5.3's `relation_footer_entries` change.
- **Footer (Patches)** — `_compute_available_bindings` in
  `src/sase/ace/tui/widgets/_keybinding_bindings.py` appends the same
  `("toggle_relation_panel", …)` entry when `app._relation_keymap` is truthy, so the
  Patches footer matches the other panes. (`_refresh_relation_footer` deliberately skips
  the Patches pane; this is its counterpart.)
- **Help modal** — `_relation_rows` in
  `src/sase/ace/tui/modals/help_modal/patches_artifact_bindings.py` appends one row for
  the toggle so every relations pane's help section gains it automatically:
  `(key_display_name(km.app.toggle_relation_panel), "Collapse / expand relations")` (27
  chars, inside the 32-char cap enforced by
  `tests/ace/tui/test_artifacts_relation_surfaces.py` and the 57-cell box width from
  `src/sase/ace/CLAUDE.md`).
- `src/sase/ace/tui/modals/help_modal/patches_bindings.py:294` — swap
  `d(a.toggle_hide_reverted)` for `d(a.patches_toggle_reverted)` on the "Show/hide
  reverted PRs" row. `agents_bindings.py:473` stays on `toggle_hide_reverted`.
- **Command palette** — `src/sase/ace/tui/commands/_app_metadata.py`:
  - change `toggle_hide_reverted`'s tabs from `CL_ONLY` to `AGENTS_AXE`;
  - add `("patches_toggle_reverted", "Toggle hide reverted", "Display", CL_ONLY, ())`;
  - add
    `("toggle_relation_panel", "Collapse relations panel", "Display", CL_ONLY, ("relations", "ancestors", "children", "relation panel"))`.

### 5.8 Docs

- `docs/artifacts_pane_visual_grammar.md`, "Relation panel slot": document the collapsed
  rail — the `.` toggle, that state is shared across panes, the rail's glyph/segment/
  separator grammar, that counts include nested descendants, that hidden counts are
  preserved, and that the relation keymap stays live while collapsed.
- `docs/ace.md:2382`: the all-tabs `.` row now reads Artifacts → collapse/expand the
  relations panel; Agents → non-run agents; Axe → axe commands. Add `X` to the Patches
  key table for reverted visibility next to `x` for submitted.
- `CHANGELOG.md` is release-please generated — do not hand-edit.

## 6. Tests

New:

- `tests/ace/tui/test_artifacts_relation_summary.py` — pure `build_relation_summary`
  unit tests: section order preserved; ancestor/child/sibling/link counts; nested
  descendant subtree counted recursively; `hidden` carried per section and totalled;
  empty view is falsy.
- `tests/ace/tui/test_artifacts_relation_collapse.py` — app-level behavior:
  - `.` collapses and re-expands on each relations pane (Patches, Beads, Files, Plans,
    Stitches), asserting the `-collapsed` class and single-line render;
  - collapsed state persists across sub-tab switches (one shared flag);
  - the collapsed rail text contains the ancestor and child counts and the declared
    labels;
  - `<` / `>` / `~` still navigate while collapsed (keymap still published);
  - the footer entry appears only when relations exist and flips label between
    `collapse relations` and `expand relations`;
  - toggling does not reload pane data (assert the pane loader / snapshot worker is not
    re-invoked and the row count is unchanged).
- `tests/ace/tui/test_artifacts_relation_key_resolution.py` — for each tab context, `.`
  and `X` each resolve to exactly one _available_ action: `.` → `toggle_relation_panel`
  on Artifacts, `toggle_hide_reverted` on Agents and Axe; `X` →
  `patches_toggle_reverted` on the Patches pane, `open_agent_cleanup_panel` on Agents
  and Axe.

Extended:

- `tests/ace/tui/test_artifacts_relation_surfaces.py` — the expected
  `CAPABILITY_HOST_ACTIONS[PaneCapability.RELATIONS]` tuple gains
  `toggle_relation_panel`; `_relation_rows` asserts the toggle row is present for every
  RELATIONS contract.
- `tests/ace/tui/artifacts_contract/harness.py` — add `toggle_relation_panel` to the
  local `_RELATION_REACHABILITY_ACTIONS` set so reachability and key-resolution
  conformance covers it on every resolved pane.
- Patches behavior: `X` toggles `hide_reverted` and `.` no longer does — extend
  whichever existing Patch action test covers `action_toggle_hide_reverted`.

Visual (per the visual-grammar extension checklist, item 7):

- Add one PNG snapshot of a collapsed relations rail on a pane that reliably shows
  ancestors and children (extend
  `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_beads.py`).
- Expect a rebaseline of existing Artifacts goldens whose footer now carries the new
  conditional entry. Inspect `.pytest_cache/sase-visual/` actual/expected/diff artifacts
  before accepting any golden, then accept with `--sase-update-visual-snapshots`.
  Expanded-panel pixels must be unchanged — a diff there means §4 rule 1 was violated.

## 7. Verification

1. `just install` first (ephemeral workspace).
2. `just check` inline while iterating.
3. `just test-visual` for the PNG suite, accepting goldens only after inspecting diffs.
4. `just check-full` before landing, run through `/sase_monitor` with a `--next` action
   — it routinely outruns a single agent turn and this change touches keymaps,
   contracts, and goldens, which is squarely in the broadening set.

## 8. Non-goals

- **No cross-session persistence.** The collapsed flag is session state, exactly like
  `hide_reverted`. Persisting it is a reasonable follow-up, not part of this change.
- **No per-pane collapse state.** One shared flag is the whole point of "generic"; a
  per-pane variant would make `.` unpredictable when switching sub-tabs.
- **No change to relation navigation, relation building, or the expanded rendering.**
- **No `../sase-core` change** — collapse is presentation state.
- **No new capability.** `PaneCapability.RELATIONS` already gates exactly the right set
  of panes.
