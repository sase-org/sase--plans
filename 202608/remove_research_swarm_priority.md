---
tier: tale
title: Remove research swarm priority specifications
goal:
  All research swarm agents use SASE's implicit queue priority without changing their
  dependency graph.
size: small
proposed_by: bbugyi200.athena.09e
create_time: 2026-08-21 09:36:05
status: wip
---

# Plan: Remove research swarm priority specifications

## Goal

Update the plugin-owned `#research_swarm` xprompt so its four agents no longer emit
runner-queue priority specifications and therefore use SASE's implicit queue priority.
Treat this as removal of the swarm's priority override feature, rather than replacement
of `20` with another explicit value: remove the `priority` input as well as every
`%wait(priority=...)` directive it feeds.

The implementation belongs in the linked `sase-research-artifacts` repository. Open it
with
`sase repo open sase-research-artifacts -r "Implement the approved research swarm priority-removal plan"`
and use the path printed by that command for all reads and writes. Follow that
repository's `AGENTS.md`.

## Implementation

1. In `src/sase_research_artifacts/xprompts/research_swarm.md`, remove the typed
   `priority` input and remove the standalone `%wait(priority={{ priority }})` directive
   from each of the `cdx`, `cld`, `final`, and `image` segments. Preserve the four-way
   swarm structure, model aliases, clan/agent IDs, optional external `wait` behavior on
   the two researchers, the lead's waits on both researchers, and the image agent's
   wait/fork relationship with the lead.
2. In `tests/test_xprompt_loading.py`, change the declared-input expectation back to
   only `prompt` and `wait`; remove tests for priority defaults and caller overrides;
   and strengthen the remaining expansion tests to assert that no segment contains a
   `priority=` clause while all optional and inter-agent dependency waits still expand
   correctly. Keep the deterministic report-target and four-segment dependency-graph
   coverage intact.
3. In `docs/xprompts.md` and `README.md`, remove the `priority` input and all claims
   that swarm members accept or honor a runner-queue priority. Retain the documented
   four-agent workflow and optional `wait` semantics. Do not add a hand-written
   changelog entry; this repository's release automation owns `CHANGELOG.md`.

## Validation

From the opened `sase-research-artifacts` repository:

1. Run `just test tests/test_xprompt_loading.py` to exercise the public plugin loader,
   frontmatter contract, segment expansion, and dependency graph directly.
2. Search the edited public contract with
   `rg -n 'priority|wait\(priority' src/sase_research_artifacts/xprompts/research_swarm.md tests/test_xprompt_loading.py docs/xprompts.md README.md`
   and confirm there are no stale swarm-priority inputs, directives, assertions, or
   documentation claims.
3. Run `just check` for the repository's complete lint, type-check, and regular test
   gate. The wheel-only lane is not required because no package manifest, entry point,
   or resource inclusion contract changes.

## Acceptance criteria

- Expanding `#research_swarm` with or without `wait` produces four segments and none
  contains `priority=`.
- `#research_swarm` no longer declares or documents a `priority` argument.
- Optional external waits still gate only `cdx` and `cld`; `final` still depends on both
  researchers; `image` still waits on and forks from `final`.
- Focused tests and `just check` pass in `sase-research-artifacts`.
