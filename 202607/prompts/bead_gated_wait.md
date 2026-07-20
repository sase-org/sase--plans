---
plan: 202607/bead_gated_wait.md
---

#fork:sase-7z.land Can you now help me ensure that phase agents that depend on other phase agents that wind up creating epic plan files (that are auto-approved, since phase agents include the `%auto` directive in their prompt) wait for epics to complete? I'm not sure if we addressed this edge case or not. If not, use your /sase_plan skill to plan the appropriate changes.
 I think we should be able to solve this by adding a new `bead=<bead_id>` kwarg to the `%wait` directive and having every phase agent depend on both the phase agents and their corresponding beads, right? This way, even after the phase agent another phase agent depends on completes by creating and launching an epic plan,  we would still not start the dependent agent until the phase agents of the sub-agent mark the dependency phase agent's corresponding sase bead as closed. 