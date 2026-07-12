---
plan: 202607/linked_repo_local_workspaces.md
---
 We currently support a `worspace.strategy` config field for `linked_repos` that accepts a value of `none` to indicate that the linked repo's primary workspace directory should always be used. Can you help me get rid of this field completely and start having all linked repos use the same workspace strategy?

- The ability to set `none` as a linked repo's workspace strategy is only used for the chezmoi repo (configured in the ~/.local/share/chezmoi/home/dot_config/sase/sase.yml file).
- The only reason that I did this is because my chezmoi repo is a linked repo that is shared by multiple main sase projects.
- This means that if two different agents working on different sase projects but the same workspace number were to try to open a chezmoi workspace currently, the second one to open the workspace would wipe out the changes from the first one.
- We can fix this however, by starting to clone linked repo workspace directories locally in our current workspace directory. Let's use the new local (already ignored by git) .sase/workspaces/ directory for this.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 

%xprompts_enabled:false
### Questions and Answers

#### Q1: Memory files

> May I regenerate the protected memory files and generated provider instruction shims required by the approved linked-repo plan?

- [x] **Yes, regenerate** — Update generated memory and shims so validation can pass.
- [ ] **No, leave unchanged** — Implement everything else and report the expected validation drift.

%xprompts_enabled:true
