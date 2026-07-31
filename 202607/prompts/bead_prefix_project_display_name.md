- **PLAN:** [../202607/bead_prefix_project_display_name.md](../bead_prefix_project_display_name.md)

 I don't understand why this epic bead (for the enabled `bob-cli` sase project) was named using the `gh_bobs-org__` prefix (see the output below for context). This epic bead was created by the `sase bead work` command, which was run as a sase background task after I approved an epic from the TUI. Can you help me diagnose the root cause of this issue and fix it? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
  
```
bryan in 🌐 athena in bob-cli on  master is 📦 v0.1.0 via 🦀 v1.97.1
❯ pwd
/home/bryan/projects/github/bobs-org/bob-cli

bryan in 🌐 athena in bob-cli on  master is 📦 v0.1.0 via 🦀 v1.97.1
❯ sase bead list
◐ gh_bobs-org__bob-cli-2 · Capture sub-bullets onto existing Obsidian tasks
◐ gh_bobs-org__bob-cli-2.3 · bob capture-tasks discovery command ← gh_bobs-org__bob-cli-2
◐ gh_bobs-org__bob-cli-2.4 · Hammerspoon task picker ← gh_bobs-org__bob-cli-2
```