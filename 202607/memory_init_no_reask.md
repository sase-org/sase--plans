---
tier: tale
title: Memory-update requests must include sase memory init without re-asking
goal: 'When the user explicitly asks an agent to update a SASE memory file, the agent
  completes the workflow by running `sase memory init` without asking for separate
  permission, while unrequested memory edits remain forbidden.

  '
create_time: 2026-07-18 19:14:01
status: wip
prompt: 202607/prompts/memory_init_no_reask.md
---

# Plan: Make explicit memory-update requests mandate `sase memory init` without re-asking

## Problem

SASE agents ask the user for permission to "initialize sase memory" far too often. The typical annoying flow: the user
explicitly asks an agent to update a memory note under `sase/memory/`, the agent makes the requested edit, and then it
stops and asks a follow-up question like "may I run `sase memory init`?" before regenerating the derived instruction
files.

The root cause is the blanket header that every generated agent instruction file starts with. The template at
`src/sase/amd/templates/AGENTS.template.md` renders this line at the top of every managed `AGENTS.md` and every provider
shim (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`), for every project root and the home root:

> IMPORTANT: You should not modify any of these memory files without approval from the user.

Because `sase memory init` rewrites `AGENTS.md`, the provider shims, and `sase/memory/README.md`, agents treat running
it as "modifying memory files" and ask for a second approval — even though the user's original request to update the
memory note already implies the regeneration step. In the sase repo specifically, the "Memory File Edits Require
Explicit User Permission" entry in `sase/memory/gotchas.md` amplifies the hesitation with its blanket "NEVER" phrasing.

The desired policy, stated by the user: **if the user explicitly asks for a SASE memory file update, the agent MUST run
`sase memory init` afterwards as part of that same request — no separate permission question.** Asking first remains
correct only when the user has _not_ explicitly requested a memory change.

## Design

Encode the policy in the one place every agent already reads — the generated instruction-file header — plus the two
supporting agent-facing documents, then regenerate this repo's derived files through the normal `sase memory init`
mechanism. No Python behavior changes are needed; `sase memory init` itself has no interactive confirmation, so this is
purely an instruction-content change. This does not cross the Rust core backend boundary: the templates and memory notes
are agent-facing instruction content owned by this repo.

### 1. Extend the managed `AGENTS.md` template header

In `src/sase/amd/templates/AGENTS.template.md`, extend the `IMPORTANT:` header paragraph so it both keeps the existing
protection and adds the mandate. Suggested wording (the implementer may polish it, keeping both halves):

> IMPORTANT: You should not modify any of these memory files without approval from the user. However, when the user
> explicitly asks you to update a SASE memory file, that request already carries the required approval for the full
> workflow: make the requested edit to the canonical note under `sase/memory/`, then you MUST run `sase memory init` to
> regenerate `AGENTS.md`, the provider instruction shims, and the memory README. Do NOT ask for separate permission to
> initialize sase memory in that case.

`src/sase/amd/templates/AGENTS.minimal.template.md` contains no such header line; verify it needs no change and leave it
alone.

### 2. Reinforce the mandate in the memory README template

In `src/sase/main/init_memory/templates/memory-README.template.md`, extend the `sase memory init` bullet in the
"Commands" section with a short clause stating that after any user-requested memory note edit, running
`sase memory init` is a mandatory follow-through that needs no additional permission.

### 3. Align the sase repo's gotcha entry

Update the "Memory File Edits Require Explicit User Permission" entry in `sase/memory/gotchas.md` (the canonical short
note, not any generated file) by appending a sentence such as:

> When the user HAS explicitly requested a memory file update in the current conversation, completing it by running
> `sase memory init` to regenerate the derived instruction files is mandatory and requires no additional permission — do
> not ask again.

Authorization note for the implementing agent: this specific edit is not an agent-invented memory change. The user's
originating request that produced this plan explicitly asked for this policy ("If I explicitly ask them to update a sase
memory file then they MUST initialize sase memory"), and approval of this plan is the user's explicit grant for exactly
this entry edit. Do not make any other memory-note edits beyond what this section describes.

### 4. Regenerate derived files via `sase memory init`

After the template and gotcha edits, run `just install` (ephemeral workspaces may have stale venvs) and then
`.venv/bin/sase memory init` from the repo root so `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, and
`sase/memory/README.md` are regenerated from the updated templates and notes. Never hand-edit those generated files; the
editable install ensures the workspace's updated templates are the ones rendered. This regeneration also keeps any
`sase memory init --check` drift validation green.

### 5. Update tests that assert the header text

`tests/main/test_init_memory_agent_docs.py::_assert_derived_managed_agents` asserts the exact rendered header prefix
(the `startswith` block containing "You should not modify any of these memory files without approval from the user.")
and must be updated to match the new template text. Search the test suite for any other assertion that embeds fragments
of the old header sentence and update those in lockstep.

## Propagation

No fleet-wide manual action is needed. Other roots (the home/chezmoi root and other projects' repos) pick up the new
header automatically the next time `sase init` / `sase memory init` runs there, which already happens via the sase
post-commit hook once this change is installed.

## Testing

- Targeted: `.venv/bin/pytest tests/main/test_init_memory_agent_docs.py` plus any other tests found asserting the header
  text.
- Manual verification: confirm the regenerated `AGENTS.md` and `CLAUDE.md` in this repo contain the new mandate
  sentence, that all provider shims are byte-for-byte copies of `AGENTS.md`, and that the updated gotcha entry appears
  in the regenerated Tier 1 section.
- Repo gate: run `just check` before finishing, per repo policy.

## Risks

- The new wording must not weaken the existing protection: the carve-out applies only when the user explicitly requested
  a memory update. Keep the "do not modify without approval" sentence intact and scope the mandate narrowly.
- Exact-text test assertions will fail if the template and tests drift; update them together.
- The `gotchas.md` edit touches a protected memory note; it is scoped and user-authorized as described above, and no
  other memory notes may be touched.
