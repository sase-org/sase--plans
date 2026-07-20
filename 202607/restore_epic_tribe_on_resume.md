---
tier: tale
title: Restore the epic tribe on resumed bead work
goal: 'Every phase and land agent launched by sase bead work remains in the @epic
  tribe, including partial, failed, retried, and join-only epic clan launches.

  '
create_time: 2026-07-20 10:00:55
status: wip
prompt: 202607/prompts/restore_epic_tribe_on_resume.md
---

# Plan: Restore the epic tribe on resumed bead work

## Context and root cause

`sase bead work` correctly renders a new epic clan with one create-only `%clan(<epic>, tribe=epic)` declaration and
renders all later members as `%id(<member>, clan=<epic>)` joiners. On a retry or partial resume,
`launch_epic_bead_work()` sees the durable clan reservation and intentionally renders every remaining phase and the land
agent as a joiner so it does not violate the single-declaration clan lifecycle.

The tribe, however, is currently transported only by the declaring prompt. The runner writes `clan_tribe` only when it
sees `tribe=` on `%clan`, while the per-segment epic metadata carries the plan reference and bead IDs but no tribe. If
the declarer fails before publishing metadata, is relaunched through the declaration-to-join retry rewrite, or is
otherwise absent from the readable clan generation, the durable clan container still supplies membership and generation
but no member supplies `clan_tribe`. Every successful joiner therefore persists `clan_tribe: null`, and ACE correctly
places the clan in the default panel even though the bead-work summary promises `Tribe: @epic`.

The observed `sase-80` state follows this path: the retained clan generation is valid, the remaining `.3`, `.5`, and
`.land` prompts are join-only, and both the earliest surviving member and the resumed members have no clan tribe
metadata. This is launch metadata loss, not an Agents-tab grouping defect.

## Carry the authoritative epic tribe with every segment

Make the epic-work launch contract carry the `epic` clan tribe alongside the existing host-only plan and bead
association fields for every phase segment and the land segment. Consume that value once during runner metadata
extraction, persist it as `clan_tribe` for the member, and remove it from the process environment with the other
launch-only epic fields so nested launches cannot inherit the association accidentally.

Keep prompt semantics unchanged: a fresh clan still has exactly one `%clan` declaration, existing clans remain
join-only, and ordinary user-created clans do not acquire special tribe behavior. The explicit segment metadata should
agree with and harmlessly reinforce the declaring prompt on a fresh launch while also being sufficient on its own for a
retry or partial resume. Preserve existing metadata across runner re-exec using the same durability rules as the other
epic and clan fields.

## Regression coverage

Extend the bead-work rendering and launch tests to prove both phase and land segment environments carry the
authoritative epic tribe without changing the single-declaration/join prompt shapes. Strengthen the partial-completion
and existing-clan cases so their join-only launches are covered explicitly.

Extend runner metadata tests with a join-only epic clan member and assert that it persists `agent_clan`, the existing
clan generation, and `clan_tribe: epic`, and that every host-only epic field is consumed. Retain coverage for fresh
declarations and runner re-exec so prompt-derived and launch-derived metadata cannot regress or leak into child
launches.

Run the focused bead-work, directive/runner metadata, clan launch, and agent environment-hygiene suites first. Then run
`just install` followed by the repository-required `just check`; if the change exposes a Rust-core boundary or shared
clan-domain mismatch during implementation, update the core contract and its tests rather than duplicating backend
behavior in Python, and rerun both repositories' applicable gates.

## Compatibility and risk

This changes only metadata emitted for trusted `sase bead work` phase and land launches. It does not rewrite existing
artifacts retroactively, relax clan name collision rules, permit redeclaration, or alter standalone tribe assignment.
The main regression risks are environment leakage and a prompt-versus-metadata precedence mismatch, so tests should
verify one-shot consumption and consistent `epic` values on both fresh and resumed launches.
