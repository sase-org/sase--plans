- **PLAN:** [../202607/bead_close_phases.md](../bead_close_phases.md)

 Can you help me make it easier to close multiple phase beads that are
associated with the same epic bead with the `sase bead close` command by adding
a new `-p|--phases` option that accepts a comma-separated list of phase names?

- For example, the `sase bead close sase-at -p 1,2,3` command should now be
  equivalent to the `sase bead close sase-at.1 sase-at.2 sase-at.3` command.
- This option should also accept ranges. So, for example, the
  `sase bead close sase-at -p 1-3` command should be equivalent to the two
  commands in the above bullet.
- Make sure we fail with a good error if the target bead provided is not an epic
  bead when this option is used (this option makes no sense unless the target is
  an epic bead).

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 