---
plan: 202607/clan_member_status_sort.md
---
 #fork:dv This fixed the bug, but now that I look at it, it doesn't look
like the agent that caused this bug even completed the feature I assigned it,
right (see #sshot)? Agents/agent families within an agent clan should always be
sorted by status (regardless of what grouping strategy is currently being used
by the agents tab):

1. Failed agents
2. Stopped agents
3. Running agents
4. Waiting agents
5. Completed agents

Can you help me fix this? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 