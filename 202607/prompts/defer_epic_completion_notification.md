- **PLAN:** [../202607/defer_epic_completion_notification.md](../defer_epic_completion_notification.md)

 Launching epics using detached background tasks is working great. There is only one problem: I receive a completion notification for the agent that proposed the epic plan before the task completes (see #sshot). I shouldn't receive that notification until the task completes and the agent's status is changed from `EPIC APPROVED` to `EPIC CREATED`. Can you help me fix this? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 