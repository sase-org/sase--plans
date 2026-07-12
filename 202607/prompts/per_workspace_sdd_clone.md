---
plan: 202607/per_workspace_sdd_clone.md
---
 sase coder agent (that implement approved plans) launches are still failing (see #sshot and recent related git commits for context). Can you help me diagnose the root cause of this issue and fix it? Keep in mind that we should continue to use a relative .sase/sdd/ file path for the plan file reference.

- This should be easy... We just need to clone a fresh copy of (if it doesn't already exist) or sync that workspace's .sase/sdd/ repo (e.g. by running the `git pull` command in that directory) during workspace preperation, right?
- This does mean that we need to make sure the .sase/sdd/tales/ plan file is committed and pushed before we attempt to prepare the coder agent's workspace directory.
- I think we use a symlink currently, but this is not ideal either. What happens when multiple agents in different workspaces modify the same sdd file in conflicting ways at the same time?

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 