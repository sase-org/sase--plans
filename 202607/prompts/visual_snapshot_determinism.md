---
plan: 202607/visual_snapshot_determinism.md
---
 GitHub Actions is failing for the sase repo. Can you run the `actstat` command to get more information about
the failing jobs, diagnose the root cause of these failures, and then fix them? These visual snapshot tests are a constant source of flakiness. Can you help me preserve their usefulness while making them much more deterministic (i.e. iff they pass on this machine or any other machine, they should almost certainly pass in CI)? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 