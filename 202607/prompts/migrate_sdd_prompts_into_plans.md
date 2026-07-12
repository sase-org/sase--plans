---
plan: 202607/migrate_sdd_prompts_into_plans.md
---
 We just finished migrating all sdd repo (cloned locally to the .sase/sdd/ directory) tales/ and epics/ plan files to a single plans/ directory. Can you now help me migrate the prompts in the prompts/ directory to the appropriate `plans/<YYmmdd>/prompts/` directory (match the months)? Make sure you move all prompts in the sdd repo to their corresponding locations and then remove the prompts/ directory completely. Also, make sure to update all references to those prompt files (I think they are referenced in plan file frontmatter). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  