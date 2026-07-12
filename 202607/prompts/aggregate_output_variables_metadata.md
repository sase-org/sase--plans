---
plan: 202607/aggregate_output_variables_metadata.md
---
 If an agent sets sase variables using the /sase_va skill, we surface those variables on the agent metadata panel in the TUI. But I think that the root agent entry on the agents tab only shows the variables that are set by the first agent child.  Can you help me fix that by aggregating all agent child sase variables in the agent metadata tab when multiple agent children set variables? See how we handle this for other sections in the agent metadata panel for inspiration. 

I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 