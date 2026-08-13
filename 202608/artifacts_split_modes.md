---
tier: tale
title: Artifacts pane split modes
goal:
  Every Artifacts sub-tab defaults to an even left/right split, and `{` / `}` cycle a
  shared narrow/even/wide split mode with wraparound.
size: medium
proposed_by: bbugyi200.athena.zl.w0
create_time: 2026-08-13 12:20:53
status: wip
---

# Artifacts Split Modes

## Goal

Every Artifacts sub-tab renders a left list panel beside a right detail panel. Today the
left panel claims most of the screen (`58%` on Stitch, `55%` on Bead / Plan / File, and
up to `80` content-driven cells on Patch), which starves the detail pane.

Two outcomes:

1. **Default to an even split.** No Artifacts sub-tab's left panel may claim more than
   half the available width by default.
2. **Add a three-mode split cycle** bound to `{` and `}`, cycling in opposite directions
   with wraparound:
   - `narrow` — left panel small, detail dominant
   - `even` — equal panels (the new default)
   - `wide` — left panel dominant, detail small (roughly today's behavior, pushed
     further)

## Design

### One mode, shared by every Artifacts sub-tab

The split mode is a single app-level reactive, not per-sub-tab state.

Rationale: the same physical keys must produce the same visual result everywhere, and a
user who prefers a wide list wants it on every pane without re-pressing `{` / `}` five
times. Per-sub-tab modes would mean the user has to track five independent states and
would make `]` (sub-tab cycle) silently change the layout. One mode is the intuitive and
reliable choice, and it collapses the whole feature down to one CSS class.

### `}` grows the left panel, `{` shrinks it

`}` moves toward `wide`, `{` moves toward `narrow`. The brace's opening points in the
direction the divider travels, which is the mnemonic. Both wrap:

```
{  wide  →  even  →  narrow  →  wide  ...
}  narrow →  even  →  wide   →  narrow ...
```

Naming follows the existing `cycle_artifacts_subtab` / `cycle_artifacts_subtab_reverse`
and `cycle_grouping_mode` / `cycle_grouping_mode_reverse` convention:
`cycle_artifacts_split` (forward, `}`) and `cycle_artifacts_split_reverse` (`{`).

### Ratios and floors

| Mode     | Left width | Left `min-width` | Left @120 cols | Left @80 cols |
| -------- | ---------- | ---------------- | -------------- | ------------- |
| `narrow` | `30%`      | `34`             | 36             | 34            |
| `even`   | `50%`      | `44`             | 60             | 44            |
| `wide`   | `70%`      | `48`             | 84             | 56            |

`30 / 50 / 70` is symmetric around the default, memorable, and leaves the minority pane
at least 30% of the screen in every mode. The right panel stays `width: 1fr` and takes
the remainder, so it never needs its own sizing rule.

The per-mode `min-width` floors exist because today's uniform floors (`48` on Stitch,
`56` on the rest) would swallow `narrow` mode entirely — `30%` of a 120-column terminal
is 36 cells, so a `56`-cell floor would clamp `narrow` back to 47% and make the key look
broken. Every Artifacts list already sets `text-wrap: nowrap; text-overflow: ellipsis`
(verified in `styles.tcss` for `#stitches-timeline`, `#beads-list`, `#plans-list`,
`#files-list`, and in the `Text(no_wrap=True, overflow="ellipsis")` row builders), so
rows degrade to ellipsis rather than wrapping at these widths.

**Textual gotcha that constrains the implementation:** `Widget._get_box_model` applies
`min_width` **first** and `max_width` **second**, so in Textual `max-width` overrides
`min-width` — the opposite of browser CSS. The four proportional panes must therefore
express their floor with `min-width` only and must not gain a `max-width`, or the floor
silently stops working on narrow terminals.

### Patch pane: the mode is a cap, not a ratio

`#list-container` is content-driven: `PatchList.WidthChanged` broadcasts the widest
formatted row and `on_patch_list_width_changed` clamps it to `43..80` cells. Forcing a
flat `50%` there would paint a wide empty gutter whenever the rows are short, so instead
the mode supplies a **cap**:

```
width = clamp(content_width, 43, min(80, floor(available_width * left_fraction)))
```

with the `43`-cell floor always winning (it is what makes the Patch info panel legible).
On a 120-column terminal in `even` mode the cap is 60, so a naturally 50-cell list is
unchanged, while a maxed-out 80-cell list is pulled back to 60 — exactly the "at most
half" outcome, with no wasted space. This must be Python, not CSS, precisely because
`min(80, 50%)` is inexpressible in a single `max-width` declaration and because Textual
lets `max-width` beat the `43`-cell floor.

### Everything else is one CSS class

`ArtifactsView` carries exactly one of `-split-narrow` / `-split-even` / `-split-wide`,
and `styles.tcss` keys the four proportional panes off it. This reuses the established
`#artifacts-view.-onboarding-active #list-container` pattern already in the file, keeps
all sizing declarative, and means pane widths cannot drift out of sync with the mode:
there is no per-pane Python width math and nothing to re-apply on mount, resize, or
sub-tab switch.

### Indicator

A fixed-width badge sits at the right end of the Artifacts sub-tab strip row, so one
indicator covers every sub-tab including Patch:

```
{████}     wide      (3 of 4 cells filled)
{██░░}     even      (2 of 4)
{█░░░}     narrow    (1 of 4)
```

Rendered as four `█` cells where the leading N use the **active sub-tab's accent color**
(so the badge harmonizes with the pane it describes) and the rest use a dim rail color;
the braces are the key hint and the frame at once. `█` is already used in the ACE TUI
and rasterizes correctly through the pinned snapshot renderer, so no new glyph-coverage
risk. The badge is 6 cells plus 1 cell of padding each side.

To keep the centered tab strip optically centered, a spacer of identical width sits at
the left end of the row.

The badge is clickable and cycles forward, mirroring `PanelTabStrip.TabClicked`.

### Rust core boundary

Split ratios, keybindings, TCSS, and Textual reactives are presentation-only per the
`rust_core_backend_boundary` memory. No `../sase-core` change is required.

## Implementation

### 1. Widget-free core — new `src/sase/ace/tui/artifacts_split.py`

Sibling to `artifact_tabs.py` and, like it, free of Textual imports so keymaps, action
availability, and tests can use it:

- `ArtifactsSplitMode = Literal["narrow", "even", "wide"]`
- `ARTIFACTS_SPLIT_MODE_ORDER: tuple[ArtifactsSplitMode, ...] = ("narrow", "even", "wide")`
- `DEFAULT_ARTIFACTS_SPLIT_MODE: ArtifactsSplitMode = "even"`
- `ARTIFACTS_SPLIT_CLASSES: dict[ArtifactsSplitMode, str]` → `-split-narrow` etc.
- `ARTIFACTS_SPLIT_LEFT_FRACTION: dict[ArtifactsSplitMode, float]` →
  `0.30 / 0.50 / 0.70`
- `ARTIFACTS_SPLIT_BADGE_FILLED: dict[ArtifactsSplitMode, int]` → `1 / 2 / 3`
- `ARTIFACTS_SPLIT_BADGE_CELLS = 4`
- `normalize_artifacts_split_mode(value: object) -> ArtifactsSplitMode` — unknown values
  fall back to the default
- `cycle_artifacts_split_mode(mode, direction: int) -> ArtifactsSplitMode` — wraps in
  both directions over `ARTIFACTS_SPLIT_MODE_ORDER`
- `artifacts_split_left_cap(mode, available_width, *, minimum, maximum) -> int` —
  returns `maximum` when `available_width <= 0`, otherwise
  `max(minimum, min(maximum, floor(available_width * fraction)))`
- `build_artifacts_split_badge(mode, accent) -> rich.text.Text` — the `{████}` badge

### 2. `src/sase/ace/tui/styles.tcss`

Replace the `width` / `min-width` declarations on `#stitches-list-container`,
`#plans-list-panel`, `#beads-list-panel`, and `#files-list-panel` with `even`-mode
values as the un-classed fallback (so nothing flashes wide before the class lands), then
add one comment-headed block of twelve class-scoped rules:

```
ArtifactsView.-split-narrow #stitches-list-container { width: 30%; min-width: 34; }
ArtifactsView.-split-even   #stitches-list-container { width: 50%; min-width: 44; }
ArtifactsView.-split-wide   #stitches-list-container { width: 70%; min-width: 48; }
```

…and the same triple for the other three left panels, using `min-width: 56` for their
`wide` rows (today's value). Use the `ArtifactsView` **type** selector rather than
`#artifacts-view` so the rules hold wherever the view is mounted. Do not add `max-width`
to any of these — see the Textual clamp-order note above.

Add `#artifacts-header` (height 1, full width), keep `#artifacts-subtabs` but change it
to `width: 1fr`, and add `#artifacts-split-badge` / `#artifacts-split-spacer` at a
shared fixed width with a comment tying the two together.

Leave `#list-container` alone; its cap is computed in Python.

### 3. `src/sase/ace/tui/widgets/artifacts/split_badge.py` (new)

`ArtifactsSplitBadge(Static)` with a `Clicked` `Message`, a `set_state(mode, accent)`
method that re-renders through `build_artifacts_split_badge`, and an `on_click` that
posts `Clicked`. It is imported directly by `view.py`; only add it to
`widgets/artifacts/__init__.py` and the matching `__init__.pyi` if something outside
that package ends up importing it.

### 4. `src/sase/ace/tui/widgets/artifacts/view.py`

- `compose`: wrap the existing `PanelTabStrip` in `Horizontal(id="artifacts-header")`
  with `Static(id="artifacts-split-spacer")` before it and
  `ArtifactsSplitBadge(id="artifacts-split-badge")` after it. `query_one` lookups for
  `#artifacts-subtabs` still resolve, since they search descendants.
- `self._split_mode: ArtifactsSplitMode = DEFAULT_ARTIFACTS_SPLIT_MODE` in `__init__`.
- `apply_split_mode(mode)`: `set_class` for each of the three classes so exactly one is
  present, store the mode, refresh the badge.
- `_refresh_split_badge()`: passes the mode plus the **active descriptor's accent**
  (`self._descriptor_by_id[self._current_subtab].accent`).
- Call `apply_split_mode(self.app.artifacts_split_mode)` from `on_mount`, and
  `_refresh_split_badge()` from `switch_to` so the accent tracks the sub-tab.
- `@on(ArtifactsSplitBadge.Clicked)` sets `app.artifacts_split_mode` one step forward.

### 5. Reactive, watcher, and the Patch cap

- `src/sase/ace/tui/app.py`:
  `artifacts_split_mode: reactive[ArtifactsSplitMode] = reactive(DEFAULT_ARTIFACTS_SPLIT_MODE)`,
  beside `current_artifacts_subtab`.
- `src/sase/ace/tui/_app_watchers.py`: `watch_artifacts_split_mode(old, new)` — return
  early when unchanged, resolve `ArtifactsView` defensively (`try/except`, matching
  `watch_current_artifacts_subtab`), call `view.apply_split_mode(new)`, then
  `self._apply_patch_list_width()`.
- `src/sase/ace/tui/actions/_event_widgets.py`:
  - `on_patch_list_width_changed` records the broadcast content width on the app and
    delegates to the new `_apply_patch_list_width()`.
  - `_apply_patch_list_width()` implements the clamp from the design section, taking
    `available_width` from the `#artifacts-view` widget's size and falling back to the
    app size.
  - `on_resize` also calls it, so the cap follows terminal resizes.
- `src/sase/ace/tui/actions/_state_init*.py`: initialize the remembered content width to
  `MIN_LIST_WIDTH`.

### 6. Actions — `src/sase/ace/tui/actions/artifacts_navigation.py`

`action_cycle_artifacts_split` / `action_cycle_artifacts_split_reverse` delegating to a
private `_cycle_artifacts_split(direction)` that assigns
`cycle_artifacts_split_mode(self.artifacts_split_mode, direction)`, placed next to the
existing `action_cycle_artifacts_subtab` pair.

### 7. Registration sites

All of these must be updated together; the repo has import-time and test-time guards
that fail loudly when they drift.

| File                                             | Change                                                                                                                                                                                                                                                        |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `keymaps/key_validation.py`                      | Add `"left_curly_bracket": "{"` and `"right_curly_bracket": "}"` to `_KEY_DISPLAY`. Textual 8.0.1 reports these exact names for `{` / `}` (confirmed via `_character_to_key`), and without the entries `is_valid_key` rejects any user rebinding onto braces. |
| `keymaps/app_keymaps.py`                         | Two `AppKeymaps` fields under a new `# Artifacts split` comment.                                                                                                                                                                                              |
| `keymaps/metadata.py`                            | `_BINDING_META` entries: `("cycle_artifacts_split", "Wider Artifact List", False)` and `("cycle_artifacts_split_reverse", "Narrower Artifact List", False)`.                                                                                                  |
| `src/sase/default_config.yml`                    | `cycle_artifacts_split: "right_curly_bracket"`, `cycle_artifacts_split_reverse: "left_curly_bracket"`, beside the existing `cycle_artifacts_subtab` pair.                                                                                                     |
| `tui/bindings.py`                                | Two `Binding(..., show=False)` entries beside the sub-tab cycle bindings.                                                                                                                                                                                     |
| `actions/artifacts.py`                           | Add both action names to `NON_PRS_ARTIFACT_ACTIONS` — without this the blanket non-Patch guard in `_app_action_availability.py` disables them on four of the five sub-tabs.                                                                                   |
| `_app_action_availability.py`                    | Return `False` for both unless `current_tab == ARTIFACTS_TAB`.                                                                                                                                                                                                |
| `commands/_app_metadata.py`                      | Two `CommandSpec` rows, category `Display`, tabs `CL_ONLY`, with aliases such as `"split"`, `"pane width"`, `"wider list"`, `"narrower list"`. The catalog raises at import if an `AppKeymaps` field has no spec.                                             |
| `commands/availability.py`                       | Add `app.cycle_artifacts_split` and `app.cycle_artifacts_split_reverse` to `_NON_PRS_ARTIFACT_COMMANDS` so the palette offers them outside the Patch pane.                                                                                                    |
| `modals/help_modal/patches_artifact_bindings.py` | One row in the `Artifact Views` section: `f"{d(a.cycle_artifacts_split_reverse)} / {d(a.cycle_artifacts_split)}"` → `"Narrow / widen the list panel"`.                                                                                                        |

The keymap loader's duplicate check only fires for user-overridden keys, and neither
brace is bound today, so the new defaults are conflict-free. Both bindings stay
`priority=False`, so a focused filter-bar `Input` still receives literal `{` / `}`.

### 8. Documentation

- `docs/ace.md`: a new `### Split Modes in Artifacts Panes` subsection after the
  `## Tab System` sub-tab paragraph, covering the three modes, the ratio table, the
  wraparound, the badge, and the Patch pane's cap semantics. Add `{` / `}` to the
  `## Keybindings: Artifacts / Patches` → `### Navigation` table.
- `docs/configuration.md`: note under `#### ace.keymaps` that `cycle_artifacts_split` /
  `cycle_artifacts_split_reverse` are remappable and that `left_curly_bracket` /
  `right_curly_bracket` are now accepted key names.

## Tests

### New — `tests/ace/tui/test_artifacts_split_modes.py`

Pure-function coverage of `artifacts_split.py`:

- `cycle_artifacts_split_mode` wraps in both directions from all three modes.
- `normalize_artifacts_split_mode` maps unknown / wrong-typed values to `even`.
- `artifacts_split_left_cap` for: `available_width=0` → `maximum`; a 120-column terminal
  in each mode; an 80-column terminal in `narrow`, where the `43`-cell floor must win
  over the `24`-cell fraction.
- `build_artifacts_split_badge` yields four block cells with the expected filled count
  and applies the supplied accent to exactly the filled prefix.

App-level coverage through the existing ACE page harness:

- `}` from the default advances `even → wide → narrow` (wraparound) and `{` reverses.
- `ArtifactsView` carries exactly one `-split-*` class after each transition.
- The class survives a sub-tab switch, and the badge accent follows the active sub-tab.
- `check_action` reports both actions unavailable on the Agents and Axe tabs, and
  available on every Artifacts sub-tab including Patch.
- Patch cap: with a broadcast content width of `80` on a 120-column app, `even` yields
  `60`, `wide` yields `80`, and `narrow` clamps up to the `43` floor.
- Clicking the badge advances the mode.

### Updated

- `tests/ace/tui/test_artifacts_scaffold.py` — confirm the two `#artifacts-subtabs`
  lookups still resolve after the header wrapper lands.
- Keymap sync tests (`tests/test_keymaps_defaults.py`,
  `tests/test_keymaps_app_bindings.py`, `tests/test_command_catalog*.py`) are
  field-count driven and should pass without edits; fix any that enumerate actions
  explicitly.

### Visual

- New `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_split.py` with three
  goldens on the Bead pane — `artifacts_split_narrow_120x40`,
  `artifacts_split_even_120x40`, `artifacts_split_wide_120x40` — plus one
  `artifacts_split_narrow_80x24` that proves the tight-terminal floors and the in-panel
  status lines still read correctly.
- Every existing Artifacts golden changes, because both the default width and the new
  header badge move pixels. Regenerate with `--sase-update-visual-snapshots` and review
  each diff by eye rather than accepting in bulk: roughly `artifacts_beads_*`,
  `artifacts_files_*`, `artifacts_plans_*`, `artifacts_stitches_*`, and any Patch-pane
  golden whose list was previously wider than half the frame.

### Verification

- `just install` first (ephemeral workspace), then `just check`.
- `just test-visual` for the PNG suite; inspect `.pytest_cache/sase-visual/` artifacts
  on any failure.
- `just check-full` before landing, run through `/sase_monitor` with a `--next` action —
  the width change touches a broad set of snapshots.
- Manual pass in a live `sase ace`: cycle `{` / `}` on all five sub-tabs at 120 and 80
  columns, confirming no clipped status line, no wrapped list row, and no layout jump
  when switching sub-tabs mid-mode.

## Risks

- **Textual clamp order.** `max_width` beats `min_width`. Any `max-width` added to the
  four proportional left panels would silently break the `narrow`-mode floor. The rules
  must use `width` + `min-width` only.
- **Narrow mode at 80 columns.** The left panel drops to 34 cells. The 2-row status
  Statics inside the list panels (`#beads-status`, `#files-status`, `#plans-status`)
  clip rather than grow. If the 80x24 golden shows truncation, add
  `text-wrap: nowrap; text-overflow: ellipsis` to those rules — the list rows themselves
  already ellipsize.
- **Wide mode squeezes Markdown detail.** At 80 columns `wide` leaves the detail pane 24
  cells. That is the point of the mode; the right panel stays `1fr` with no floor so the
  `Horizontal` can never overflow.
- **Golden churn.** The header badge changes every Artifacts snapshot. Regenerating is
  mechanical but must not be used to paper over an unintended layout regression.

## Acceptance criteria

1. On every Artifacts sub-tab, the left panel occupies at most half the available width
   by default.
2. `}` and `{` cycle `narrow ↔ even ↔ wide` in opposite directions with wraparound, on
   every Artifacts sub-tab, and are inert on Agents and Axe.
3. The mode is shared across sub-tabs and survives sub-tab switches and terminal
   resizes.
4. The split badge shows the active mode on every sub-tab, in the active sub-tab's
   accent, and cycles on click.
5. Both actions are rebindable under `ace.keymaps.app`, appear in the command palette
   and the help modal, and `{` / `}` are accepted as configured key names.
6. `just check-full` and `just test-visual` pass with reviewed goldens.

## Out of scope

- Persisting the mode across ACE restarts, and a `default_config.yml` option for the
  starting mode. The mode is session state defaulting to `even`.
- Per-sub-tab split modes.
- Mouse drag-to-resize of the divider.
- Any change to the Agents or Axe tab layouts, which have their own content-driven
  sidebar sizing.
