- **PLAN:** [../202607/artifact_tranche_zero_and_generic_sidecar_roles.md](../artifact_tranche_zero_and_generic_sidecar_roles.md)

 Can you help me fix the defects described by the
`Tranche zero: four small, independent defect fixes` section of the
artifact_refs_and_inspector.md research repo file?

- You should fix all of the defects except for the one described by the
  `Surface research in Plans.` bullet.
- That bullet reveals a deeper problem that you should try to figure out and
  fix. Namely: The `<project>--research` sidecar repos are custom user-defined
  sidecar repos. Their shouldn't be any code that references them directly.
- Do your best to make any existing logic that references that repo generic so
  it works with other user-defined sidecar repos.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 