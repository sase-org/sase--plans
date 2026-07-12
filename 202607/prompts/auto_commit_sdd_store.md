---
plan: 202607/auto_commit_sdd_store.md
---
 It looks like some changes to the sase-org/sdd repo (checked out locally in the .sase/sdd/ directory) are not committed and pushed to GitHub. Can you help me fix this by having the commit finalizer look for file changes in this repo? Also, let's make it so any command that modifies sdd repo files (ex: the `sase bead close` command) automatically commits the changes to this repo and pushes to GitHub (so the finalizer should just be a fallback--in case we miss something here).

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  