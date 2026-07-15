---
plan: 202607/per_project_research_sidecars.md
---
 It looks like when we migrated the `sase--research` sidecar repos to custom sidecar repos (see the sase-60 epic bead for context), the entry for that sidecar repo stopped being added to agent instruction files.

- As a result agents do not know about this repo and thus do not know that they should use the `sase repo open` command to open it.
- Also, it looks like the research sidecar repo that you configured for all of my repos (in the ~/.local/share/chezmoi/home/dot_config/sase/sase.yml file in my chezmoi repo), you defined the `sase-org/sase--research` repo for all repos. This is NOT correct.
- Every one of my enabled sase projects should be configured to use their own research (e.g. `bobs-org/bob-cli--research` for the bob-cli project and `bbugyi200/actstat--research` for the actstat project) sidecar repo, which the `sase init` command should automatically create if necessary in the GitHub organization that the sase project lives in.
- We should stop defining the configuration for this sidecar repo in my chezmoi repo and instead define it (with a very similar/identical description) in each of my sase project repos (this way the `sase init` command generates the same agent instruction files for each of these projects regardless of what machine it is run on).

Can you help me fix this? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 