---
plan: 202607/fix_silent_failed_agent_workspace_leak.md
---
 We don't seem to ever be releasing sase workspace claims anymore. The sase agent named "7e", for example, is using workspace number 27 even though there are only three sase agents running on the sase project right now. Yesterday, we made it so failed agents no longer release their workspaces until dismissed, but this was supposed to only be for failed agents. Can you help me diagnose the root cause of this issue and fix it? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  