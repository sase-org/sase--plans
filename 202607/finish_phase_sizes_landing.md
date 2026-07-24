---
tier: epic
title: Finish and land the five-size phase-size epics
goal: The remaining five-size documentation and bead-record gaps are repaired, later
  work is integrated, both sase-8w.7 and sase-8w are closed only after final verification,
  post-close Symvision cleanup is complete, and both linked epic plans are marked
  done.
phases:
- id: reconcile
  title: Reconcile the remaining records and public guidance
  depends_on: []
  size: small
  description: '''Reconcile the remaining records and public guidance'' section: fix
    the stale xprompt and SDD guidance, correct the sase-8w.7.1 commit note, search
    for equivalent leftovers, and run the full SASE checks.'
- id: land
  title: Reverify integration and land both epics
  depends_on:
  - reconcile
  size: medium
  description: '''Reverify integration and land both epics'' section: integrate later
    work, rerun end-to-end checks, close sase-8w.7 and sase-8w in order, perform post-close
    Symvision cleanup, and mark both linked plans done.'
parent_bead: sase-8w.7
parent: sase/repos/plans/202607/finish_phase_sizes.md
create_time: 2026-07-23 20:33:35
status: done
bead_id: sase-8w.7.4
---

# Plan: Finish and land the five-size phase-size epics

## Goal

Repair the remaining public-documentation and bead-record gaps found while landing `sase-8w.7`, recheck integration
against everything that has landed since the epic began, and close both `sase-8w.7` and its parent `sase-8w` only after
the five-size feature is demonstrably complete. Finish the post-close Symvision cleanup and mark both linked epic plans
done.

## Audit baseline

The completed implementation is already present on `sase` `master`/`origin/master` and linked `sase-core`
`master`/`origin/master`. There is no current ChangeSpec or PR branch. The relevant landed commits are:

- `sase-core`: `f9d9c37` (`sase-8w.1`) and `32a146d` (`sase-8w.7.1`).
- `sase`: `18ca7cb96` (`sase-8w.2`), `a5c5d0398` (`sase-8w.3`), `39f90d376` (`sase-8w.5`), `f19e031dd` (`sase-8w.4`),
  `39c9fe7dc` (`sase-8w.7.2`), `b638df32f` (`sase-8w.7.1`), and `c2fd04336` (`sase-8w.7.3`).

The audit read every child bead and the linked plans, inspected these commit diffs and the live Rust/Python/TUI source,
and rebuilt the local Rust binding. `just check` passed, including the visual snapshot suite, and `just rust-check`
passed in the linked core. The build emits a pre-existing warning that the linked core source version is `0.9.0` while
the published Python dependency window expects `0.12.x`; the development build explicitly permits this and both full
check lanes pass, so do not lower the dependency window as part of this landing.

No non-epic commit landed after the first `sase-8w.7` commit (`39c9fe7dc`), and the branch is identical to its base.
Recheck this at implementation time. Earlier unrelated work has no phase-size overlap.

The audit nevertheless found these concrete unfinished items:

1. `docs/xprompt.md` still says only small/medium/large phase aliases exist and documents the retired fallbacks
   `@cheaper`, `@default`, and `@smartest`.
2. `docs/sdd.md` still permits an explicit phase model when work is non-consequential or exercises the feature. The
   authoritative five-size guidance reserves explicit phase models for a model specifically requested by the user and
   directs exercise/observation-only work to `size: xsmall`.
3. `sase bead show sase-8w.7.1` still records unreachable pre-rewrite commit `5feb67a1c`; its landed work spans
   `32a146d` in `sase-core` and `b638df32f` in `sase`.
4. `sase-8w` and `sase-8w.7` remain in progress, and their linked plans `202607/phase_sizes.md` and
   `202607/finish_phase_sizes.md` remain `status: wip`.

Preserve the canonical behavior already covered by the passing checks:

| Size     | Phase alias            | Implicit target          | `#plan` |
| -------- | ---------------------- | ------------------------ | ------- |
| `xsmall` | `@xsmall_phase_worker` | `@cheaper`               | no      |
| `small`  | `@small_phase_worker`  | `@cheap`                 | no      |
| `medium` | `@medium_phase_worker` | `codex/gpt-5.6-sol@high` | no      |
| `large`  | `@large_phase_worker`  | `@smart`                 | yes     |
| `xlarge` | `@xlarge_phase_worker` | `@smartest`              | yes     |

## Phases

### Phase 1: Reconcile the remaining records and public guidance

**ID:** `reconcile`  
**Size:** `small`  
**Depends on:** none

Update `docs/xprompt.md` so its launch-scoped model-alias override examples and explanation cover all five size-specific
aliases and the canonical fallback table above. The examples should demonstrate current policy rather than recommend a
retired large-to-`@smartest` mapping.

Update `docs/sdd.md` so phase `model` guidance matches `src/sase/main/plan_explain.py` and the Rust schema descriptions:
set an explicit phase model only when the user's prompt requested that model; use `size: xsmall` for phases that merely
exercise or observe a SASE agent feature and do no consequential work. Search all shipped docs and source again for
three-size domains, retired size-to-alias routing, and the removed testing-model exception. Inspect every match in
context and preserve intentional three-value test fixtures or unrelated small/medium/large vocabularies.

Correct the notes for `sase-8w.7.1` with `sase bead update` so they name both landed commits and their repositories,
removing unreachable hash `5feb67a1c`. Re-open `sase-8w.7`, each of its child beads, `sase-8w`, and the original
children whose commit notes were previously repaired; verify every note describes a reachable landed commit.

Run focused documentation formatting checks and then `just check`. Commit the documentation changes with the appropriate
SASE bead ID and leave the bead store synchronized through the normal `sase bead` workflow.

### Phase 2: Reverify integration and land both epics

**ID:** `land`  
**Size:** `medium`  
**Depends on:** `reconcile`

Re-run the integration scan before closing. Starting with the first commit mentioning `sase-8w.7`, inspect every later
non-epic commit on the working branch and, if a PR/ChangeSpec now exists, the base branch as well. Update any newly
landed code or documentation that should consume the five-size feature or duplicates/conflicts with it. Do not
manufacture an integration edit when later work remains unrelated.

Re-open both epics and all children, inspect the final diffs and live source, run `just install`, `just check`, and
`just rust-check`, and confirm the legacy SQLite migration, five alias routes, `#plan` gating, five-chip rendering, and
five-size validation still pass. Treat a repeatable failure as blocking.

Perform the landing in this order:

1. Close the nested epic explicitly with `sase bead close sase-8w.7`.
2. Close its parent with `sase bead close sase-8w`. Closing a plan cascades to active children, which is why the nested
   epic must be closed first and verified independently.
3. Only after both are closed, use `/sase_memory_read` to load the Symvision memory and run `just symvision` (the recipe
   exists). Remove expired `sase-8w.7` and `sase-8w` epic-symbol whitelist entries and any unused code Symvision
   reports; do not reopen or suppress either epic to retain a whitelist. Re-run relevant tests and `just check`.
4. Open the plans sidecar through `/sase_repo` and set `status: done` in both `202607/phase_sizes.md` and
   `202607/finish_phase_sizes.md`.

Finally, verify with `sase bead show` that both epics and every child are closed, confirm both plan frontmatters are
`status: done`, confirm every intended commit is reachable, and ensure the `sase`, `sase-core`, and plans worktrees are
clean except for intended committed results. No landing work remains until all of these conditions hold.
