---
plan: '[../202607/commit_view_plan_toggle.md](../commit_view_plan_toggle.md)'
---
 Can you help me make all plan files viewable from the "Commit" panel that is shown when a particular set of commits are selected (supported on the "Commits" sub-tab of the "Artifacts" tab as well as when using the `v` keymap from the agents tab)?

- We should add a new `p` keymap to this panel that temporarily shows the plan file associated with the given commit (parsed from the `SASE_PLAN` sase commit tag, if present; otherwise, show a good error when this keymap is used).
- The user should be able to press `p` again to reload the commit that we were viewing.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
