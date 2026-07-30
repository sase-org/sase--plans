- **PLAN:** [../202607/task_beads.md](../task_beads.md)

 Can you help me implement a variant of the
`Delete the capture prohibition and give discovered work a destination`
recommendation of the sase_beads_close_integrity_and_capture.md file in the
research sidecar repo?

- I like the idea of having all epic phase agents file `PROPOSED FOLLOW-UP:`
  notes.
- The epic lander agent can then choose whether or not it wants to create a new
  bead with the (NEW) `task` type. This bead's status should then be changed to
  `ready`.
- **QUESTION:** What does the `ready` status mean now? **ANSWER:** You should
  add a built-in lumberjack chop that checks for any ready task beads and, if
  found, sends a custom sase gate notification that allows the user to either
  close the bead with a reason or launch an agent to work the bead.
- The `open` status is still useful since it allows us to leave beads in a draft
  (i.e. not yet ready to be proposed to the user) status while sase agents
  finish creating them.
- Remove the `#bd/next` xprompt, which isn't used anymore.
- If the user chooses the ready `task` bead sase gate option (which should be
  the default) to launch an agent, a detached background task (like we use when
  approving epics) should be run that marks those beads as `in-progress` (in as
  few commits as possible--which should be pushed to GitHub) and then lauches
  sase agents using a new `#bd/work_task` xprompt (which you should create in
  the appropriate location with the appropriate contents).
- Also, we should add a new recommendation to the builtin sase/memory/sase.md
  file (that gets added to sase managed project repos automatically by the
  `sase memory init` command) to tell agents that they can (and SHOULD) create
  sase `task` beads (and change their statuses to `ready` once done) in the
  following scenarios assuming that the prompt did not explicitly tell them not
  to create beads directly (like we should for epic phase worker agents, for
  example):
  - If a linter or test is flakey/failing but you didn't cause the issue, create
    a task bead instead of ignoring the failure outright.
  - If a sase memory file (which are used to construct agent instruction files
    in sase-managed project repos) or skill contains out-of-date information
    that you believe should be updated, create a task bead to make those
    updates.
  - If a tool / command / script that the project the agent is working on (i.e.
    was launched in) is responsible for has a bug or there is a clear objective
    improvement that can be made to help future agents, that agent should create
    a task bead to fix the bug in / make the improvement to that tool / command
    / script.
- Make sure you give this new `task` bead type excellent visual representation
  on all surfaces that currently display sase beads.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 