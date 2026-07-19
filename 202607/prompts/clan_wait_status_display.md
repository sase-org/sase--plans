---
plan: 202607/clan_wait_status_display.md
---
 When we wait for an agent clan, we should show all of the statuses of that agent's clan members as values of the `Wait:` field in the agent metadata panel on the "Agents" tab of the `sase ace` TUI. But we currently just show the clan name with the `?` symbol (see #sshot). We also need to make sure that the `Wait:` field makes it clear that we are really waiting for all of a particular clan's members to complete (so it is clear that the list of agents we are waiting for is potentially dynamic). Can you help me fix this / improve this field? Also, make sure that waiting for agent clans using the `%wait` directive is really supported (I think it is)! If not, you should add support. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 