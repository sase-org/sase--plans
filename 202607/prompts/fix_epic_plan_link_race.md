- **PLAN:** [../202607/fix_epic_plan_link_race.md](../fix_epic_plan_link_race.md)

 Can you help me figure out why launching this epic failed (see the output below), fix the underlying issue, and then re-run that command to launch the epic? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.


```
❯ sase bead work ~/.sase/plans/202607/beads_sidecar_repo.md -y

Epic plan  /home/bryan/.sase/plans/202607/beads_sidecar_repo.md
✓ Validated       tier: epic · 10 phases · 16 dependency edges
✓ Store           sidecar_repos · beads at /home/bryan/projects/github/sase-org/sase/sase/repos/plans/beads
✓ Archived        /home/bryan/projects/github/sase-org/sase/sase/repos/plans/202607/beads_sidecar_repo.md (already archived)
Error: failed to commit bead_id sase-a7 to approved plan /home/bryan/projects/github/sase-org/sase/sase/repos/plans/202607/beads_sidecar_repo.md
Resume with:
  sase bead work /home/bryan/projects/github/sase-org/sase/sase/repos/plans/202607/beads_sidecar_repo.md --yes
```