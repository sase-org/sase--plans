- **PLAN:** [../202607/agent_page_artifacts.md](../agent_page_artifacts.md)

 Can you help me make the agent pages (and agent family pages) that are
viewed from the `--agents` sidecar repo's GitHub page (see
~/tmp/screenshots/20260727_160315.png for an example agent page) attach links to
useful artifacts?

- Any agent with commits associated with it should link to those commits from
  the corresponding agent's page.
- We should also link to the agents sidecar repo GitHub pages all of an
  agent's/agent family's neighbors. See how we determine agent neighbors for the
  `NEIGHBORS` section in the agent metadata panel on the agents tab in the TUI
  for context and inspiration.
- Also, we should start displaying any sase variables that an agent sets via its
  /sase_var skill on this page as well.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 