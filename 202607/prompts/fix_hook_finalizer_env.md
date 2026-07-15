---
plan: 202607/fix_hook_finalizer_env.md
---
 sase agents keep failing for some unknown reason (there's not a good failure message in the TUI). When I relaunch these failed agents, they always work. I suspect that these agents are failing because I launched them before updating sase and they intentionally fail when they detect that a different version of sase is being used than the version that the launcher was using. Can you help me confirm or deny my suspicion about these failing agents, diagnose the true root cause, and fix it? If I'm correct that these agents intentionally fail, can we start automatically relaunching them since that seems to work every time?

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 