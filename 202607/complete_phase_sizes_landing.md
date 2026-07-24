---
tier: tale
title: Complete and land the five-size phase-size epic chain
goal: Correct the final public routing inconsistency, revalidate the integrated five-size
  feature, and close sase-8w.7.4 plus its parent epics with post-close Symvision cleanup
  and all linked plans marked done.
bead: sase-8w.7.4
parent: sase/repos/plans/202607/finish_phase_sizes_landing.md
create_time: 2026-07-23 21:20:06
status: wip
---

- **PROMPT:** [202607/prompts/complete_phase_sizes_landing.md](prompts/complete_phase_sizes_landing.md)

# Plan: Complete and land the five-size phase-size epic chain

## Audit baseline

The land audit for `sase-8w.7.4` found that the behavioral five-size feature is present and tested, but the epic chain
is not ready to close yet.

The current `sase` checkout is `master` at `f22a49f0d`, identical to `origin/master`, with no ChangeSpec or PR branch.
The plans sidecar is `main`, identical to `origin/main`. The linked `sase-core` checkout is `master`, identical to
`origin/master`. Starting at the first `sase-8w.7.4` commit (`4c3fde93e`), the only later `sase` commit is the epic's
own `f22a49f0d`; there are no later non-epic commits to integrate. No linked-core commit landed after that start time.
Recheck all three repositories immediately before landing because this baseline can change.

Every child bead named by `sase bead show sase-8w.7.4` is closed. The audit inspected both child commits and their live
source:

- `4c3fde93e` updates `docs/sdd.md` and `docs/xprompt.md` with the five phase-worker aliases, their canonical fallbacks,
  and the rule that an explicit phase model is only for a user-requested model. The repaired `sase-8w.7.1` note now
  names both reachable commits: `32a146d` in `sase-core` and `b638df32f` in `sase`.
- `f22a49f0d` replaces fixed sleeps in `tests/ace/tui/test_family_member_relaunch.py` with observable async state waits.
  This is a valid full-suite stability fix, but it did not perform the landing actions promised by `sase-8w.7.4.2`.

The audit also reopened `sase-8w.7`, `sase-8w`, and all of their phase children; inspected the relevant landed commits
and live Rust, Python, TUI, documentation, and regression-test source; rebuilt the local Rust binding; and ran
`just check` and `just rust-check`. Both complete lanes pass. The build emits the already-documented warning that the
linked core source version is `0.9.0` while the published Python dependency window expects `0.12.x`; development builds
explicitly permit this, so do not lower the dependency window as part of this landing.

Two concrete gaps remain:

1. `crates/sase_core/src/plan/validate.rs` still tells schema consumers that `large` and `xlarge` route directly through
   `@smart` and `@smartest`. The implementation correctly routes all sizes through their size-specific role aliases;
   `@smart` and `@smartest` are only the implicit fallbacks of the last two aliases.
2. `sase-8w.7.4`, `sase-8w.7`, and `sase-8w` are all still in progress. Their linked plans
   `202607/finish_phase_sizes_landing.md`, `202607/finish_phase_sizes.md`, and `202607/phase_sizes.md` all still have
   `status: wip`.

Preserve this canonical behavior:

| Size     | Phase alias            | Implicit target          | `#plan` |
| -------- | ---------------------- | ------------------------ | ------- |
| `xsmall` | `@xsmall_phase_worker` | `@cheaper`               | no      |
| `small`  | `@small_phase_worker`  | `@cheap`                 | no      |
| `medium` | `@medium_phase_worker` | `codex/gpt-5.6-sol@high` | no      |
| `large`  | `@large_phase_worker`  | `@smart`                 | yes     |
| `xlarge` | `@xlarge_phase_worker` | `@smartest`              | yes     |

## Phase 1: Correct the Rust schema guidance

Open the linked `sase-core` repository through `/sase_repo`. Update `PHASE_SIZE_DESCRIPTION` in
`crates/sase_core/src/plan/validate.rs` so it says that the five sizes route through `@xsmall_phase_worker`,
`@small_phase_worker`, `@medium_phase_worker`, `@large_phase_worker`, and `@xlarge_phase_worker`. If the text discusses
ultimate targets, distinguish those aliases from their implicit fallbacks instead of presenting `@smart` and `@smartest`
as direct size routes.

Strengthen the existing schema-field test in the same module so this public explanation cannot regress to a partial
alias ladder. Exercise the focused Rust test, then run the linked core's full check lane. Commit the linked-core change
through the normal SASE commit workflow and verify the resulting commit is reachable from its remote base.

## Phase 2: Revalidate integration and feature behavior

Re-run the integration scan from `4c3fde93e^` through the current `sase` branch, excluding commits belonging to
`sase-8w.7.4`. If a ChangeSpec or PR now exists, inspect its base branch as well. Scan linked `sase-core` from the epic
start time and inspect the plans-sidecar changes since its first `sase-8w.7.4` record. Integrate any newly landed work
that should consume the five-size feature or duplicates/conflicts with it; do not manufacture an edit for unrelated
work.

Reopen `sase-8w.7.4` and both children, then `sase-8w.7`, `sase-8w`, and every child shown beneath those parents.
Confirm each promised note is reflected in reachable commits and live source. Rebuild the linked binding with
`just install`, then run `just check` and `just rust-check`. Verify in source and tests that:

- Rust and Python accept and round-trip all five sizes.
- legacy three-size SQLite constraints are relaxed idempotently without losing records, dependencies, columns, or
  indexes;
- every size routes through its size-specific phase-worker alias and explicit phase models still win;
- only `large` and `xlarge` receive `#plan`;
- the five-value presentation order and chip palette cover `xsmall` through `xlarge`;
- public docs and both schema explanations use the canonical routing and authoring policy.

A repeatable failure or an unreachable/mismatched bead note blocks landing and must be fixed before proceeding.

## Phase 3: Land the epic chain

Only after Phases 1 and 2 are complete, perform the landing in this exact order:

1. Run `sase bead close sase-8w.7.4` and verify the nested epic and both of its children are closed.
2. Run `sase bead close sase-8w.7` and independently verify it and all of its children are closed.
3. Run `sase bead close sase-8w` and verify the root epic and its full child tree are closed.
4. After all three closes, load `symvision.md` through `/sase_memory_read` and run `just symvision`. Remove every
   expired epic-symbol whitelist entry and any genuinely unused code it reports according to the Symvision decision
   hierarchy; do not reopen or suppress an epic to retain a whitelist. Re-run the exact failing lane and then
   `just check` after any source edit.
5. Open the plans sidecar through `/sase_repo` and set `status: done` in `202607/finish_phase_sizes_landing.md`,
   `202607/finish_phase_sizes.md`, and `202607/phase_sizes.md`. Persist the intended sidecar updates through its normal
   workflow.

Finish by reopening all three epics, verifying every descendant is closed, confirming all three plan frontmatters are
`status: done`, confirming every intended commit is reachable, and checking that `sase`, `sase-core`, and the plans
sidecar are clean and synchronized with their expected remote bases.
