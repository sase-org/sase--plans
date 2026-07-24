---
tier: epic
title: Finish and land five-size epic phase support
goal: 'The xsmall/xlarge phase-size feature handles legacy SQLite stores, every shipped
  reference describes the five-size routing accurately, and sase-8w is closed only
  after post-close Symvision cleanup and final verification.

  '
phases:
- id: migrate
  title: Wire legacy SQLite phase-size relaxation
  depends_on: []
  size: medium
  description: '''Wire legacy SQLite phase-size relaxation'' section: expose and invoke
    the Rust size-constraint relaxation on compatibility database open, with regression
    coverage for existing three-size stores.

    '
- id: docs
  title: Reconcile five-size public documentation
  depends_on: []
  size: small
  description: '''Reconcile five-size public documentation'' section: replace stale
    three-size and old-alias guidance across the shipped SASE manuals and check later
    non-epic work for newly relevant integration points.

    '
- id: land
  title: Verify and land sase-8w
  depends_on:
  - migrate
  - docs
  size: small
  description: '''Verify and land sase-8w'' section: run end-to-end checks, correct
    stale child commit notes, close sase-8w, perform required post-close Symvision
    cleanup, and mark the original epic plan done.'
parent_bead: sase-8w
parent: sase/repos/plans/202607/phase_sizes.md
create_time: 2026-07-23 19:12:58
status: done
bead_id: sase-8w.7
---

# Plan: Finish and land five-size epic phase support

## Audit context and invariants

The original epic `sase-8w` added `xsmall` and `xlarge` around the existing `small`, `medium`, and `large` phase sizes.
Its landed implementation commits are:

- `sase-core`: `f9d9c37a452602a9021c5170892e94346f302390` (`sase-8w.1`).
- `sase`: `18ca7cb9684c5b24ca311b6a8e8f8706a3a13f85` (`sase-8w.2`), `a5c5d0398e31032622ca93624fddc95d8a1bcc58`
  (`sase-8w.3`), `f19e031dd1aebbb6ebb1b86fd4385f02d91c5901` (`sase-8w.4`), and
  `39f90d3764eb050ae869f711a62e36f467874d64` (`sase-8w.5`).

The implementation already validates and serializes all five Rust and Python values, routes them through
`@xsmall_phase_worker` through `@xlarge_phase_worker`, adds `#plan` only for `large` and `xlarge`, renders the five-chip
palette in canonical order, and updates `sase plan validate --explain`. Preserve this canonical routing:

| Size     | Phase alias            | Implicit target          | `#plan` |
| -------- | ---------------------- | ------------------------ | ------- |
| `xsmall` | `@xsmall_phase_worker` | `@cheaper`               | no      |
| `small`  | `@small_phase_worker`  | `@cheap`                 | no      |
| `medium` | `@medium_phase_worker` | `codex/gpt-5.6-sol@high` | no      |
| `large`  | `@large_phase_worker`  | `@smart`                 | yes     |
| `xlarge` | `@xlarge_phase_worker` | `@smartest`              | yes     |

The audit found two unfinished areas:

1. `sase-core` defines and tests `needs_size_check_relax_migration()` and `size_check_relax_migration_sql()`, but no
   Python binding or database-open path invokes them. Opening an existing compatibility database whose `issues.size`
   constraint is `size IN ('small', 'medium', 'large')` leaves that schema unchanged, and an `xsmall` insert fails with
   `sqlite3.IntegrityError`.
2. `docs/ace.md`, `docs/beads.md`, `docs/configuration.md`, `docs/llms.md`, and `docs/sdd.md` still contain three-size
   lists and the retired alias routing (`small -> @cheaper`, `medium -> @default`, `large -> @smartest`).

The only non-epic `sase` commit after the first `sase-8w` commit was `2464be5462bd99580d0a91b2802abea3560e9064`
(`sase-8v.4`). It changes agent sidecar publication and has no phase-size or model-routing overlap. Recheck the current
history at implementation time, but do not manufacture an integration edit when later changes remain unrelated. There
were no later `sase-core` commits after `sase-8w.1` at audit time.

## Wire legacy SQLite phase-size relaxation

Open the linked `sase-core` repository with `/sase_repo` before reading or editing it. Keep schema-detection and
migration policy in Rust, consistent with the repository's Rust core backend boundary. Add a narrow `sase_core_rs` API
in `crates/sase_core_py/src/lib.rs` for the already-landed size-check detector and migration SQL, including binding
registration, documented API inventory, and binding-level tests. Prefer names consistent with the existing bead facade
surface.

In `src/sase/bead/db.py`, add an open-time migration between adding the `size` column and executing the current schema.
Read the stored `issues` table SQL, call the Rust detector, and execute the Rust-owned rebuild SQL only for a legacy
three-value size constraint. Keep the Python layer a thin adapter; do not copy the table-rebuild SQL or independently
reimplement the detector. Verify that:

- Fresh databases still accept both bookend sizes.
- Databases without a `size` column take the existing add-column migration and do not run the rebuild unnecessarily.
- Existing three-size databases preserve issues, dependencies, current columns, indexes, and foreign-key validity,
  become idempotently current on reopen, and accept both `xsmall` and `xlarge`.
- Already-current databases are unchanged.

Add focused Rust binding tests and Python regression coverage in `tests/test_bead/test_db.py`. Run `just install` before
Python checks so the binding includes the new API, run the `sase_core` Rust suite, and run the focused database/binding
tests.

## Reconcile five-size public documentation

Update all stale phase-size and alias references in:

- `docs/ace.md`
- `docs/beads.md`
- `docs/configuration.md`
- `docs/llms.md`
- `docs/sdd.md`

Document the canonical five-value order, the mint/sky/gold/rose/violet chip ramp where presentation is discussed,
`#plan` for `large` and `xlarge` only, and the five alias fallbacks from the invariant table above. Update CLI `--size`
value tables, Models-panel alias/bucket membership and order, completion catalogs, configuration examples, role-alias
explanations, plan frontmatter guidance, and the `xsmall` replacement for explicit cheap testing models. Preserve
intentional three-value test fixtures or examples that are testing a subset rather than claiming the supported domain.

Search the current source and docs again for hard-coded three-size domains and old routing prose. Inspect every match in
context so unrelated uses of small/medium/large (for example performance dataset sizes or model tiers) are not changed.
Re-run markdown formatting/lint through `just check`.

## Verify and land `sase-8w`

First re-open the original epic and every child with `sase bead show` and confirm the repair phases did not invalidate
any earlier acceptance criteria. The notes for `sase-8w.2` and `sase-8w.4` currently name unreachable pre-rewrite hashes
(`297d5b977` and `102338042`); update those child notes to the landed hashes listed in this plan so the bead record is
accurate.

Run `just install`, `just check`, and `just test-visual`. The pre-plan audit observed one unrelated parallel TUI timing
failure in `test_commits_pilot_drives_live_filter_bar_detail_copy_and_toggles`, which passed immediately in isolation;
do not waive a repeatable failure. Exercise a throwaway five-size epic plan through the workspace-installed CLI and
confirm validation, dry-run directives, planning gates, alias resolution, five-chip rendering, and legacy-database
migration. Remove the throwaway file.

When all checks are green, close the original epic with:

```bash
sase bead close sase-8w
```

Only after it is closed, use `/sase_memory_read` for the Symvision memory and run `just symvision` if that recipe
exists. The close expires `sase-8w` epic-symbol whitelist entries; remove every stale whitelist entry and unused
symbol/code that Symvision reports, then rerun the relevant tests and `just check`. Do not reopen or suppress the epic
merely to retain a whitelist.

Finally, open the plans sidecar with `/sase_repo` and set `status: done` in the frontmatter of the original epic plan at
`202607/phase_sizes.md` (the plan shown by `sase bead show sase-8w`). Verify the epic is closed, all children remain
closed, the original plan is marked done, all worktrees are clean except for intended committed results, and no required
landing work remains.
