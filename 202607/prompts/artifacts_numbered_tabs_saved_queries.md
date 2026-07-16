---
plan: 202607/artifacts_numbered_tabs_saved_queries.md
---
 Can you help me start numbering the sub-tabs on the "Artifacts" tab and mapping those numbers to keymaps that (when triggered) jump to the corresponding sub-tab (see how the similar keymaps on the "SASE Admin Center" panel work for inspiration)?

- We currently use the `0-9` keymaps for saved queries, but let's start using the star (`*`) character as a prefix for those keymaps.
- When the user presses `*` (from the `PRs` tab only for now--this keymap should not be active anywhere else) they should be presented with a menu of their current saved PR search queries. They should be able to select which saved query slot they would like to load using a single key press.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 