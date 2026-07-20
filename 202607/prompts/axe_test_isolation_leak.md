---
plan: 202607/axe_test_isolation_leak.md
---
 I think there is still something wrong with the `wait_checks` sase chops. This agent, for example, should have been launched since the agent it is waiting for is already complete, right (see #sshot for context)? Can you help me diagnose the root cause of this issue and fix it? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 