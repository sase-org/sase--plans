---
tier: tale
title: Complete and land the git index-lock recovery epic
goal: 'Every remaining SDD Git mutation uses the shared bounded retry and safe stale
  index-lock recovery policy, configuration behavior is consistent, and sase-77 is
  closed only after focused and full validation succeeds.

  '
create_time: 2026-07-19 10:56:55
status: done
prompt: 202607/prompts/sase_77_completion.md
---

# Plan: Complete and land the git index-lock recovery epic

## Context and audit findings

Epic `sase-77` has four closed children, represented in the primary repository by commits `725782988` (`sase-77.1`),
`4060ac645` (`sase-77.2`), `09fa3fe1e` (`sase-77.3`), and `c7b84dc46` (`sase-77.4`). The shared policy, the principal
VCS and workspace-provider funnels, the named long-tail callers, and broad unit and integration coverage are present.
The closed child notes for the first three phases contain hashes that are not objects in the repository, so landing
should reconcile those notes with the commit-message-backed hashes above.

The non-epic commits that landed after the first epic commit only affect ACE clan/family state, keymaps, Vim-search
extraction, rendering, and documentation. They neither add nor modify Git command runners, so no interleaved feature
needs adoption of the retry helper.

The source audit did find incomplete epic work. Several SDD flows still bypass `run_sdd_git_write()` and call the
low-level `run_sdd_git()` directly while performing index-mutating operations:

- bare-git SDD initialization stages and commits files in `sase.sdd._commit_bare_git`;
- repository integration fetches and rebases, continues or aborts rebases, and is reused by the managed bead-sync
  worker;
- sidecar clone fast-forwarding runs `git pull`;
- store materialization and sidecar initialization retain generic low-level runners for their Git transactions.

Consequently, a planted stale `.git/index.lock` can still defeat representative SDD initialization, integration, and
materialization flows despite the epic's codebase-wide guarantee. In addition, the SDD compatibility delay resolver
falls directly back to the default schedule, so `SASE_GIT_LOCK_RETRY_DELAYS` does not configure SDD writes when the
legacy `SASE_SDD_GIT_LOCK_RETRY_DELAYS` variable is absent. This duplicates parsing that the shared module was intended
to own.

## Phase 1: Finish the SDD retry funnel

Route every remaining SDD runner that can execute an index-mutating command through the shared retry policy while
preserving its existing native result, timeout, checked-error, transaction-lock, and best-effort push semantics. Favor
one SDD adapter/funnel that repository integration, the managed bead-sync worker, bare-git initialization, sidecar
fast-forwarding, and materialization can reuse instead of adding new per-module retry closures. Keep pure repository
discovery and clone-only probes direct when they cannot encounter an existing checkout's index lock, and leave an
accurate one-line rationale at deliberately unwrapped sites.

Remove or correct comments that currently claim the low-level SDD runner already supplies shared retry behavior.
Preserve the store write lock as the outer transaction guard; the shared retry driver remains the recovery layer
underneath it and must not introduce nested flock acquisition.

Make the legacy SDD delay variable an override for SDD callers, but delegate to the shared `git_lock_retry_delays()`
result when it is unset. This makes the global variable apply consistently while retaining compatibility and a clear
precedence rule. Avoid another independent parser unless compatibility requires it for the explicitly set legacy value.

## Phase 2: Prove the missing paths and re-audit

Add focused tests with tiny retry schedules and real planted lock files for the missed behavior. At minimum, cover a
bare-git SDD initialization add/commit, an SDD repository rebase or pull path used by sidecar/bead integration, and the
default-vs-legacy delay-variable precedence. Extend the managed bead-sync or materialization suites where that is the
clearest way to prove their runner now uses the shared policy. Assert successful final operations and lock removal, not
merely that the helper was called.

Repeat the source sweep for direct `subprocess` Git invocations, `_run_git` helpers, and `run_sdd_git()` callers.
Classify every remaining direct call as a pure read/clone operation or route it through the policy; ensure no mutation
path relies only on a comment. Run `just install` before repository validation, then run the focused lock/SDD/bead
suites and `just check`. Resolve every failure without weakening the stale-lock safety rules or the existing TUI
responsiveness guarantees.

## Phase 3: Land and clean up the epic

Only after Phases 1 and 2 pass, reconcile the incorrect commit hashes in the closed child bead notes with the four
commit-message-backed epic commits. Then perform the requested landing sequence in this exact order:

1. Close the parent with `sase bead close sase-77`.
2. Run `just symvision`. Because closure expires the `sase-77` epic-symbol exemptions, remove the three stale `sase-77`
   entries from the `Justfile`, address any newly reported unused public surface in the shared retry module, and rerun
   Symvision until clean.
3. Open the canonical plans sidecar through `sase repo open plans`, resolve the linked `202607/git_index_lock_retry.md`,
   and set only its frontmatter `status` to `done`.

Finish with `just check` after all source and `Justfile` cleanup. Verify `sase bead show sase-77` reports the epic
closed, every child remains closed, the linked plan reports `status: done`, and both the primary checkout and plans
sidecar contain only the intended landing changes.

## Risks and constraints

Do not close the epic early: Symvision cleanup is intentionally meaningful only after the whitelist expires. Do not
replace the repository transaction flock with retry sleeps or acquire the same flock recursively. Retry tests must use
short explicit schedules so they remain deterministic. Preserve the existing rule that only a canonical, unchanged or
old-enough `index.lock` may be removed; non-index locks are retried but never deleted.
