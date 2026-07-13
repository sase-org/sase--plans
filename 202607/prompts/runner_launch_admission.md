---
plan: 202607/runner_launch_admission.md
---
 The sase agent named 7u started too early I think. Other agents launched before the one and only agent that was running completed. Can you help me confirm my suspicion, diagnose the root cause, and fix this? You can lead the design on this one but I'm thinking that the solution is likely that we need to update the active runner count for these types of agents any time a new agent is launched. Make sure this doesn't hurt performance too much. Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 