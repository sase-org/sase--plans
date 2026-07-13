---
plan: 202607/research_swarm_lead_month_dir.md
---
  Can you help me fix the `#research_swarm` xprompt to make some things clear to the final / consolidator research agent?

- This agent should create the new directory inside a directory of the form `<YYmmdd>` (represents the current month). We should give this directory to the agent explicitly using xprompt shell expansion with the `date +%y%%m` command.
- Make sure this agent understands that it is the lead researcher and as such it should perform its own research, which should be merged into the final result.
- Use best prompting practices here to make sure that we keep this prompt concise and meaningful. Every token in context is either helping us or hurting us.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  