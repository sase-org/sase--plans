---
plan: 202607/commit_hooks.md
---
 sase already supports a project-local `precommit_command` configuration field that specifies a command that should be run before the `sase commit` command makes a git commit. Can you help me add a new config field that does the same thing but runs the command after the commit has been created and pushed?

- Let's rename the `precommit_command` field to `commit_hooks.before` and name the new field `commit_hooks.after`.
- Set this new field in a new sase.yml file at the root of my chezmoi repo to `chezmoi update -a --force` (so my chezmoi changes are always applied to this machine after committing).
- Make sure you update any references to the old `precommit_command` sase config field.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 