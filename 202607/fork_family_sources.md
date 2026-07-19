---
tier: tale
title: Family-aware
goal: '#fork resolves agent family names to a dedicated family source and injects
  the full chat transcript of every completed family member, rendered in a clearly
  delimited, attributed, sequential format that downstream agents can consume without
  confusion or duplicated context.

  '
create_time: 2026-07-19 15:13:31
status: wip
prompt: 202607/prompts/fork_family_sources.md
---

# Plan: Family-aware `#fork` sources with full per-member transcripts

## Context and current behavior

The `#fork` xprompt workflow (`src/sase/xprompts/fork.yml`) resolves its `name` arguments in
`src/sase/scripts/agent_chat_from_name.py` and renders the injected history in `src/sase/history/chat_fork.py` (via
`src/sase/scripts/fork_history.py`).

`_resolve_fork_source` in `agent_chat_from_name.py` currently distinguishes three shapes:

1. **Tribe refs** (`@name`) — resolved through the wait-dependency index. `TribeCandidate.kind` is only ever `"agent"`
   or `"clan"` (`src/sase/core/wait_dependency_resolution/_types.py`), so tribes never surface family entities and are
   out of scope here.
2. **Clans** (`find_agent_clan`) — produce a structured `kind="clan"` source carrying every member's
   name/path/artifact_dir. The renderer (`_format_clan_fork_source`) intentionally shows only member prompts plus reply
   statistics, not full replies.
3. **Everything else** — falls through to `kind="agent"`. A family base name lands here: `resolve_resume_agent_name`
   (`src/sase/agent/names/_lookup_resolution.py`) maps it via `most_recent_completed_family_member` to the **newest
   completed member only**, and the injected history is that single transcript flattened by `load_chat_for_resume`.

Consequences of the current fallthrough for family names:

- Only the tip member's conversation is injected with attribution. Earlier members' conversations appear **only if** the
  tip's prompt happens to contain a `#fork:<ancestor>` reference (the plan-accept coder path prepends
  `#fork:<base>--plan` — see `src/sase/axe/run_agent_exec_plan_accept.py` around line 616 — but manual
  `%n(parent, suffix)` attaches do not), and even then the expansion merges all turns into one flat transcript with no
  member boundaries or attribution.
- There is no family analogue of the clan wire shape, so duplicate-parent validation (`_ForkSource.transcripts()`)
  cannot detect `#fork(myfam, myfam--code)` resolving to overlapping transcripts.

`find_agent_family` / `AgentFamily` / `AgentFamilyMember` in `src/sase/agent/names/_lookup_groups.py` already provide
everything needed to enumerate the newest family generation: member names, artifact dirs, timestamps (chain order),
outcomes, and a legacy recovery path for rootless generations.

## Format research summary

Research into how to best present prior conversation transcripts to an agent (Anthropic's long-context and
context-engineering guidance, multi-agent handoff writeups) converges on these principles:

1. **Long context first, live query last.** Transcripts belong near the top of the prompt with the active request at the
   end. The existing `_wrap_fork_history` layout (`# Previous Conversations` … `# New Query`) already does this; keep
   it.
2. **Unambiguous per-document boundaries with metadata.** Each transcript should be wrapped in a clearly delimited block
   carrying source metadata (who produced it, role, outcome, when). The repo's established delimiter system is markdown
   headings plus `**User:**`/`**Assistant:**` turn markers, and the entire downstream parsing pipeline
   (`parse_chat_turns`, `parse_flat_turns`, `extract_previous_conversation_turns` in `src/sase/history/chat_resume.py`)
   consumes exactly that format, including on recursive fork-of-fork loads. Stay with markdown heading structure rather
   than switching to XML-style tags: it keeps nested re-parsing working, matches the existing clan and multi-agent fork
   formats, and honors the uniform-agent-runtimes rule (no provider-specific formatting).
3. **Attribution / narrative casting.** A forked agent reading raw "Assistant" turns can conflate prior agents' actions
   with its own. The guidance block must state explicitly that each transcript is a _named prior agent's_ conversation,
   not the reader's own history, and name which agent said what.
4. **Explicit relationship semantics.** Clans are independent/parallel; families are strictly sequential. The guidance
   must say that members form a chain in chronological order, each member continuing the previous member's work, and
   that the last member represents the family's final state.
5. **No duplicated context.** Dumping overlapping transcripts degrades reasoning. Because family member prompts
   frequently embed `#fork:<ancestor>` refs, naive per-member expansion would inject ancestor conversations twice.
   Deduplicate so each transcript appears exactly once.

## Design

### 1. Resolver: new `kind="family"` fork source (`src/sase/scripts/agent_chat_from_name.py`)

In `_resolve_fork_source`, after the clan branch and before the plain-agent fallthrough, attempt
`find_agent_family(name)`. (`find_agent_family` already returns `None` for explicit member references like `cx--plan`
and for bare names with no family metadata, so explicit-member forks and legacy exact-name forks keep their current
single-agent behavior unchanged.)

When a family is found, build a `_ForkSource` with:

- `kind="family"`, `name=<base>`.
- **Member inclusion policy:** include every member of the newest generation whose outcome is `SUCCESS_OUTCOME` and
  whose `done.json` `response_path` resolves to a readable transcript (validated with the existing
  `_validate_readable_transcript`). Members that are still running, failed, or lack a transcript are _not_ rendered but
  are reported in the wire as excluded entries (name + status) so the renderer can surface the chain's true state. If
  **zero** members qualify, raise the existing `RuntimeError(f"No agent with chat history found for: {name}")` so
  bare-fork fallback/error behavior stays consistent with clans.
- Members ordered by artifact timestamp (chain order — `find_agent_family` already sorts this way).
- `path` compatibility field set to the newest included member's transcript (mirrors the clan source's tip-path seam
  consumed by `fork.yml`'s `path` output).

Extend `_ForkSource`:

- `to_json_data()` gains the family shape:
  `{"kind": "family", "name", "members": [{"name", "path", "artifact_dir", "outcome"}], "excluded": [{"name", "status"}]}`.
- `transcripts()` returns each included member's `(name, path)` pair so the existing atomic duplicate-parent validation
  in `_resolve_agent_chat_sources` catches overlaps such as `#fork(myfam, myfam--code)` or a family plus an agent that
  shares a transcript.

Tribe resolution (`_resolve_tribe_fork_source` / `_fork_source_from_tribe_candidate`) is untouched: the wait index
enrolls tagged family members as individual agent entities, never as family aggregates.

### 2. Renderer: family section format (`src/sase/history/chat_fork.py`)

Accept `"family"` in `_fork_source_kind`, dispatch to a new `_format_family_fork_source`, and make the single-source
path use it too (only the single-_agent_ case keeps the bare `# Previous Conversation` fast path; a lone family source
gets the full structured format).

Target rendering for a family source (heading depths shown for the multi-source case; the section plugs into the
existing `## Source N of M` scheme):

```markdown
## Source 1 of 1 — agent family `cx`

- **Members shown:** 2 of 3 (sequential chain, oldest first)
- **Not shown:** `cx--fix` (running)

Family members ran as one sequential chain: each member continued the previous member's work, and the last member
reflects the family's final state. These are transcripts of prior agents' conversations, not your own — attribute
decisions to the named member when it matters.

### Member 1 of 2 — agent `cx--plan`

- **Outcome:** `completed` · **Model:** `anthropic/claude-fable-5` · **Launch:** `20260718010101`
- **Transcript:** `~/.sase/…/planner-chat.md`

**User:**

<full sanitized prompt>

**Assistant:**

<full reply>

### Member 2 of 2 — agent `cx--code`

- **Outcome:** `completed` · …

**User:** …
```

Implementation notes:

- Per-member metadata reuses the helpers `_format_clan_member` already uses: `format_metadata_model` plus
  `agent_meta.json`/`done.json` reads keyed off `artifact_dir`. Unlike clans, the member body is the **full transcript**
  rendered through `load_chat_for_resume` (sanitized turns, recursive external-fork expansion) — no reply omission and
  no truncation, per the feature's purpose.
- **Deduplication:** render members oldest-first and thread one shared `visited` set (the canonical absolute paths of
  _all_ included member transcripts, plus paths already expanded) into each `load_chat_for_resume` call — the function
  already accepts `_visited` and silently strips refs whose target was already loaded. This guarantees a member's
  `#fork:<ancestor>` reference cannot re-inject a transcript that is already present as an earlier member section, while
  refs to _outside_ conversations still expand inline in the earliest member that cites them.
- **Guidance text:** the top-level guidance paragraph built in `build_fork_injected_history` needs a family-aware
  variant. Multi-source wording currently claims every source is independent; adjust so it distinguishes "Source
  sections are independent parents" from "members inside a family section are sequential". Keep the existing closing
  rule that the New Query takes precedence.

### 3. Workflow definition

`src/sase/xprompts/fork.yml` needs no changes: `sources_json` is an opaque JSON pass-through and the `type: agent` input
already completes family base names (the `%wait:`/`#fork:` completer covers agent/family/clan/tribe targets).

## Testing

- `tests/test_agent_chat_from_name.py`
  - Family base name resolves to a `kind="family"` source containing every completed member in chain order, with `path`
    pointing at the tip transcript; explicit member names (`cx--plan`) still resolve as single-agent sources.
  - A family whose tip is running yields the completed prefix as members plus an excluded entry.
  - A family with no completed members raises the standard no-chat-history error.
  - Duplicate validation: `#fork(cx, cx--code)` fails with the same-transcript error.
  - The legacy rootless-generation recovery shape still resolves.
- `tests/test_fork_history.py`
  - Family block renders full member replies (contrast with the clan test asserting omission), metadata lines,
    oldest-first ordering, and the sequential-chain guidance.
  - Dedup: a child transcript containing `#fork:<parent-member>` does not duplicate the parent's turns; a ref to an
    unrelated outside conversation still expands.
  - Mixed parents (family + agent, family + clan) render under the multi-source scheme with corrected guidance wording.
- `tests/test_fork_workflow.py` — extend end-to-end coverage if it exercises source kinds through the workflow steps.
- Run `just check` before finishing.

## Risks and edge cases

- **Prompt size.** Full transcripts for long families can be large. This is the explicit intent of the feature; the
  format's "Members shown" header and per-member transcript paths give the consumer agent what it needs to skip or
  re-read selectively. No truncation logic in this change.
- **Behavior change.** `#fork <family>` output grows from one flattened transcript to a structured multi-member block.
  The single-agent and clan paths are untouched, and explicit member names keep the old behavior, which bounds the blast
  radius.
- **Fork-of-fork re-parsing.** The injected block is later re-parsed by
  `extract_previous_conversation_turns`/`parse_flat_turns` when a descendant forks the forked agent. The member sections
  keep `**User:**`/`**Assistant:**` markers under the `# Previous Conversations` region, which those parsers already
  flatten; add a regression test only if the existing fork workflow test surfaces a gap.
- **Member metadata gaps.** `agent_meta.json`/`done.json` may be partial for old artifacts; all metadata reads must
  degrade to `unknown` exactly as `_format_clan_member` does.
