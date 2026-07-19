---
tier: tale
title: Agents-tab fork support for clans and tribes
goal: 'Pressing f on an agent clan or tribe opens a correct #fork prompt, inheriting
  a VCS xprompt workflow only when every real member shares one launch context.

  '
create_time: 2026-07-19 10:55:50
status: done
prompt: 202607/prompts/agents_tab_group_fork.md
---

# Plan: Agents-tab fork support for clans and tribes

## Context and outcome

The Agents tab already binds `f` through `edit_hooks` to `action_fork_agent`, but that action assumes the selection is
one real agent. A clan row is a synthetic `Agent` container with no ordinary prompt name or response path, and a
selected tribe is represented by whole-panel focus, for which `_get_selected_agent()` deliberately returns `None`. The
`#fork` workflow now accepts both bare clan names and `@tribe` references, so the TUI should expose those scopes without
changing the existing single-agent and agent-family behavior.

The completed interaction should prefill `#fork:<clan> ` for a selected clan generation and `#fork:@<tribe> ` for a
selected tribe panel. Active groups are valid because an explicit `#fork` target supplies the deferred wait semantics;
the TUI should not require every member to be done before opening the editor. Untagged/empty panels and stale scopes
should fail with a clear warning rather than falling through to an unrelated row.

## Fork scope and membership

Introduce a small, testable fork-scope representation in the Agents action layer that captures the prompt reference,
user-facing label/history key, and the real agents used for VCS consensus. Resolve scopes in this order:

1. A focused whole panel with a non-empty panel key is a tribe and maps to `@<panel-key>`.
2. A selected synthetic clan container maps to its `agent_clan` name and its matching generation members.
3. Otherwise retain the current real-agent and plan-family resolution, including coder-follow-up selection for
   `PLAN DONE` / `TALE DONE` rows.

Use the already-loaded panel/tree state for membership. Expand clan containers to real agents from the same clan
generation, gather all real agents in the focused tribe (including members nested under clan/family presentation rows),
exclude synthetic summaries, and de-duplicate by identity in stable display order. Snapshot the resolved reference and
member identities when `f` is pressed, then revalidate the scope before opening the prompt if VCS inspection crosses an
asynchronous boundary. Do not add filesystem or resolver work to the Textual event loop.

## Shared VCS workflow inheritance

Extract the existing per-agent smart VCS resolution into reusable logic and add a group consensus helper. For each real
member, resolve the launch VCS tag with the same rules used for an individual fork: raw-prompt parent fallback,
branch/ChangeSpec substitution, `#pr` agent-name substitution, and display-name humanization. Compare the resulting
workflow and ref canonically rather than by incidental spelling such as colon versus parenthesized syntax.

Prepend one representative resolved VCS tag before the group `#fork` call only when every member has a resolvable tag
and all canonical workflow/ref pairs are identical. If any member lacks a VCS workflow, cannot produce the smart ref, or
resolves to a different workflow/ref, omit the VCS prefix entirely. This strict rule also prevents a group whose members
occupy different PR branches from accidentally prefilling one member's branch. Keep any data-scaled raw prompt reads off
the UI thread (or consume an existing safe cache), and return to the UI thread only to show the prefilled prompt after
the captured scope is still valid.

## Command availability and user-facing help

Update Agents-tab command applicability so `edit_hooks` / `f` is available for a valid clan container and for focused
expanded or collapsed tribe panels, in addition to its current real-agent cases. Carry enough focused-panel context into
the pure command predicate to distinguish an actual named tribe from the untagged panel if the existing context booleans
are insufficient. Keep the keymap/default-config source unchanged because `f` is already the registered binding, but
revise command/help wording where needed so discovery describes forking an agent, clan, or tribe rather than only one
chat.

## Verification

Add focused tests around the action/scope helpers and command availability:

- clan rows produce `#fork:<clan>` for both active and completed aggregate states, use only the selected generation's
  real members, and preserve existing family-root/coder-follow-up behavior;
- focused expanded and collapsed named tribe panels produce `#fork:@<tribe>`, while untagged, empty, or stale panel
  focus warns and opens no prompt;
- a clan or tribe whose members all resolve to the same canonical VCS workflow/ref gets that tag before `#fork`;
- mixed workflow types, refs/branches, or tagged-versus-untagged members omit the VCS prefix, including smart `#pr`
  substitutions that diverge by member;
- synthetic clan containers do not count as members, membership is stable and de-duplicated, and command-palette
  availability matches direct-key behavior;
- existing single-agent, running-agent, plan-family, VCS substitution, and wait/fork tests remain green.

Run the narrow Agents action, VCS-resolution, keymap/help, and command availability test modules during development.
Before handoff, run `just install` as required for an ephemeral workspace, then run `just check` to exercise formatting,
lint/type checks, and the complete test suite.
