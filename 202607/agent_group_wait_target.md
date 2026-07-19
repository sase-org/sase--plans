---
tier: tale
title: Agent clan and tribe wait targets
goal: 'The Agents tab can start a new prompt waiting on a selected clan or named tribe,
  with the same selection safety, VCS-context handling, and UI discoverability as
  clan and tribe forks.

  '
create_time: 2026-07-19 14:09:50
status: wip
prompt: 202607/prompts/agent_group_wait_target.md
---

# Plan: Agent clan and tribe wait targets

## Context and behavior contract

Clan and tribe references are already valid wait dependencies in the launch backend: a clan name waits on the applicable
clan generation, while an `@tribe` reference uses tribe wait semantics. The Agents-tab action that prefills a new prompt
with `%w:<target>` does not expose those capabilities. It currently checks marked agents and otherwise calls
`_get_selected_agent()`, so a selected clan container has no usable agent name and a focused tribe panel reports
`No agent selected`.

The default keymap calls this wait-for action with uppercase `W` through the shared `add_tag` action; lowercase `w` is a
different action that edits the wait dependencies/time of one existing STARTING, WAITING, or RUNNING agent. Implement
clan and tribe targeting for the wait-for action that is analogous to `f`, without turning lowercase `w` into a bulk
mutation of every group member.

The interaction contract should be:

- Preserve marked-agent precedence. If marks exist, `W` continues to create the existing comma-separated wait directive
  from the marked set.
- With no marks, a selected real agent or family continues to produce its canonical prompt name, a selected clan
  produces `%w:<clan>`, and a focused named tribe panel produces `%w:@<tribe>` in both collapsed and expanded panel
  states.
- Reject untagged panels, empty/stale clan or tribe scopes, and in-panel grouping banners with precise warnings instead
  of falling back to a remembered row.
- Preserve the current display/history context conventions, using labels such as `wait(builders)` and `wait(@builders)`
  and stable group-oriented history keys.

## Shared target scope and prompt preparation

Generalize the immutable selection snapshot introduced for clan/tribe forks into a prompt-target scope that both fork
and wait-for actions can consume. Keep one definition of real-member filtering, clan-generation membership, tribe-panel
membership, stable identity ordering, and scope comparison rather than implementing a second, subtly different group
resolver. The scope must retain the prompt reference, user-facing label, history key, member identities, and the member
data needed for VCS-context resolution.

Route `action_wait_for_agent` through this scope resolver after its existing marked-agent branch. Build the wait prefill
from the resolved reference rather than requiring an individual `agent_name`. For group targets, inherit a VCS workflow
tag only when every real, de-duplicated member resolves to the same canonical workflow/ref, matching the strict
consensus behavior of group forks; otherwise emit only the wait directive. Preserve the established smart VCS behavior
for a single agent/family and the existing marked-set behavior.

Any raw prompt reads, VCS parsing, or other disk-backed context resolution must run outside Textual's event loop using
the existing worker pattern. Take an immutable scope/agent-list snapshot at keypress, then re-resolve and compare the
current selection before opening the prompt bar so navigation during the worker cannot apply a result to a different
clan or tribe. Handle worker scheduling failures with a user-visible error and no leaked coroutine. Factor the common
fork/wait prompt preparation only as far as it keeps action-specific directive formatting and messages clear.

## Availability and discoverability

Expose the contextual wait-for action consistently wherever clan and tribe forks are currently advertised:

- Update command metadata so the shared `add_tag` command is represented on the Agents tab as add-tag on ChangeSpecs and
  wait-for-agent/clan/tribe on Agents, with useful wait aliases.
- Add an Agents-tab availability predicate that honors marked-set precedence, allows named clan containers and named
  tribe panels, rejects untagged panels and generic grouping banners, and retains valid single-agent/family targets.
- Add footer labels for `wait for clan` and `wait for tribe` alongside the existing fork labels, without exposing a
  hidden remembered agent when a whole panel is selected. Keep marked-set labels/precedence coherent.
- Update the Agents help text to describe agent, marked-set, clan, and tribe wait targets and retain the explicit
  distinction between `W` (start a new prompt with a wait) and `w` (edit one current agent's wait state).

## Tests and verification

Extend the focused Agents action tests to cover active and completed clan containers, selected-generation membership,
collapsed and expanded named tribe panels, exact `%w:<clan>`/`%w:@<tribe>` prefills, unanimous and mixed VCS contexts,
synthetic-row filtering, marked-agent precedence, invalid/untagged scopes, and selection revalidation after asynchronous
preparation. Retain the existing single-agent, family-root, and marked-set expectations as regression coverage.

Add command-palette availability/dispatch and keybinding-footer assertions for named clans and tribes, including
negative cases for generic group banners and untagged panels. Update affected Agents PNG snapshots when the new footer
affordances are intentional. Run the focused wait/fork, command availability, footer, and visual snapshot suites during
development, then run `just check` for the required full repository verification.

No backend wait-resolution or Rust-core change is expected: clan and tribe wait dependency semantics already exist and
are covered independently. If implementation discovers a mismatch between the prompt reference emitted by the TUI and
those existing resolvers, fix and test the shared backend contract instead of adding TUI-only interpretation.
