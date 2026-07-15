---
plan: 202607/sdd_sidecar_unmanaged_repo_guard.md
---
 Why was a plan file commited to the `sase-github--sdd` repo when the sase-github repo is not sase managed (i.e. does not set the `is_sase_managed` field in its sase.yml file)? See #sshot for context. This plan file lives in the sase/repos/plans/202607/github_ci_core_source_alignment.md file in our sase/repos/plans/ (the GitHub repo is named `sase-org/sase--plans`) sidecar repo. Can you help me diagnose this issue and make sure this never happens again? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 