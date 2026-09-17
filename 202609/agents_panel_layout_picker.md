---
tier: tale
title: Move the Agents panel-layout toggle from leader mode into the grouping picker
goal:
  Users can toggle split and merged Agents panels with oo while retaining independent
  grouping modes and retiring the old leader shortcut.
size: medium
proposed_by: bbugyi200.apollo.0a.f0
create_time: 2026-09-17 14:42:53
status: wip
---

# Move the Agents panel-layout toggle from `,g` to `oo`

## Outcome and scope

On the Agents tab, the first `o` opens the existing grouping picker. A second lowercase
`o` switches between separate tribe panels and one merged panel, closes the picker, and
returns to the Agents view. Repeating the complete `oo` interaction switches back. This
is the existing `,g` functionality moved into the picker.

Implement this as one tale of medium size: modal behavior, action routing, shortcut
retirement, command discovery, documentation, and their regression tests are one bounded
UI change. The work does not need independently landed phases.

The Project/Date/Status/Machine selection and the split/merged panel layout are
independent axes. Toggling panels must preserve the selected grouping mode and its
persistence behavior. Retain the layout toggle's existing session-only lifetime,
fold/focus cleanup, and toast. This is presentation and routing work in the Python TUI;
reuse its existing action and core-backed model calls. No new shared domain logic, Rust
API, persistent preference, modal-keymap subsystem, or global `oo` binding is needed.

## Current implementation

- `src/sase/ace/tui/modals/agent_grouping_modal.py` exposes four mode choices and
  returns `GroupingMode | None`. It consumes unknown printable keys, including `o` and
  `O`, and guards completion with `_dismiss_once`.
- `src/sase/ace/tui/actions/agents/_grouping.py` opens that modal from
  `action_choose_agent_grouping`, rejects stacked modals, and applies a selected mode
  after checking that the current tab is still Agents.
- `src/sase/ace/tui/actions/agents/_panel_navigation.py` already implements
  `action_toggle_agent_panel_grouping`. It flips `_agent_panels_grouped`, tears down
  fold hints, disarms isolation restore, clears whole-panel fold intents, clears
  expanded-panel/group/attempt focus, invalidates the panel cache, and refreshes the
  cached Agents display once. Preserve these semantics.
- `_agent_panels_grouped` starts false in
  `src/sase/ace/tui/actions/_state_init_agents.py`. It is distinct from
  `_grouping_mode`; false means split by tribe and true means merged.
- The old leader shortcut is distributed across `default_config.yml`,
  `keymaps/mode_keymaps.py`, the leader handler, command metadata, help, and the leader
  footer. `keymaps/registry.py` already filters relocated leader overrides and warns
  only when an old override is present. Use that mechanism so config deep merges cannot
  resurrect the removed command.
- Existing picker tests explicitly assert that a second `o` does nothing. Those
  expectations must change; mode-selection tests must continue to exercise the four
  existing modes.

All abbreviated TUI paths below are relative to `src/sase/ace/tui/`.

## Interaction contract

1. Keep the four mode rows in their existing order, with the current mode selected
   initially and its current-state badge intact. Add a separate **Panel layout** section
   containing one selectable `[o]` row after them. The row describes the next action and
   current state: **Merge panels** when split, **Split panels by tribe** when merged. Do
   not represent layout as a fifth `GroupingMode`.
2. Lowercase `o`, clicking the layout row, or moving to it with `j/k` or arrows and
   pressing Enter all perform the same one-shot toggle and dismiss the modal. Navigation
   remains bounded and scrolls the focused row into view. Include the row within the
   modal's bounded scrollable content so it works in small terminals.
3. Dismiss the modal before applying the action, as mode selection already does. Retain
   the exactly-once completion guard across direct keys, Enter, and clicks. Cancel
   (`Esc` or `q`) never changes either state. Unknown printable keys remain contained.
   Uppercase `O` remains inert; check lowercase `o` before the existing case-normalized
   mode-letter handling.
4. `p/d/s/m` still select only a grouping mode and dismiss. Opening/cancelling the
   picker and reselecting the current mode retain their existing no-op behavior. Opening
   the picker alone must never toggle panels.
5. The opener remains configurable through `ace.keymaps.app.choose_agent_grouping`. The
   internal toggle key stays literal `o`, like the existing fixed `p/d/s/m` choices.
   With a rebound opener, the sequence is `<configured opener>`, then `o`; it is `oo`
   only with defaults. An unbound opener leaves palette access available and must not
   advertise an invalid chord.
6. Retire the old leader shortcut completely, including stale remapped overrides. It
   must disappear from leader help, footer, repeat dispatch, and command entries. Other
   leader commands retain their behavior. This is a complete requested migration without
   a retained compatibility branch, so no feature flag is needed.
7. Preserve Agents-only scope, prompt/editor key ownership, and modal-stack guards.
   Artifacts retain their existing `o/O` cycling, including two forward cycles for `oo`;
   other tabs and unrelated modals gain no layout-toggle binding. The action remains
   available with an empty list or only one visible panel, just as today.

## Implementation steps

### 1. Extend the picker result and render the layout choice

In `modals/agent_grouping_modal.py`, introduce a small typed action result, such as
`AgentGroupingAction.TOGGLE_PANELS`, and use `GroupingMode | AgentGroupingAction | None`
for modal completion. Keep `AGENT_GROUPING_CHOICES` as the four mode descriptors. Accept
current panel layout as a keyword constructor argument and use it to render the layout
row's current state and action label. Read that state anew whenever the picker opens.

Extend focus, click, Enter, refresh, and scroll handling to cover the layout row without
assuming that every row contains a mode. Preserve existing mode row IDs and give the
layout row a stable semantic ID for tests. All activation paths use the same dismissal
guard. Update modal guidance/footer and, if necessary, scoped `AgentGroupingModal` CSS
in `styles.tcss` for the separate section and narrow views.

In `actions/agents/_grouping.py`, pass the current `_agent_panels_grouped` state into
the modal. After cancellation and current-tab guards, route the typed toggle result to
`action_toggle_agent_panel_grouping()` and real mode results to
`_set_agents_grouping_mode()`. Never feed the toggle into mode persistence or duplicate
the toggle implementation. Do not call `_refresh_current_tab()` after the toggle: the
existing action owns its cache invalidation and display refresh. No disk reads,
background reload, subprocess, or new persistence worker belongs in this keypress path.

### 2. Retire leader routing and preserve palette discovery

Remove `toggle_agent_panel_grouping` from both `src/sase/default_config.yml`'s leader
defaults and `keymaps/mode_keymaps.py`'s `LeaderModeKeymaps` defaults. Remove its branch
in `actions/agent_workflow/_leader_mode.py`, label/tab entries in
`commands/_mode_commands.py`, and leader help/footer rows in
`modals/help_modal/agents_bindings.py` and `widgets/_keybinding_modes.py`.

Add the retired setting to `keymaps/registry.py`'s relocated-leader filtering, with a
warning explaining that the action now lives under the Agents grouping picker
(`choose_agent_grouping`, then `o`). Ignore the old override rather than mapping it onto
the opener or reintroducing a hidden leader alias. Do not edit users' configuration
files. Remove every direct dictionary lookup of the retired key.

Preserve a searchable direct palette action by adding a catalog entry in
`commands/catalog.py` with ID `agents.toggle_panel_grouping`, label **Toggle agent panel
layout**, category **Grouping**, and Agents-only scope. Its executor is the existing
`app_action` named `toggle_agent_panel_grouping`. Advertise
`(registry.app.choose_agent_grouping, "o")` using the existing sequence formatter. If
the opener is unbound, expose the command with an empty sequence and display; the
palette still executes it directly. Use aliases including panel, tribe, split, merge,
and the action name. Remove the old `leader.toggle_agent_panel_grouping` catalog entry.
This requires no new global binding, AppKeymaps field, executor kind, or synthetic key
replay.

Update the chooser's search aliases in `commands/_app_metadata_display.py` so
panel-layout searches can also find the picker. Verify catalog scope and direct action
guarding with no selected agent and on non-Agents tabs.

### 3. Update documentation and help together

- In `modals/help_modal/agents_bindings.py`, add the configured opener-plus-`o` sequence
  to the Grouping section, labelled **Toggle tribe panels split/merged**. Respect the
  help box's 32-character description limit and unbound-key handling.
- In `docs/ace.md`, update Agents navigation, Grouping Modes, and the old `,g` leader
  row. Explain that panel layout and grouping mode are independent and that `oo` toggles
  and closes the picker. Remove the obsolete leader row.
- In `docs/configuration.md` and the chooser comment in `default_config.yml`, document
  the picker-local `o`, configurable opener, and migration from the old leader setting.
  Keep Artifacts cycle documentation accurate.
- Keep the persistent footer convention: this unconditional layout action belongs in the
  picker/help/palette, not a new always-visible footer binding.

## Regression coverage and acceptance criteria

Extend existing suites rather than introducing a new harness:

- `tests/ace/tui/modals/test_agent_grouping_modal.py`: both layout labels/states;
  lowercase `o` returns the typed action exactly once; keyboard navigation plus Enter
  and clicking the layout row return the same result; current grouping badge remains
  correct; all four mode keys still work; cancel, `O`, and unknown-key containment still
  work. Remove `o` from the unknown-key test. Check that the additional row remains
  reachable in a constrained viewport.
- `tests/ace/tui/test_agent_grouping_picker.py`: a real default `oo` sequence closes the
  picker and toggles once, and a second complete sequence restores the layout. Use a
  deterministic multiple-tribe fixture to assert rendered separate/merged panels as well
  as state. Confirm `_grouping_mode` and mode-save scheduling are unchanged by the
  toggle. Replace the old second-`o`-is-inert assertion with a direct attempt to reopen
  while a modal is present, preserving stack protection. Cover a rebound opener, prompt
  ownership, and a result arriving after a tab change. Existing `O` and Artifacts
  routing regressions must remain green.
- Retain and run `tests/ace/tui/test_agent_panel_collapse_state.py` and
  `test_agent_panel_isolation_revert.py`. At the new picker route, assert delegation
  happens once; these existing action tests verify fold intent clearing and isolation
  cleanup without copying that implementation into modal tests.
- Update `tests/test_keymaps_defaults.py`,
  `tests/test_keymaps_registry_loading_legacy.py`,
  `tests/ace/tui/test_leader_keymap_dispatch.py`, and
  `tests/ace/tui/test_leader_keybinding_footer.py`: no default or remapped stale leader
  entry survives; the migration warning is useful; removed `g` dispatch and a remembered
  stale `g` cannot toggle; unrelated leader commands still dispatch without missing-key
  errors.
- Update `tests/test_command_catalog_build.py` and applicable catalog guards,
  `tests/test_command_availability_scope.py`, `tests/test_command_execution.py`, and
  `tests/test_keymaps_display_help_agents.py`: one replacement direct command, correct
  default/rebound/unbound sequence display, Agents-only availability, exactly-once
  action execution, accurate help, and no obsolete leader entry. Preserve all existing
  keymap coverage invariants.
- Search for `toggle_agent_panel_grouping`, `,g`, and double-`o` picker assumptions
  across source, tests, and docs. Distinguish text input containing `oo` from actual
  picker interactions; do not rewrite prompt/search typing or Artifacts sequences.

Use Textual/AcePage observable-state waits, not fixed sleeps. Keep focused fixtures
isolated from real grouping-preference files. Test the semantic boundaries above; avoid
redundant assertions that merely restate rendering implementation.

## Verification and completion

After implementation, read the current `lint_and_test.md` reference memory, run the
focused suites named above plus the existing grouping-mode and key-resolution
regressions, then run `just fix` followed by `just check`. Use `/sase_monitor` if the
repository check takes a long time, following the required verification handoff rather
than repeatedly polling a long-running command. Apply the documented `just check-full`
escalation rules if test selection requires them.

Inspect the expanded picker in split and merged states at normal and constrained
terminal sizes. If existing PNG snapshots cover a changed help/leader surface, run the
affected visual subset and inspect diffs before updating intentional goldens. The prior
implementation reported unrelated `sase ace`/`sase tui` title drift; do not assume that
accounts for a new failure or bulk-accept unrelated snapshots.

Completion requires functional `oo` in both directions, preserved grouping and cleanup
semantics, fully retired leader routing (including stale config), correct help/palette
behavior with custom and unbound openers, and passing applicable checks with any
independent verification failures reported accurately. Implementation starts only after
this plan is approved.
