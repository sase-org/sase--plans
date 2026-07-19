---
tier: tale
title: Finish sase-7f traceability and landing
goal: 'Epic sase-7f is fully reconciled with current master, its child records point
  to canonical reachable commits, the integrated smart-summary behavior remains green,
  and the epic is closed with post-close Symvision hygiene and its plan marked done.

  '
create_time: 2026-07-19 14:21:51
status: wip
prompt: 202607/prompts/finish_sase_7f_landing.md
---

# Plan: Finish sase-7f traceability and landing

## Verified baseline

Epic `sase-7f` is still open. Its linked plan is `sase/repos/plans/202607/land_sase_73.md`, whose frontmatter remains
`status: wip`. Both children are closed:

- `sase-7f.1` records canonical commit `b1d192ea5f7c24e9b9b28a3963144dfb2e5a0545`, which updates `docs/ace.md`,
  `docs/agent_families.md`, and the deterministic clan visual fixture.
- `sase-7f.2` still records pre-integration commit `21a1d39df31172df9406c6a62f234afcc755992f`. That object is no longer
  on `master`; reachable commit `f9084fcd727ad8e65bbc10b8779386c982267f9e` has the same subject, bead metadata, and
  byte-identical patch on the current parent. Its patch updates the xprompt-save test to patch the public
  `git_lock_retry_delays` helper introduced while the epic was active.

The original feature epic `sase-73` and its three children are closed, their notes resolve to canonical commits
`c433dc7590a64bfa186f311c89b4b75482d63683`, `c85cdd7a369c9c79aa0be9e7a9044f7597ac41c3`, and
`4665110c7e86f301d45c5288039afa150b39dd32`, and its linked plan `sase/repos/plans/202607/smart_summary_folding.md`
already reports `status: done`.

The source audit confirmed the implementation rather than relying on bead notes. `_fold_language.py` owns the shared
glyph, count, fold-header, and scanning-tail presentation. `_member_roster.py` always renders non-empty numbered rosters
and jump maps. Tribe summaries omit empty sections, request reply/slow-call presence at every level, reserve runtime
statistics for level 4, and distinguish bounded triage, grouped inspection, and unbounded forensics. Clan summaries use
one unknown-content scanning tail and omit known-empty disk sections. Family summaries omit absent xprompt/prompt
sections while preserving meaningful pending replies. The cross-kind contract tests pin empty-section omission,
always-present rosters and jump maps, exact scanning-tail behavior, and distinct adjacent levels.

Commits after the first `sase-7f` commit were reviewed through `origin/master`. Later `%id`/clan naming, prompt-target
completion, query shortcuts, clan/tribe fork actions, agent-catalog, Axe, and git-lock changes either preserve the
updated documentation or do not touch the summary renderers. The public git-lock rename is integrated by `f9084fcd7`.
The latest base commit, `fe76c789f28065b5fede6bcb8554dabc166a0d91`, only splits agent kill/cleanup actions; it leaves
the summary modules unchanged and passes the overlapping clan-cleanup and summary tests.

Verification on that integrated tree produced:

- 142 focused renderer, cross-kind contract, member-jump, fold-mode, section-navigation, and xprompt-save tests passed.
- 22 clan, family, tribe, interaction, and retry PNG tests passed.
- 87 post-fast-forward clan cleanup, kill-action, contract, and member-jump tests passed.
- All formatting and lint stages, including Symvision, passed; committed-plan validation passed 2,885 files with no
  errors or warnings.
- The broad test lane reached 19,318 passes and 7 skips. Its three unrelated failures all passed on exact serial rerun:
  two update-command assertions were sensitive to a long `/dev/shm` pytest path wrapping a diagnostic, and one deep
  archive test fetched twice only under full-suite load.
- `just check` currently stops at `sase validate` because five generated `sase_run` provider skill files in the external
  chezmoi source are stale. Do not expand this landing into unrelated dotfile regeneration.

## Refresh integration and repair traceability

Refresh the base reference immediately before landing and inspect any commit added after
`fe76c789f28065b5fede6bcb8554dabc166a0d91`. Fast-forward only with a clean worktree. For each new commit, compare its
paths and behavior with the smart-summary renderers, documentation, member-jump/fold state, and overlapping visual
fixtures. Integrate any real overlap before proceeding; do not assume the audit above remains current if `origin/master`
advanced again.

Repair the one remaining traceability defect:

```bash
sase bead update sase-7f.2 --notes "COMMIT: f9084fcd727ad8e65bbc10b8779386c982267f9e"
```

Then run `sase bead show sase-7f`, `sase bead show sase-7f.1`, and `sase bead show sase-7f.2`. Confirm both children are
closed, the dependency is satisfied, and both note hashes resolve to commits whose subjects contain their bead IDs. Do
not alter the already-correct `sase-7f.1` note.

## Revalidate the current integrated tree

Run `just install` first if the follow-up workspace has not already been installed. Re-run the cross-kind contract,
member-jump, fold-mode, section-navigation, clan/family/tribe renderer, xprompt-save, and six overlapping visual suites.
If a newer base commit touches clan/tribe/family actions or presentation, add its focused tests to this set. Keep all
tests that touch `sase-73` or `sase-7f` strictly green.

Run `TMPDIR=/dev/shm just check`. If the only failure is the known external generated-skill drift, record it without
modifying the chezmoi source, run `just validate-committed-plans` and the test lane directly, and use a short unique
pytest `--basetemp` when checking the update-command tests. Rerun any load-sensitive failure by exact node ID, serially,
with the same code revision. Treat a repeatable failure or any failure in epic-touched code as remaining work and fix it
before landing.

## Final phase: close, run post-close Symvision, and mark the plan done

This phase must remain last and must not start until traceability, base-branch integration, and focused validation are
green.

1. Re-run `sase bead show sase-7f` and both children. Confirm the epic is open, both children are closed, and the child
   notes resolve to canonical commits on current `master`.
2. Close the epic with `sase bead close sase-7f`.
3. Only after the close succeeds, use `/sase_memory_read` to review `symvision.md`, then run `just symvision`. Remove a
   stale `sase-7f(...)` epic-symbol entry if Symvision reports one, delete genuinely dead public code and its dead
   helpers/tests, or make file-local symbols private. Do not add a replacement whitelist merely to silence the linter.
   The current `Justfile` has only the active `sase-7i(release_chop_once_per_keys)` epic-symbol entry; preserve it
   unless its own bead state independently makes Symvision reject it.
4. Re-run the exact Symvision command after any cleanup. If main-repository files changed, run their relevant tests and
   `just check` as required; distinguish the known external generated-skill drift from regressions introduced here.
5. Open the plans sidecar through `sase repo open plans`, then change only the frontmatter of `202607/land_sase_73.md`
   from `status: wip` to `status: done`. The original `smart_summary_folding.md` plan is already done and should remain
   unchanged. Make this plan-status edit the final state change.
6. Finish by showing `sase-7f` again, verifying `status: done` at the top of `land_sase_73.md`, and checking both the
   main checkout and plans sidecar for unintended changes.

If closing the epic exposes Symvision work, that cleanup is part of the landing and must be completed before the plan is
marked done.
