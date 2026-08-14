---
tier: tale
title: Consume pooled model aliases once per real agent invocation
goal:
  Make default and explicit pooled-alias launches rotate their user-visible models
  fairly while keeping root, step, and chat metadata aligned with the model actually
  invoked.
size: medium
proposed_by: bbugyi200.athena.00i.f0
create_time: 2026-08-14 07:29:20
status: wip
---

# Plan: Consume pooled model aliases once per real agent invocation

## Problem

The shipped `@default` alias delegates to the two-member `@smarter` pool, but recent
no-`%model` launches do not alternate their user-visible model. The latest completed
launch artifacts show the same pattern repeatedly: the top-level `ace-run` metadata
records `claude/opus`, while `prompt_step_main.json` and the inner workflow chat record
`codex/gpt-5.6-sol`, which is the model that produced the answer.

The composed launch path advances the same pool twice:

1. `extract_directives_and_write_meta()` resolves with `consume=True` while preparing
   top-level runner metadata.
2. `run_execution_loop()` wraps the prompt in an anonymous workflow whose prompt step
   calls `invoke_agent()`, which resolves with `consume=True` again.

With a two-member pool, those two advances return the cursor to its starting position on
every SASE launch. The existing tests exercise metadata resolution and direct
`invoke_agent()` calls separately, so both lanes pass while the real runner composition
keeps selecting the same user-visible member.

## Implementation

1. Make a real provider invocation the only authoritative consumption boundary. Factor
   the provider/model/effort result into a small typed launch-selection value that can
   be resolved once and then passed through without re-resolving an alias. Top-level
   runner metadata preparation should perform a non-consuming preview; the prompt-step
   execution path should consume the pool under the existing machine-wide lock
   immediately before the provider call. Direct `invoke_agent()` callers should continue
   to resolve and consume internally when they do not supply a pre-resolved selection.

2. Teach `WorkflowExecutor` and `invoke_agent()` to share that authoritative selection.
   Replace the prompt step's current non-consuming display lookup followed by a second
   consuming invocation lookup with one consuming resolution whose concrete provider,
   model, effective effort, effort explicitness, and alias provenance are reused for
   both the step marker and the provider call. Keep internal handoff data out of Jinja
   workflow inputs and serialized user context. Each actual prompt step should consume
   once; script-only workflows and display/validation/doctor paths should consume zero
   times.

3. Reconcile launch metadata with the authoritative selection before invoking the
   provider. Update the root `agent_meta.json` for the primary anonymous prompt step and
   write the same provider/model/effort/alias values to `prompt_step_*.json` and chat
   metadata, so `sase agent list`, the ACE row, and saved transcripts all identify the
   model that actually answered. Preserve the separate requested-versus-execution
   provider behavior used by `SASE_LLM_EXEC_PROVIDER`, and retain the existing
   best-effort semantics for alias/config-derived effort versus explicit `%effort`.

4. Preserve phase and retry semantics. Runner re-exec/resume must not consume during
   metadata reconstruction; an actual retried provider call must follow the existing
   retry-model rules. Plan-feedback, question, and accepted-plan coder handoffs should
   use their own generated model directives and artifacts rather than accidentally
   inheriting a stale initial selection. Existing explicit wrapper `%model` behavior for
   flattened workflows must remain intact.

5. Add composed regression coverage rather than only resolver-level tests. Exercise the
   runner metadata plus anonymous-workflow prompt-step path against an available
   two-member pool and assert that:
   - two consecutive no-`%model` launches invoke opposite `@smarter` members and move
     the cursor once per launch;
   - a direct `%model:@smarter` launch and a no-directive `@default` launch share the
     pool owner's cursor;
   - root metadata, prompt-step markers, and chat metadata match the concrete model
     passed to the provider;
   - concurrent launches receive serialized selections without duplicate cursor
     consumption;
   - runner metadata re-exec, preview/display resolution, and a workflow with no agent
     step do not advance the cursor.

   Keep the focused alias-state and direct-invocation tests, adjusting assertions that
   currently describe metadata preparation as the authoritative consuming lane.

6. Update `docs/llms.md` to state that round-robin accounting advances once per real LLM
   invocation, that `@default` and direct/delegating references share the pool-owning
   alias cursor, and that launch/step metadata reflects the consumed selection.

## Verification

Run the focused load-balancing, invocation, runner-metadata, workflow-executor, and
chat-metadata tests that cover the changed paths. Then run `just install` followed by
`just check`. If scoped selection escalates because the shared invocation or workflow
modules are broadening paths, run `just check-full` through `/sase_monitor` as required
by the repository instructions.
