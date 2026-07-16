---
plan: 202607/plans_pane_all_projects_upgrade.md
---
 Can you help me make the "Plans" sub-tab of the "Artifacts" tab in the TUI way more useful and upgrade the user interface?

- The syntax highlighting is difficult to read for some text in the right pane (see #sshot for context).
- Make sure we show all properties in the frontmatter of the plan file in the right pane.
- We should start listing plans sorted by recency, with most recently proposed plans at the top.
- The plans listed in the left pane should almost never need to be wrapped across multiple lines (figure out how to make these more concise without losing information and/or make this pane wider).
- When I initially load this tab now, there are no results whereas I would expect all plans from all known enabled sase projects to be shown. You should fix this bug.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
  