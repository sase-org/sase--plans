---
plan: 202607/artifacts_tab.md
---
 I want to rename the "PRs" tab to "Artifacts" and make "PRs" a sub-tab (one of four) on that tab.

- The "PRs" sub-tab should function in the same way that it does today.
- The new "Commits" tab should provide a surface that combines the beauty and same options that the `sase vcs log` command does, but with much better support for commit messages and diffs (see how the "Commits" panel on the agents tab handles this for inspiration).
- The new "Bugs" sub-tab Will allow the user to create and view and modify bugs that are managed by an external system like GitHub issues, which should be supported to start.
- The new "Plans" tab should provide a good way to view and manage sase projects (depending on what sase project we are filtering on). This tab should also provide a good way to view and manage beads (which are stored in the same `plans` sidecar repo) with the goal of eventually allowing an epic bead to be associated with an external bug.
- Make sure that every feature you add provides real practical value to the user. No fluff.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Can you help me implement an (ambitious) MVP for this new functionality? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 