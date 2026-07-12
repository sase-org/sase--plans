---
plan: 202607/hold_failed_agent_workspaces.md
---
 Currently when an agent fails, its workspace directory is still released. Often when an agent fails, it has not made any commits although it finished all of the work that it needed to successfully. The problem is that because we release the workspace, there's a strong possibility that another agent will start running and claim the workspace. The workspace preparation process will clear any file changes that the previous failed agent made, making it impossible to recover those changes or difficult at least. Can you help me stop releasing the workspace directory when agents fail and instead only release that workspace directory after the user dismisses that failed agent from the agents tab?

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  