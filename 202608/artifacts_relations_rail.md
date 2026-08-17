---
tier: tale
title: Collapse Artifacts relations by default behind a self-explaining rail
goal:
  Every Artifacts sub-tab starts with its relation panel collapsed, and the collapsed
  rail states its own expand key and responds to a click, so the panel is never hidden
  knowledge.
size: medium
proposed_by: bbugyi200.athena.057
create_time: 2026-08-17 17:46:58
status: wip
---

# Plan: Collapse Artifacts relations by default behind a self-explaining rail

## Problem

The host-owned relation panel (`RelationPanel`,
`src/sase/ace/tui/widgets/artifacts/relation_panel.py`) is mounted at the bottom of the
list column on every Artifacts pane that declares `PaneCapability.RELATIONS` (Stitch,
Patch, Bead, Plan, File). It starts **expanded** and can grow to 24 rows
(`.artifacts-relation-panel` in `src/sase/ace/tui/styles.tcss`), so a single link row
costs a five-line bordered box that permanently shrinks the list the user actually came
to read.

`.` (`toggle_relation_panel`) already collapses it to a one-line rail, but two things
keep that from being usable as a default:

1. The default is expanded
   (`artifacts_relations_collapsed: reactive[bool] = reactive(False, ...)` in
   `src/sase/ace/tui/app.py:175`), and there is no configuration for the starting state.
2. The collapsed rail never says how to get the panel back. It renders
   `▸  > 2 children  ·  1 plans`, where `>` is the _descendant navigation_ key, not the
   expand key. The only place the expand key appears is the conditional footer
   (`expand relations`), which is a crowded lane the eye does not associate with the
   rail. Nothing on the rail reacts to a mouse click either.

Collapsing by default without fixing (2) would hide a whole feature. Both halves ship
together.

## Design

### The relations rail (collapsed — the new default)

```
 ▸ .  expand   ·   < 1 ancestors  ·  > 2 children  ·  ~ 1 siblings  (1 hidden)
 └────┬───────┘    └──────────────────────┬──────────────────────┘  └───┬───┘
  control chip            navigation segments (unchanged)          hidden count
```

Anatomy, left to right, built in `_build_collapsed_rail`:

1. **Control chip** — `" ▸ {key} "` rendered `bold #1a1a1a on {accent}`. This is the
   same chip grammar `build_shell_scope` uses for the pane identity chip
   (`src/sase/ace/tui/widgets/artifacts/shell.py`), so it reads as a solid, pressable
   object rather than text. `▸` is the collapsed disclosure triangle. `{key}` is the
   **resolved display name** of `toggle_relation_panel`, never a hardcoded `.`.
2. **Verb** — `" expand"` in style `{accent}` (not dim, not bold). Dim is the codebase's
   "secondary information" register; this is the one thing on the rail that must be
   read, so it stays at full accent weight.
3. `_SEPARATOR` (`"  ·  "`, dim).
4. **Navigation segments** — unchanged from today: `{key}` bold `#FFAF00`, `{count}`
   bold `{accent}`, `" {label.lower()}"` dim, joined by `_SEPARATOR`.
5. **Hidden count** — unchanged trailing `" (N hidden)"` in `dim #808080`.

Two key registers, deliberately distinguished by color so the rail never reads as one
undifferentiated key soup:

- **chip / dark-on-accent** = a control that acts on this rail (expand).
- **amber `#FFAF00`** = a navigation key that moves the selection (existing grammar,
  identical to the `[key]` prefixes on expanded rows).

**Truncation order is part of the design.** The rail Text stays `no_wrap` with
`overflow="ellipsis"`. Because the affordance is leftmost, an 80-column terminal
ellipsizes relation counts and never the expand chip — the narrow case is exactly when a
user most needs to be told how to get their space back.

An empty summary still returns `None` and the panel stays hidden (unchanged): a
selection with no relations has nothing to expand.

### The relations panel (expanded)

The panel keeps its rows exactly as they render today. It gains a frame that names
itself and carries the reverse affordance at zero row cost, using the border-title
support the codebase already relies on (`list_panel.border_title = "Beads"`;
`border-title-color` appears throughout `styles.tcss`):

- `border_title` = `"▾ RELATIONS"` — `▾` is the expanded disclosure triangle, mirroring
  the rail's `▸`.
- `border_subtitle` = `"{key} collapse"`, bottom-right. A key hint in a border subtitle
  is already this codebase's idiom: `scroll.border_subtitle = "ctrl+d/u scroll"` in
  `src/sase/ace/tui/modals/plugin_action_confirm_modal.py:502`.

Same disclosure vocabulary in both states, so the two states are legibly the same
control.

### Mouse

- Clicking the **collapsed** rail expands it. `RelationPanel.on_click` posts a
  `RelationPanel.Clicked` message, mirroring `ArtifactsSplitBadge`
  (`src/sase/ace/tui/widgets/artifacts/split_badge.py`), handled in `ArtifactsView`
  exactly like `_on_split_badge_clicked` (`widgets/artifacts/view.py:349`).
- Clicking the **expanded** panel does nothing. Rows are not click targets today, and
  collapsing the panel out from under a click aimed at a row would be hostile.
- Hover raises the collapsed rail's background so the pointer discovers it before the
  click. `:hover` is already used in `styles.tcss` (`.admin-center-home-row:hover`,
  `.statistics-tile:hover`).

### Starting state

New config field `ace.artifacts.relations_expanded` (bool, **default `false`**), a
direct sibling of the existing `ace.axe_description_expanded` and semantically identical
to it: it seeds the session, and the toggle key changes only the current session. This
is a config field, not a feature flag — `sase/memory/sase_flags.md`: "If users are meant
to choose the value forever, it was never a feature flag. Add a normal config field
instead."

The session flag stays app-global and shared across sub-tabs. That cross-sub-tab
behavior is documented and covered by
`tests/ace/tui/test_artifacts_relation_collapse.py::test_collapsed_state_persists_across_subtabs`;
only the seed changes.

## Implementation

Run `just install` first — workspaces are ephemeral and dependencies may have moved.

### 1. One default constant

`src/sase/ace/tui/_artifact_tab_model.py` — add next to `DEFAULT_ARTIFACTS_SUBTAB`:

```python
DEFAULT_ARTIFACTS_RELATIONS_COLLAPSED: bool = True
```

Re-export it from `src/sase/ace/tui/artifact_tabs.py` (import + `__all__`), which is the
path `app.py` already imports these defaults through.

Use it as the single source of truth at all four current literal sites, so the default
can never disagree with itself:

- `src/sase/ace/tui/app.py:175` —
  `artifacts_relations_collapsed: reactive[bool] = reactive(DEFAULT_ARTIFACTS_RELATIONS_COLLAPSED, recompose=False)`.
- `src/sase/ace/tui/widgets/artifacts/relation_panel.py` — the two
  `getattr(app, "artifacts_relations_collapsed", False)` fallbacks (in
  `refresh_relation_panel` and `relation_footer_entries`).
- `src/sase/ace/tui/widgets/_keybinding_bindings.py:490` — same fallback.

`_artifact_tab_model` is Textual-free and already imported by `relation_panel.py`, so no
import cycle and no import-time cost.

### 2. Config field

- `src/sase/default_config.yml` — under `ace: artifacts:`, add
  `relations_expanded: false` with a short comment saying `.` toggles it for the current
  session only (match the comment style of `axe_description_expanded` above it).
- `src/sase/config/sase.schema.json` — add to the `ace.artifacts` properties object:

  ```json
  "relations_expanded": {
    "type": "boolean",
    "description": "Whether the Artifacts relation panel starts expanded; the configured toggle key changes this only for the current ACE session.",
    "default": false
  }
  ```

  `ace.artifacts` is `additionalProperties: false`, so the schema entry is required for
  the field to load at all.

- `src/sase/ace/tui/actions/_state_init_late.py` — seed it beside the existing
  `axe_description_expanded` block, from the `merged`/`ace_cfg` dict already loaded
  there. Read `ace_cfg["artifacts"]["relations_expanded"]` defensively (both levels may
  be absent or non-dict), require `isinstance(value, bool)`, and set the reactive
  backing attribute directly, matching the neighbouring style:

  ```python
  self._reactive_artifacts_relations_collapsed = not relations_expanded
  ```

  `_init_app_state` runs from `AceApp.__init__`, before compose, so the first relation
  render already sees the configured value — no first-frame flip.

**Perf constraint (`sase/memory/tui_perf.md` rules 8 and 9):** config is read exactly
once, in the existing startup call that already calls `load_merged_config()`. Never call
`load_merged_config()` from `refresh_relation_panel`, `update_relations`, or
`_build_collapsed_rail`; those are render paths.

### 3. Rail rendering

`src/sase/ace/tui/widgets/artifacts/relation_panel.py`:

- Extend the app-keymap helper so the toggle key resolves like the mode keys do. Today
  `_mode_keys_from_app` returns `Mapping[RelationRole, str]`; add a sibling
  `_toggle_key_from_app(app) -> str` using the same `registry.app` + `key_display_name`
  lookup with fallback `"."`. (The help modal already resolves it this way:
  `key_display_name(km.app.toggle_relation_panel)` in
  `modals/help_modal/patches_artifact_bindings.py:317`.)
- Thread the resolved key through `update_relations` → `_refresh_collapsed` →
  `_build_collapsed_rail` as a new keyword argument with a `"."` default, so the pure
  builder stays directly unit-testable.
- Build the chip, verb, separator, then the existing segments, per the anatomy above.

### 4. Expanded frame

Also in `relation_panel.py`, in `_refresh_content` / `_refresh_collapsed`:

- Expanded: set `self.border_title = "▾ RELATIONS"` and
  `self.border_subtitle = f"{toggle_key} collapse"`.
- Collapsed: clear both (`""`) — the collapsed rail has no border to hang them on.
- `clear()` clears both too.

Style them in `styles.tcss` on `.artifacts-relation-panel`. `border-title-color` and
`border-subtitle-align: right` are both already used in that stylesheet; confirm
`border-subtitle-color` parses under the pinned `textual==8.0.1` before relying on it,
and simply leave the subtitle unstyled if it does not. Use a static color from the
existing palette rather than mutating `self.styles` per refresh — per-render style
mutation is not worth a per-pane accent here.

### 5. Click and hover

- `relation_panel.py`: add `class Clicked(Message)` to `RelationPanel` and an `on_click`
  that posts it **only** when the panel currently has the `-collapsed` class.
- `src/sase/ace/tui/widgets/artifacts/view.py`: add an `@on(RelationPanel.Clicked)`
  handler beside `_on_split_badge_clicked` that calls `event.stop()` then the app's
  `action_toggle_relation_panel()`. That action is already Artifacts-guarded and already
  refreshes the panel, keymap, and footer.
- `styles.tcss`: add

  ```
  .artifacts-relation-panel.-collapsed:hover {
      background: #29293A;
  }
  ```

  `#29293A` is the hover background this stylesheet already uses
  (`.admin-center-home-row:hover`, `.statistics-tile:hover`); do not introduce a new
  hover color.

### 6. Not in scope

- **Per-sub-tab collapse state.** The flag stays global; that is existing documented,
  tested behavior and the user asked for a default, not a new axis of state.
- **Persisting session toggles to disk.** Matches `axe_description_expanded`: config
  seeds, the key toggles for the session only.
- **Rail label pluralization** (`1 plans`, `2 children`). The rail prints the
  contract-declared label verbatim by design, labels come from third-party providers as
  well as core panes, and English irregulars (`Children`) defeat any rule small enough
  to be worth writing. Leave it; do not invent grammar in the renderer.
- **Rust core.** The relation panel is presentation — Textual state, rendering, and
  keybindings — and its layout module (`src/sase/core/artifact_relation_layout.py`) is
  already Python-side. No `../sase-core` wire, binding, or API change is needed.

## Tests

Add to `tests/ace/tui/test_artifacts_relation_collapse.py` unless noted.

Unit (pure, no app):

- `_build_collapsed_rail` puts the chip and the word `expand` before the first
  navigation segment, and still renders one line.
- The rail prints the **passed** toggle key, not a hardcoded `.` (assert with a
  non-default key such as `^r`).
- An empty view still returns `None` (existing test — keep it green).

App-level:

- **Default is collapsed** on each relations pane. Extend the existing pane-matrix test
  (patches / stitches / beads / files / plans) to assert the panel carries `-collapsed`
  and the rail is one line _before_ any keypress, then that `.` expands it and `.`
  collapses it again. Every existing assertion of the shape
  `assert page.app.artifacts_relations_collapsed is False` at startup flips; audit the
  whole file rather than patching the first failure.
- The configured toggle key appears in the on-screen rail, and rebinding
  `toggle_relation_panel` changes it (an app-level test that goes through the keymap
  registry, guarding against a hardcoded `.` regression).
- `ace.artifacts.relations_expanded: true` starts **expanded**; absent config starts
  collapsed. Use the config-loading fixtures already used by config-sensitive TUI tests
  and remember `clear_config_cache()` (`sase/memory/tui_perf.md` rule 8).
- Clicking the collapsed rail expands it (`await page.click("#beads-relation-panel")`,
  `AcePage.click` exists at `src/sase/ace/testing/ace_page.py:275`); clicking the
  expanded panel leaves it expanded.
- Narrow-terminal guard: at 80 columns the rendered rail still contains the expand chip
  and verb.
- Existing coverage that must stay green unchanged: the relation keymap stays live while
  collapsed (`<` still navigates), the footer label flips `collapse relations` /
  `expand relations`, and collapse state persists across sub-tabs.

Schema/config:

- `tests/test_config_schema.py` — a case asserting `ace.artifacts.relations_expanded`
  validates as a boolean and that `default_config.yml` still matches the public schema
  (`test_default_config_matches_public_schema` covers the latter; confirm it passes).

Visual snapshots (`just test-visual`):

- `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_beads.py::test_artifacts_beads_collapsed_relations_rail_png_snapshot`
  reaches the collapsed state by pressing `.`; with the new default that press _expands_
  it. Rewrite the test to assert the default-collapsed state directly (drop the press,
  or press twice if the intent is to prove the round trip), and keep a
  `wait_for_svg_contains` anchored on rail text that still exists.
- Regenerate every golden the new default and the new frame change:
  `just test-visual -- --sase-update-visual-snapshots`, then **look at each changed
  PNG** before accepting it — this change is judged on how it looks. Expect at least the
  artifacts beads / plans / files / stitches / split goldens under
  `tests/ace/tui/visual/snapshots/png/` to move (the relation panel appears in the list
  column of any golden whose selected entry has relations). Report the exact list of
  regenerated goldens in the handoff.

## Docs

- `docs/artifacts_pane_visual_grammar.md` — the "Relation panel slot" section owns the
  rail spec. Update the paragraph describing the `▸` rail to the new anatomy (chip,
  verb, two key registers, truncation order), state the new default, and document the
  expanded frame's `▾ RELATIONS` title and collapse subtitle.
- `docs/ace.md` — the relation-panel paragraph (~line 155) should say the panel starts
  collapsed as a rail and how to expand it (key **and** click). The `.` row in the
  keymap table (~line 2405) still reads correctly; verify rather than rewrite. Add the
  new config field near the `ace.axe_description_expanded` prose (~line 2158) if that
  section lists starting states.
- `docs/configuration.md` — add `relations_expanded` to the `ace.artifacts` block: the
  example YAML (~line 643), and a table row plus field description in the same shape as
  the existing `ace.artifacts.stitches` entry (~line 746).

Do **not** touch `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction
shims; nothing here requires a memory change.

## Verification

1. `just install`
2. `just check` inline while iterating.
3. `just test-visual` for the PNG suite; inspect `.pytest_cache/sase-visual/`
   actual/expected/diff artifacts on any failure.
4. `just check-full` before landing — it outruns a single agent turn, so run it **only**
   through `/sase_monitor` (`sase monitor start --command 'just check-full' …`) with a
   `--next` action, never inline.
5. Eyeball the result in a real terminal at both 120 and 80 columns: `sase ace`, open
   Artifacts, walk Stitch / Patch / Bead / Plan / File, confirm the rail reads clearly,
   `.` round-trips, and a mouse click on the rail expands it.
