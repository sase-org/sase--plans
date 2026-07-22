- **PLAN:** [../202607/agent_group_clan_collapse_precedence.md](../agent_group_clan_collapse_precedence.md)

 When we use the `H` keymap from inside an agent tribe panel group that has expanded agent clans, we should prioritize fully collapsing them all before collapsing the group. For example, if the user pressed `H` in #sshot, I would expect the `sase-8k` clan to be collapsed, not the `Running` group. We already have similar logic that does this correctly for agents and agent families that are expanded within groups when this keymap is used but we don't seem to do this for agent clans that are expanded. Can you help me fix this? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
