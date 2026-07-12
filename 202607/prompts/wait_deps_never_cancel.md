---
plan: 202607/wait_deps_never_cancel.md
---
 Currently, when I kill an agent that other agents are waiting for, those waiting agents fail (see #sshot). This is not correct. These agents should continue waiting, potentially forever, until all agents that they are waiting for (not all of which might exist yet--and when they do exist, they might fail and/or get restarted) have completed. Can you help me fix this? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
   