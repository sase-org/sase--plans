---
plan: 202607/epic_phase_sizes_and_child_epics.md
---
 I want to start requiring sase agents that create plans with an epic
tier (via the /sase_plan skill) to start having to (enforced by the
`sase plan validate` command) specify a `size` property for each phase in the
epic: `small`, `medium`, or `large`. Can you help me implement this?

- A phase of size small should be treated exactly the way we treat every phase
  now.
- A phase of size medium should have the `#plan` xprompt added to the end of the
  prompt so the agent is requested to create a plan before implementing (which
  will be auto-approved since phase agents have `%auto` in their prompts).
- A phase of size large will have the same prompt as a phase with size medium
  but will configure the corresponding phase bead (and thus the corresponding
  phase agent) to use the `@smartest` model alias for its model.
- We should instruct agents creating these epic plans to use a phase of size
  medium if they think that the phase is potentially a lot of work and justifies
  its own plan file. They should be told to use a size large if they suspect
  that the plan file will be sized large enough that it will be assigned an epic
  tier.
- As a part of this change we should start supporting epic beads that have other
  epic/phase beads as their parent. This way phase agents and the epic lander
  agent can create epic plans/beads and have them associated back to the main
  epic bead / plan file.
- We should try to make the association of epic beads with their parents
  automatic by setting an environment variable that has the value associated
  with the phase or epic bead that the agent is responsible for.
- We should use the same naming schema for child epics that we do for phase
  beads.
- For example, if an epic bead `foo-5` has 3 phases and the 2nd phase creates
  and epic, that epic bead should be named `foo-5.2.1`. If the epic lander agent
  created an epic plan, the epic bead would be named `foo-5.4` (since the epic
  lander should be associated directly with `foo-5`, not a phase bead).
- Make sure that the sase bead show command has excellent support for epic beads
  that have parents.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 