---
plan: 202607/periodic_ace_update_checks.md
---
 When we start the TUI, currently there is a check for outdated sase packages and an update notification is shown as a toast to the user if any packages have available updates. We also set a visual indicator in the right-hand corner if updates are available. The problem is if the user leaves the TUI open for a long time they're never made aware of updates. Can you help me start running the same check automatically every 10 minutes iff the update indicator is not already set? Make sure this is fast and does not block the TUI. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 