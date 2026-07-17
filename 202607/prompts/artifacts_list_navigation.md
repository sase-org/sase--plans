---
plan: 202607/artifacts_list_navigation.md
---
 Can you help me make the `g` / `G` / `<ctrl+d>` / `<ctrl+u>` / `<ctrl+f>` / `<ctrl+b>` keymaps work on the sub-tabs of the "Artifacts" tab (except for the "PRs" tab, which already has good navigation) to make navigating the list of entries in the left panel (i.e. commits, bugs, or plans) easier?

- `g` / `G` should jump to the first / last entry in the list, respectively.
- `<ctrl+d>` / `<ctrl+u>` should jump down / up 10 entries, respectively.
- `<ctrl+f>` / `<ctrl+b>` should jump down / up 5 entries, respectively.
- We should also add support for the special apostrophe functionality (supported elsewhere in the TUI) to these sub-tabs.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
