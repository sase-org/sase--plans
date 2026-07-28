- **PLAN:** [../202607/queued_status_for_all_runner_slot_waits.md](../queued_status_for_all_runner_slot_waits.md)

 I thought we got rid of this use for `WAITING` in favor of the new `QUEUED` status, right? See #sshot for context. Can you help me fix this so we always use `QUEUED` when agents are waiting because of the configured maximum allowed number of running agents? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 