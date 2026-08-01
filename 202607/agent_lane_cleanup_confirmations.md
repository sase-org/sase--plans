---
tier: tale
title: Agent-lane labels in cleanup confirmations
goal: Every TUI confirmation for killing or dismissing agents identifies the affected
  work by agent lane, while a running sequential-family member kill also identifies
  the exact member, without changing cleanup scope or execution behavior.
create_time: 2026-07-24 17:59:16
status: done
---

- **PROMPT:** [prompts/202607/agent_lane_cleanup_confirmations.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/agent_lane_cleanup_confirmations.md)
- **AGENTS:**
  - [bbugyi200.athena.jp](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.jp/README.md)
  - [bbugyi200.athena.jp--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.jp.md#member-code)
- **COMMITS:**
  - [339e06f](https://github.com/sase-org/sase/commit/339e06f651ca5268f340a4910646dc19ff491006) — fix(ace): summarize cleanup confirmations by agent lane

# Plan: Agent-lane labels in cleanup confirmations

## Context and behavior contract

The Agents tab now treats a standalone agent or an entire sequential agent family as one agent lane, but the
kill/dismiss confirmation builders still enumerate concrete cleanup targets. Planner-backed cleanup expands workflow
parents into their steps, and sequential families may contain several historical members, so a panel with only a few
lanes can produce a long confirmation containing internal workflow-step and completed-family-member names.

Keep the concrete target expansion intact for cleanup planning, counts, cascading, optimistic removal, persistence, and
notification dismissal. Change only the confirmation projection:

- Show one stable, presentation-ready lane name for each affected lane, in the same first-seen order as the concrete
  targets.
- Treat a standalone agent, a top-level workflow and its internal steps, and each direct clan member as their respective
  lanes.
- Treat every member of a sequential agent family as one lane using the family reference name, including plan-workflow
  roots, rename-on-attach roots, family members nested in clans, and machine-qualified presented names.
- Never expose workflow-step names or completed sequential-family member names in dismiss listings.
- When a concrete running target is a sequential-family member, show both the lane name and that exact presented
  family-member name so the destructive action is unambiguous. Do not add a redundant member label when the lane and
  concrete names are identical.
- Deduplicate repeated/cascaded concrete targets within each Kill or Dismiss section without changing the underlying
  lists passed to the confirmed action. A lane may still appear in both sections when the operation kills its active
  member and dismisses earlier completed members.
- Retain useful scope/action headers and concrete cleanup counts unless they themselves contain per-target names.
  Non-agent confirmations, including AXE background-command termination, remain unchanged.

## Implementation

1. Add a pure TUI presentation helper near the Agents kill/cleanup actions that resolves arbitrary concrete `Agent` rows
   against the loaded Agents tree and returns ordered confirmation entries.
   - Reuse the model's presented family/clan identity methods and parent/tree metadata rather than parsing display
     strings.
   - Resolve workflow descendants to their owning lane, sequential-family rows to their family lane, and clan
     descendants to the direct lane inside the clan rather than to the clan container.
   - Provide a single formatting path for standalone lane entries and the running-family-member exception, with
     defensive fallbacks for incomplete legacy rows or missing parents.
2. Route every agent kill/dismiss confirmation builder through the shared projection.
   - Update bulk marked, focused panel, group, clan, tribe, custom-selection, and planner-backed cleanup confirmations
     that converge on `_present_bulk_kill_modal`.
   - Update the separate focused-panel/global dismiss-all and kill-and-dismiss-all builders.
   - Update focused running-agent kill, focused kill-and-edit, marked kill-and-edit, and wait-relaunch kill
     confirmations so their subjects use the same lane/member contract while preserving action-specific context such as
     the replacement wait specification.
   - Keep the concrete `killable` and `dismissable` collections and callbacks byte-for-byte equivalent in meaning; only
     construct modal subject text from the lane projection.
   - Do not alter `ConfirmDialog` styling/parsing or reuse the agent helper for the AXE background-command
     `ConfirmKillModal` call site.
3. Add focused regression coverage for the projection and all distinct confirmation paths.
   - Cover standalone lanes, top-level workflows with loaded hidden steps, completed sequential families, an actively
     killed family member, family lanes nested in clans, presented machine-qualified names, stable
     ordering/deduplication, and a lane represented in both Kill and Dismiss sections.
   - Update existing panel-, tribe-, clan-, group-, marked-, dismiss-all-, kill-all-, focused-kill-, kill-and-edit-, and
     wait-relaunch assertions so they require lane-only names and reject leaked workflow/family descendants.
   - Add or update a representative ACE PNG modal snapshot modeled on the reported tribe-panel case, proving that many
     concrete cleanup targets render as the small lane roster and that the dialog remains readable.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral SASE workspace.
2. Run the focused unit/integration tests for agent cleanup confirmations, panel scoping, clan/tribe cleanup,
   family-member relaunch, marked kill-and-edit, wait relaunch, and confirm dialog behavior.
3. Run `just test-visual` and inspect any intentional updated PNG snapshot plus failure artifacts under
   `.pytest_cache/sase-visual/`.
4. Run the mandatory full `just check`.
5. Re-read the final call-site search for `ConfirmKillModal`, `ConfirmKillAllModal`, and `ConfirmDismissAllModal` to
   verify that every agent kill/dismiss subject builder uses the shared lane projection and that only non-agent
   confirmations remain outside it.
