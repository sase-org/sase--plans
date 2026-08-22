---
tier: tale
title: Move final-declaration guidance into generated SASE memory
goal:
  Stop mutating every provider prompt with final-declaration prose while preserving
  reliable declaration submission through always-loaded generated instructions and the
  host recovery turn.
size: medium
proposed_by: bbugyi200.athena.0am
create_time: 2026-08-22 12:21:43
status: wip
---

# Plan: Move final-declaration guidance into generated SASE memory

## Context and confirmed behavior

The primary invocation path currently calls `append_finalizer_end_turn_instructions()`
after resolving a finalizer plan, so the `SASE Final Declaration` block becomes part of
every eligible provider query and saved prompt. The same guidance belongs in SASE's
always-loaded instruction context instead: the packaged `memory-sase.template.md`
generates `sase/memory/sase.md`, which `sase memory init` (and the memory step of bare
`sase init`) inlines into `AGENTS.md` and copies to every provider instruction shim.

The host fallback already has the requested behavior. After a normal provider response,
the finalizer controller publishes a fresh context and, when a required declaration is
missing or stale, spends one bounded recovery turn whose prompt explicitly tells the
agent to use `/sase_final`. The implementation should preserve this mechanism and add a
direct regression assertion for that instruction rather than redesigning it.

## Implementation

1. Move the canonical end-of-turn guidance into generated short-term memory.
   - Add a `SASE Final Declaration` section to
     `src/sase/main/init_memory/templates/memory-sase.template.md`, outside the
     project-name conditional so both project and home instruction roots receive it.
   - Retain the operational contract: use `/sase_final` as the last normal action;
     inspect context and submit when required; do not mutate files or repositories after
     a successful submission; repair validation failures when possible; and exempt the
     intentional plan, monitor, pipe, and question handoffs that terminate mechanically.
   - Keep the generated template as the source of truth. Do not hand-edit generated
     instruction files as an alternative to changing the template.

2. Remove per-query prompt mutation without disturbing finalizer setup.
   - Delete `append_finalizer_end_turn_instructions()` and its public export from
     `src/sase/finalizers/declaration.py`.
   - Remove its import and call from `src/sase/llm_provider/_invoke.py`, while retaining
     finalizer-plan persistence, agent metadata projection, turn-nonce minting, and the
     post-response controller/recovery flow.
   - Ensure the query passed to the provider and written to prompt artifacts remains the
     preprocessed user query, with no appended `SASE Final Declaration` section.

3. Regenerate the explicitly authorized memory outputs.
   - After installing the workspace in editable mode, run `sase memory init --no-commit`
     so the changed packaged template regenerates the canonical `sase/memory/sase.md`,
     then refreshes `AGENTS.md` and the provider instruction shims through the supported
     workflow.
   - Inspect the generated diff to confirm the declaration section appears once in the
     short memory and once in each inlined agent-instruction document, with no unrelated
     generated drift. Do not manually edit those generated outputs.

4. Replace prompt-injection expectations with memory and fallback regressions.
   - Update invocation/finalizer tests that currently assert `/sase_final` was appended
     to provider prompts; assert instead that the provider receives the unchanged
     processed prompt while finalizer plans and metadata are still created and generic
     finalizers still run.
   - Extend memory-initialization coverage to verify the default generated project and
     home `sase/memory/sase.md` notes contain the final-declaration contract and that
     the text is propagated into managed `AGENTS.md`/provider shims.
   - Strengthen the existing missing-declaration recovery test to assert that its one
     recovery prompt explicitly directs the agent to use `/sase_final`, then publishes
     fresh context and accepts a valid submission. Preserve the clean-context and
     intentional-handoff tests proving recovery is not spent unnecessarily.
   - Keep custom `memory.sase_template` behavior intact: it remains a documented full
     template override. A custom template may replace default instructional prose, while
     the host recovery turn remains the correctness fallback whenever a declaration is
     actually required.

5. Align documentation with the new source and fallback model.
   - Update the generated-memory descriptions in `docs/init.md` and `docs/memory.md` to
     include the final-declaration guidance carried by `sase/memory/sase.md`.
   - Update the finalizer lifecycle descriptions in `docs/llms.md`,
     `docs/commit_workflows.md`, and `docs/configuration.md`: normal turns learn the
     terminal action from generated agent instructions, `/sase_final` exits early when
     no payload is required, and the host opens one bounded recovery turn explicitly
     requesting `/sase_final` when a required submission is missing or stale.
   - Remove wording that claims the host injects or asks for `/sase_final` in the
     primary provider prompt.

## Verification

1. Run focused tests for generated memory rendering and propagation, provider invocation
   prompt preservation, finalizer foundation behavior, Codex artifact capture, and the
   declaration recovery channel.
2. Run `sase memory init --check` after regeneration to prove the generated memory and
   agent instruction files are current and idempotent.
3. Run `just check` as the required repository-wide lint and diff-scoped test gate. If
   it escalates or reports unusual selection, use `/sase_monitor` for `just check-full`
   as required by repository instructions.

## Acceptance criteria

- No primary provider query or saved prompt is modified by a hard-coded
  `SASE Final Declaration` append step.
- The default generated `sase/memory/sase.md` contains the complete terminal-action
  contract for `/sase_final`, and initialization propagates it to agent instruction
  files for project and home contexts.
- A missing or stale required declaration still triggers exactly one recovery turn that
  explicitly tells the agent to use `/sase_final`; clean contexts and intentional
  handoffs do not trigger that recovery.
- Finalizer-plan persistence, metadata, nonce handling, submission validation, and
  finalizer execution remain unchanged.
- Generated-memory checks, focused tests, and `just check` pass.
