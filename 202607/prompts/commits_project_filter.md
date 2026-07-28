- **PLAN:** [../202607/commits_project_filter.md](../commits_project_filter.md)

 Can you help me add first-class support for project filters to the
"Commits" sub-tab of the "Artifacts" tab?

- We should add a new `project:<project>` filter
- When the user uses the `p` keymap on that page to filter projects, the new
  `project:<project>` filter should be added to the current filter query.
- Remove other indications of what project we are currently filtering for (the
  `project:<project>` in the filter bar should be the user's only indication).
- Make sure that we pre-load the filter bar query appropriately (i.e. if we
  automatically filter by the current project, insert `project:<project>` in the
  filter bar when loading the commits tab).

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
