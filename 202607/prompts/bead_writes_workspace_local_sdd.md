---
plan: 202607/bead_writes_workspace_local_sdd.md
---
 It looks like some sase agents are modifying the sdd repo that is checked out in the primary workspace instead of their own workspace directory (see #sshot for context). This is NOT allowed. They should instead sync their local .sase/sdd repo, make their bead changes there using the `sase bead` command (which might be the command that needs fixing), and then the finalizer should ensure that those bead changes are committed (although ideally the `sase bead` command is already taking care of the commit for us).

Can you dig into this, diagnose the root cause, and fix the issue? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 