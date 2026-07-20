---
tier: tale
title: Add bead keyword support to wait directives
goal: Parse, validate, expose, and safely rewrite repeatable bead conditions on %wait
  without changing existing agent, time, runner, bare-wait, or template semantics.
create_time: 2026-07-20 11:24:45
status: done
prompt: 202607/prompts/wait_bead_kwarg.md
---

# Plan: Add bead keyword support to wait directives

## Context and scope

Phase `sase-87.2` introduces the directive-layer representation needed by later phases of the bead-gated wait epic. This
phase is deliberately limited to parsing and prompt editing: it does not persist bead waits into runner metadata,
resolve bead status, render ACE waiting state, or emit bead waits from `sase bead work`.

The current `%wait` implementation separates raw match collection, semantic value resolution, the public
`PromptDirectives` model, multi-prompt bare-wait handling, and canonical prompt editing. The change should preserve that
separation so downstream phases can consume a stable `wait_beads` field without coupling directive parsing to the bead
store.

## Directive collection and validation

- Extend `_CollectedDirectives` in `src/sase/xprompt/_directive_collect.py` with raw bead keyword values and accept
  `bead` alongside `time` and `runners` in the `%wait(...)` named-argument allowlist. Preserve existing duplicate-key
  rejection within one occurrence and collect `bead=` values across multiple `%wait` occurrences in source order. Update
  the unsupported-key diagnostic so it names all three supported keywords.
- Add a focused resolver in `src/sase/xprompt/_directive_values.py` that requires every `bead=` value to be a non-empty,
  whitespace-free token, raises a clear `DirectiveError` that identifies `%wait(bead=...)` for malformed values, and
  performs order-preserving deduplication.
- Call the resolver from `src/sase/xprompt/_directive_extract.py` and expose its result as `wait_beads: list[str]` on
  `PromptDirectives` in `src/sase/xprompt/_directive_types.py`, including the public field documentation. Keep bead
  values independent of positional agent-name expansion and template lookup.

## Bare-wait and template isolation

- Preserve `%wait(bead=<id>)` and `%w(bead=<id>)` as bead-only occurrences: they must not synthesize a positional empty
  argument, look up the most recently named agent, or become a bare wait during multi-prompt preflight/rewrite.
- Verify mixed occurrences still behave compositionally: positional agents continue through tribe/template validation;
  `time=` and `runners=` retain their existing rules; bead values accumulate and deduplicate separately. Add regression
  assertions around `has_bare_wait_directive` and `rewrite_bare_wait_directives` so future parser changes cannot
  accidentally rewrite bead-only waits.

## Directive editing round-trip

- Extend `PromptWaitDirective` in `src/sase/xprompt/directive_edit.py` with an ordered bead collection, include it in
  truthiness, and serialize each entry as a `bead=` keyword in the canonical `%wait(...)` form while retaining agent,
  time, and runner ordering.
- Cover canonical editing with mixed agent/time/runner/bead values and bead-only values, then parse the rewritten prompt
  to prove that `wait_beads` round-trips without loss or accidental agent dependencies.

## Tests and validation

- Extend the focused directive tests under `tests/test_directives_wait.py` for accepted bead-only and mixed forms,
  repetition across occurrences, order-preserving deduplication, aliases, malformed empty/whitespace values, updated
  unsupported-key guidance, and separation from agent-name templates.
- Extend `tests/test_multi_prompt_reference_directives.py` or the existing multi-prompt wait tests with direct bead-only
  bare-wait detection and rewrite regressions.
- Extend `tests/test_directive_edit.py` for bead-only and mixed canonical serialization and extraction round-trips.
- Run the focused pytest files first, then run the repository-mandated `just check` after ensuring the workspace
  dependencies are installed with `just install`.

## Acceptance criteria

- Extracting repeated `%wait(bead=...)` directives yields one ordered, deduplicated `PromptDirectives.wait_beads` list
  and leaves `PromptDirectives.wait` unchanged unless positional agents were supplied.
- Empty or whitespace-containing bead tokens fail with actionable directive errors, while unsupported keywords report
  `bead`, `runners`, and `time` as the supported set.
- Bead-only waits neither resolve nor rewrite as bare agent waits and do not trigger agent-name template resolution.
- Prompt editing preserves bead conditions and produces a form that extracts back to the same agents, beads, time, and
  runner threshold.
- Focused tests and the full `just check` suite pass.
