- **PLAN:** [../202607/sase_beads_memory.md](../sase_beads_memory.md)

 The /sase_beads skill is bloated and shouldn't be a skill at all since
it represents semantic memory more than it does procedural. Can you help me
migrate it to a new sase/memory/sase_beads.md file that gets auto-added (like
the sase/memory/sase.md file does, since this long-term memory needs to be
available to all sase-managed projects) and make it MUCH more concise?

- Try not to lose too much of the value that the current contents provide but
  keep in mind that every token in context either helps or hurts us so
  conciseness is critical.
- Make sure to remove all existing copies of the /sase_beads skill. This spans
  multiple LLM providers and includes the files in my chezmoi repo and in my
  home directory (since the `chezmoi apply` command does not delete removed
  files).
- Make sure you give the new memory file an excellent description (via the
  `description` property in that file's frontmatter).

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 