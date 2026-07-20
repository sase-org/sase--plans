---
tier: tale
title: Fix agent-family fork transcript resolution
goal: 'Forking a completed agent family injects every available family-member conversation
  exactly once, including real plan-to-code handoffs whose root done marker summarizes
  the final child.

  '
create_time: 2026-07-19 21:53:44
status: done
prompt: 202607/prompts/fix_family_fork_transcripts.md
---

# Plan: Fix agent-family fork transcript resolution

## Context and root cause

The built-in `#fork:<family>` workflow expands a family into ordered member sources in
`src/sase/scripts/agent_chat_from_name.py`. The resolver currently requires every included member to have a successful
`done.json` and takes that marker's `response_path` as the member transcript. That assumption does not match the
plan-chain artifact lifecycle:

- A planner or question member saves its own completed handoff conversation in `agent_meta.json` as `chat_path` before
  the next family member starts.
- Intermediate members may not receive their own `done.json` at all.
- Finalization writes the final child's aggregate `done.json` to both the final artifact and the family root so the root
  reflects the whole family's terminal state.

Consequently, a normal plan-to-code family can expose the coder response through both the planner root and coder
artifact. The duplicate-source validation then correctly observes the same canonical path twice but rejects a family
produced by SASE itself. Explicitly forking the planner child also resolves to the coder transcript because the generic
lookup prefers the propagated root done marker. The existing family fixtures hide this mismatch by giving every member a
member-specific successful done marker.

## Member-owned transcript resolution

Introduce one clear resolver policy for sequential family members while leaving ordinary agents and clans on their
existing compatibility paths:

- Treat a readable metadata `chat_path` as the authoritative transcript for a member that completed a handoff, even when
  that member has no done marker or its root artifact carries a later aggregate marker.
- Use a successful, readable `done.json.response_path` for the terminal family member and as a compatibility fallback
  for legacy family artifacts that do not record a metadata chat path.
- Do not attribute a propagated done response to the wrong named member. Keep failed, running, missing-transcript, and
  unreadable-transcript exclusions meaningful, and retain the duplicate canonical-path guard for genuinely overlapping
  sources.
- Apply the same member-aware precedence to explicit family-child references, so `#fork:<family>--plan` selects the
  planner handoff while `#fork:<family>--code` selects the final coder conversation. Preserve the historical
  done-before-meta behavior for non-family agents.

Keep the existing family wire format and oldest-first history rendering intact; the fix should correct source ownership
rather than weaken validation or dedupe away legitimate family members.

## Regression coverage

Replace or extend the resolver fixtures with artifact layouts that mirror the real handoff lifecycle:

- A two-member plan/code family whose root metadata points to the planner chat, whose root and coder done markers both
  summarize the coder, and whose base reference resolves to distinct planner and coder sources.
- Explicit planner and coder references from that layout, proving each child resolves to its own conversation.
- An intermediate handoff member with a readable metadata chat and no done marker, proving completed sequential context
  is retained instead of reported as running.
- Legacy done-only family members and genuine duplicate aliases, proving the compatibility fallback and safety rejection
  remain in force.
- Workflow-level `#fork:<family>` expansion proving planner and coder replies are each injected once, in chain order,
  from the realistic propagated-root layout.

Run the focused resolver, fork-history, and fork-workflow tests first. Then run `just install` followed by the
repository-required `just check`, addressing any task-owned failures and documenting only genuinely unrelated blockers
if the full check cannot complete.

## Risks and boundaries

The main risk is mistaking a live artifact's metadata for a completed handoff. Limit metadata precedence to recognized
family-member artifacts and rely on the fact that the runner persists `chat_path` only after saving that member's
handoff conversation. Do not change family naming, wait semantics, aggregate done-marker propagation, history
formatting, or the clan fork contract as part of this fix.
