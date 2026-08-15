---
tier: tale
title: Fix child epic launches from clanned phase agents
goal:
  Clanned epic phase agents can launch approved child epics on their own monitor lane
  without resolving to a sibling phase.
size: small
proposed_by: bbugyi200.athena.01y
create_time: 2026-08-15 05:31:17
status: wip
---

- **PROMPT:**
  [prompts/202608/fix_child_epic_clan_lane.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/fix_child_epic_clan_lane.md)

# Fix child epic launches from clanned phase agents

## Goal

Allow an epic phase agent such as `sase-m6.6` to propose an approved child epic and
start its host-owned epic-launch monitor without confusing the phase with another member
of its parent epic clan.

## Root cause

The epic-launch monitor derives its lane from the proposing agent's metadata. When the
agent has no `agent_family`, it passes the agent's name through `agent_family_base()`.
That parser intentionally supports historical dotted numeric family members, so it
interprets the current clan member name `sase-m6.6` as legacy family member `.6` and
returns `sase-m6`. Monitor lane resolution then matches every `sase-m6.<phase>` record
and chooses the newest sibling, which was `sase-m6.10` in the reported failure.
Promotion consequently tries to turn `sase-m6.10` into the `sase-m6` family and rejects
the mismatched identity.

The artifact metadata already contains the disambiguating fact: current rows carry
`agent_clan`, while legacy parallel-family rows carry `agent_family_parallel: true`.
That structured group metadata must take precedence over ambiguous dotted-name syntax.

## Implementation

1. Update the epic-launch lane derivation in `src/sase/bead/epic_launch.py` so a
   proposing agent that belongs to a clan uses its exact agent name as the monitor lane.
   Preserve an explicit, non-parallel `agent_family` as the authoritative lane for
   genuine family members, retain the historical name-parser fallback for metadata that
   has neither clan nor family identity, and recognize legacy parallel-family metadata
   as clan-equivalent.
2. Add focused regression coverage in `tests/test_bead/test_epic_launch.py` that models
   `sase-m6.6` inside clan `sase-m6` and asserts the child-epic monitor is attached to
   lane `sase-m6.6`, not the clan prefix. Cover both current `agent_clan` metadata and
   legacy `agent_family_parallel` metadata, while retaining coverage that an explicit
   genuine family continues to resolve to its family lane.

## Verification

Run the focused epic-launch tests, including the new clan/family disambiguation cases.
Then run `just install` followed by the repository-required `just check` gate and
confirm the diff contains only the intended lane-resolution and regression-test changes.
