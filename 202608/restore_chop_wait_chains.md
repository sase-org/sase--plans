---
tier: tale
title: Restore sequential AXE chop agent wait chains
goal:
  Chained chop proposals launch through typed admission with durable named-agent waits,
  including correct relinking across admission-time skips.
size: medium
proposed_by: bbugyi200.athena.0d5
---

# Plan: Restore sequential AXE chop agent wait chains

## Problem and root cause

The `toobig_split` chop is still emitting a correct linear `wait_on` chain, and the
ordinary chop planner still renders each dependency as `%wait:<full-agent-name>`. The
regression begins when an active `%if` routes the batch through typed admission:

- the typed planner converts an in-batch named wait into a logical-unit edge;
- that edge becomes satisfied when the predecessor reaches a terminal admission state,
  including `launched` (the process was spawned), rather than when the agent finishes;
- AXE dispatch rebuilds the agent prompt with `agent_unit_dispatch_prompt()`, which
  intentionally omits typed waits because generic admission normally owns them.

Consequently, typed admission preserves proposal launch order but removes the runner's
named-agent dependency. The admitted agents all start without `waiting_for` metadata and
can do work concurrently. The live `toobig-40` clan demonstrates this exact state.

This is an AXE chop adaptation bug, not a generic Rust admission bug. Generic typed
launches and mixed Agent/Proc plans should retain their current semantics; the fix
belongs in the Python AXE dispatch boundary that already owns chop-specific routing,
environment, clan promotion, and launch records.

## Implementation

1. Preserve the chop dependency in durable unit dispatch metadata.
   - In `src/sase/axe/chop_proposal_launch.py`, associate each accepted proposal's
     effective `wait_on` reference with the predecessor's typed logical unit while
     building `unit_dispatch_metadata`.
   - Derive this from the already filtered/relinked `PreparedChopProposal` set so
     once-per removals continue to use their existing effective predecessor.
   - Keep enough proposal-index/ID information to derive the same relationship for an
     older in-flight AXE bundle that lacks the new convenience field.

2. Restore a named-agent wait when an AXE Agent unit is dispatched.
   - In `src/sase/axe/chop_typed_admission.py`, add one shared resolver that starts at
     the unit's structured chop predecessor and consults durable admission unit receipts
     for the actual launched identity.
   - If `%if` skipped or condition-errored an intermediate unit (or another terminal
     outcome produced no launch identity), walk that unit's own predecessor link until
     the nearest successfully launched ancestor is found. If none exists, dispatch
     without a named wait. Guard malformed/missing mappings and cycles explicitly
     instead of emitting a wait for a nonexistent agent.
   - Use the existing logical edge to guarantee that each predecessor has reached a
     terminal admission state before this resolution. Then add exactly one
     `%wait:<actual-agent-name>` directive to the rebuilt AXE prompt while preserving
     workspace qualification, `%id`/`%clan` promotion, model/effort, `%auto`,
     finalizers, and runner-capacity directives.
   - Resolve from the durable bundle rather than closure-only state so a detached
     coordinator restart produces the same chain and never duplicates a wait.

3. Keep AXE launch records aligned with the prompt that actually ran.
   - Reuse the effective-wait resolver for immediate launch callbacks and detached
     lifecycle reconstruction, recording the relinked `wait_on`/`wait_name` rather than
     the statically skipped member.
   - Leave proposal previews deterministic before admission; after runtime filtering,
     ensure persisted launch rows expose the actual predecessor identity used by the
     agent runner.

4. Document the two-layer AXE behavior.
   - Update `docs/axe.md` to state that a structured `wait_on` serves both as an
     admission-order edge and, for an admitted Agent, as a restored named-agent wait;
     admission-time skips relink to the nearest launched ancestor.
   - Update the typed-admission overview in `docs/architecture.md` to distinguish this
     chop-specific behavior from generic logical waits, which remain consumed by the
     coordinator.

## Regression coverage

- Extend `tests/test_axe_chop_proposal_launch_clan_dispatch.py` with a typed,
  `toobig_split`-shaped multi-member chain and assert that the first launched member has
  no named wait while each later member's captured prompt waits for the actual previous
  launched identity.
- Cover a skipped middle member and multiple leading skips: the next survivor must wait
  for the nearest launched ancestor, or have no wait when no ancestor launched. Assert
  that it never waits for the skipped member's planned name.
- Exercise a fresh dispatcher/coordinator view of the same durable bundle so restart
  recovery reconstructs the same effective wait.
- Add non-clan coverage for structured chop `wait_on` so the repair is protocol-wide,
  not hard-coded to `toobig-*` clans.
- Extend typed lifecycle reconciliation coverage to verify persisted launch descriptors
  report the effective `wait_on` and `wait_name` after a skip.
- Keep the generic typed-launch assertions that dispatch prompts omit waits unchanged,
  proving the behavior is scoped to AXE chop proposals.

## Verification and acceptance

Run `just install` before repository checks, then run the focused AXE proposal,
admission, lifecycle, and typed-identity tests followed by `just check`. Use
`just check-full` through `/sase_monitor` only if the scoped selector escalates or the
change reaches a broadening surface.

The change is complete when:

- every admitted member of a structured chop chain after the first records a named wait
  for the nearest earlier member that actually launched;
- skipped members allocate no agent and leave no phantom wait target;
- members can still be launched up front and remain visible as queued/waiting while
  their predecessors work;
- clan declarer promotion, project routing, once-per relinking, runner-capacity waits,
  and detached admission recovery retain their current behavior; and
- no Rust-core or `bugyi-chops` change is required.
