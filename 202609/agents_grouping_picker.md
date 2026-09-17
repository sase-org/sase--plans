---
tier: tale
size: medium
title: Choose Agents grouping from a single-key picker
goal:
  Replace Agents-tab grouping cycling with an elegant direct-choice panel and remove its
  reverse-grouping shortcut.
proposed_by: bbugyi200.apollo.09
create_time: 2026-09-17 12:35:58
status: wip
---

# Plan: Agents grouping picker

## Outcome and scope

On the main **Agents** tab, `o` opens **Group agents by**. One further keypress chooses
Project (`p`), Date (`d`), Status (`s`), or Machine (`m`), applies that strategy, and
closes the panel. Bare `O` has no Agents-tab action. Help and the command palette
describe choosing a strategy and expose no Agents cycling command.

Implement this as one medium tale: the work is a bounded Textual interaction, keymap
routing, presentation, and regression coverage that one coding agent can complete
together. No independently landed stages or feature flag are needed for this explicitly
requested replacement.

Keep the existing four grouping strategies, enum values, default, saved-mode file,
per-mode folds, query/filter behavior, and panel organization. Grouping applies to the
whole Agents tab, across its existing panels. The separate Artifacts tab, including its
Agents pane, retains its current capability-gated behavior; its forward/reverse grouping
actions remain available where they are today. Prompt editor Vim `o`/`O` and prefixed
commands on other tabs retain their own meanings.

This is TUI presentation and interaction work. Reuse existing grouping and storage
functions; do not change grouping domain rules, add a Python backend, or migrate shared
backend logic. No Rust API or linked-repo change is required by this design.

## Grounding in the current code

- `src/sase/ace/tui/actions/agents/_grouping.py` currently dispatches both cycling
  actions to Agents, Patches, and other grouping-capable Artifacts panes. Its Agents
  branch changes the mode, swaps the per-mode fold registry, clears stale group-banner
  focus, schedules a coalesced asynchronous save, and refilters cached agents. Preserve
  those effects when applying an explicitly selected mode.
- `src/sase/ace/tui/models/agent_groups/_buckets.py` defines `GroupingMode`: `STANDARD`,
  `BY_DATE`, `BY_STATUS`, and `BY_MACHINE`. `STANDARD` is presented to users as Project.
  Grouping algorithms remain unchanged.
- `src/sase/ace/tui/actions/agents/_loading_filter.py::_refilter_agents` reuses cached
  data and restores selection by identity. It already handles pending first load.
- `src/sase/ace/grouping_strategy.py` loads and saves `grouping_mode.txt`. Existing save
  scheduling in the grouping mixin keeps disk I/O outside the event loop and coalesces
  requests so the most recent mode wins.
- Keymaps have synchronized surfaces: `src/sase/default_config.yml`,
  `keymaps/app_keymaps.py`, `keymaps/metadata.py`, fallback `tui/bindings.py`,
  `keymaps/registry.py`, `_app_action_availability.py`, and command metadata. Paths
  without a full prefix here are under `src/sase/ace/tui/`.
- `keymaps/registry.py::_CONTEXTUAL_APP_DUPLICATES` already supports shared physical
  keys for actions whose tab scopes are disjoint.
- `modals/property_picker_modal.py` demonstrates direct accelerators, keyboard
  navigation, row clicks, and containment of unused printable keys.
  `modals/prompt_submit_choice_modal.py` and `styles.tcss` show the existing compact
  chooser appearance. Reuse their interaction/styling conventions without forcing
  grouping into a duration or schema-property API.
- Agents help is in `modals/help_modal/agents_bindings.py`. Its boxes must retain the
  established 57-character width and 32-character description limit. The persistent
  grouping hint is in `widgets/agent_info_panel.py`.

## Interaction and visual design

Use a focused `AgentGroupingModal(ModalScreen[GroupingMode | None])` with four stable
choices. The modal returns a value; its caller owns applying that value. A small
immutable presentation table should supply mode, mnemonic, label, and subtitle to both
rendering and selection dispatch.

| Key | Label   | Existing mode | Subtitle                                      |
| --- | ------- | ------------- | --------------------------------------------- |
| `p` | Project | `STANDARD`    | Projects and their Patches                    |
| `d` | Date    | `BY_DATE`     | Recent date buckets and time groups           |
| `s` | Status  | `BY_STATUS`   | Attention first, then activity and completion |
| `m` | Machine | `BY_MACHINE`  | This machine and remotes, grouped by status   |

All four choices remain visible for an empty list, a single project, or a machine
without remote agents. No fetching or counting is needed to render the picker.

Illustrative layout; use real widgets and theme styles, not a literal text box:

```text
╔══════════════════════════════════════════════════════════════╗
║                       Group agents by                        ║
║  Press a letter to switch grouping.                           ║
║                                                              ║
║  ▸ [p] Project                                    Current    ║
║        Projects and their Patches                            ║
║                                                              ║
║    [d] Date                                                  ║
║        Recent date buckets and time groups                   ║
║                                                              ║
║    [s] Status                                                ║
║        Attention first, then activity and completion         ║
║                                                              ║
║    [m] Machine                                               ║
║        This machine and remotes, grouped by status           ║
║                                                              ║
║  ↑/↓ or j/k move · Enter select · Esc cancel                   ║
╚══════════════════════════════════════════════════════════════╝
```

- Center the panel over the existing dimmed modal backdrop. Target roughly 66 terminal
  columns, auto height, maximum viewport width minus margins, and a bounded height. Use
  the existing double primary border, surface background, clear title hierarchy, aligned
  mnemonic badges, and muted subtitles. Keep the styling specific to this modal so other
  choosers do not change.
- The current mode has a persistent text badge **Current**. Initially highlight that
  row. Keyboard focus uses a separate pointer and subtle background; moving focus never
  moves the Current badge or changes the underlying view.
- Use theme variables for text, muted text, surface, accent, and selection colors.
  Current and focus must be distinguishable without color. No animation, preview pane,
  search field, or additional Apply button is needed for four choices.
- At 80x24, all choices and the footer should fit without clipping. At narrower or
  shorter sizes, wrap subtitles and let the choices scroll while retaining title,
  footer, key badges, and labels. Avoid fixed minimum widths that exceed the viewport.
  Scrolling a highlighted row into view must not animate.
- Lowercase `p/d/s/m` select immediately, without Enter. Up/down and `j/k` move focus
  with clamping at the ends; Enter selects the focused row. A click on any part of a
  choice row selects it. Escape or `q` cancels. Clicking the backdrop does not choose
  anything.
- Unknown keys, including `o` and `O`, do not change grouping or leak into underlying
  Agents actions. Repeated opener presses cannot stack the modal. Contain the opening
  event so an immediate `o`, then choice-key sequence works without a timing-dependent
  extra press; guard duplicate result delivery. Preserve standard app-wide
  emergency/terminal behavior; do not globally swallow unrelated control shortcuts.
- Choosing the already-current mode simply closes the modal: no refilter, write,
  notification, selection movement, or fold-state mutation. Cancel has the same
  no-mutation guarantee. Normal background refresh may continue while it is open.

## Implementation work

### 1. Give Agents a distinct action and retire its cycling routes

Add `choose_agent_grouping`, default `o`, to `ace.keymaps.app` in
`src/sase/default_config.yml`, `AppKeymaps`, binding metadata, and the fallback
bindings. Expose **Choose agent grouping** in the command catalog's Grouping category,
scoped to Agents, with useful aliases such as grouping, project, date, status, and
machine.

Keep `cycle_grouping_mode` and `cycle_grouping_mode_reverse` with their existing `o`/`O`
defaults for Artifacts. Remove Agents from both command scopes and their actual dispatch
branches. Guard both old actions on Agents so a direct call, stale override, or palette
execution cannot cycle grouping. Restrict the new action to the main Agents tab. Respect
focused inputs, prompt editing, and active modals; the command-palette selection must
still work after the palette dismisses.

Add only the justified disjoint pairs between the new action and the two existing
cycling actions to `_CONTEXTUAL_APP_DUPLICATES`. Preserve ordinary duplicate-key
validation. Verify actual Textual key dispatch in a mounted app, not just the registry's
list of bindings, because both tabs intentionally share `o`.

The new action is independently configurable and supports the established unbound
sentinel. Existing `cycle_grouping_mode*` overrides continue to configure Artifacts;
they no longer control Agents. Document `choose_agent_grouping` as the Agents setting.
Do not add an alias that revives Agents cycling or remap users' config files. With
default configuration, `O` has no navigation-mode action on Agents.

### 2. Apply an explicit mode through one state transition

Replace Agents-only cycle helpers/order with an explicit typed setter, for example
`_set_agents_grouping_mode(mode)`, preserving the existing per-mode registry helper and
save queue. Do not implement direct selection by cycling repeatedly.

The chooser callback rechecks that it is applying to Agents, handles `None` and the
current mode as no-ops, and applies against current app state rather than an agent/index
captured when the panel opened. A real change must:

1. Set `_grouping_mode` to the chosen existing enum value.
2. Restore/create that mode's fold registry and clear `_current_group_key`, whose old
   banner identity belongs to a different tree.
3. Refilter cached Agents once through `_refilter_agents`, preserving the focused agent
   by identity where visible and the existing fallback where it is hidden or gone.
   Respect saved destination folds; do not expand groups just to force the previous
   selection into view. Preserve marks, filters, and panel state.
4. Schedule the existing coalesced asynchronous mode save and retain one brief
   `Grouping: by ...` feedback notification for an actual change. The info-panel label
   remains the persistent confirmation.

Dismiss the modal before rebuilding the underlying view. Returning from cancel must
restore normal list keyboard focus. Keep first-load/empty-list behavior on the existing
refresh path and introduce no synchronous I/O or reload on opening, navigating, or
choosing. Preserve last-request-wins saves across rapid reopen and selection. A save
failure must not crash or roll back the active UI; handle both false return values and
exceptions, and show a concise warning if saving the latest Agents choice fails.
Suppress stale failure feedback when a newer choice is already pending. Keep this
feedback scoped to Agents so unrelated persistence behavior is unchanged.

### 3. Add the chooser and synchronize discoverability

Implement the modal and its scoped styling as described above. Keep presentation
metadata local to the TUI; retain `GroupingMode` as the canonical value type. Use
existing modal result/callback conventions and event containment. Do not build a new
general-purpose picker framework for four options.

Update Agents help to advertise the configured chooser key as **Choose grouping** and
list `p/d/s/m` with their meanings. Remove its `o / O` cycling instruction; preserve
useful explanations of date and machine grouping. Keep box widths and description
lengths within the help renderer's constraints.

Update `widgets/agent_info_panel.py` and its test helpers to display the new configured
key. An unbound chooser should omit the empty key hint. Keep this always-available tab
action in help and the info hint rather than adding it to the conditional bottom footer.
Update `docs/configuration.md` and stale comments that claim all grouping surfaces cycle
with `o/O`.

### 4. Verify behavior, routing, persistence, and appearance

Use the existing tests and AcePage/Pilot fixtures. Replace obsolete Agents cycle
expectations with direct-choice coverage; preserve the substantial existing fold,
selection, and persistence assertions instead of discarding them.

Required behavioral coverage:

- `o` opens exactly one picker without changing the mode; every shortcut selects its
  corresponding enum from every starting mode. Test current-mode and cancel as strict
  no-ops, including write/refilter counts.
- Navigation changes only highlight, Enter and row clicks select once, unrelated
  printable keys are contained, and keyboard focus returns after dismissal.
- Default `O` and both old cycling action IDs cannot mutate Agents state, including a
  custom binding for the retired reverse action. The palette contains the new chooser
  and neither old cycling command on Agents.
- Test actual shared-key routing after tab switches, a custom chooser opener, unbinding,
  legitimate tab-disjoint overrides, and rejection of a real conflict. Assert prompt
  editor `o/O` still work and other modals do not open a nested picker.
- Test selection preservation through reorder, old banner focus, round-trip per-mode
  folds across multiple panels, empty/filtered lists, first load, and a background
  refresh changing the selection/list while the picker is open.
- Retain off-thread/coalesced save tests and startup restoration for all four enum
  values. Cover latest-choice persistence under rapid selections, a failed save without
  a frozen UI, and subsequent successful saving.
- Retain Artifacts/Patches forward/reverse cycling and capability-gating tests,
  including independence of Agents and Artifacts grouping preferences.

Relevant existing suites include `tests/ace/tui/test_agent_grouping_cycle.py`,
`test_changespec_grouping_cycle.py`, `test_changespec_grouping_integration.py`,
`test_state_init_grouping_defaults.py`, the info-panel tests under
`tests/ace/tui/widgets/`, `tests/test_keymaps_patch_grouping_binding.py`,
`test_keymaps_registry_loading.py`, `test_command_catalog.py`, and
`test_command_availability_scope.py`. Add focused modal/routing tests as needed.

Several existing Agents PNG tests use `page.press("o", "o")` to reach Status. Change
these to `page.press("o", "s")` in the Agents, clans, group-clan-collapse,
group-lane-collapse, and panel-clan-collapse snapshot suites. Search for all such call
sites; do not rewrite `o` interactions belonging to other panels. Existing
post-selection grouping images should remain unchanged.

Add PNG coverage for the new picker at 120x40 and 80x24, including light and dark
themes, a non-default current mode, and keyboard highlight on a different row. Use a
smaller viewport layout test to catch overflow and unreachable choices. Inspect
generated images for visual hierarchy, aligned keys, readable muted text, no clipping,
and clear Current versus focus state. Only accept intentional new or changed goldens
after inspection.

Before finishing implementation, follow `lint_and_test.md`: run formatting/fixes,
`just check`, and `just test-visual` scoped to the added/affected snapshot cases.
Inspect `tools/select_tests --explain` if necessary and explicitly run any relevant
behavioral suites omitted by the scoped lane. Use `/sase_monitor` for long checks; use
`just check-full` only if the documented broadening/escalation criteria require it.
Planning itself changes no tracked implementation file and requires plan schema
validation rather than running the application test suite.

## Acceptance criteria

The feature is complete when `o` opens the polished picker, one letter reliably selects
any existing strategy, default `O` has no Agents action, current-mode and cancel are
side-effect-free, saved folds and selection behave correctly, and the preference
survives restart. Help, palette, config, and info hints agree on the new action. Other
tabs retain their current behavior, and the behavioral and visual checks pass with
reviewed images.
