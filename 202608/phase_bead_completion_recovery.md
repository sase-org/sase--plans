---
tier: tale
title: Recover unfinished phase agents before finalization
goal: "A phase-bound agent cannot be recorded as successfully completed while its
  assigned phase bead is still open: it gets one bounded chance to finish, close, or
  hand off, and otherwise fails loudly with durable diagnostics.

  "
size: medium
proposed_by: bbugyi200.athena.0br
create_time: 2026-09-09 20:00:17
status: wip
---

# Plan: Recover unfinished phase agents before finalization

## Context and root cause

The `sase-s9.3` run was associated with `phase_bead_id: sase-s9.3` and returned while
that bead was still `in_progress`. Its `just install` command exceeded the provider's
foreground tool timeout and continued as a provider-native background task. The
provider's scheduled-wakeup attempt then failed, and the normal model turn ended before
`just check`, `sase bead epic-symbols sase-s9.3`, or `sase bead close sase-s9.3` ran.

The dirty repository correctly triggered SASE's declaration-recovery turn. That turn is
intentionally restricted to `/sase_final`, so it committed the implementation but could
not resume verification or close the bead. Phase beads are intentionally excluded from
commit-time auto-close, and the runner currently has no invariant connecting a
successful provider return to the assigned phase's terminal state. The finalizer
therefore succeeded and the agent was recorded as `DONE` with an open phase. Another
actor later closed the bead; this plan changes the runtime behavior, not that historical
state.

## Behavioral contract

Add a provider-neutral completion check for runs whose `agent_meta.json` declares a
`phase_bead_id`:

1. Immediately after the ordinary provider turn returns, but before host finalizers run,
   resolve the phase from the exact workspace-bound bead store recorded in agent
   metadata.
2. Continue normally when the phase is already closed or the run has an intentional
   pending handoff marker.
3. When the phase is still open, invoke exactly one focused completion-recovery turn
   using the same provider, model, invocation options, and artifact directory. Tell the
   recovery agent to inspect the preceding live reply and phase state, finish only the
   missing verification, run the epic-symbol check, and close the phase with an honest
   verification note. If work still requires a long command or user input, it must use
   the SASE monitor/questions/plan handoff rather than provider-native background or
   wakeup tools. It must not submit a final declaration; finalization follows after this
   guard succeeds.
4. After recovery, proceed when the phase is closed, and preserve an intentional handoff
   when one was created. If the phase remains open, is missing, or its status cannot be
   proved from the workspace store, raise a typed completion error so the run is marked
   failed instead of `DONE`.
5. Persist the recovery prompt and response in the run artifact directory, merge the
   recovery response and usage into the invocation result, and treat an existing
   recovery-prompt artifact as an exhausted one-shot budget. This makes failures
   diagnosable and prevents workflow retries from silently granting unlimited recovery
   turns.

This guard must not auto-close a phase or invent verification evidence. Task-bead commit
auto-close remains unchanged, as do normal runs without `phase_bead_id` and all
intentional plan, monitor, pipe, or question handoffs.

## Implementation

- Add a focused agent-runtime module for phase completion recovery. Keep metadata and
  workspace-store resolution, recovery artifact persistence, prompt construction,
  one-shot accounting, status revalidation, and the typed failure in that module.
- Reuse the existing pending-handoff detector and response/usage merge helpers. Resolve
  the bead through the workspace directory and number in `agent_meta.json`, not through
  a potentially stale primary-store projection or a subprocess call.
- Insert the guard in the shared LLM invocation pipeline between the normal
  `provider.invoke(...)` result and `run_finalizers(...)`. This ordering lets a phase
  close participate in the subsequent finalizer dirty-state snapshot and keeps the
  behavior uniform across Claude, Codex, Grok, and other runtimes.
- Keep declaration recovery separate. Its narrow commit-declaration contract remains
  unchanged; the new guard establishes phase completion before that protocol starts.

## Tests and verification

- Add unit coverage for no phase association, an already-closed phase, and every
  intentional handoff bypass.
- Cover an open phase that closes during recovery, including prompt/response artifacts
  and merged token usage.
- Cover an open phase that creates a handoff, an open phase that remains open, an
  unavailable or missing workspace bead, and a retry after the one-shot recovery budget
  has already been spent. Assert that unresolved cases fail rather than appearing
  successful.
- Add an invocation-pipeline ordering test proving completion recovery runs before
  finalizers and that ordinary non-phase invocations retain their current path.
- Run focused tests for the new recovery module and finalizer declaration behavior, then
  run `just install` followed by `just check` as required for repository changes.

## Non-goals

- Do not make phase beads eligible for commit-time auto-close.
- Do not add provider-specific handling for Claude background tasks or scheduled-wakeup
  tools.
- Do not change the historical `sase-s9.3` close event or broaden this work to parent
  epic/land-agent completion semantics.
