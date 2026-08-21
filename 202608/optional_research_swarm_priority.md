---
tier: tale
title: Add an opt-in priority to
goal:
  Let callers optionally set one runner-queue priority across all four research swarm
  agents while omission preserves SASE's implicit priority.
size: small
proposed_by: bbugyi200.athena.09e.f0
create_time: 2026-08-21 11:08:49
status: wip
---

# Plan: Add an opt-in priority to `#research_swarm`

## Goal

Add a nullable `priority` input to the plugin-owned `#research_swarm` xprompt. When a
caller supplies the input, render that exact runner-queue priority onto all four swarm
agents (`cdx`, `cld`, `final`, and `image`). When the caller omits it, render no
`%wait(priority=...)` directive, leaving every agent on SASE's implicit queue priority.

This is intentionally different from the former priority input, whose default of `20`
made the override unconditional. The new input must default to YAML `null`, and its
Jinja guard must check `priority is not none` rather than truthiness so the valid
explicit value `0` is not mistaken for omission.

The implementation belongs in the linked `sase-research-artifacts` repository. Open it
with:

```bash
sase repo open sase-research-artifacts -r "Implement the approved opt-in research swarm priority plan"
```

## Implementation

1. Update `src/sase_research_artifacts/xprompts/research_swarm.md`:
   - Add a typed `int` input named `priority` with `default: null` and documentation
     explaining that it applies to every swarm member, lower numbers start first, and
     omission retains SASE's implicit priority.
   - In each of the four top-level segments, conditionally render exactly one
     `%wait(priority={{ priority }})` only when `priority is not none`.
   - Keep the existing `wait` input independent: it must continue to add the external
     agent dependency only to `cdx` and `cld`; the lead must still wait on both
     researchers, and the image agent must still wait on and fork from the lead.
   - Do not introduce a replacement default such as `10` or `20`; omission must be
     represented by the absence of the priority directive.

2. Update `tests/test_xprompt_loading.py` to lock down the input and rendering contract:
   - Expect `priority` as the third typed input, with `int` type and a `None` default.
   - Preserve the existing four-segment identity, model, report-target, dependency, and
     fork assertions.
   - Prove that omitting `priority` leaves all four expanded segments free of
     `priority=` and of unresolved Jinja syntax.
   - Prove that a supplied ordinary value (for example `5`) renders exactly one
     `%wait(priority=5)` in each segment.
   - Add explicit coverage for `priority=0` so the conditional cannot regress to a
     truthiness check.
   - Prove that `priority` composes with `wait`: all four agents receive the priority,
     while only the two researchers receive the caller-provided dependency and the
     internal lead/image dependency graph remains unchanged.

3. Update the user-facing contract in `README.md` and `docs/xprompts.md`:
   - List `priority` as an optional integer with no default override.
   - State that a supplied value applies to all four agents and that lower values start
     first.
   - State clearly that omission uses SASE's implicit queue priority, while `wait`
     continues to gate only the two researchers.

## Validation

From the opened `sase-research-artifacts` repository:

1. Run `just install` if the linked checkout's development environment is not current.
2. Run `just test tests/test_xprompt_loading.py` for the focused input, conditional
   rendering, zero-value, and dependency-graph coverage.
3. Search the xprompt, tests, and docs for stale claims that priority defaults to `20`
   or is always emitted.
4. Run `just check` for the repository's full lint, type-check, and normal test gates.

## Non-goals

- Do not change SASE's global implicit queue-priority value or `%wait` validation.
- Do not change model aliases, clan/tribe identity, deterministic report targets,
  segment ordering, or the research artifact layout.
- Do not add priority to only a subset of the swarm; an explicit value applies to all
  four agents.
