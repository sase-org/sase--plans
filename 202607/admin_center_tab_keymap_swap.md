---
create_time: 2026-07-13 09:07:19
status: wip
prompt: 202607/prompts/admin_center_tab_keymap_swap.md
tier: tale
---
# Swap SASE Admin Center main-tab and sub-tab keymaps

## Objective

Make `Tab` / `Shift+Tab` move forward and backward through the SASE Admin Center's six main tabs, while `]` / `[` move
forward and backward through a pane's local sub-tabs (currently the Projects state filters). Keep numbered and mouse tab
selection intact, and do not change the configurable `next_tab` / `prev_tab` keymaps used by the underlying ACE main
tabs.

## Design and implementation

1. **Give the Admin Center ownership of `Tab` / `Shift+Tab`.**
   - Replace the Admin Center modal's bracket bindings with priority `Tab` and `Shift+Tab` bindings that reuse the
     existing wrapping `action_next_center_tab` / `action_prev_center_tab` actions.
   - Use modal-level priority so focused widgets cannot turn these keys into focus traversal or pane-local actions. This
     works with ACE's existing `check_action` modal guard, which suppresses the hidden app's own priority tab-switch
     actions while a modal is active.
   - Preserve direct numeric selection, mouse selection, tab ordering, active-pane focus restoration, and persisted
     active-tab behavior.

2. **Move Projects state-sub-tab navigation to brackets.**
   - Rebind the Projects pane's forward/reverse state-filter actions from `Tab` / `Shift+Tab` to `]` / `[`, retaining
     the current order and modulo wrapping across Active, Sibling, Inactive, and All.
   - Update the Projects filter input's special key handling so printable brackets are intercepted and routed to the
     owning Projects pane's state-filter actions. Let `Tab` / `Shift+Tab` reach the Admin Center's priority bindings,
     ensuring the same hierarchy works whether the project list or filter input has focus.

3. **Remove legacy bracket forwarding and resolve focused-input collisions.**
   - Remove the Config, Updates, and XPrompts filter inputs' special forwarding of `[` / `]` to the Admin Center. On
     panes without sub-tabs, brackets should no longer change main tabs and focused text inputs should be free to accept
     them as filter text.
   - Remove the XPrompts filter's bare-`Tab` fallback for inline loading, because `Tab` must consistently mean next
     Admin Center tab. Retain the explicit `Ctrl+I` binding for environments that report it distinctly; terminals that
     encode `Ctrl+I` as the same byte as Tab will now follow the newly requested Tab navigation behavior.
   - Leave other bracket-driven interfaces, such as Help panel tabs and notification tag tabs, unchanged because they
     are separate modal hierarchies rather than Admin Center sub-tabs.

4. **Update visible guidance and nearby documentation.**
   - Revise the Admin Center module description, pane/filter docstrings, and inline comments to describe the new key
     hierarchy.
   - Update the Config, Logs, Projects, Tasks, Updates, and XPrompts hint lines: advertise `Tab` / `Shift+Tab` for main
     Admin Center tabs, and advertise `[` / `]` for Projects state sub-tabs.

5. **Strengthen behavioral and visual regression coverage.**
   - Update Admin Center integration tests to exercise forward/backward Tab navigation and wrapping while representative
     lists and text inputs have focus, proving the keys neither traverse focus nor switch the hidden ACE main tab.
   - Update Projects unit/integration tests to use brackets for state-filter cycling in both directions, including wrap,
     and to verify the focused filter forwards brackets locally while Tab leaves Projects without changing its state
     filter.
   - Update Logs and XPrompts collision tests so Tab switches Admin Center tabs, brackets no longer perform legacy main
     navigation, and explicit `Ctrl+I` XPrompt loading remains covered without treating bare Tab as load.
   - Regenerate only the Admin Center PNG goldens whose rendered hint lines intentionally change, and inspect the
     actual/expected/diff artifacts before accepting them.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run focused Admin Center, Projects, Logs, Updates, Config, and XPrompts TUI tests while iterating.
3. Run `just test-visual`, inspect any Admin Center snapshot diffs, and update only intentional hint-text goldens.
4. Run the required full `just check` and confirm the worktree contains only the planned source, test, and intentional
   snapshot changes.
