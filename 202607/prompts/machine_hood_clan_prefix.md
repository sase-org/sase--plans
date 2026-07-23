- **PLAN:** [../202607/machine_hood_clan_prefix.md](../machine_hood_clan_prefix.md)

 The sase-8k epic bead seems to have introduced a few bugs (see what the `is.cdx` sase agent is working on, for example--and make sure your work doesn't solve the same problem or conflict with that agent's work). A new one I've discovered is that we don't seem to always hide the machine name hood when we should (i.e. when shown to a user in the TUI on the machine corresponding with that hood). See #sshot for an example of this bug (notice the `athena.sase-8t` sase agent name has the `athena.` agent hood prefix when it should not). Can you help me diagnose the root cause of this issue and fix it? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 