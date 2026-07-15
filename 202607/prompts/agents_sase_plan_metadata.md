---
plan: 202607/agents_sase_plan_metadata.md
---
 We recently added the new goal field to the metadata panel on the agents tab, which is shown when the agent has proposed a plan, which always has the goal property in their frontmatter now. The top-level fields in the agent metadata panel do not feel like the right place for this though. Can you help me add a new `SASE PLAN` section to the metadata panel that is shown when an agent has proposed a plan? 

- We should stop showing the "Goal:" section in the metadata panel. Instead the `goal` will be listed under this section.
- We will also list two other fields: `tier` and `path`
- `tier` should be either `plan`, `epic`, or `none` (if the user specified that the plan should be approved but not committed).
- `path` should be the path to the plan file, which should be relative to the agent's workspace directory (e.g. `sase/repos/plans/202607/ace_prompt_input_demo.md`) or, iff the plan file was not committed, absolute (e.g. `~/.sase/plans/202607/ace_prompt_input_demo.md`).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
