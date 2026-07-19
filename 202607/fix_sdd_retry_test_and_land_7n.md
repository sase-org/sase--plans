---
tier: tale
title: Repair the SDD retry regression and land sase-7n
goal: 'The non-lock Git failure regression measures SASE retry behavior without intercepting
  Python subprocess polling, all newly landed work is reconciled with the agent-ID
  and clan grammar, and epic sase-7n is closed with clean post-close Symvision state
  and a finalized plan.

  '
create_time: 2026-07-19 16:10:52
status: wip
prompt: 202607/prompts/fix_sdd_retry_test_and_land_7n.md
---

# Plan: Repair the SDD retry regression and land sase-7n

## Context

The landing audit verified all three `sase-7n` children and their authoritative commits:

- `23f5be3` in the linked `sase-core` repository moved repeat planning and editor/LSP metadata to `%id|%i`, exposed
  `clan=`, removed `%name|%n` support, and preserved fenced, disabled, and adjacent-inline literal regions.
- `dc5b7ea8b` in the primary repository removed the temporary Python `%id`-to-`%name` repeat-planner bridge and
  strengthened binding coverage.
- `c9a7757` in the plans repository finalized the original `sase-7g` plan after that epic and all four children were
  closed.

Current source inspection confirmed the intended parser, create-only clan declaration, join-or-create membership, retry
demotion, bead epic rendering, and Rust editor behavior. `just rust-check` passed in the linked core. In the primary
repository, formatting, Ruff, mypy, pyscripts, Symvision, and toobig passed; the complete test run produced 19,407
passes, seven skips, and one failure.

The failure is `tests/test_sdd_commit.py::test_commit_sdd_files_does_not_retry_non_lock_128`. It patches
`sase.sdd._git_contention.time.sleep`, which mutates the shared standard-library `time` module. On Python 3.14,
`subprocess.run(..., timeout=)` uses `time.sleep` for its own bounded polling, so the mock records hundreds of
subprocess polling sleeps even though the invalid pathspec never reaches the Git-lock retry helper. The production
classifier correctly treats this non-lock exit 128 as non-retryable; the test observes the wrong boundary.

The audit also found intentional concurrent state that must be preserved. Successor epic `sase-7o` is actively replacing
the positional family form and standalone tribe directive with `%id` keyword arguments. Its managed provider skill
copies already contain the new family syntax, so `sase skill init --force` from the current primary source would
overwrite in-flight successor work. Chezmoi commit `5c111214` proves those provider files had already been correctly
regenerated for `sase-7n` before `sase-7o` began. Do not treat the current generated-skill drift as unfinished `sase-7n`
work.

## Phase 1: Make the non-lock retry regression observe the retry boundary

Replace the process-global sleep monkeypatch with a focused observation of the SASE Git-lock retry boundary. Keep
coverage for the public `commit_sdd_files` behavior and ensure an invalid pathspec raises immediately without a retry.
Prefer an assertion on the helper invocation/outcome or a module-local time proxy that cannot affect `subprocess`; do
not weaken the production retry classifier or remove the non-lock regression.

Run the exact failing test serially, the related Git-lock and SDD contention tests, and the full primary test suite.
Re-run `just install` before repository checks so the local Rust binding comes from the audited linked-core checkout.
Run `just check`; if its only failure is `sase init --check` reporting the five `sase_run` provider files while
`sase-7o` is still active, preserve those files and record the scoped code/test results rather than regenerating older
syntax. If `sase-7o` has finished and the canonical source has landed, update to that landed state and require the full
check to pass.

## Phase 2: Reconcile newly landed concurrent work

Immediately before landing, re-run the commit-window audit from the first `sase-7n` work through current `master`,
excluding `sase-7n`'s own commits. Review actual diffs, not only subjects. In particular, re-check whether any `sase-7o`
commits or release automation have reached the base branch while this follow-up was running. Integrate only changes that
have actually landed on the base branch; do not merge or copy from unmerged automation branches and do not overwrite
active linked-repository worktrees.

Reconfirm the three `sase-7n` children are closed and their notes match the source, the legacy repeat bridge remains
absent, and the primary, linked-core, plans, and opened chezmoi worktrees contain no unintended changes from this
follow-up.

## Final phase: Close and finalize epic sase-7n

Only after the regression and integration audit are complete, perform the landing in this order:

1. Run `sase bead close sase-7n`.
2. Read `symvision.md` through the `sase_memory_read` workflow, then run `just symvision`. Remove stale `sase-7n`
   whitelist entries and any unused code reported after the epic exemption expires, and re-run affected checks.
3. Change `status: wip` to `status: done` in the frontmatter of
   `${SASE_SDD_PLANS_DIR}/202607/finish_id_directive_clan_integration.md`.

Finish with `sase bead show sase-7n`, focused regression checks, `just check` after any primary source cleanup, and
worktree summaries for the primary, linked-core, plans, and chezmoi repositories. Do not edit canonical memory or
generated provider instruction shims. Do not regenerate provider skills over active `sase-7o` work.

## Risks

- Patching an attribute on the shared `time` module reproduces the current false failure; the replacement test must
  isolate its observer from Python's subprocess implementation.
- `sase-7o` deliberately supersedes parts of the family/tribe grammar while this landing completes. Only base-branch
  commits are integration inputs, and its in-flight managed files must be preserved.
- Closing `sase-7n` expires its Symvision allowances immediately, so the post-close scan and cleanup cannot be deferred.
