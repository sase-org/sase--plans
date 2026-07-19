---
tier: epic
title: Finish the agent-ID and clan grammar integration
goal: 'The Rust core, Python launch layer, editor integrations, and post-epic changes
  all use the completed %id|%i and declare-once clan grammar without a legacy repeat-planner
  bridge, after which epic sase-7g is closed and its plan is finalized.

  '
phases:
- id: core-grammar
  title: Bring the Rust core onto the new directive grammar
  depends_on: []
  description: '''Rust-core grammar and editor parity'' section: migrate the shared
    directive registry, repeat fan-out planner, and editor-facing metadata and tests
    to %id|%i plus the clan= join form.'
- id: python-cutover
  title: Remove the compatibility bridge and integrate concurrent work
  depends_on:
  - core-grammar
  description: '''Python cutover and concurrent-change integration'' section: rebuild
    against the migrated core, remove the temporary legacy rewrite, and verify the
    epic against TUI, tribe, retry, bead, documentation, and completion changes that
    landed after it began.'
- id: land
  title: Verify and land epic sase-7g
  depends_on:
  - python-cutover
  description: '''Final verification and epic landing'' section: run complete validation,
    close sase-7g, clean its expired Symvision allowances, and mark the original epic
    plan done.'
create_time: 2026-07-19 15:04:52
status: done
bead_id: sase-7n
---

# Plan: Finish the agent-ID and clan grammar integration

## Context

Epic `sase-7g` implemented its four phases in the primary `sase` repository:

- `f62815452` renamed the launch directive from `%name|%n` to `%id|%i` and retained intentional compatibility only for
  persisted-display and strip paths.
- `985b1c0d1` added `%id(<id>, clan=<clan>)`, derived hood-qualified names, and prompt-local validation.
- `dea236963` made `%clan` create-only, made `clan=` join-or-create, and migrated retry, bead-epic, tagging, and launch
  lifecycle behavior.
- `09f9151b6` completed retry fixes and end-to-end coverage.

The child bead notes refer to pre-rewrite commit hashes, but these four current commits carry the matching `sase-7g.N`
IDs and are the authoritative history. All four child beads are closed.

The landing audit found one substantive unfinished boundary. The linked `sase-core` repository still registers
`%name|%n`, describes `%clan` with its old join semantics, and makes the Rust repeat planner recognize canonical
`"name"` rather than `"id"`. The primary repository therefore contains a temporary
`_rewrite_id_for_legacy_repeat_planner()` adapter in `src/sase/core/agent_launch_facade.py`. The Rust editor registry is
also used by the xprompt LSP, so it still advertises the removed spelling and omits `clan=` from `%id` argument
completion. This is especially relevant because grouped editor-completion work landed in `sase-core` after the epic's
first commit.

Several other primary-repository changes landed during the epic window: kind-aware editor agent catalogs and prompt
target completion, clan wait-status and group-wait support, TUI action/completion module splits, canonical tribe
persistence, documentation refreshes, and AXE/SDD fixes. Most were already rebased underneath later epic phases, but the
final integration must explicitly verify that their current code paths use the new grammar and semantics.

Long-term memory still documents the old grammar, but memory files are outside this plan: they require explicit user
permission to edit. Likewise, personal chezmoi xprompts and the standalone Neovim plugin remain separately identified
follow-ups unless their behavior is supplied through the Rust LSP changed here.

## Rust-core grammar and editor parity

Open the configured `sase-core` linked repository through the `/sase_repo` workflow before reading or modifying it, and
obey its local `AGENTS.md` release-version rules.

Update the shared Rust directive metadata so `%id` with alias `%i` is the only advertised and canonical identity
directive. `%name` and `%n` must not remain launch-planner aliases; legacy launch prompts are rejected by the primary
Python parser, while non-launch history/display compatibility remains Python-owned. Rewrite the identity description to
cover explicit agent IDs and family attachment. Update `%clan` metadata to describe a create-only clan declaration, and
expose `clan=` as an argument candidate for `%id` while retaining `tribe=` on `%clan`.

Migrate the Rust repeat fan-out parser from canonical `"name"` to `"id"`, including function names, comments, and tests.
`%id` and `%i` must be stripped directly and provide `repeat_name` without translation; directive-looking text inside
fenced, disabled, and adjacent-inline literal regions must remain untouched. Update editor completion, hover,
diagnostic, and planner tests so they assert the new registry and do not accidentally preserve `%name|%n` as supported
spellings. Keep wire field names such as `repeat_name` when they describe the resulting agent name rather than the
surface directive.

Run Rust formatting, workspace tests, and clippy through the primary repository's Rust helper targets (or equivalent
Cargo commands in the opened core checkout), and leave the linked core worktree in a state the next phase can build.

## Python cutover and concurrent-change integration

Run `just install` in the primary repository so `sase_core_rs` is rebuilt from the migrated linked core before testing.
Remove the temporary repeat-planner version bridge from `src/sase/core/agent_launch_facade.py`, including its effective
directive scan and `%id`-to-`%name` rewrite helpers. Strengthen the Rust binding tests so a `%repeat` prompt using both
`%id` and `%i` is handled in one planner call and literal old-looking tokens are preserved only where the scanner is
supposed to ignore them.

Re-audit current source rather than trusting commit summaries:

- Confirm canonical Python tables advertise `id/i`, emit actionable migration errors for `name/n`, and retain the
  intentional strip/history compatibility tests only.
- Confirm `%id(<id>, clan=<clan>)` derives plain, dotted, forced-reuse, and `@`-template names; rejects missing IDs,
  family attachment, `%clan`, and `%tribe` conflicts; and marks declaration versus join membership correctly.
- Confirm launch planning atomically enforces one create-only declaration, supports joiners-only implicit creation and
  later joins, and keeps clan generation and tribe metadata consistent with the canonical tribe-persistence changes that
  landed during the epic.
- Confirm all retry/relaunch entry points still demote declarations after the TUI action-module splits, including
  template-clan resolution, and that clan wait/group-target completion code accepts the derived names.
- Confirm bead epic rendering declares once, joins thereafter, and emits join-only prompts on re-work; confirm tagging
  updates declaration prompts without inventing declarations on joiners.
- Confirm the post-start documentation refreshes, Python directive completion refactor, generated skill sources, demos,
  schemas, and emitters contain the new grammar. Old spellings should remain only in migration/compatibility tests,
  historical plans/data, or literal examples explicitly demonstrating rejection.

Run focused Rust/Python tests for the planner boundary, directive parsing, multi-prompt clan lifecycle, retry editing,
bead epic launch/re-work, tagging, group completion/waits, stored-prompt compatibility, and generated skill sources. Any
gap discovered by this audit is part of this phase rather than a deferred follow-up.

## Final verification and epic landing

Re-run `sase bead show sase-7g` and every child to ensure the hierarchy remains complete and map the rewritten commit
hashes above to their bead IDs. Verify both the primary and linked-core worktrees contain only the intended integration
changes.

Run the core formatting/test/clippy suite, then in the primary repository run `just install`, `just check`, and
`just test-visual`. Do not accept visual snapshot changes merely to make the suite pass; the epic is intended to
preserve rendered clan/tribe behavior.

Only after all verification passes, execute the landing operations in this exact order:

1. Close the original epic with `sase bead close sase-7g`.
2. Read `symvision.md` through `/sase_memory_read`, run `just symvision` if the target exists, and remove stale
   `sase-7g` whitelist entries plus any unused code it reports. Re-run affected checks after cleanup.
3. Change `status: wip` to `status: done` in the frontmatter of `sase/repos/plans/202607/id_directive_clan_kwarg.md`
   (using the effective plans path rather than assuming a checkout relative location).

Finish with `sase bead show sase-7g`, a clean focused test run, `just check` after any post-close source edits, and a
worktree summary for both repositories. Do not edit canonical memory or generated instruction files without the separate
user approval required by repository policy.

## Risks

- The Rust directive registry serves both launch fan-out and editor/LSP behavior; keeping an undocumented legacy alias
  there would silently reintroduce the old spelling, while removing the wrong compatibility path could break persisted
  display in Python.
- Tests must rebuild the local extension before judging the Python facade; otherwise an old installed wheel can make a
  correct source change look broken or preserve the bridge accidentally.
- The final close causes `sase-7g` Symvision exemptions to expire immediately, so cleanup must follow the close and be
  validated before the original plan is marked done.
- The linked core and primary repository are separate Git worktrees. Preserve unrelated changes and report each worktree
  independently; use only the authorized SASE commit workflow if commits are requested by the execution environment.
