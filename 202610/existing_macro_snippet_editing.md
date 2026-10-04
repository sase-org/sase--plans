---
tier: epic
title: Edit existing macros and snippets from the Ctrl+G x / Ctrl+G t location picker
goal: 'From the prompt input, `Ctrl+G x` / `Ctrl+G t` (and `gx` / `gt`) offer an `e`
  (existing) row in the location picker that opens a beautiful fuzzy finder over every
  macro or snippet definition in every supported file. Picking one opens it in the
  existing mini-macro / snippet pane for in-place editing, or starts a guided override
  when the definition is read-only. Every path that would redefine an existing macro
  or snippet, whatever destination the user chose, shows an accurate warning that
  names where the name already lives and whether the new copy will take effect.

  '
phases:
- id: macro-redefinition
  title: Accurate macro redefinition analysis and warnings
  depends_on: []
  size: medium
  description: 'macro-redefinition: make the mini-macro target catalog see every loader
    source, use the runtime loader as the oracle for the active definition, add a
    shared macro redefinition helper, rewrite the name-step verdicts with the shared
    warning copy (including the shadowed in-place edit case), and refresh the warning
    at save time.

    '
- id: snippet-redefinition
  title: Provenance-accurate snippet redefinition analysis and warnings
  depends_on: []
  size: medium
  description: 'snippet-redefinition: replace the inverted, partial `snippet_collision`
    check with a helper built on the provenance snippet catalog''s real layer order
    (built-in, plugin, user, overlay, project, and macro-derived sources plus aliases),
    then wire it into the snippet name step, its loaded body, and the save-time warning.

    '
- id: picker-existing-row
  title: Existing row and override mode in the save-location picker
  depends_on: []
  size: medium
  description: 'picker-existing-row: teach the shared choice builders and picker modal
    to render an optional `e` Existing action row and an override mode. In override
    mode, rows are filtered by whether the name fits and badged with an injected after-save
    outcome. The live flows do not pass these options yet.

    '
- id: existing-finder
  title: Existing-definition fuzzy finder modal
  depends_on:
  - macro-redefinition
  - snippet-redefinition
  size: medium
  description: 'existing-finder: build the pure entry model, entry builders, ranking,
    and verdict copy for macros and snippets, then build the shared presentation-only
    finder modal with Rust fuzzy highlighting, status chips, a debounced off-thread
    preview, and back/cancel results.

    '
- id: wire-macro-existing
  title: Wire the existing path into the mini-macro flow
  depends_on:
  - picker-existing-row
  - existing-finder
  size: medium
  description: 'wire-macro-existing: connect the `e` row, the finder, in-place edits,
    and the read-only override detour into `_MiniMacroLocationFlow`. Add replace-draft
    and dirty-guard semantics for an already-open pane, refresh the hint label, and
    update the shared picker docs and the mini-macro docs.

    '
- id: wire-snippet-existing
  title: Wire the existing path into the snippet flow
  depends_on:
  - wire-macro-existing
  size: medium
  description: 'wire-snippet-existing: mirror the macro wiring in `_SnippetLocationFlow`
    using the provenance catalog, add replace-draft semantics to the snippet pane,
    refresh the hint label, and update the snippet authoring docs.'
proposed_by: bbugyi200.athena.0w7
create_time: 2026-10-04 06:32:56
status: wip
bead_id: sase-1fv
---

- **PROMPT:** [prompts/202610/existing_macro_snippet_editing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/existing_macro_snippet_editing.md)
- **BEAD:** [sase-1fv](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1fv/README.md)

# Plan: Edit existing macros and snippets from the Ctrl+G x / Ctrl+G t picker

## Background (current behavior)

- `Ctrl+G x` / `Ctrl+G Ctrl+X` / `gx` calls `request_mini_macro_target_pane()`
  (`src/sase/ace/tui/widgets/_prompt_input_bar_mini_macro_pane.py`). The app then runs
  `_MiniMacroLocationFlow`
  (`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_mini_macro_pane.py`): it pushes
  `SaveLocationPickerModal` right away, loads the rows, the `MiniMacroTargetCatalog`,
  and the choices off-thread, then opens `MiniMacroNameModal` and finally the pinned
  mini-macro pane.
- `Ctrl+G t` / `Ctrl+G Ctrl+T` / `gt` follows the same path through
  `_SnippetLocationFlow`
  (`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_snippet_pane.py`), then
  `SnippetNameModal`, then the snippet pane.
- The picker (`src/sase/ace/tui/modals/save_location_picker_modal.py`) only presents
  data. Its rows come from the pure builders `macro_location_choices` and
  `snippet_location_choices` (`src/sase/ace/tui/modals/save_location_choices.py`). Macro
  hotkeys are `p P h H 1-9` and snippet hotkeys are `c p h 1-9`; `j k q +` are reserved.
  **`e` is free in both pickers.** Keys typed while the picker is still loading are
  buffered: the first key becomes the pending pick and the rest become type-ahead.
- The only way to edit an existing definition today is to remember its exact name and
  location, choose that location, type the name, and get the `Edit` verdict.

### Redefinition-warning gaps found while designing (fixed by phases 1–2)

1. **The snippet precedence is inverted.** `snippet_collision()`
   (`src/sase/macro/snippet_targets.py`) treats
   `[user sase.yml, sase_*.yml overlays, project sase/sase.yml]` as first-wins. At
   runtime, though, `load_snippet_catalog()` (`src/sase/snippet/catalog.py`) replays the
   config layers `default → plugin:* → user → overlay:* → project`, and later layers
   win. As a result, saving to the project config while the user config defines the
   trigger warns "will be shadowed by ~/.config/sase/sase.yml", which is the opposite of
   what happens.
2. **Snippet checks ignore built-in and plugin `default_config.yml` `ace.snippets` and
   alias names.** Only user, overlay, and project files plus macro-derived triggers are
   checked.
3. **The macro catalog misses some loader sources.** `load_mini_macro_target_catalog()`
   (`src/sase/ace/tui/modals/mini_macro_target_catalog.py`) only parses the unified save
   rows. Plain macros that `get_all_macros()` finds elsewhere (legacy fallback dirs,
   registry-backed project copies) never appear, so redefining them gives no warning.
   For catalog-only entries, `precedence=macro.discovery_rank or 1000` mixes a
   higher-wins rank into a lower-wins scale.
4. **Shadowed in-place edits look like success.** In `MiniMacroNameModal`,
   `Edit #x at <dest>` is a green verdict even when `<dest>`'s copy is shadowed by a
   higher-precedence definition, so the edit will never take effect. Fork verdicts also
   never say whether the new copy will win.
5. **Save-time warnings go stale.** The save confirmation shows the `save_warning`
   captured when the name was chosen and never recomputes it.

## Design

### Interaction flow

```
Ctrl+G x / Ctrl+G t
  └─ Location picker ─────────────── (unchanged create path: p/P/h/H/c/1-9/↵ → name step → pane)
       └─ e  ✎ Edit existing …  ──► Existing finder (fuzzy)
                                      ├─ editable definition ─► pane opens with that body (edit in place)
                                      ├─ read-only definition ─► picker in OVERRIDE mode ─► name step (prefilled, warns) ─► pane
                                      ├─ incompatible ─► Enter refused, verdict explains why
                                      ├─ ⇧Tab ─► back to the picker (query remembered for the next `e`)
                                      └─ Esc ─► cancel, focus returns to the origin pane
```

- Type-ahead still works: pressing `e` while the picker loads, then typing `rev`, opens
  the finder with query `rev`.
- If a pane of the same kind is already open, the row reads `Switch to existing macro…`
  or `Switch to existing snippet…`. Picking a definition replaces the pane's draft
  (body, frontmatter, and target). A dirty draft goes through the existing discard
  confirmation first. Picking the definition the pane already targets just focuses the
  pane and shows `Already editing #name`.
- Picker keys are built in, not configurable keymaps, so `src/sase/default_config.yml`
  needs no change.

### Picker: the Existing section (phase `picker-existing-row`)

```
 New mini-macro · where should it live?                    ● Location › ○ Name
 ── Existing ──
 e  ✎ Edit existing macro…            148 macros
 ── Project · sase ──
 p  📁 Project macros   sase/macros/                ★ last used
 P  📄 Project config   sase/sase.yml
 ── Home ──
 h  📁 Home macros      ~/sase/macros/
 H  📄 User config      ~/.config/sase/sase.yml
 ── Plugins & built-in (4) · + to show ──
 → fuzzy-find 148 macros across 9 files · edit in place or override read-only ones
 e existing · p P h H pick · ↵ last used · j/k move · + plugins · esc cancel
```

- The section sits at the top, so it is always in the same place and never moves when
  plugins expand. The label uses its own lavender accent (`bold #D7AFFF`) and the `✎`
  icon. The count badge is dim.
- The existing row is never the `★` default and never the fallback default. The "No
  writable destinations found" branch only looks at destination rows. With zero
  definitions the row is disabled: `no macros yet` / `no snippets yet`.

### Finder (phase `existing-finder`)

```
╭────────────────────────────────────────────────────────────────────────────────╮
│ ✎ Edit existing macro                                     ✓ Existing › ● Find │
│ ❯ rev▏                                                                12 / 148 │
│ ┌ Matches ─────────────────────────────┐ ┌ Preview ───────────────────────────┐ │
│ │ #review              ● active        │ │ #review · sase/macros/review.md    │ │
│ │   sase/macros/review.md              │ │ shadows ~/sase/macros/review.md    │ │
│ │ #review              ◐ shadowed      │ │  1 ---                             │ │
│ │   ~/sase/macros/review.md            │ │  2 input:                          │ │
│ │ #review_pr           🔒 built-in     │ │  3   - name: diff                  │ │
│ │   …/sase/default_macros/review_pr.md │ │  4 ---                             │ │
│ │ #sase/revert_swarm   ✗ swarm         │ │  5 Review {{ diff }} carefully…    │ │
│ └──────────────────────────────────────┘ └────────────────────────────────────┘ │
│ ✓ Edit #review in place · sase/macros/review.md                                │
│ ↑↓ ^n/^p move · enter open · ^d/^u scroll preview · ⇧tab locations · esc cancel│
╰────────────────────────────────────────────────────────────────────────────────╯
```

- There is one row per _physical definition_, so shadowed copies are listed too. Matched
  characters are highlighted from the Rust fuzzy runs.
- With an empty query, rows sort as compatible before incompatible, then name
  alphabetically, then active before shadowed, then precedence. With a query, rows sort
  by fuzzy match on the name (`tier`, `-score`, length, name), then the same tiebreaks.
  Rows that match only on the display path come after every name match.
- Snippet rows use `⇥ trigger`. For their second line they show the config file,
  `built-in default_config.yml`, `plugin <module>`, or `from #macro`.

Status chips are shared by both kinds:

| Chip                                                            | Style     | Meaning                                                                | Enter                         |
| --------------------------------------------------------------- | --------- | ---------------------------------------------------------------------- | ----------------------------- |
| `● active`                                                      | green     | runtime winner, writable                                               | edit in place                 |
| `◐ shadowed`                                                    | `#D7AF5F` | writable, but another definition wins (2nd line: `shadowed by <path>`) | edit in place, with a warning |
| `🔒 built-in` / `🔒 plugin` / `🔒 read-only` / `🔒 from #macro` | dim gold  | not writable here                                                      | override detour               |
| `✗ swarm` / `✗ workflow` / `✗ skill` / `✗ memory`               | dim red   | not a simple mini target                                               | refused; verdict explains why |

### Shared warning copy (phases 1, 2, 4)

`<ref>` is `#name` for macros or `⇥ trigger` for snippets. `<path>` is a short display
path or an origin label such as `built-in default_macros/x.md`, `plugin sase_github`, or
`#macro (macro snippet)`. The name-step verdict, the finder verdict, and the save
confirmation all use the same sentences:

| Situation                                                   | Kind    | Copy                                                                                                        |
| ----------------------------------------------------------- | ------- | ----------------------------------------------------------------------------------------------------------- |
| fresh name                                                  | success | `✓ Create <ref> at <dest>`                                                                                  |
| edit in place, destination is active                        | success | `✓ Edit <ref> in place · <dest>`                                                                            |
| edit in place, destination is shadowed                      | warning | `⚠ <ref> in <dest> is shadowed by <winner> — edits here won't take effect`                                  |
| exists elsewhere, new copy will win                         | warning | `⚠ <ref> already exists in <active>[ (+N more)] — saving to <dest> will override it`                        |
| exists elsewhere, new copy will lose                        | warning | `⚠ <ref> already exists in <active> — a copy in <dest> would be shadowed by <winner> and won't take effect` |
| active definition is outside the save destinations (macros) | warning | `⚠ <ref> already exists in <active> — saving to <dest> adds another definition`                             |
| trigger is currently an alias (snippets)                    | warning | `⚠ ⇥ <t> is an alias of ⇥ <source> — saving defines ⇥ <t> directly`                                         |
| finder: read-only                                           | warning | `⚠ <ref> is <origin> (read-only) — Enter picks where your override should live`                             |
| incompatible                                                | error   | `✗ Cannot open <ref>: <reason>`                                                                             |

When the definition that wins lives in a writable row other than `<dest>`, keep the
existing mini-macro `· ⇧tab to pick <label>` suffix, reworded to
`· ⇧tab to edit it in <label> instead`.

### Architecture decisions

- **Precedence logic stays where it already lives.** Macro discovery (`get_all_macros`)
  and snippet provenance (`load_snippet_catalog`) are owned by Python in this repo. The
  new snippet helper therefore goes in `src/sase/snippet/` next to the catalog and the
  mutation service that the CLI uses, and the macro helper extends the existing
  mini-macro catalog. No `sase-core` change is needed. Do not reimplement fuzzy
  matching: use `sase.core.fuzzy_facade` (`fuzzy_match`, `fuzzy_sort_key`).
- **Modals only present data.** The finder receives immutable entries plus an injected
  `preview_loader`, and it never imports from `actions/`. This follows the picker's
  existing contract.
- **Performance rules** (see the TUI perf memory): no disk I/O on the event loop or per
  keystroke. Catalogs and entries load off-thread, and the macro flow reuses the catalog
  it already loads. Finder ranking is in-memory, with rendered rows capped at 200 and a
  `shown / total` counter. Previews are debounced with `DetailPanelDebouncer`, read via
  `asyncio.to_thread`, cached per entry id, and dropped if stale after the await.
  Silence programmatic `OptionList.highlighted` echoes with the guard-flag pattern the
  name modals use.
- **File size.** `snippet_name_modal.py` (676 lines) and `mini_macro_name_modal.py`
  (602) are near the `toobig` thresholds (`1000 850 700`). Put new logic in new modules
  and keep every touched file under 700 lines.

### Non-goals

- The `gX` / `Ctrl+G X` unified save panel, the Snippets panel (`gT`) add flow, and
  `sase snippet add` keep their current collision handling.
- No warning about inactive machine overlays (a `sase_*.yml` overlay selected for
  another machine). A destination outside the active layers keeps today's "highest
  precedence" treatment.
- Editing workflows, skills, memory macros, or macro swarms from the finder. They are
  listed but refused, with a reason.

---

## Phase `macro-redefinition`: Accurate macro redefinition analysis and warnings

Files: `src/sase/ace/tui/modals/mini_macro_target_catalog.py`, a new
`src/sase/ace/tui/modals/mini_macro_redefinition.py` (keeps the catalog and name modal
under the size limit), `src/sase/ace/tui/modals/mini_macro_name_modal.py`,
`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_macro_mini_io.py`, and
`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_macro_mini.py`.

1. **Complete the catalog.** `_load_catalog_only_definitions` must also yield plain
   macros (not skill, memory, or workflow) from `get_all_macros(project)` whose source
   is not already represented by a row definition. Give them
   `compatibility="read_only"`, or `"incompatible"` with the swarm reason when
   `macro_has_segment_separators`. Dedupe by `(name, normalized source path)`, where the
   normalized path is `resolve_macro_write_target(path).write_path`. With chezmoi, rows
   point at chezmoi source paths while the loader reports deployed paths, so comparing
   raw paths produces duplicates. Add an optional
   `MiniMacroDefinition.origin_label: str | None` (`"built-in"`, `"plugin"`,
   `"read-only"`, or `None` for writable rows), derived from the row group
   (`Built-in (dev)`, `Plugin directories`) or set to `"read-only"` for catalog-only
   sources.
2. **Use the loader as the oracle for the active definition.** In
   `_annotate_precedence`, mark as `effective` the definition whose normalized source
   path (and `entry_name` for config entries) matches
   `get_all_macros(project)[name].source_path`. Order it first, then the rest by row
   precedence. Fall back to pure precedence when nothing matches. Derive `shadowed_by` /
   `shadows` from that order. Stop putting `discovery_rank` on the lower-wins precedence
   scale: give catalog-only definitions a precedence after all rows (for example
   `1000 + ordinal`), since ordering among them is only a tiebreak once the oracle has
   decided the winner.
3. **Add the shared helper** in `mini_macro_redefinition.py`:

   ```python
   @dataclass(frozen=True, slots=True)
   class MacroRedefinition:
       name: str
       destination: MiniMacroDestinationTarget
       destination_definition: MiniMacroDefinition | None
       active: MiniMacroDefinition | None          # loader-oracle winner today
       others: tuple[MiniMacroDefinition, ...]     # every other definition, active first
       winner_after_save: str | None               # display of what still beats <dest>; None → dest wins
       active_outside_rows: bool                   # active source is not a save row → don't claim a winner

   def macro_redefinition(catalog, name, row) -> MacroRedefinition
   def macro_redefinition_warning(redef) -> str | None   # shared copy table; None when nothing to warn
   ```

   `winner_after_save` reuses `destination_target_for_name(...).resolution.shadowed_by`,
   which is the existing row-based `resolution_after_save`, mapped to a display path.

4. **Rewrite `_build_mini_macro_verdict`** on top of the helper, using the shared copy
   table. Keep the action semantics (`edit` / `fork` / `override` / `create`) and the
   incompatible errors. New behavior: an `edit` whose destination definition is not
   active becomes a **warning** with a `save_warning`. Fork and override messages state
   the after-save outcome and the `(+N more)` count.
5. **Refresh the warning at save time.** Add
   `refresh_mini_macro_save_warning(project, target) -> str | None` in
   `_prompt_bar_save_macro_mini_io.py`. It reloads the rows and catalog off-thread,
   finds the row whose `location.path` equals `target.location_path`, and returns
   `macro_redefinition_warning(...)`. In
   `on_prompt_input_bar_mini_macro_pane_save_requested`, run it in the same off-thread
   step as `load_mini_macro_save_disk_state` (the project comes from
   `self._prompt_context`, or `None` in home mode). Pass the fresh value as the confirm
   state's `warning`. If the refresh raises or finds no row, fall back to
   `mini_macro_save_warning(target)`. A save that was already confirmed must never be
   blocked by the refresh.

Tests: extend `tests/ace/tui/modals/test_mini_macro_target_catalog.py` with catalog-only
plain macros, chezmoi dedupe, oracle-based `effective`, and catalog-only ordering.
Extend `tests/ace/tui/modals/test_mini_macro_name_modal.py` with the shadowed-edit
warning, both fork outcomes, `(+N more)`, and the override copy. Add save-time refresh
tests next to the existing mini save tests in
`tests/ace/tui/actions/test_prompt_save_macro*.py`. Update any copy assertions and PNG
goldens that change (`tests/ace/tui/visual/test_ace_png_snapshots_mini_macro.py`).

Acceptance: every situation in the shared copy table that applies to macros renders as
specified in the name step and in the save confirmation, and `just check` passes.

## Phase `snippet-redefinition`: Provenance-accurate snippet redefinition analysis and warnings

Files: `src/sase/snippet/models.py`, `src/sase/snippet/catalog.py`, a new
`src/sase/snippet/redefinition.py`, `src/sase/ace/tui/modals/snippet_name_modal.py`
(move logic out so it stays under 700 lines),
`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_snippet_pane.py`,
`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_macro_snippets.py`, and
`src/sase/macro/snippet_targets.py`.

1. **Record layer order on the catalog.** Add
   `SnippetCatalog.layer_paths: tuple[str | None, ...] = ()`: the active config layers
   in runtime (later-wins) order, from `_load_raw_layers`, with `None` for the default
   and plugin layers. Add `SnippetSourceContribution.layer: str | None = None`, filled
   with the layer name (`default`, `plugin:<module>`, `user`, `overlay:<file>`,
   `local`). Both fields are additive with defaults. Do not change `display_path`, which
   the editor-helper wire depends on.
2. **Add the helper** in `src/sase/snippet/redefinition.py`:

   ```python
   @dataclass(frozen=True, slots=True)
   class SnippetDefinitionSite:
       trigger: str
       kind: SnippetSourceKind
       path: str | None
       display: str            # short path, "built-in default_config.yml", "plugin <module>", "#macro (macro snippet)"
       template: str
       writable: bool
       active: bool
       shadowed_by: str | None # display of the winner when not active

   @dataclass(frozen=True, slots=True)
   class SnippetRedefinition:
       trigger: str
       destination_path: str
       destination_site: SnippetDefinitionSite | None
       active: SnippetDefinitionSite | None
       others: tuple[SnippetDefinitionSite, ...]  # active first, then descending runtime precedence
       winner_after_save: SnippetDefinitionSite | None  # None → destination wins
       alias_of: str | None

   def snippet_definition_sites(catalog) -> tuple[SnippetDefinitionSite, ...]
   def snippet_redefinition(catalog, trigger, destination_path) -> SnippetRedefinition
   def snippet_redefinition_warning(redef, destination_display: str) -> str | None
   ```

   Rank contributions as macro-derived `xprompt` (lowest), then `default`, then each
   `plugin`, then each layer path in `layer_paths` order. The project layer is last and
   wins. Match the destination to a layer by comparing both
   `resolve_macro_write_target(p).read_path` and `.write_path`, so chezmoi source paths
   match deployed layer paths. A destination that matches no active layer (an
   out-of-discovery `ace.snippet_config_path`) ranks highest, which preserves today's
   convention. Also check `catalog.alias_provenance` for `alias_of`. Count a site as
   writable only when its path maps to a `SnippetConfigLocation` with no
   `disabled_reason`.

3. **Snippet name step.** `SnippetNameModal` takes the loaded `SnippetCatalog` instead
   of the derived maps and builds its verdict from `snippet_redefinition` and the shared
   copy table. The current "exists here" case becomes `✓ Edit ⇥ t in place` when the
   destination is active, or the shadowed-edit warning when it is not. `existing_body`
   for a definition that does not live at the destination now comes from
   `redef.active.template`, which is in memory and so also works for built-in, plugin,
   and macro-derived sources that have no file path. Set `derived_from` when the active
   kind is `xprompt`. The prefix-match list may read from catalog sites instead of
   reading per-match files from disk.
4. **Flow.** `_SnippetLocationFlow.load_and_deliver` loads
   `load_snippet_catalog(project)` off-thread, keeps it on the flow, and passes it to
   the name modal. This replaces `_load_derived_snippet_catalog`. `redeliver_for_name`
   reuses the stored catalog.
5. **Refresh the warning at save time.** In `_prompt_bar_save_macro_snippets.py`,
   recompute the warning off-thread with a fresh `load_snippet_catalog(project)` plus
   `snippet_redefinition_warning`, as in the macro phase, and fall back to
   `_snippet_save_warning(target)` on any error.
6. Delete `snippet_collision`, `SnippetCollision`, and `_SnippetTriggerMatch` from
   `snippet_targets.py` once nothing uses them, and remove their `__all__` entries and
   tests. Leaving them would trip symvision's unused-symbol check.

Tests: add `tests/snippet/test_redefinition.py` covering project-beats-user,
overlay-beats-user, user-beats-plugin/default, macro-derived lowest, built-in/plugin
detection, chezmoi path mapping, an out-of-discovery configured destination, aliases,
and every copy row. Update `tests/ace/tui/modals/test_snippet_name_modal.py`,
`tests/ace/tui/actions/test_prompt_snippet_location_flow.py`,
`tests/ace/tui/actions/test_prompt_save_snippet_pane.py`, and
`tests/macro/test_snippet_targets.py`. Update the snippet-name PNG goldens
(`tests/ace/tui/visual/test_ace_png_snapshots_snippet_name.py`) whose copy changes.

Acceptance: with user and project configs both defining `todo`, choosing the user config
warns that the copy would be shadowed by the project config, and choosing the project
config warns that it will override the user config. A trigger defined only in a built-in
or plugin `default_config.yml` warns with that origin. `just check` passes.

## Phase `picker-existing-row`: Existing row and override mode in the save-location picker

Files: `src/sase/ace/tui/modals/save_location_choices.py`,
`src/sase/ace/tui/modals/save_location_picker_modal.py`, and
`src/sase/ace/tui/styles.tcss` (if you need styles beyond Rich text).

1. **Choice model.** Add `EXISTING_CHOICE_ID = "__existing__"`, add `"existing"` to
   `SaveLocationKind`, and add
   `@dataclass(frozen=True, slots=True) class ExistingRowSpec: count: int; switching: bool = False`.
   `macro_location_choices` and `snippet_location_choices` gain keyword-only
   `existing: ExistingRowSpec | None = None`. When it is given (and override mode is
   off), prepend one choice: section `Existing`, hotkey `e`, label
   `Edit existing macro…` / `Edit existing snippet…` (or `Switch to existing …` when
   `switching`), badge `N macros` / `N snippets`, preview
   `→ fuzzy-find N macros across M files · edit in place or override read-only ones`
   (the builders already know the row count M), and `disabled_reason="no macros yet"` /
   `"no snippets yet"` when `count == 0`. It must never be `is_default` and never count
   toward hotkey digits.
2. **Override mode.** Both builders gain keyword-only `override_name: str | None = None`
   and `shadowed_by: Mapping[str, str] | None = None` (choice id → display of the
   definition that would still win). When `override_name` is set:
   - Omit the Existing row.
   - For macros, disable rows whose namespace would change the callable name
     (`rebase_name_for_destination` result differs from `override_name`) with reason
     `saves as #<rebased> — can't override #<name>`.
   - Badge rows found in `shadowed_by` with `⚠ shadowed by <x>`. They stay selectable.
   - Default precedence is current → last used → the first selectable row that is not
     shadowed (display order) → the first selectable row. For snippets, `★ configured`
     still sits right after current.
   - Badge the default with `★ override`.
3. **Modal.** Render `kind == "existing"` with the `✎` icon and a `bold #D7AFFF` label.
   Leave the existing row out of `_default_id`'s first-selectable fallback and out of
   the "No writable destinations found" check. Hints start with `e existing` and the `e`
   is dropped from the generic `… pick` letters. Buffered loading resolves `e` like any
   other hotkey: `SaveLocationPick(EXISTING_CHOICE_ID, typeahead)`.
4. The live flows do not pass these options yet. They are wired in the
   `wire-macro-existing` and `wire-snippet-existing` phases, so this phase changes
   nothing visible on its own.

Tests: `tests/ace/tui/modals/test_save_location_choices.py` covers the row presence,
count, switching label, disabled-empty state, ordering, non-default, override filtering,
badges, and default. `tests/ace/tui/modals/test_save_location_picker_modal.py` covers
the hotkey, buffered `e` plus type-ahead, hints, and that Enter still picks the `★`
default. Add PNG goldens for a macro picker and a snippet picker with the Existing row
and one override-mode picker in
`tests/ace/tui/visual/test_ace_png_snapshots_save_location_picker.py`.

## Phase `existing-finder`: Existing-definition fuzzy finder modal

Files: new `src/sase/ace/tui/modals/existing_definition_entries.py` (pure), new
`src/sase/ace/tui/modals/existing_definition_finder_modal.py`,
`src/sase/ace/tui/modals/__init__.py` (exports), and `src/sase/ace/tui/styles.tcss` (an
`ExistingDefinitionFinderModal` block styled like `MiniMacroNameModal`: `thick $primary`
border, `$surface` background, success/warning/error verdict classes).

1. **Entry model and builders (pure, no Textual):**

   ```python
   ExistingStatus = Literal["active", "shadowed", "read_only", "incompatible"]

   @dataclass(frozen=True, slots=True)
   class ExistingDefinitionEntry:
       entry_id: str                 # stable: "macro:<norm path>:<entry or ''>:<name>" / "snippet:<layer|kind>:<path|''>:<trigger>"
       kind: Literal["macro", "snippet"]
       name: str
       reference: str                # "#review" / "⇥ todo"
       display_path: str
       origin_label: str | None      # "built-in", "plugin <module>", "from #macro", …
       status: ExistingStatus
       shadowed_by: str | None
       reason: str | None            # read-only/incompatible explanation (chip text + verdict)
       precedence: int
       preview_text: str | None      # in-memory body (snippets); None → lazy preview_loader

   def macro_existing_entries(catalog: MiniMacroTargetCatalog) -> tuple[ExistingDefinitionEntry, ...]
   def snippet_existing_entries(sites: Sequence[SnippetDefinitionSite]) -> tuple[ExistingDefinitionEntry, ...]
   def rank_existing_entries(entries, query, *, limit=200) -> tuple[RankedExistingEntry, ...]  # entry + name/path runs
   def existing_entry_verdict(entry) -> tuple[Literal["success","warning","error"], str]  # shared copy table
   ```

   For macros, status comes from the `macro-redefinition` catalog: `editable` +
   effective is `active`, `editable` + not effective is `shadowed`, `read_only` is
   `read_only`, `incompatible` is `incompatible` (chip from `workflow_kind` or `swarm`).
   For snippets, status comes from `SnippetDefinitionSite`: writable + active is
   `active`, writable + not active is `shadowed`, and anything not writable is
   `read_only`. Ranking follows the Design section and uses `fuzzy_match` /
   `fuzzy_sort_key`.

2. **Modal**
   `ExistingDefinitionFinderModal(kind, entries, *, initial_query="", preview_loader=None)`
   returns `ExistingDefinitionPick(entry_id, query)`, `ExistingFinderBack(query)`, or
   `None`:
   - Layout as in the Design mock: title `✎ Edit existing macro` /
     `✎ Edit existing snippet`, stepper `✓ Existing › ● Find`, a `❯` query input with a
     right-aligned `shown / total` counter, a Matches list (two-line rows with chips and
     highlighted fuzzy runs), a Preview panel, the verdict line, and the hints line.
   - Keys: typing filters; `↑`/`↓`/`Ctrl+N`/`Ctrl+P` move the highlight (focus stays in
     the input); `Enter` opens; `Ctrl+D`/`Ctrl+U` scroll the preview; `⇧Tab` goes back;
     `Esc` cancels. `Enter` on an `incompatible` row is refused and leaves the verdict
     showing.
   - Empty states: `No macros match ‘<q>’ · ⇧tab to create one`, and
     `No macros yet · ⇧tab to create one`.
   - Preview: a header line (`<ref> · <display_path>`, plus `shadowed by …` /
     `shadows …` / origin), then the body. Macro bodies use `markdown_document_syntax`
     (`src/sase/ace/tui/util/frontmatter_syntax.py`) and snippet templates render as
     plain text. Entries with `preview_text` render right away. Others go through the
     debounced, off-thread, cached, staleness-checked `preview_loader` (show `Loading…`
     until ready, and a dim error line on failure).

Tests: `tests/ace/tui/modals/test_existing_definition_entries.py` (builders for every
status, ids, ranking order including path-only matches and empty-query order, runs, and
verdict copy) and `tests/ace/tui/modals/test_existing_definition_finder_modal.py`
(filtering, counter, navigation, Enter results, refusal on incompatible, back and cancel
results, preview caching and staleness with a stub loader). Add PNG goldens for a macro
finder with a query showing every chip, a snippet finder, and the empty-match state in a
new `tests/ace/tui/visual/test_ace_png_snapshots_existing_finder.py`.

## Phase `wire-macro-existing`: Wire the existing path into the mini-macro flow

Files: `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_mini_macro_pane.py`,
`src/sase/ace/tui/widgets/_prompt_input_bar_mini_macro_pane.py`,
`src/sase/ace/tui/widgets/_prompt_input_bar_snippet_pane.py` (public dirty-guard
wrapper), `src/sase/ace/tui/widgets/_prompt_input_bar_g_prefix_hint_metadata.py`,
`docs/ace.md`, and `docs/prompt.md`. Move flow logic into a new sibling module if
`_prompt_bar_mini_macro_pane.py` would pass 700 lines.

1. **Picker data.** `_load_and_show` passes
   `existing=ExistingRowSpec(count=<distinct macro-kind names in the catalog>, switching=<a mini pane is open>)`
   to `macro_location_choices`, and stores `macro_existing_entries(catalog)`, computed
   off-thread right next to the choices.
2. **`e` pick.** `_on_pick` sends `EXISTING_CHOICE_ID` to
   `_open_existing_finder(query)`. The query is the remembered finder query or else
   `pick.typeahead`. `preview_loader` wraps the existing
   `_load_definition_markdown(definition)`, keyed by entry id.
3. **Finder results:**
   - `None`: refocus the origin pane.
   - `ExistingFinderBack(query)`: remember the query, reopen the normal picker with
     `highlight_id=EXISTING_CHOICE_ID`, and redeliver choices.
   - Editable pick: build
     `MiniMacroNameResult(action="edit", destination= destination_target_for_name(row, name, destinations=catalog.destinations), definition=definition, existing_definition=<active>, save_warning=macro_redefinition_warning(...))`.
     If the open pane already targets the same `write_path` and name, focus it and
     notify `Already editing #name`. If a mini pane is open, call
     `origin_bar.confirm_discard_dirty_auxiliary(proceed)` and then
     `_apply_mini_macro_name_result(..., replace_draft=True)`. Otherwise apply normally.
   - Read-only pick: open the override picker
     (`SaveLocationPickerModal(kind="macro", title=f"Override #{name} · where should your copy live?")`).
     Its choices use `override_name=name` and a `shadowed_by` map built from
     `macro_redefinition` for each row. On pick, call the existing
     `_open_name_step(row, name)` with the flow in override mode: name results apply
     with `replace_draft=True` and the dirty guard, and `⇧Tab` from the name step
     returns to the _override_ picker. The name step's override verdict (from
     `macro-redefinition`) is where the warning shows.
   - Origin lost at any stage: use the existing
     `Prompt pane is no longer available - mini-macro discarded` handling.
4. **Widget.** `open_mini_macro_target_pane(..., replace_draft: bool = False)`: when a
   mini pane exists and `replace_draft` is set, replace text, frontmatter, and target
   (`clean_hash` from the loaded content, `changed_on_disk=False`), keep
   `_mini_macro_focus_restore` so that closing still returns to the original prompt
   pane, focus the pane in INSERT, and refresh the title and frontmatter panel. Thread
   `replace_draft` through `_apply_mini_macro_name_result`. Add a public
   `confirm_discard_dirty_auxiliary(proceed)` on the bar that delegates to
   `_confirm_discard_dirty_snippet`.
5. **Hint label.** Change `open mini-macro…` to `new / edit mini-macro…` (the retarget
   label is unchanged). Update any g-prefix hint goldens that change.
6. **Docs.** In `docs/ace.md`'s "Save location picker" section, document the Existing
   section, the `e` row in the key table (both pickers), the finder keys and chips,
   in-place versus override behavior, switching while a pane is open, and override mode.
   Also update the mini-macro paragraph and the `Ctrl+G x` / `gx` keymap table rows
   ("Open, retarget, or edit an existing mini-macro pane"). In `docs/prompt.md`, add one
   sentence to the `gx`/`gt` paragraph. Run `just fmt`.

Tests: extend `tests/ace/tui/actions/test_prompt_mini_macro_location_flow.py` with `e` →
finder → editable opens a pane with the loaded body; shadowed edit carries the warning;
read-only → override picker (namespaced rows disabled) → name step → pane with the
built-in body; back and cancel; type-ahead query; origin loss; and replacing a dirty or
clean open pane (including the same-target no-op). Extend
`tests/ace/tui/widgets/test_prompt_stack_mini_macro_pane_lifecycle.py` for
`replace_draft`. Add a live-flow PNG golden for the picker → finder sequence.

## Phase `wire-snippet-existing`: Wire the existing path into the snippet flow

Files: `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_snippet_pane.py` (split into
a sibling module if it would pass 700 lines),
`src/sase/ace/tui/widgets/_prompt_input_bar_snippet_pane.py`,
`src/sase/ace/tui/widgets/_prompt_input_bar_g_prefix_hint_metadata.py`, and
`docs/ace.md`.

1. **Picker data.** `_build_snippet_picker_tables` receives
   `existing=ExistingRowSpec(count=len(catalog.entries), switching=<a snippet pane is open>)`.
   Entries come from `snippet_existing_entries(snippet_definition_sites(catalog))`,
   built from the catalog that `snippet-redefinition` already loads in
   `load_and_deliver`. Snippet previews are in memory, so no loader is needed.
2. **`e` pick, results, and override.** Mirror `wire-macro-existing`:
   - An editable pick maps the site path to its `SnippetConfigLocation` (comparing both
     raw and `resolve_macro_write_target` read/write paths) and builds
     `SnippetNameResult(trigger, target=snippet_save_target_for_location(location), exists=True, existing_body=site.template, derived_from=None, save_warning=snippet_redefinition_warning(...))`.
   - A read-only pick (built-in, plugin, macro-derived, or an unwritable file) opens the
     override picker (`Override ⇥ <t> · where should it live?`). Its choices use
     `override_name` and a `shadowed_by` map from `snippet_redefinition`. It then opens
     the prefilled `SnippetNameModal`, whose `existing_body` comes from the catalog
     template.
   - Back, cancel, origin-loss, same-target, and dirty-guard behavior match the macro
     flow, using the existing snippet copy (`… - snippet discarded`).
3. **Widget.** `open_snippet_target_pane(..., replace_draft: bool = False)` replaces the
   draft body and target in place, keeps `_snippet_focus_restore`, and is threaded
   through `_apply_snippet_name_result`.
4. **Hint label.** Change `new snippet…` to `new / edit snippet…`.
5. **Docs.** In `docs/ace.md`, add the existing path to "Authoring a snippet from the
   prompt bar" (step 1) and update the `Ctrl+G t` / `gt` keymap table rows. Run
   `just fmt`.

Tests: extend `tests/ace/tui/actions/test_prompt_snippet_location_flow.py` and
`tests/ace/tui/widgets/test_prompt_stack_snippet_pane_lifecycle.py` with the same
scenario matrix as the macro phase. Include overriding a plugin- or built-in-defined
trigger and a macro-derived trigger. Add a snippet picker → finder live-flow PNG golden.

## Verification (every phase)

- Run `sase tool run check` (the wrapped `just check`). Hand it to `/sase_monitor` if it
  may exceed the synchronous limit. Do not run `just check-full` unless explicitly told
  to.
- A phase that changes rendered output runs targeted
  `just fix-tui-screenshots -- <selectors>` (through `/sase_monitor` when long) and
  inspects every created or updated golden before finishing.
- After the final phase, do a live check with `sase screenshot` (`--keep`, drive
  `Ctrl+G x`, `e`, type a query) to confirm the picker → finder → pane flow looks as
  designed.
