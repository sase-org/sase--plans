---
plan: 202607/companion_sdd_finalizer_prompting.md
---
 Though sase automatically commits plans and beads to the `sase--plans` repo for some changes, we should mostly be treating the `sase--plans` and `sase--research` companion repos just like linked repos when it comes to commits (agent file changes should be caught by the finalizer and the agent should be prompted to commit the changes), but I'm seeing a lot of `sync uncommitted SDD store changes` commits to these repos and am not seeing diffs of the changes in the file panel or any file entries in the "Deltas:" metadata panel field (see #sshot). Can you help me diagnose the root cause of this issue and fix it?

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  