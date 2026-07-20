---
plan: 202607/admin_config_commit_push_prompt.md
---
 When the user makes configuration changes from the config tab in the sase admin center panel using the `e` (edit) keymap and they have chezmoi configured with sase, we currently write the changes correctly and we apply them to chezmoi. We are supposed to prompt the user whether they would like to commit and push as well though and we do not do that. Can you help me fix this (make sure we use the same y/n prompt popup that we do elsewhere for this same use-case)? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
