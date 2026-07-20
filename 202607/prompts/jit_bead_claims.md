---
plan: 202607/jit_bead_claims.md
---
 Can you help me start marking beads as in-progress right before the
agent responsible for working that bead (phase worker agent for phase beads or
epic lander agents for epic beads) is launched?

- We currently mark all phase beads as in-progress as soon as we launch the epic
  (i.e. run the `sase bead work <epic_bead>` command) and never (I don't think
  at least) mark the epic bead as in-progress.
- We should be able to accomplish this by adding a new `bead=<bead_id>` kwarg to
  the `%id` directive that specifies that `<bead_id>` should be marked as
  in-progress before that agent is launched (fail with a good error message
  otherwise).

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
