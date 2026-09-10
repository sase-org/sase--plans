---
tier: epic
title: Repair post-start plan archives before landing sase-z2
goal: Publish the two recoverable bead-linked plans created after sase-z2 began and
  prove the plans sidecar is current.
parent_bead: sase-z2
phases:
- id: repair-archives
  title: Repair and verify the post-start plan archives
  depends_on: []
  description: 'repair-archives: publish the two recoverable bead-linked plans and
    verify their canonical, sidecar, and remote state.'
  size: xsmall
proposed_by: bbugyi200.athena.sase-z2.land
create_time: 2026-09-10 12:48:58
status: done
bead_id: sase-z2.5
---

- **PROMPT:** [prompts/202609/repair_post_start_plan_archives.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/repair_post_start_plan_archives.md)
- **PARENT:** [202609/durable_plan_archive_publication.md](https://github.com/sase-org/sase--plans/blob/main/202609/durable_plan_archive_publication.md)
- **BEAD:** [sase-z2.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-z2/sase-z2.5.md)

# Plan

Repair only the post-start drift found by the `sase-z2` landing audit. The live
`sase bead doctor` report currently identifies these recoverable bead-linked plans:

- `plan:202609/shared_format_bridge.md`, owned by `sase-x7.5.1`
- `plan:202609/weighted_queue_capacity.md`, owned by `sase-z4`

Run the confirmed `sase bead doctor --fix-plan-archive` repair through the supported
archive machinery. Verify that both canonical sources and the published plans-sidecar
copies contain their correct `bead_id` values, that a fresh remote view contains both
documents, and that a subsequent doctor run no longer reports either plan as missing or
recoverable.

Preserve the remaining source-missing and invalid legacy-plan findings. Their landing
dispositions are already tracked separately by `sase-bw` and `sase-zb`; do not normalize
or invent metadata for those files as part of this tale.
