---
plan: 202607/toggle_all_metadata_folds.md
---
 Can you help me make sure that the `zZ` keymap either completely opens all folds (i.e. highest fold level) or completely closes all folds (lowest fold level) depending on the current fold state?

- If we are currently using a fold level below the maximum, we should open all folds; otherwise, we should close all folds.
- I'm not sure exactly what functionality this keymap had before but replace it with the requested functionality.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
