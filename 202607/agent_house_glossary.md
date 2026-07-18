---
tier: tale
title: Add "Agent House" term to the SASE glossary
goal: 'The SASE glossary memory defines "Agent House" as the generic term covering
  both standalone agents and agent families, and every generated agent instruction
  file carries the new entry.

  '
create_time: 2026-07-18 18:49:08
status: wip
prompt: 202607/prompts/agent_house_glossary.md
---

# Plan: Add "Agent House" term to the SASE glossary

## Context

SASE docs and UI copy frequently need to talk about "an agent or an agent family" as a single concept. This comes up
constantly because creating an agent family hijacks the original agent's name: the first `%n(parent, suffix)` attachment
renames the original agent with its own suffix and reserves the bare family name as a pure container. As a result, a
bare name can denote either a plain agent or a family, and prose that says "agent" usually means both. The user wants a
glossary term — **Agent House** — that names this union so future writing does not have to spell out "agent or agent
family" every time.

Authorization note: memory-file edits require explicit user permission. The user personally requested this exact
glossary addition and approves this plan through the PlanApproval gate; that gate response is the user's explicit grant
for this specific edit to `sase/memory/glossary.md`. No other memory files may be touched.

## Change

Edit `sase/memory/glossary.md` only. Insert one new entry, keeping the file's existing conventions:

- **Placement:** alphabetical, between the existing **Agent Hoods** and **Agent Instruction Files (aka agents.md
  files)** entries.
- **Format:** bold term on its own line ending with two trailing spaces (hard line break), followed by a short prose
  definition wrapped to the file's existing ~120-column width.

Proposed entry text (the implementing agent may polish wording but must preserve the meaning):

> **Agent House** An agent house is a generic term for either a standalone agent or an agent family. Because creating an
> agent family hijacks the original agent's name (the bare name is reserved as the family container), a bare name can
> denote either a plain agent or a family; "agent house" covers both without spelling out "agent or agent family" each
> time.

Scope guardrails:

- The term covers agents and agent families only — it does not include agent clans, hoods, or tribes.
- This is a documentation-only change: no code, keymap, or configuration changes are needed, and no new CLI surface is
  introduced.

## Regenerating the instruction-file shims

The top-level agent instruction files (`AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`) are generated
from the memories in `sase/memory/` by `sase init`. Never hand-edit the shims. After editing the glossary memory, run
`sase init` from the repo root so the shims pick up the new entry; the post-commit hook runs it as well, but running it
before verification keeps `sase validate` (part of `just check`) green.

## Verification

1. `just install` — required first in a possibly-stale workspace.
2. `sase init` — regenerate the instruction-file shims from the updated memory.
3. `just check` — must pass; this covers the prettier markdown format check (`fmt-md-check`), `sase validate` (SASE
   initialization consistency), and the test suite.
4. Spot-check that the new **Agent House** entry appears identically in `sase/memory/glossary.md` and the generated
   `AGENTS.md`/`CLAUDE.md`, placed between **Agent Hoods** and **Agent Instruction Files**.

## Risks

- Prettier may rewrap the inserted paragraph; run `just fmt` (or `just fmt-md`) and accept its formatting rather than
  fighting it.
- If `sase init` produces shim diffs beyond the glossary entry (stale shims from unrelated memory drift), keep them —
  the generator is the source of truth — but mention the extra diff in the final report.
