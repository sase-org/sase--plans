---
tier: tale
title: Add configurable runner capacity to the research swarm
goal: Every research-swarm agent uses a configurable runner limit that defaults to 16.
size: small
proposed_by: bbugyi200.athena.0hz
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0hz](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0hz.md)
  - [bbugyi200.athena.sase-z2.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-z2.1/README.md)
  - [bbugyi200.athena.sase-z2.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-z2.3/README.md)
  - [bbugyi200.athena.sase-z2.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-z2.4/README.md)
  - [bbugyi200.athena.sase-z3.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-z3.2/README.md)
- **COMMITS:**
  - [d6b1163](https://github.com/sase-org/sase/commit/d6b116360301dda75ed92c63ca61eb64b2c4c717)
    — fix(bead): publish epic plan archives after launch
  - [b8ac9f3](https://github.com/sase-org/sase/commit/b8ac9f39290d48a3710277541c438a0cd154d107)
    — feat(beads): add plan archive doctor repair
  - [93de142](https://github.com/sase-org/sase/commit/93de1427708a8fc8b6b1bcb02d1ee944ab57bed1)
    — fix(beads): skip invalid plan archive sources
  - [0dc3536](https://github.com/sase-org/sase/commit/0dc3536782a38da976262be332cc603a0e915b3f)
    — feat(ace): adopt equals model shortcuts

# Add configurable runner capacity to `#research_swarm`

## Goal

Give every agent launched by the `#research_swarm` xprompt an explicit runner-queue
capacity, defaulting to `16`, while preserving the swarm's existing wait dependencies,
optional priority behavior, and positional argument compatibility.

The canonical xprompt is owned by the `sase-research-artifacts` plugin. No change is
needed in the main `sase` repository or the `chezmoi` configuration: SASE already
supports the `runners` field on `%queue`, and the plugin already uses `%queue` for its
optional priority.

## Implementation

1. Update `src/sase_research_artifacts/xprompts/research_swarm.md` in the
   `sase-research-artifacts` repository:
   - Add a typed `runners` integer input with a default of `16` and a description that
     explains it controls runner-queue admission for every swarm member.
   - Append the new input after the existing `priority` input so existing positional
     invocations continue to bind their third argument to `priority`.
   - Replace each segment's priority-only conditional queue rendering with exactly one
     `%queue(...)` directive that always sets `runners={{ runners }}` and adds
     `priority={{ priority }}` only when priority was supplied. Apply this uniformly to
     the `cdx`, `cld`, `final`, and `image` agents without disturbing their `%wait`,
     `%clan`, `%id`, model, or fork directives.

2. Extend `tests/test_xprompt_loading.py` in `sase-research-artifacts` to lock down the
   new contract:
   - Assert the input remains typed and ordered as `prompt`, `wait`, `priority`, then
     `runners`, with `runners` defaulting to the integer `16`.
   - Update the raw four-segment directive assertions to require the runners-aware queue
     template on every segment.
   - Assert omission renders exactly one `%queue(runners=16)` on each agent and no
     priority, while an explicit `runners` value overrides `16` on all four agents.
   - Retain coverage for explicit and zero priority, adapting it to expect a single
     combined queue directive such as `%queue(runners=16, priority=5)`.
   - Retain the wait-composition checks so the two researchers still honor the optional
     external wait and the lead/image dependency graph remains unchanged when queue
     arguments are present.

3. Synchronize the public contract in `README.md` and `docs/xprompts.md`:
   - Add `runners` to the input documentation as an integer defaulting to `16`.
   - Explain that it applies to all four swarm agents and may be overridden per swarm
     invocation, while `priority` remains optional and SASE's implicit priority is used
     when it is omitted.

## Verification

From the `sase-research-artifacts` repository:

1. Run `just test tests/test_xprompt_loading.py` to exercise input parsing, default and
   overridden runner rendering, optional priority composition, and the existing swarm
   dependency graph.
2. Run `just check` to execute the plugin's complete lint, type-check, and non-wheel
   test suite.

The wheel resource inventory is unchanged, so the slow `just test-wheel` packaging lane
is not required for this content-only update.
