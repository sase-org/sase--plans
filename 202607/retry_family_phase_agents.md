---
tier: tale
title: Safe retries for family-phase agents
goal: Family-phase retries relaunch from the family base identity, while invalid forced-reuse
  launches fail before any prior agent state or artifacts are removed.
create_time: 2026-07-20 16:37:03
status: wip
prompt: 202607/prompts/retry_family_phase_agents.md
---

# Plan: Safe retries for family-phase agents

## Context

Automatic plan/tale agents are displayed as derived family members such as `sase-8a.3--plan`, but their replayable raw
prompt and `%auto` lifecycle are rooted at the family base name (`sase-8a.3`). The Agents-tab retry and kill-and-edit
paths, including marked-agent kill-and-edit, currently seed editable prompts from the concrete derived row. Mobile retry
likewise allocates from that derived name. Those paths can therefore generate user-facing `%id` values containing the
reserved `--` family separator instead of letting the trusted auto-plan launcher recreate the derived phase. In
addition, the TUI forced-reuse path wipes the requested name and its artifacts before normal launch-name validation
rejects invalid syntax, turning a rejected relaunch into destructive cleanup.

## Implementation

Normalize retry and relaunch seed identities through the existing `agent_family_base` contract at the entry boundaries.
Kill-and-edit prompt preparation should use the family base before applying forced reuse, so both focused and
marked-agent flows inherit the correction while retaining existing clan/name-template handling. Plain Agents-tab retry
and mobile retry should allocate their `.r@` names from the same normalized base, but should continue to record the
concrete source member in provenance fields and rely on the retained `%auto` directive to recreate trusted
`--plan`/`--code` members. Ordinary agents and clan members must keep their current behavior, and user-editable launches
must not gain the internal family-name bypass or relaxed `--` validation.

Refactor the TUI launch-body forced-reuse preparation so all non-mutating prompt parsing and launch-name preflight
needed to reject malformed or reserved family-separator names happens before `wipe_names_for_forced_reuse`. Preserve the
existing post-wipe validation and dispatch behavior for accepted reuse requests, including multi-prompt collection,
history recording, error reporting, and worker-thread execution. A syntax or early prompt-parse rejection must
record/follow the established failure path without calling the wipe hook.

## Tests and validation

Extend the retry-edit tests with derived `--plan` rows to prove kill-and-edit seeds forced reuse of the base name and
plain retry asks the allocator for the base before rewriting the prompt. Cover the shared marked-agent preparation
behavior through its helper contract. Extend mobile lifecycle tests to prove family-phase retries allocate and launch
from the base while preserving the concrete source-agent provenance. Add launch-body regressions showing invalid
forced-reuse family names—and malformed forced-reuse multi-prompts that never become launchable—perform no wipe, while
valid single and multi-prompt reuse still wipes exactly the expected names and launches normally.

Run the focused retry, mobile lifecycle, launch validation, and non-blocking launch-body test modules during
development. After implementation, run `just install` as required for an ephemeral workspace, then `just check` for the
full repository validation.

## Risks

The normalization must distinguish true family-member suffixes via `agent_family_base` rather than splitting arbitrary
hyphenated or clan names. Preflight must not accidentally treat confirmed forced reuse as an ordinary collision before
cleanup; it should reject syntax and malformed requests without weakening the full collision checks that run after
accepted cleanup. Multi-prompt parsing must remain fence/frontmatter aware so names are neither missed nor wiped from
non-launchable content.
