---
plan: 202607/phase_agent_metadata.md
---
 When phase agents run (that work phases of an epic plan), we seem to show the same "SASE PLAN" section when those agents are selected that we show for the agent that created the plan. This is not correct. Can you help me fix this?

- Only the agent that created the epic and the agent that lands the epic should show the "SASE PLAN" section.
- We should show only the phase that the current phase agent is working when a phase agent is selected.
- Luckily for us this phase data that we can take from the frontmatter of the plan file should be the same as the bead description. Let's start using the frontmatter of the plan file to construct the `Bead:` field of the agent metadata panel when a phase agent is selected (this should be much faster than reading sase beads). This bead data is all we should show for phase agents.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
