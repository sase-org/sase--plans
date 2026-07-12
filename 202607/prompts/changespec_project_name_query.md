---
plan: 202607/changespec_project_name_query.md
---
 We currently don't respect the PROJECT_NAME project spec field when using the `project:<project>` ChangeSpec query filter. For example, in #sshot, the `project:gh_sase-org__sase` query should match no ChangeSpecs. Instead, `project:sase` should match the ChangeSpecs shown in the screenshot. Can you help me diagnose the root cause of this issue and fix it? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 