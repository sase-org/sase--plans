---
tier: tale
title: Make initial research swarm members explicitly independent
goal: "Each initial researcher knows its peer exists and is explicitly instructed not to
  seek or read that peer's work, while the lead continues to synthesize both reports.

  "
size: small
proposed_by: bbugyi200.athena.0gj.f0
create_time: 2026-09-05 18:57:54
status: wip
---

# Explicit independence for the two initial research swarm members

## Goal and scope

Give each of the two initial members of `#research_swarm` its own explicit independence
instruction. Each member must know which other researcher is working on the same request
and must not attempt to read that researcher's report. The lead continues to read both
reports and consolidate their findings.

This is a `tale` with `size: small`: one coding agent can make a focused prompt and
documentation change in one linked plugin repository and verify it with the existing
expansion machinery and checks.

Open the implementation repository before reading or editing it, and follow its
`AGENTS.md` instructions. Use only the path this command prints:

```bash
sase repo open sase-research-artifacts -r "Implement explicit research swarm independence"
```

The intended changed files, relative to that repository, are:

- `src/sase_research_artifacts/xprompts/research_swarm.md`
- `docs/xprompts.md`

## Current behavior and design choice

The swarm has four segments: `cdx` using `@research_a`, `cld` using `@research_b`,
`final` using `@xlarge`, and `image`. The first pair receive the research request and
`#research(suffix=a)` / `#research(suffix=b)`. The lead waits on both researchers, reads
their transcripts to find their reports, reads both reports, adds research, and
preserves their suffixes when organizing the consolidated output.

The swarm description and lead prompt call the pair independent, but neither initial
segment tells its recipient that its peer exists or forbids reading the peer's report.
Separate filenames prevent the pair from choosing the same output path; they do not
communicate the intended separation of research.

Put the instruction directly in both initial swarm segments. Independence belongs to
these roles, while the generic `#research` input `suffix` is a naming constraint and
does not imply that another researcher exists. Keep the generic xprompt unchanged. Two
short explicit paragraphs are sufficient; no helper xprompt or new input is needed.

The restriction concerns the peer's work **from this swarm**, not every existing
research file ending in `__a.md` or `__b.md`. Researchers choose their own stems, and
other swarms reuse the same suffixes, so do not invent a peer's exact report path or
instruct agents to search for all files bearing the peer's suffix.

## Implementation

### 1. Give each initial segment peer context and a direct prohibition

In `src/sase_research_artifacts/xprompts/research_swarm.md`, place a prose block after
each initial member's scheduling directives and before its existing
`{{ prompt }} #research(suffix=...)` content. It must render unconditionally, outside
the optional `wait` and `priority` Jinja conditionals.

Use this identity paragraph in the `cdx` segment:

> You are researcher A in a two-researcher swarm. The other researcher,
> `research.{@1}.cld`, is independently investigating the same request and will write a
> report ending in `__b.md`. Your report must end in `__a.md`.

Use the symmetric paragraph in the `cld` segment:

> You are researcher B in a two-researcher swarm. The other researcher,
> `research.{@1}.cdx`, is independently investigating the same request and will write a
> report ending in `__a.md`. Your report must end in `__b.md`.

Follow each identity paragraph with the same instruction:

> Conduct your research independently and form your own conclusions. Do NOT attempt to
> locate, open, read, or otherwise consult the other researcher's report from this
> swarm, even if it becomes available before you finish. Do not obtain that peer's
> findings indirectly through its chat transcript, summaries, or requests to the peer.
> You may independently use the same external sources, shared input material, and
> unrelated prior research. You may check filenames or file existence to avoid
> overwriting your own output, but do not inspect the peer's report contents. If you
> encounter its filename, leave the report alone. The lead researcher will read both
> reports and synthesize their findings after you have both finished.

Keep peer IDs as plain prose identifiers using the existing `{@1}` marker, not artifact
citations, fork instructions, new waits, or file includes. The message itself must not
import the peer's work or create a dependency on it. Preserve the existing topic and
`#research(suffix=a)` / `#research(suffix=b)` lines so their expansion contracts remain
intact. The duplicated instruction should be identical in the two segments.

Leave the lead and image segments unchanged. In particular, the lead must retain its
instruction to read both transcripts and both reports, and its suffix-preserving report
organization steps.

### 2. Document the two independent roles

In `docs/xprompts.md`, update the initial researcher descriptions under
`#research_swarm` and add a short explanation that each researcher is told about its
peer but must not seek or read the peer's report or obtain its findings through the
peer's transcript. Clarify that this applies to the current swarm and that combining the
two reports is the lead's responsibility. Retain the current inputs, model assignments,
suffix descriptions, scheduling behavior, and final report layout.

## Verification

Use the existing tests; this prose change does not require new tests that lock in
sentence wording. `tests/test_xprompt_loading.py` already checks segment boundaries,
typed inputs, scheduling, distinct suffixes across two dispatches, lead dependencies,
and the generic research input branches.

From the opened plugin repository, run:

```bash
just check
git diff --check
```

The plugin's `just check` owns environment setup and runs ruff, mypy, and the default
pytest suite. Use it instead of starting an ad hoc `uv run pytest` environment. Its
setup can build the local Rust binding; use `/sase_monitor` for a long-running check or
build rather than repeatedly polling it. No primary SASE source files change, so the
primary repository's check lane is not part of this implementation.

Also inspect rendered prompts with the plugin environment and the same public APIs used
by its tests: `load_xprompts_from_plugins`, `expand_single_xprompt`, and
`split_segments_protecting_fences`. Check default arguments and combined optional `wait`
/ `priority` arguments. Use `expand_xprompt_swarms_with_metadata` for two identical
dispatches, as in the existing distinct-suffix test, to verify peer IDs receive the
matching dispatch's marker. These are one-off inspection checks; they must not launch
research agents or read real research reports.

Confirm all of the following:

- Four segments still render, with the original topic in both initial segments.
- Each initial segment identifies the opposite peer and correct own/peer suffixes, and
  contains the full no-read instruction regardless of optional arguments.
- Each dispatch's prose peer references use its own clan marker, not the other
  dispatch's marker.
- The peer context is ordinary prose and introduces no peer wait or automatic
  transcript/report inclusion. Only the existing lead waits on both researchers.
- Neither lead nor image receives the new restriction; the lead still explicitly reads
  both reports and transcripts.
- Existing suffix naming, collision handling, generic `#research` behavior, and the
  final layout remain unchanged.
- The final diff contains only the two intended plugin files.

## Limits and completion criteria

This is an explicit agent instruction, not a filesystem access-control mechanism. It
closes the missing prompt contract without changing runtime permissions, the research
provider, or core backend behavior. There are no new CLI options, feature flags, memory
changes, packaging changes, or manual changelog edits.

The implementation is complete when both researchers receive the correctly scoped peer
context and prohibition, the lead retains its synthesis role, documentation matches the
behavior, and the verification above passes.
