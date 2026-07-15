---
plan: 202607/fresh_workspace_sidecar_clone.md
---
 Can you help me ensure that we remove the local sase/repos/ directory and all of its contents during workspace directory preperation (before launching the agent) and that the sase/repos/plans/ sidecar repo is freshly cloned each time. I keep seeing rebase conflicts in the sase/repos/plans/ repo that is cloned to workspace directories right after that workspace is initialized, which shouldn't be possible with a fresh clone. Try to make sure this is as fast as possible without allowing the possibility of conflicts/errors that would not have happened if we did a fresh, normal git clone of the repo--I'm not sure what can be done here, but think about it.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
