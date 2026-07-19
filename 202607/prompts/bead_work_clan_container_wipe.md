---
plan: 202607/bead_work_clan_container_wipe.md
---
 I'm trying to restart the sase-6v epic but it is not working (see the output below). Can you help me diagnose the root cause of this issue and fix it? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
  
```
❯ sase bead work sase-6v -y
Epic sase-6v is already ready; retrying remaining non-closed phases.
Epic sase-6v — Script-only chops with structured launch proposals: 2 phase agent(s) in 2 wave(s) plus 1 land agent (sase-6v.land).
  Clan: sase-6v · Tribe: @epic
  Wave 0: sase-6v.8 → sase-6v.8
  Wave 1: sase-6v.9 → sase-6v.9
  Land waits on: sase-6v.8, sase-6v.9
Error: forced reuse cleanup left agent name 'sase-6v' reserved after rebuild; resolve the conflicting owner and retry
```