---
plan: 202607/commits_filter_bar.md
---
 The `f` (filter) keymap on the "Commits" sub-tab of the "Artifacts" tab
doesn't work (pressing `<enter>` after inputting a filter does nothing). Can you
help me fix this and also improve the filtering functionality on this tab?

- Let's start using `/` to trigger this filtering functionality.
- Add support for filtering commits by repo as well as by date / whatever we
  claim to offer now.
- Add good completion for this new filtering bar that we should make appear at
  the top of this tab after the / key is pressed.
- Typing in new this new filter bar should also update the commits live if
  possible but make sure that we are conscious of performance here (and never
  block the TUI).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 