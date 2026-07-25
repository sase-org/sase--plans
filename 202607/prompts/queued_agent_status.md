- **PLAN:** [../202607/queued_agent_status.md](../queued_agent_status.md)

 For agents that are queued, I'd like to start using a new `QUEUED` agent status that replaces the current `WAITING` status that we use for queued agents. This better matches the agent status counts that we use. For example, in #sshot, we should replace `WAITING ▶10/10` with `QUEUED`. Make sure we give `QUEUED` a distinct agent status color. I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
