---
plan: 202607/canonical_sase_directories.md
---
 We already store sase repos in a sase/repos/ directory.

- I would like to start storing other sase directories like the xprompts/ directory and the memory/ directory in the sase/ directory as well.
- This should apply to both project local versions of these directories and the home/chezmoi versions.
- Also let's start storing project-local sase.yml files in this (project-local) sase/ directory as well.
- Make sure you check what sase projects are enabled right now and update all references.
- Also make sure that all documentation is up to date and that if any infographic images need to be updated, that you use an epic plan with phases dedicated to updating those images using GPT image.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
