- **PLAN:** [../202607/bead_created_by_attribution.md](../bead_created_by_attribution.md)

 Can you help me start correctly recording the `created_by` field on new
sase beads that get created?

- Currently, we seem to always set this field to bryanbugyi34@gmail.com (my
  email). This is not correct.
- Instead for epic and phase beads we should always set this field to the name
  of the agent that proposed the epic plan file.
- For task beads, we should use the name of the sase agent that created the bead
  (or bryanbugyi34@gmail.com, if I created the bead manually).
- When the `sase bead show <bead>` command is run, we should start rendering a
  link to the `agents` sidecar repo GitHub page that corresponds with the agent
  that created this bead.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 