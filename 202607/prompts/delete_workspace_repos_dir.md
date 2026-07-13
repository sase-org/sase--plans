---
plan: 202607/delete_workspace_repos_dir.md
---
 We currently delete the sase/repos/linked/ directory during workspace directory preperation in order to ensure that linked repos that were opened by previous agents in this workspace are not visible to the new agent we are about to launch. Can you help me start just deleting the entire sase/repos/ directory instead? This means that we will need to automatically clone the companion repos that are configured to auto-clone (currently just the `plans` repo) after deleting this directory but before launching the new agent. Make sure we are smart about this auto-cloning so we don't take too much of a performance hit for this.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  