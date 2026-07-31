- **PLAN:** [../202607/bead_id_shorthand.md](../bead_id_shorthand.md)

 We currently need to specify the full bead ID (ex: `sase-a1`) for all `sase bead` sub-commands that take bead IDs as arguments (e.g. the `sase bead show` command, the `sase bead update` command, etc...). Can you help me make it so we can just use the alphanumeric part of the bead ID (ex: `a1`) as a shorthand? This should be easy enough since any full bead ID is expected to contain a dash, whereas the alphanumeric parts of bead IDs never will. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
