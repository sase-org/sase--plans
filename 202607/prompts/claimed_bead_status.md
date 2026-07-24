- **PLAN:** [../202607/claimed_bead_status.md](../claimed_bead_status.md)

 Can you help me add a new `claimed` status to sase beads?

- We should start automatically changing a bead's status to `claimed` when an
  agent is started that used the `%id` directive's `bead` kwarg, but has a
  `WAITING` status for some reason (for example, another sase agent that it is
  waiting for might not be completed yet). We already do something similar for
  fully launched agents with the `in_progress` status. See that code for
  inspiration and context.
- If a `WAITING` agent is killed for some reason, we should automatically change
  the corresponding bead's status to `open`, but make sure that the
  `sase bead work` command is resiliant and continues to check if the actual
  agents are running; it should ignore the `claimed` status in that case (and
  probably in general--I don't _think_ there is a need for the `sase bead work`
  command to know about this new status).
- The idea is that this will give us better visibility into which beads already
  have sase agent's associated with them.
- This status should have great visual representation by CLI commands that show
  beads (ex: the `sase bead list` command and multiple other `sase bead`
  commands).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 