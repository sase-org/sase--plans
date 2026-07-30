- **PLAN:** [../202607/family_scoped_agent_provenance.md](../family_scoped_agent_provenance.md)

 We currently determine which agent data to publish to the agent sidecar repo and which agent to link to via the SASE_AGENT commit tag by using the agent that created the commit and then gathering all data related to that agent and adjacent agents. Can you help me start using and linking to the agent family instead of the agent family member?

- For example, where we might link to the `foo--code` agent before this change, we should link to the `foo` agent family after this change.
- For agents that do not belong to agent families, we should not change this behavior at all.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 