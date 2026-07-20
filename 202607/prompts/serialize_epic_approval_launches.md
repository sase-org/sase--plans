---
plan: 202607/serialize_epic_approval_launches.md
---
 I just tried to approve two epics at the same time. This resulted in the epic beads that were created being assigned the same exact ID and that caused a conflict. Can you help me fix this by always running the operations that happen after an epic is approved in sequence when multiple epics are approved at the same time? This way the first epic's beads are already committed and pushed before we start creating the second epic. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
