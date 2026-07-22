- **PLAN:** [../202607/shutdown_recovery_and_toobig_relaunch.md](../shutdown_recovery_and_toobig_relaunch.md)

 %xprompts_enabled:false
# Previous Conversation

**User:**

I'm having a hard time launching this agent (see the output below). Is this just an issue where data got left behind because I shut down this machine while sase agents were running and Axe was running? If so can you just clean up the data and re-run the command to re-launch this epic?
```
❯ sase bead work sase-8k -y
Epic sase-8k is already ready; retrying remaining non-closed phases.
Epic sase-8k — Hidden agents sidecar repo with machine agent hoods: 4 phase agent(s) in 4 wave(s) plus 1 land agent (sase-8k.land).
  Clan: sase-8k · Tribe: @epic
  Wave 0: sase-8k.3 → sase-8k.3
  Wave 1: sase-8k.6 → sase-8k.6
  Wave 2: sase-8k.7 → sase-8k.7
  Wave 3: sase-8k.8 → sase-8k.8
  Land waits on: sase-8k.3, sase-8k.6, sase-8k.7, sase-8k.8
Error: agent name 'sase-8k.3' is reserved by a family container and cannot be force-reused; dismiss or clean up the container's members, then retry
```

**Assistant:**

I’ll inspect the bead/agent state first to distinguish stale family metadata from a genuinely active run. I’m using the SASE beads and agent-status skills because they define the supported read/cleanup workflow; if the reservation is stale, your request already authorizes cleaning that specific epic state and relaunching it.

The epic itself has no live `sase-8k` agents, while phases 3/6/7/8 remain open. So this is consistent with stale naming state, not a surviving worker. I’m tracing the supported name-registry cleanup command now so I remove only the stale `sase-8k.3` family and don’t disturb the unrelated live `sase-8m` epic.

The reservation comes from a two-member family: the dismissed `sase-8k.3--plan` member and `sase-8k.3--code`. The code member is marked active in persisted metadata, but its PID no longer exists and it has no completion record—exactly the stale-after-shutdown pattern. I’m using SASE’s forced-reuse cleanup routine on that family root now; it also releases its workspace claim and rebuilds the name registry.

The stale family reservation is gone cleanly. The epic relaunch command has passed that preflight and is still starting its agents; I’m waiting for the command to finish, then I’ll verify the new `sase-8k` rows are live.

Yes—this was stale state left after shutdown. `sase-8k.3--code` was marked active, but its PID no longer existed.

I removed the stale `sase-8k.3` family artifacts and name reservations, then reran:

```bash
sase bead work sase-8k -y
```

The relaunch succeeded:

- `sase-8k.3`: running
- `sase-8k.6`, `.7`, `.8`: waiting on their preceding waves
- `sase-8k.land`: waiting on all phases

All five agents are now present and behaving normally.

---

%xprompts_enabled:true
# New Query

 Can you help me track down and fix any other issues that might have been caused by this shutdown? Also, (and this might be related) can you fix the `toobig_split` chop (see #sshot). Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
