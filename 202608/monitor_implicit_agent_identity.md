---
tier: tale
title: Pin implicit monitor starts to the calling agent
goal:
  Implicit monitor starts inherit the exact caller's family context and workspace
  without selecting unrelated artifacts.
size: medium
proposed_by: bbugyi200.athena.sase-ll
bead: sase-ll
create_time: 2026-08-15 15:35:44
status: wip
---

- **BEAD:**
  [sase-ll](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ll/README.md)

# Pin implicit monitor starts to the calling agent

## Problem

`sase monitor start` currently derives its implicit target by parsing `SASE_AGENT_NAME`
as an agent-family member name. That is unsafe because SASE bead and phase names can end
in syntax that the legacy family parser also recognizes. For example, a phase agent
named `sase-m6.6.1.5` is collapsed to `sase-m6.6.1`. Resolving that broader lane selects
its newest matching artifact, which can be a sibling phase, the epic land agent, or a
settled monitor. The start can therefore fail promotion with `FamilyAttachError`,
inherit the wrong workspace, and build a follow-up from the wrong conversation.

The implicit CLI contract is that the calling agent is the target. A durable monitor
lane may still be an existing family, but it must be derived from the exact caller's
artifact metadata rather than guessed from the spelling of the caller's name.

## Implementation

1. Separate exact caller identity from family-lane identity in the monitor store/start
   APIs. Preserve the existing family-oriented default used by monitor inspection and
   stop operations, while giving monitor start a way to read the exact `SASE_AGENT_NAME`
   without applying `agent_family_base()`.
2. Resolve an implicit start against that exact caller artifact first. Derive the
   durable lane from the selected artifact's `agent_family` metadata when it already
   belongs to a family, or from the exact caller name when it is still a bare agent. Use
   the durable lane consistently for the per-lane start lock, replay/conflict detection,
   request fingerprint, suffix allocation, and created monitor identity, while retaining
   the exact selected artifact as the source of workspace, claim, model, and follow-up
   metadata. Keep explicit `--agent` requests and the host epic launch path able to
   target an existing lane as they do today.
3. Update the CLI start handler so its implicit target and default cwd come from the
   exact caller, and clarify internal request/helper documentation where needed. Do not
   require callers to pass `--agent`/`--lane`; the documented ergonomic path must work.
4. Add regressions covering both ambiguous identity shapes reported on `sase-ll`: a
   numeric epic-phase name whose apparent suffix must not select a sibling/land
   artifact, and an existing family member whose broader family has a newer settled
   monitor. Assert that the monitor inherits the caller's numbered workspace and
   metadata, is attached to the correct durable family, and neither promotes nor
   launches from the unrelated newest artifact. Retain coverage for explicit family
   targets and default stop/inspection behavior.

## Verification

Run the focused monitor store, monitor start, and CLI handler test modules containing
the new regressions. Then run `just install` as required for an ephemeral workspace and
`just check`. If the scoped selector escalates or reports unusual coverage, use the
repository's monitored `just check-full` workflow before closing the bead.

Close `sase-ll` only after the regressions demonstrate that implicit starts preserve the
exact caller's workspace and family context and the repository checks pass.
