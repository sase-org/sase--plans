- **PLAN:** [../202608/task_bead_plus_one.md](../task_bead_plus_one.md)

 Can you help me add a +1 feature to sase task beads and a new
/sase_new_task xprompt skill that agents should now be instructed to use when
creating new task beads?

- A +1 on a sase task bead should be used to indicate that another agent has
  experienced the same issue.
- When this functionality is used, we should require a note (with evidence
  including artifacts if possible) be left by the agent leaving the +1.
- When this functionality is triggered we should increment a counter and leave a
  (visually distinct) labeled note on the bead (using the note provided by the
  agent).
- The /sase_new_task xprompt skill should instruct agents to first use the
  `sase memory read` command to read the sase/memory/sase_beads.md file and then
  use the `sase bead list` command to look for any existing sase task beads that
  are similar enough to be considered duplicates of the task the agent was about
  to create. If found the agent should be instructed to +1 the task bead with an
  appropriate note.
- The agent should also look for any epics that are in-progress that might have
  caused this issue. If so, the agent should leave a note on the epic bead with
  details on the issue that the agent uncovered. Make sure that the epic lander
  agent's prompt instructs the agent to review the epic bead's notes (we might
  already do this, but I'm not sure).
- The agent should NOT create a new task bead in either of the two cases
  described above.
- As a part of this change, we should stop allowing task beads to be created
  without a `size` attribute (which should support the same values that phase
  bead `size` attributes support). Consequently, we should no longer need the
  `@task_worker` model alias (use the same model aliases from the `phase_bucket`
  that we do for epic phase agents). Make sure the /sase_new_task xprompt skill
  gives good advice on what sizes should be used.
- Think VERY HARD about what instructions should be added to the new
  /sase_new_task xprompt skill with the goal of making it as useful as possible
  while still being concise. Remember that every token in context either helps
  or hurts us.
- Make sure that this new +1 feature for task beads has excellent support on all
  UI surfaces that show information related to task beads (e.g. in the TUI, the
  `sase bead show` command, etc...).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
