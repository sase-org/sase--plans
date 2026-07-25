- **PLAN:** [../202607/lane_neighbors_section.md](../lane_neighbors_section.md)

 We already added a tribe members/clan members/family members section to
the agent tribe/agent clan/agent family metadata panel summaries, respectively,
with numeric keymaps that help the user jump to the corresponding member. Can
you help me now start adding a new `NEIGHBORS` section to agent lane metadata
panels (i.e. to agent family metadata panels and sometimes, if a single agent
owns the lane, to agent metadata panels)?

- This section should list all the same neighbors that neighbors panels, which
  is triggered by the `~` (neighbors) keymap, shows for that agent lane.
- Each neighbor should have a numeric keymap listed to the left of it (using the
  same style / logic that we do for tribe/clan/family member sections).
- Because there are potentially many neighbors, we should only show the first
  three in the first fold level.
- We will need to add support for fold levels to single agents (agent
  families/clans/tribes all already support this) for this.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
