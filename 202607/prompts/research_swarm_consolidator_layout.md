---
plan: 202607/research_swarm_consolidator_layout.md
---
 Can you help me change the prompt for the Research Consolidator Agent (the 3rd one) defined in the Research Swarm xprompt that's defined in my chezmoi repo? This agent should start creating a directory in the directory that it used to create a file in, named the same as the research file it intends to create (with the .md file extension removed). Instead of deleting the research files created by the previous two agents, this agent should rename them to use `<name>/<name>__a.md` and `<name>/<name>__b.md`. The final research markdown file produced by this agent should be named `<name>/<name>.md` (In the appropriate research directory of course). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
