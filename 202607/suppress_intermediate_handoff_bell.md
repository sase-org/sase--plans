---
tier: tale
title: Suppress intermediate agent handoff bells
goal: 'Plan and question handoffs continue into their follow-up phase without emitting
  the provider''s terminal-completion bell, while genuine final agent completions
  and actionable plan or question notifications retain their existing alerts.

  '
create_time: 2026-07-21 16:23:54
status: done
---

- **PROMPT:** [202607/prompts/suppress_intermediate_handoff_bell.md](prompts/suppress_intermediate_handoff_bell.md)

# Plan: Suppress intermediate agent handoff bells

## Context and diagnosis

The previous handled-plan notification fix targeted ACE's unread-notification polling, but the bell that remains at a
`WORKING TALE` transition comes from a different subsystem. Recent real workflow output contains three literal BEL bytes
immediately after the planner is terminated by `sase plan propose`; every byte is emitted by
`run_bam_command("Agent reply received")` during successful LLM postprocessing. The bytes appear between the planner's
`Received SIGTERM` line and the coder prompt, so they are replayed into the same terminal exactly as the approved tale
starts. Comparable plan chains contain one three-bell batch at the intermediate planning handoff and another at true
final completion.

The durable `.sase_plan_pending` marker is written and fsynced before the proposal command sends SIGTERM, and the runner
does not consume it until after the provider invocation and its postprocessing return. The equivalent
`.sase_questions_pending` marker follows the same lifecycle. Those markers therefore provide a deterministic way to
distinguish an intentional continuation handoff from a terminal agent reply without inferring from status text, provider
exit behavior, or notification timing.

## Implementation

Define one shared, read-only pending-handoff classification for the plan and question markers, and use it wherever the
runner currently needs to protect an intentional handoff from ordinary SIGTERM cleanup. Have successful LLM
postprocessing consult that classification before playing the `Agent reply received` sound. A visible invocation with no
pending handoff should retain the existing completion sound; `suppress_output` should retain its current stronger
suppression behavior.

Only the audio side effect should be skipped. Prompt/response logging, chat-history persistence, provider metrics,
handoff normalization, follow-up creation, and final completion notifications must continue unchanged. Keep the
plan-ready and question-ready desktop/tmux alerts, along with ACE's actionable-notification bell, separate: they notify
the user that input is required, whereas the suppressed sound incorrectly announces an intermediate provider exit as if
the whole agent were done.

## Tests and validation

Add focused postprocessing coverage for both pending marker types, proving that each suppresses the bam command while
logging and history behavior remain available. Cover an ordinary visible success with no marker so the final-completion
sound cannot be accidentally removed, and retain coverage for `suppress_output`. Exercise the shared marker predicate
through the runner signal tests so plan and question handoffs remain protected and a normal SIGTERM remains an ordinary
kill.

Run the focused LLM-postprocessing and runner-signal test modules first. Then run `just install` followed by the
repository-required `just check`. As a final regression audit, exercise or inspect a plan-chain handoff and verify the
intermediate segment contains no `Agent reply received` BEL bytes while the terminal coder completion still contains its
normal completion batch.

## Risk controls

Tie suppression to the durable marker in the invocation's own artifacts directory rather than to a global status or
agent name. Treat missing artifacts and marker-read errors as no handoff so unrelated invocations fail open to the
existing sound. Centralizing the marker set prevents provider postprocessing and runner SIGTERM handling from drifting,
and covering both existing handoff types avoids fixing tales while leaving the identical question-continuation defect.
