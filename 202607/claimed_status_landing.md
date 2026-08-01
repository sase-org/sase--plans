---
tier: tale
title: Finish and land the claimed bead status epic
goal: Close the remaining schema and presentation gaps, validate the integrated claimed-status
  lifecycle, and land epic sase-8y cleanly.
bead: sase-8y
create_time: 2026-07-24 18:13:07
status: done
---

- **PROMPT:** [prompts/202607/claimed_status_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/claimed_status_landing.md)
- **PARENT:** [202607/claimed_bead_status.md](https://github.com/sase-org/sase--plans/blob/main/202607/claimed_bead_status.md)
- **BEAD:** [sase-8y](https://github.com/sase-org/sase--beads/blob/main/pages/sase-8y/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-8y.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-8y.land/README.md)
  - [bbugyi200.athena.sase-8y.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-8y.land.md#member-code)
- **COMMITS:**
  - [d0495f1](https://github.com/sase-org/sase/commit/d0495f1cba07b4706cc7696a1561d9fa0a0c3343) — fix: finish claimed status landing cleanup (sase-8y)

# Finish and land the claimed bead status epic

## Context

Epic bead `sase-8y` adds a durable `claimed` bead status for the interval in which a bead-carrying agent is alive but
waiting to start model execution. Its seven child beads are closed and their implementation commits are present:

- `sase-8y.1`: Rust status support, commit `6dc2a990`
- `sase-8y.2`: Rust claim/release mutations, commit `793234e7`
- `sase-8y.3`: Python status/read surfaces, commit `5ca1756f`
- `sase-8y.4`: runner claim lifecycle, commit `408b7894`
- `sase-8y.5`: reconciler and doctor advisory, commit `bd7ad46a`
- `sase-8y.6`: ACE/clan visuals, actual commit `cf1d3aa4` (the bead note's `50d31a571` short hash is stale or otherwise
  not present)
- `sase-8y.7`: documentation and agent guidance, commit `3b5937b9`

The audit confirmed the central behavior in source: `StatusWire::Claimed` and `Status.CLAIMED`; compare-and-swap wait
claim/release mutations and PyO3 bindings; canonical-store claim helpers; pre-wait claim, pre-execution promotion,
durable `bead_claim_promoted`, shutdown release, and handoff suppression; beads-first stale-claim reconciliation; doctor
reporting; CLI/mobile/ACE/clan presentation; and generated skill/documentation updates.

Concurrent commits since the epic began were also reviewed. The model-panel test split, agent-sync cache work, effort
preservation, documentation refreshes, ACE sync changes, and release-preparation commits do not duplicate or conflict
with the claimed-status lifecycle. The documentation refresh already names the claimed count, and the release branches
already include the epic commits.

Two acceptance gaps remain:

1. `sase-core/crates/sase_core/src/bead/schema.rs` still constrains status to only `open`, `in_progress`, and `closed`
   in `BEAD_SQLITE_SCHEMA`, `issue_type_migration_sql`, and `size_check_relax_migration_sql`. Those are status lists in
   the bead module and reject `claimed` for consumers that use the Rust compatibility schema or rebuild a table through
   either migration.
2. `src/sase/ace/tui/widgets/artifacts/plans_detail.py` hard-codes the claimed readiness glyph/color in an inline map
   even though the epic plan requires claimed status presentation to come from `src/sase/bead_status_presentation.py`.

The linked epic plan is `sase/repos/plans/202607/claimed_bead_status.md`. Repository access rules apply: open
`sase-core` and `plans` with the `/sase_repo` skill before reading or modifying them, and use only the paths that skill
prints.

## Phase 1: Complete the Rust claimed-status schema contract

In the `sase-core` linked repository, add `claimed` between `open` and `in_progress` in every status `CHECK` inside
`crates/sase_core/src/bead/schema.rs`, including:

- `BEAD_SQLITE_SCHEMA`
- `issue_type_migration_sql`
- `size_check_relax_migration_sql`

Strengthen the schema unit tests so they prove that a fresh compatibility database accepts a `claimed` issue and that
both table-rebuild migration fragments retain/accept claimed status rather than merely checking for generic schema text.
Preserve the existing migration shapes and column-copy behavior.

Run `cargo fmt`, `cargo clippy --all-targets --all-features`, and `cargo test --all-targets --all-features` (or the
repository's documented equivalent if it differs).

## Phase 2: Remove the remaining presentation duplication

In the primary `sase` repository, update the ACE Plans detail readiness chip so bead statuses (`closed`, `claimed`, and
`in_progress`) obtain their TUI glyph and Rich color from `bead_status_presentation(issue.status)`. Keep the
readiness-only states (`blocked`, `ready`, `waiting`, and `unknown`) local.

Extend the focused Plans rendering tests to assert that a claimed readiness chip reflects the shared presentation
metadata. Avoid a snapshot update unless rendered output intentionally changes; this refactor should preserve pixels.

Run `just install` before repository checks, then run the focused affected tests and `just check`, as required for
source changes in this repository.

## Phase 3: Re-audit integration and behavior

Review the final diffs in both repositories and repeat the narrow searches that found the gaps:

- no Rust bead schema status `CHECK` may omit `claimed`;
- no claimed status glyph/color mapping in the ACE Plans detail path may duplicate the shared presentation source;
- `sase bead work` must remain status-blind and schedule every non-closed phase;
- claim/release error handling must remain advisory and compare-and-swap safe.

Run focused claimed-status tests in both repositories if the full checks did not already exercise them. Confirm the
primary checkout and both linked repositories contain no unrelated modifications.

## Phase 4: Close and finalize the epic

This is the final phase and must happen only after Phases 1-3 pass.

1. Run `sase bead show sase-8y` once more and confirm all seven child beads are closed.
2. Close the epic with `sase bead close sase-8y`.
3. After closure, run `just symvision` if that recipe exists. Epic-symbol allowances for `sase-8y` expire at closure. If
   Symvision reports stale allowlist entries, private-use violations, or unused code, read the `symvision` long-term
   memory through `/sase_memory_read` before changing anything, remove only the stale entries/unused code it identifies,
   and rerun `just symvision`. If those fixes change primary-repository files, rerun `just check`.
4. Open the `plans` sidecar through `/sase_repo` and change only the linked epic plan frontmatter from `status: wip` to
   `status: done`.
5. Verify with `sase bead show sase-8y` that the epic is closed, verify the plan frontmatter is `status: done`, and
   report all validation results plus the `sase-8y.6` note-hash discrepancy.
