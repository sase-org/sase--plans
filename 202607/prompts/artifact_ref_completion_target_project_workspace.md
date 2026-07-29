- **PLAN:** [../202607/artifact_ref_completion_target_project_workspace.md](../artifact_ref_completion_target_project_workspace.md)

 No completion triggers when I type `@res` in the prompt input widget
(see ~/tmp/screenshots/20260729_161932.png). I expected `@research:` to be
offered in the prompt input widget completion menu since `@research` is
configured as a sidecar repo for this project (see the sase-av epic bead for
context). Can you help me fix this?

- FWIW, this completion seems to be working for `@plans` artifacts in the prompt
  input widget.
- This is probably because I said to ignore the parts of the research that
  mentioned the `sase--research` repo since that is a custom user-defined (by
  me) sidecar repo.
- But that doesn't mean that it shouldn't trigger artifact completion. We should
  trigger this completion and support artifact resources from all custom sidecar
  repos.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 