---
tier: epic
title: Canonical plan BEAD associations
goal: "New epic plan links are stored once in the canonical BEAD header while new and
  legacy plans remain safe to refresh, resume, diagnose, and repair.

  "
phases:
  - id: core-plan-bead-association
    title: Rust-owned plan bead association contract
    depends_on: []
    size: medium
    description:
      "core-plan-bead-association: implement and bind canonical-header-first plan bead
      resolution, then use it in bead-doctor ownership checks."
  - id: shell-canonical-bead-links
    title: Canonical BEAD links throughout the SASE shell
    depends_on:
      - core-plan-bead-association
    size: medium
    description:
      "shell-canonical-bead-links: migrate link writes, resume, refresh, repair,
      diagnostics, tests, and documentation to the shared core contract."
create_time: 2026-09-09 19:53:03
status: wip
---

- **PROMPT:**
  [prompts/202607/canonical_bead_plan_links.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/canonical_bead_plan_links.md)

# Make the `BEAD` header the canonical plan-to-bead association

## Goal

Stop writing the redundant `bead_id` YAML frontmatter property when
`sase bead work <plan-file>` creates an epic. Store new epic-to-plan associations only
in the plan's top-level `BEAD` Markdown bullet, while preserving safe resume,
duplicate-prevention, refresh, doctor, and repair behavior for both new plans and
already-archived plans that still use legacy `bead_id` or `bead` frontmatter.

## Current behavior and risks

- `src/sase/bead/epic_from_plan.py` currently writes `bead_id` into frontmatter, then
  derives a `BEAD` header section from it. Removing only the frontmatter assignment
  would make later header refreshes delete the new bullet and would make plan-file
  retries create a duplicate epic.
- `src/sase/bead/cli_work_from_plan_helpers.py` detects resumable epics only through
  `bead_id` frontmatter.
- `src/sase/sdd/plan_header_writes.py` treats frontmatter as the durable source and
  treats a header-only `BEAD` section as stale.
- `src/sase/bead/design_ref_repair.py` and the linked `sase-core` repository's
  bead-doctor ownership check identify a plan owner only from legacy frontmatter.
- Existing archived plans can contain `bead_id`, historical `bead`, a matching generated
  `BEAD` section, or only one of those representations. They must remain readable and
  resumable; this change must not rewrite or delete their legacy frontmatter as an
  incidental migration.

## Design decisions

- The canonical `BEAD` header label is authoritative whenever it is present. When no
  `BEAD` section exists, read `bead_id` first and historical `bead` second for backward
  compatibility. This preserves the old legacy precedence while allowing new header-only
  plans.
- Put that precedence and validation in the Rust core plan-document contract and expose
  it through the existing PyO3 boundary. The Python application should consume one typed
  adapter instead of reimplementing the rule across launch, refresh, and repair paths.
- Keep the existing plan-header wire schema unless the implementation actually changes a
  current payload. A new scalar resolver/binding does not by itself require changing the
  schema version.
- Do not remove, rewrite, or reject existing `bead_id`/`bead` fields. They remain
  accepted compatibility input. Do not change the separate `parent_bead` association or
  tale-plan proposal stamping in this work.
- A header-only association must survive all normal provenance refresh paths. Refresh
  may update its hosted page target or degrade it to an unlinked label when the bead
  store proves it missing, but absence of legacy frontmatter is no longer a reason to
  remove the section.
- Keep the current transactional boundary: the complete epic graph exists before the
  `BEAD` link is committed; rollback restores the exact original plan before any runner
  is spawned; preserved/retriable state retains the header-only link.

## Phase 1: Rust-owned plan bead association contract

Phase ID: `core-plan-bead-association`

Dependencies: none.

Work in the linked `sase-core` repository:

1. Add a reusable plan-document API near the existing SDD plan-header contract that
   returns the associated bead ID using this precedence: canonical `BEAD` section,
   legacy `bead_id` frontmatter, then legacy `bead` frontmatter. Reuse the existing
   header parser, trim accepted legacy scalar values, return no association when none
   exists, and surface malformed documents through the existing plan error conventions
   instead of silently inventing an owner.
2. Export the API from the core plan surface and add a small PyO3 binding that returns
   `str | None`. Update the binding inventory/documentation and binding parity tests. Do
   not edit release-owned Cargo versions.
3. Replace the bead-doctor `read_plan_owner` frontmatter-only implementation with the
   shared resolver. Extend `crates/sase_core/tests/bead_read_parity.rs` so ownership
   diagnostics recognize a canonical header-only plan, retain both legacy fallbacks,
   honor the canonical header when both representations exist, and still report a
   genuinely mismatched owner.
4. Add focused pure-Rust tests for missing, canonical linked/unlinked, prettier-wrapped,
   legacy-only, and mixed-representation documents. Confirm that existing plan-header
   parse/render/upsert behavior and wire version stay unchanged.

Acceptance criteria:

- Rust exposes one tested source of truth for reading a plan's bead association.
- Header-only plans participate in bead-doctor owner validation.
- Legacy plans still resolve with the previous `bead_id`-before-`bead` fallback.
- `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`,
  and `cargo test --workspace` pass in `sase-core`.

## Phase 2: Use canonical `BEAD` links throughout the SASE shell

Phase ID: `shell-canonical-bead-links`

Dependencies: `core-plan-bead-association`.

Work in the main SASE repository after the core API is available:

1. Add a typed Python adapter for the Rust plan-bead resolver alongside the existing
   plan-header adapter. Route every plan-owner/link reader through it, including:
   - `linked_bead_id_if_present` and the direct duplicate guard used by epic creation;
   - `refresh_bead_plan_section`; and
   - `src/sase/bead/design_ref_repair.py`. Update repair wording such as “frontmatter
     names another bead” to representation-neutral plan/`BEAD` language.
2. Refactor `src/sase/sdd/plan_header_writes.py` to support explicitly upserting a
   `BEAD` section from a supplied bead ID, including hosted-page target resolution and
   the existing known-ID optimization. Make the general refresh path resolve the current
   association through the core adapter and preserve header-only sections. Keep
   legacy-only refresh as a compatibility backfill and keep the operation idempotent.
3. Change `create_and_launch_epic_from_plan` to upsert and commit the canonical `BEAD`
   section directly, without calling `set_frontmatter_fields` for `bead_id`. Use the
   shared resolver for duplicate prevention. Update user-facing link, stale-link, retry,
   and commit-failure messages to name the `BEAD` link rather than instructing users to
   remove `bead_id`.
4. Update focused tests across plan-header writes/refresh, epic creation, plan-file
   create/resume/dry-run/concurrency/store rollback, and design-reference repair. Assert
   the actual archived document shape: newly-created epics have a canonical `BEAD`
   section and no `bead_id` frontmatter; retries resume the same epic; legacy
   frontmatter-only plans still resume and are backfilled on refresh; duplicate guards
   accept either representation; header-only links survive refresh and
   rollback/preservation boundaries; and hosted/unhosted target behavior remains intact.
5. Update `docs/sdd.md` and `docs/beads.md` to describe the `BEAD` bullet as the
   canonical epic link and resume token, describe `bead_id` as compatibility input
   rather than current output, and replace stale-link/push workflow wording that still
   promises a frontmatter write. Keep the validation docs clear that legacy managed
   properties remain accepted.
6. Install the editable project with the linked core (`just install`), run the focused
   Rust/Python tests while iterating, then run `just rust-check` and the mandatory
   main-repository `just check`. Inspect the final diffs in both repositories to ensure
   no Cargo version fields, generated memory files, or unrelated user changes were
   touched.

Acceptance criteria:

- A successful new epic plan archive contains `- **BEAD:** ...` and does not gain a
  `bead_id:` frontmatter line.
- Re-running the original or archived plan resumes the existing epic and cannot create a
  duplicate.
- Existing `bead_id` and `bead` plans remain readable, refreshable, and diagnosable
  without an automatic destructive migration.
- Header refresh, bead doctor, and design-reference repair agree on the same
  canonical-header-first association.
- Transactional rollback and state-preservation behavior remains unchanged except for
  the representation of the persisted plan link.
- The linked-core and main-repository full check suites pass.

## Out of scope

- Removing legacy frontmatter from already archived plans.
- Rejecting `bead_id` or `bead` in plan validation.
- Changing bead IDs in runtime metadata, environment variables, commit footers, or
  bead-store records.
- Changing `parent_bead`, `PARENT`, phase dependency, launch scheduling, or hosted
  bead-page URL formats.
