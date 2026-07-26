- **PLAN:** [../202607/detached_epic_launch.md](../detached_epic_launch.md)

 Launching the epic associated with the `l4.cld` sase agent, after
approving the epic from the TUI, just failed. Launching an Epic should now
involve running the `sase bead work` command as a sase task (using the new
`sase task` command) command. Can you help me diagnose the root cause of this
issue and fix it?

- Additionally, we need a way to support epic approvals when there is no TUI
  running.
- Let's accomplish this by adding support for a new "detached" type of sase
  background task.
- All epic approvals, regardless of whether done through the TUI or some
  external source (e.g. our sase-telegram integration), should be launched by
  running the `sase bead work` command as a new "detached" task.
- Make sure detached tasks have excellent support across all surfaces that show
  background tasks (e.g. the TUI, the `sase task` command, etc...).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 