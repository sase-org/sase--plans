---
tier: tale
title: Close retired flag bead sase-z5 to unbreak just check-full
goal:
  just check-full passes on a clean master tree because flag bead sase-z5, whose
  weighted_queue_capacity flag was already removed from the code, is closed.
size: small
proposed_by: bbugyi200.athena.0ij--1
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0ij](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ij.md)
  - [bbugyi200.athena.sase-zl.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-zl.1/README.md)
- **COMMITS:**
  - [e40da3e](https://github.com/sase-org/sase/commit/e40da3e1aa29f8518bb54bae45f2739a71ae2998)
    — feat(monitor): record continuation baseline measurements

# Close Retired Flag Bead sase-z5 To Unbreak `just check-full`

## Problem

`just check-full` fails on a clean master checkout at the `_lint-flags` gate
(`tools/check_feature_flags` rule 8):

```text
rule 8: live flag bead 'sase-z5' has no definition (key 'weighted_queue_capacity');
created 2026-09-10T01:42:39Z by bbugyi200.athena.sase-z4.2 — add the registry
definition or close the bead
```

Root cause: epic sase-z4 phase sase-z4.5 retired the `weighted_queue_capacity` feature
flag — commit `0afe85be4` ("feat(xprompt): complete weighted queue rollout") deleted the
Off branch and removed the registry definition from
`src/sase/feature_flags/registry.py`, making the On branch unconditional on master. The
phase worker was only authorized to close its own phase bead, so it left flag bead
sase-z5 open and recorded on sase-z4.5 (note #2) that land/triage should close sase-z5
or otherwise resolve it "before the orphan grace window expires". The sase-z4 land agent
deferred resolving sase-z5 until after acceptance (sase-z4 note #1), the 24-hour
`ORPHAN_BEAD_GRACE` (`src/sase/feature_flags/orphan.py`) has now expired, and rule 8
escalated its warning to an error, breaking `just check-full` for every agent on a clean
tree.

## Fix

The retirement that bead sase-z5 tracks is already complete on master, so the correct
FlagTriage disposition is **Close** ("the flag was already removed"). No code changes
are needed — do NOT re-add a registry definition for `weighted_queue_capacity`.

1. Close the bead with a resolution note (the default resolution `done` is correct
   because the removal work happened):

   ```bash
   sase bead close sase-z5 --note "Retirement already completed by sase-z4.5: commit 0afe85be4 deleted the Off branch and removed the weighted_queue_capacity registry definition, making the On branch unconditional on master. Closing per sase-z4.5 note #2 because the 24h orphan grace expired and check_feature_flags rule 8 was failing just check-full on clean master. If weighted-capacity acceptance (sase-z4.6.5) later needs a flag again, create a new one with sase flag new."
   ```

2. Verify: re-run `just check-full` through the `/sase_monitor` skill using the
   `TESTING`/`TESTED` status pair (never inline — it routinely outruns one agent turn).
   Every gate must pass before this work is considered done.

## Constraints And Non-Goals

- Do not close or otherwise modify flag beads sase-z6 (`ace_unified_agents`, created by
  sase-xe.16.11.7.6) or sase-z9 (`completion_managed_install_recipe`, created by
  sase-z8.2). Their flags are in-flight: their registry definitions are expected to land
  with their epics' branches, and closing those beads now would make
  `check_feature_flags` rule 7 fail the moment those definitions land. They currently
  produce rule 8 warnings only.
- Known risk: sase-z6 crosses its 24-hour orphan grace at 2026-09-11T05:53Z and sase-z9
  at 2026-09-11T14:39Z. If the verification run fails on rule 8 for either of those
  beads, that is a distinct failure owned by their still-active epics — do not "fix" it
  by closing their beads or hand-adding registry definitions. Report the failure to the
  user instead of treating this work as landed.
- No git-tracked file changes are expected from this plan. If the verification run fails
  for any other reason, report the failure rather than improvising fixes.
