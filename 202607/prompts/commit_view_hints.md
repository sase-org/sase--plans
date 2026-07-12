---
plan: 202607/commit_view_hints.md
---
 Can you help me add support to the `v` (view) keymap on the "Agents" tab of the `sase ace` TUI for "COMMITS:" field (in the agent metadata panel) commit entries?

- When the corresponding hints are selected, the user should be shown the full commit message and a pretty diff of the changes made by the selected commit.
- For example, in #sshot the 52c99ca5d and 0fed546e32f4 git commits should have hints rendered next to them if the user presses `v`.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 