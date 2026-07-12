---
plan: 202607/prepare_workspace_sdd_link.md
---
 The sase agent named "22" just failed. Can you help me diagnose the root cause of this issue and fix it? See the ~/.sase/plans/202607/fix_plan_chain_sdd_ref_resolution.md plan for inspiration, but you'll need to make an adjustment to that plan. Namely:

- We should still use the relative `.sase/sdd/` file path for the plan.
- We should be able to make this work by always ensuring (during workspace preperation) that the .sase/sdd/ repo is up-to-date (e.g. by running the `git pull` command in that directory) before any prompt validation is performed.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  