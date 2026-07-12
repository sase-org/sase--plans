---
plan: 202607/is_sase_managed.md
---
 Yesterday we fixed an issue with the `sase memory init` command where it was attempting to initialize memory files for project repos that are not managed by sase as a main active project (this repo, for example). This worked but the problem has a larger scope than I was originally thinking. The `sase sdd init` command in particular should not attempt to initialize an SDD repo or really do anything for a project that is not an active main sase project. Can you help me fix this by migrating the recently added `memory.enabled` boolean field to a new `is_sase_managed` boolean field that is used the same way but is also respected by the `sase sdd init` command? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 