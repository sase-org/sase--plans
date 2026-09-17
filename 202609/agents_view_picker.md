---
tier: tale
title: Agents view and layout picker
goal:
  Replace the Agents tab panel-cycle and layout shortcuts with one polished p picker
  that offers direct view and layout choices while preserving the old layout swap as pp.
size: medium
proposed_by: bbugyi200.apollo.0e
create_time: 2026-09-17 15:30:17
status: wip
---

# Plan: Agents view and layout picker

## Outcome and scope

On the Agents tab, `p` opens an **Agent view** picker. One further key selects File,
Tools, None, or either existing panel layout. `pp` performs the old `p` layout swap
exactly once. Bare `[` and `]` no longer change the Agents detail view.

Implement this as one coherent, medium-sized tale: presentation state, a small Textual
modal, keymap integration, documentation, and behavioral/visual tests. It does not need
separately landed phases. No implementation changes are authorized until this plan is
approved through the plan proposal flow.

Keep the existing vertical layout and its two proportions. No horizontal/equal split,
new saved preference, backend API, or generic picker framework is needed. Panel
selection and dimensions are Textual presentation state, so this work belongs in this
repository; reuse existing tool capabilities and file/tool loaders without duplicating
domain behavior from Rust core.

## Grounding in the current implementation

- Commit `73e4318edf` introduced the `o` grouping picker. Follow
  `modals/agent_grouping_modal.py`, `actions/agents/_grouping.py`, their styles, and
  mounted tests for direct-key dispatch, current-versus-focused styling, cancellation,
  mouse selection, and protection against repeated dismissal. All these paths are under
  `src/sase/ace/tui/`.
- `widgets/_agent_detail_panels.py` owns `DetailPanelMode`: `AUTO` displays File,
  `TOOLS` displays Tools, and `INFO` hides both secondary panels. Keep these internal
  enum values; display the third choice as **None** instead of the current `collapsed`
  label.
- `widgets/agent_detail.py` stores `_layout_swapped`. False gives metadata/file or
  metadata/tools `3fr/7fr`; true gives `7fr/3fr`. The old
  `actions/agents/_panel_detail.py::action_toggle_layout` rejects INFO mode and a
  missing secondary panel. Preserve that applicability for the swap.
- File-mode application deliberately invalidates cached file identity/list state before
  refreshing. `tests/ace/tui/test_panel_mode_cycle_refresh.py` protects the stale-file
  regression after navigating while File was hidden.
- Detail state is session-local and normally survives row navigation. Summary rows and
  pinned historical attempts can force metadata-only rendering.
  `tools/sources.py::supports_slow_tool_sources` detects tool-capable entries without
  probing files, including supported workflow/family roots.
- The top info bar already shows `[view: ...]`; its grouping neighbor advertises the
  configured `o` key. Artifacts separately owns `p` for project selection and `[` / `]`
  for pane navigation. Help and other modals have local uses too.

## Interaction design

### One key after opening

| Picker key | Label                      | Effect                                                         |
| ---------- | -------------------------- | -------------------------------------------------------------- |
| `f`        | File                       | Select `AUTO`; preserve the layout preference.                 |
| `t`        | Tools                      | Select `TOOLS`; preserve the layout preference.                |
| `n`        | None                       | Select `INFO`; metadata fills the detail area.                 |
| `1`        | Metadata larger            | Set the existing 70% metadata / 30% File or Tools layout.      |
| `2`        | File larger / Tools larger | Set the existing 30% metadata / 70% File or Tools layout.      |
| `p`        | Swap sizes                 | Toggle the existing layout once, as the old app-level `p` did. |

The layout percentages describe the existing flex proportions; borders and padding still
consume terminal cells. Layout choices affect only proportions, never the viewing mode.
Mode choices never alter the remembered proportions. Do not simulate a direct choice by
repeatedly cycling through other modes.

Every valid choice dismisses the picker immediately and applies once. Selecting the
already-current mode or layout also closes, but performs no reload, scroll reset, or
refresh work. Opening, moving the cursor, and cancelling make no changes; there is no
live preview and no separate confirmation step.

Use fixed modal-local keys, as in the grouping picker. The opener is configurable as
`ace.keymaps.app.choose_agent_view`, default `p`; rebinding it to `g`, for example,
means `gp` swaps sizes. The inner `p` stays `p`, per the requested design. Help must
derive the opener from the registry rather than hard-code compound shortcuts.

Support Up/Down and `j`/`k`, Enter, and clicking a whole enabled row. Start the keyboard
cursor on the current view. Skip disabled choices during keyboard navigation. Esc and
`q` cancel and restore the previous focus. Unknown printable keys, including `[` and
`]`, stay inside the modal. Direct letters may accept uppercase as the grouping picker
does. Rapid `pp` must produce one modal and one swap, with a dismissal guard preventing
duplicate callbacks.

### Availability and honest feedback

- Open only from the Agents tab when a selected detail row exists. Do not open over
  another modal or steal text from prompt, query, or search inputs or other active
  prefix modes. Match command-palette applicability to the action. Empty Agents, group
  banners, and whole-tribe focus do not expose this action; defensive direct invocation
  should provide a short explanation instead of failing.
- Keep File available for an ordinary row even when it currently has no file. The
  subtitle can say `No file currently; metadata fills the space`. Selecting File retains
  the existing automatic expansion and later content-arrival behavior.
- Tools is available when the existing capability predicate supports the row, even when
  no calls have arrived yet. Use the existing loading/empty Tools view. Otherwise keep
  its row visible, dimmed, with `Unavailable for this entry`. Do not perform discovery,
  network calls, or disk reads to populate the picker.
- For synthetic clan/proc summaries and pinned historical attempts, keep the existing
  metadata-only restrictions. The picker may open for these selected rows, but shows
  None as the effective view and disables File, Tools, and size actions with
  `Summary view` or `Historical attempt` explanations. Choosing None there is a no-op
  and must not overwrite the remembered ordinary-row preference.
- Disable `1`, `2`, and `p` when there is no visible secondary panel, including None
  mode and File with no content. State why: `Choose File or Tools first` or
  `No file to resize`, as appropriate. Retain the saved layout preference; label it
  `Saved` rather than `Current` when it is not applied. A disabled direct key or click
  leaves the picker open and displays its reason inline without a toast storm. This
  preserves the old swap's no-change behavior in these contexts.
- Capture the selected row identity and attempt at opening. On submission, re-resolve
  the current selection and capabilities; reject a stale result if selection, attempt,
  tab, or summary context changed while the picker was open. Close with a brief
  `Selection changed; reopen Agent view` notification in that case. Never mutate the
  last-rendered agent merely because it is still cached. If identity is unchanged but a
  formerly available action has become unavailable, close without applying it and
  explain the new restriction. A background content update must never turn a disabled or
  stale size choice into a hidden state change.

### Visual design

Use the grouping picker's palette and double-border treatment, with a centered card
approximately 68 columns wide, max-width 94%, max-height 90%. Keep the title and footer
visible and make the choice body scroll at short terminal heights. Use two labeled
sections, **View** and **Layout**, separated by whitespace and a muted divider. Row
height should follow wrapping rather than clip at narrow widths.

Example, with File and its default layout active:

```text
╔════════════════════════════════════════════════════════════╗
║                        Agent view                          ║
║  Press a key to apply.                                     ║
║                                                            ║
║  VIEW                                                      ║
║  > [f] File                                   Current      ║
║        Files and diffs                                     ║
║    [t] Tools                                               ║
║        Tool calls and activity                             ║
║    [n] None                                                ║
║        Metadata fills the detail area                      ║
║                                                            ║
║  LAYOUT                                                    ║
║    [1] Metadata larger     Metadata 70% / File 30%          ║
║    [2] File larger         Metadata 30% / File 70%  Current ║
║    [p] Swap sizes          File larger → Metadata larger   ║
║                                                            ║
║  ↑/↓ or j/k move · Enter select · Esc cancel                 ║
╚════════════════════════════════════════════════════════════╝
```

Use bright keycaps and a left focus rail/background; use the existing success color and
an explicit `Current` badge for the selected setting in each section. Current setting
and keyboard focus remain independently visible. Disabled rows retain readable labels
and reasons. Dynamically say File or Tools in layout rows; when neither is active, use
`File / Tools`. Text and shape, not color alone, communicate focus, selection, and
disabled state. Keep this a quiet, fast chooser: no decorative animation or content
preview that loads files behind the modal.

Add the configured opener to the existing top-bar view chip, e.g. `[view: file (p)]`,
matching the grouping chip. Hide the key when unbound or when the current context cannot
open the picker. Display `none` consistently in the normal-row chip and picker; retain
distinct existing tribe/summary descriptions.

## Implementation sequence

1. **Make detail changes explicit and idempotent.** Add a public typed mode setter and a
   layout setter/read-only state projection to the existing detail widget or a narrowly
   scoped presentation helper. Reuse `_apply_panel_mode` and the existing layout
   classes. Route swaps and explicit size selections through one layout application
   path. Ensure both active metadata surfaces (normal and committed-search overlay) and
   File/Tools receive the correct classes across `1 → t → 2 → f` and hide/show
   transitions; stale classes on hidden surfaces must not resurrect an old ratio.
   Preserve search text, file selection, attempt pinning, tool expansion, and scroll
   where the existing operation can. Keep generation/identity protection, and preserve
   the selected attempt when dispatching any necessary detail update. Treat a same-mode
   selection as a no-op only when the detail belongs to the selected row/attempt; do not
   let that shortcut retain another row's stale file content after rapid navigation.
   Opening the picker must not synchronously flush pending content loaders. Change only
   obsolete cycle helpers and call sites made unnecessary by this feature; avoid
   unrelated refactoring.

2. **Build and connect the picker.** Add `modals/agent_view_modal.py`, its small
   immutable choice/result types, scoped styles, and the normal lazy exports in
   `modals/__init__.py` and `_export_table.py`. Take a lightweight state snapshot into
   the modal; return a typed mode/layout/swap result or cancellation. Implement the
   action and callback in `actions/agents/_panel_detail.py` or a focused new mixin if
   needed to respect module size limits. Centralize the presentation capability
   calculation so displayed reasons and action guards agree. Apply only after dismissal
   and current-context validation. Use `_update_agents_info_panel()` and
   `_refresh_agent_footer_bindings_only()` when needed, instead of
   `_refresh_agents_display()` and a list rebuild.

3. **Replace the public shortcuts and discovery surfaces.** Update `default_config.yml`,
   `keymaps/app_keymaps.py`, `keymaps/metadata.py`, `bindings.py`,
   `_app_action_availability.py`, the command metadata/availability modules, Agents
   help, and `widgets/agent_info_panel.py`. Expose the new `choose_agent_view` action as
   `Choose agent view and layout` with search terms including panels, file, tools, none,
   metadata, and layout. Remove the three old app actions (`toggle_thinking`,
   `toggle_thinking_reverse`, `toggle_layout`) from configurable binding/catalog/help
   surfaces. Put their names in the existing `_RETIRED_APP_KEYS` handling so stale
   overrides load quietly but cannot restore the removed bracket cycles or hijack `p`.
   The old widget-level swap behavior remains callable by the modal; do not leave a
   stale app action that bypasses it. Document retirement and the new opener setting.
   This is the requested complete replacement, with no retained legacy execution branch
   or temporary feature flag. Allow the new opener and `pick_artifacts_project` to share
   `p` by tab scope without weakening same-tab duplicate validation. Update
   `docs/ace.md` key tables and metadata/tools prose plus `docs/configuration.md`;
   explain every chord, None, disabled layouts, and session-local preferences. Preserve
   all other-tab and modal-local uses of these letters and brackets.

4. **Exercise the workflow and polish the actual rendering.** Add focused mounted-app
   and visual coverage, update affected old fixtures, inspect the resulting images, and
   run the required checks below. Keep implementation and documentation consistent
   before considering the tale done.

## Validation and acceptance

Use the existing AcePage and PNG snapshot infrastructure with deterministic local
fixtures. Test observable state and rendering, not just that a setter was called.

- **Routing:** `p` opens once without changing mode/layout; `pf`, `pt`, `pn`, `p1`,
  `p2`, and `pp` select the stated outcome and close. Test the swap in both directions
  for both File and Tools. Test rapid consecutive presses. Bare `[` / `]` do nothing on
  Agents. Artifacts keeps its project picker and bracket pane switching, the `o` picker
  still handles its local `p`, and existing modal/input owners retain their keys. Custom
  opener/unbound config, retired overrides, contextual duplicate validation, help,
  top-bar hints, and command palette agree.
- **Modal behavior:** current/focused markers differ, direct keys and mouse work,
  navigation/Enter work, Esc/`q` return focus unchanged, unused keys do not leak, and
  disabled choices leave all state unchanged with a visible explanation. Selecting the
  current choice must not reset scrolling or start loader work.
- **Transitions and races:** exercise File → Tools → None → another agent → File,
  preserving `test_panel_mode_cycle_refresh.py`'s fresh-file guarantee. Cover both
  ratios across modes, no-file expansion and later file arrival, empty but tool-capable
  entries, unsupported Tools, summary rows, pinned attempts, and a selection
  removed/replaced during modal lifetime. Test normal and committed metadata-search
  surfaces. Maintain existing navigation/persistence and zoom semantics; do not
  accidentally persist these choices to disk.
- **Responsiveness:** opening, navigating, resizing, and cancelling perform no
  filesystem/network discovery or subprocess work. Verify mode changes refresh only the
  required detail/info/footer surfaces, using existing async loaders; they must not
  rebuild the agent list or rescan agent storage. Include a targeted guard/spied
  assertion for this boundary in the mounted regression.
- **Visual review:** add deterministic screenshots for the default File picker, Tools
  with metadata emphasized, and None with disabled layout choices. Include a compact
  viewport such as 60×18 as well as a normal 100×32 view. Inspect the PNGs to check
  wrapping, current/focus contrast, disabled reasons, scrolling, and footer visibility.
  Update existing Tools/slow-tools visual interactions from bracket presses to `pt`/`pn`
  where appropriate; keep local bracket uses in other screens intact. Refresh only
  intentionally affected golden images.
- Read `lint_and_test.md` through `sase memory read` when implementing. Run focused
  picker, detail, keymap, command, and affected visual tests, then `just check`. Use the
  repository's `/sase_monitor` procedure if verification becomes long; run `just fix`
  (or at least `just fmt`) before handing checks to a monitor. Run `just check-full`
  only when required by the documented broadening/landing rules, through the monitor.
  Report actual checks and any unresolved failures.

The feature is complete when a user can discover the picker from the view chip, choose a
mode or an existing layout in one further key, use `pp` with the old swap semantics, and
navigate or cancel without stale content, accidental commands, or unexpected changes to
selection and layout preferences.
