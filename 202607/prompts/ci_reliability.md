---
plan: 202607/ci_reliability.md
---
 GitHub Actions is failing for the sase repo. Can you run the `actstat` command to get more information about
the failing jobs, diagnose the root cause of these failures, and then fix them? You can raise the perf floor a bit if needed, but make sure there isn't a major performance regression we need to fix. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
