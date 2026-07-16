---
tier: tale
goal: 'Projects and Artifacts sub-tab strips render the selected label in all capitals,
  while inactive labels retain their normal display spelling and the PR acronym is
  always shown as PRs when inactive.

  '
create_time: 2026-07-16 06:33:26
status: wip
prompt: 202607/prompts/uppercase_active_subtabs.md
---

# Plan: Uppercase active Projects and Artifacts sub-tabs

## Product outcome

Make selection state easier to scan in the two nested tab strips called out by the user:

- **SASE Admin Center → Projects:** the selected label is `PROJECTS`, `REPOS`, or `WORKSPACES`; inactive labels remain
  `Projects`, `Repos`, and `Workspaces`.
- **Artifacts:** the selected label is `PRS`, `COMMITS`, `BUGS`, or `PLANS`; inactive labels remain `PRs`, `Commits`,
  `Bugs`, and `Plans`. In particular, the current incorrect inactive spelling `Prs` disappears.

This is a display-only change. Sub-tab IDs (`projects`, `repos`, `workspaces`, `prs`, `commits`, `bugs`, `plans`),
keyboard/click navigation, content lifecycles, persistence, and project/PR data remain unchanged.

## Current implementation and scope boundary

Both surfaces build `PanelTab` definitions and render them through the shared
`src/sase/ace/tui/widgets/panel_tab_strip.py::PanelTabStrip`. The strip currently changes only the active label's style;
it always renders `PanelTab.label` verbatim. Projects and Artifacts both derive those labels with `str.title()`, which
produces the unwanted `Prs` spelling.

`PanelTabStrip` is also used by the Admin Center's top-level numbered tabs and by the Help modal. Making uppercase
selection a global default would silently alter those unrelated surfaces. The behavior therefore needs to be an explicit
strip-level rendering option enabled only by the two requested nested tab strips.

This stays within Textual presentation code and does not cross the Rust core backend boundary. The render path remains
pure and synchronous: it adds only a local string transformation during an already-required strip rebuild, with no I/O,
background work, extra refresh path, or event-loop blocking.

## Implementation design

1. **Add opt-in active-label capitalization to `PanelTabStrip`.**
   - Extend the constructor with a clearly named keyword-only boolean such as `uppercase_active`, defaulting to `False`
     for backward compatibility.
   - In `_build_content()`, derive the rendered label from the canonical `PanelTab.label`: uppercase it only when the
     option is enabled and that tab is active; otherwise render the canonical label unchanged.
   - Continue computing `_tab_ranges` and `_line_width` from the text actually rendered. This preserves centered click
     hit-testing even if a future label's uppercase form changes display length. `set_active_tab()` and `set_tabs()`
     should continue to rebuild through the same single path, so keyboard changes, clicks, and programmatic switches
     cannot diverge.
   - Do not mutate the immutable `PanelTab` entries or replace canonical labels during a switch; inactive tabs must
     reliably return to their original spelling.

2. **Enable the behavior for the Projects sub-tab strip.**
   - Pass the new option when `ProjectsPane` composes `#projects-subtabs`.
   - Keep the existing canonical labels `Projects`, `Repos`, and `Workspaces`, allowing the shared renderer to produce
     uppercase only for `_active_subtab` as `_switch_to_subtab()` updates the strip.

3. **Give Artifacts an acronym-aware canonical label and enable uppercase selection.**
   - Replace the `tab.title()` assumption used to construct `_ARTIFACT_TABS` with an explicit display-label mapping (or
     equivalently focused helper) whose `prs` entry is `PRs` and whose other entries remain `Commits`, `Bugs`, and
     `Plans`.
   - Pass the new uppercase-active option to `#artifacts-subtabs`. The canonical `PRs` label will then render as `PRs`
     while inactive and `PRS` while selected, while all other labels follow the same active/inactive rule.
   - Keep command-palette labels and all internal `prs` identifiers out of scope; they already use the correct `PRs`
     display spelling and do not participate in this strip-rendering defect.

## Tests and visual verification

Add focused behavioral coverage before re-baselining snapshots:

1. **Shared strip rendering contract** (`tests/ace/tui/test_config_center_tabs.py` or a focused panel-strip test
   module):
   - With the new option enabled, assert the initial active label is uppercase and inactive labels retain their
     canonical spelling.
   - Call `set_active_tab()` and assert the old label returns to canonical spelling while the new one becomes uppercase.
   - Exercise an acronym-style canonical label (`PRs`) so the contract explicitly distinguishes inactive `PRs` from
     active `PRS`.
   - Preserve/extend the existing numbered-strip assertion to prove the default remains opt-out, so Admin Center and
     Help labels do not change unintentionally.
   - Keep click-range assertions tied to the rendered text so capitalization cannot desynchronize hit targets or
     centering math.

2. **Projects integration** (`tests/ace/tui/test_projects_pane.py` or the existing Projects sub-tab tests): assert the
   default strip shows active `PROJECTS`, then switch or click to Repos/Workspaces and verify exactly one label is
   uppercase while prior labels return to title case. Retain the existing content-switch and focus assertions to prove
   this remains a presentation-only change.

3. **Artifacts integration** (`tests/ace/tui/test_artifacts_scaffold.py`): assert the default active spelling is `PRS`,
   switch to another sub-tab, and verify inactive `PRs` plus the newly uppercase active label. Exercise the existing
   programmatic/click message path so lifecycle behavior and label state stay synchronized.

4. **PNG snapshots:** regenerate only intentional label diffs with `--sase-update-visual-snapshots`, then inspect the
   actual PNGs before accepting them. The affected families include Projects/Repos/Workspaces Admin Center snapshots
   (and the inventory picker if its underlying strip is visible) plus Artifacts PR onboarding, Commits, Bugs, and Plans
   snapshots. Confirm each image has exactly one uppercase nested-tab label, that inactive `PRs` is spelled correctly,
   and that centering/dividers do not shift unexpectedly. Run the full visual suite afterward to discover any other
   currently visible instance of these shared strips rather than guessing the complete golden set.

## Validation sequence

- Run focused non-visual tests for `PanelTabStrip`, Projects sub-tabs, and the Artifacts scaffold.
- Run the targeted Projects and Artifacts visual snapshot modules, update only expected goldens, and inspect the PNG
  output.
- Run `just install` before repository checks as required for an ephemeral workspace.
- Finish with `just check` and `just test-visual` so lint, typing, all behavioral tests, and all exact-pixel snapshots
  pass together.

## Risks and guardrails

- **Unrequested global capitalization:** prevented by a default-off strip option and a regression assertion for an
  existing top-level numbered strip.
- **`Prs` returning after a switch:** prevented by storing `PRs` as the canonical Artifacts display label and deriving
  active `PRS` only at render time.
- **Click targets drifting from visible text:** prevented by calculating ranges and total line width from the
  transformed render, with existing click tests retained.
- **Snapshot churn hiding unrelated visual changes:** update only label-related failures, inspect changed goldens, and
  leave unrelated goldens untouched.

## Out of scope

- Uppercasing the main ACE tab bar, the Admin Center's top-level Config/Logs/Projects/etc. tabs, or Help modal tabs.
- Changing internal sub-tab identifiers, keybindings, colors, separators, lifecycle activation, data loading, or tab
  persistence.
- Broadly changing other occurrences of `PRs`; only the Artifacts nested strip currently derives the incorrect `Prs`
  label.
