- **PLAN:** [../202607/bead_work_cleanup_confirm.md](../bead_work_cleanup_confirm.md)

 Can you help me make the `sase bead work` command more resiliant by having it automatically cleaning up any old agents related to or named the same as the agents that the command intends to launch?

- This should include dismissing / killing agents if necessary.
- Make sure that we are very clear with the user about which agents we will dismiss / kill.
- Additionally, confirm with the user via a y/n prompt before performing any destructive actions like this.
- The user should be able to override this y/n prompt, but not via the existing `-y|--yes` option. They will need to use the new `-Y|--yes-to-all` option to override this (and the y/n prompt that the `-y|--yes` option was overriding).
- See the command output below for an example of a failure that I wouldn't expect to happen after these changes.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
  

### Failed `sase bead work` Command Output
```
❯ sase bead work sase-8u.4 -y
Epic sase-8u.4 is already ready; retrying remaining non-closed phases.
Epic sase-8u.4 — Finish and land capitalized snippet aliases: 2 phase agent(s) in 2 wave(s) plus 1 land agent (sase-8u.4.land).
  Clan: sase-8u.4 · Tribe: @epic
  Wave 0: sase-8u.4.2 → sase-8u.4.2
  Wave 1: sase-8u.4.3 → sase-8u.4.3
  Land waits on: sase-8u.4.2, sase-8u.4.3
Error: forced reuse cleanup could not resolve any concrete members for agent family 'sase-8u.4.2'
```