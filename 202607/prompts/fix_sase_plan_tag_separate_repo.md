---
plan: 202607/fix_sase_plan_tag_separate_repo.md
---
 When sase agents commit after a tale is approved (i.e. the plan file is copied from the ~/.sase/plans/
directory to the .sase/sdd/tales/ directory and then committed to the .sase/sdd/ repo--which is associated with its own GitHub repo), the `sase commit` command is supposed to include a `SASE_PLAN` git commit tag in the message of the form `SASE_PLAN=tales/<YYmmdd>/<plan_name>.md`, but that doesn't seem to be happening anymore (I think before we included the `sdd/` in the `SASE_PLAN` tag value, but we can drop that now). Can you help me diagnose the root cause of this issue and fix it?

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  