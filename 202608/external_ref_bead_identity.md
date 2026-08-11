---
tier: tale
title: External issue identity for beads
goal:
  Beads carry one optional, project-qualified external issue identity that round-trips
  across Rust and Python, rejects duplicates atomically, and supports cross-project-safe
  links.
size: medium
proposed_by: bbugyi200.athena.sase-jd.1
bead: sase-jd.1
create_time: 2026-08-10 19:24:39
status: done
---

- **PROMPT:**
  [prompts/202608/external_ref_bead_identity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/external_ref_bead_identity.md)
- **PARENT:**
  [202608/external_artifact_ingestion.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_artifact_ingestion.md)
- **BEAD:**
  [sase-jd.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jd/sase-jd.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-jd.1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-jd.1.md)
- **COMMITS:**
  - [fd93aab](https://github.com/sase-org/sase/commit/fd93aab1d4c850d10fddb13330108d4e0627a0a1)
    — feat(beads): surface external refs in Python workflows

# External issue identity for beads

## Goal

Add `external_ref` as the durable, project-qualified identity of the external issue a
bead mirrors. The field must round-trip through the Rust canonical bead store and all
Python compatibility surfaces, remain optional for ordinary beads, reject duplicate
non-empty identities atomically, support create/update/clear CLI operations on every
bead type, participate in history and search, and power cross-project-safe external
issue links without changing the existing epic-only `find_bug_links` behavior.

## Invariants and representation

- The canonical external issue spelling is `bug:<stable-project-key>#<issue-id>`, where
  the project component is the ProjectSpec directory key rather than its user-facing
  display name. Known keys, aliases, configured display names, and GitHub issue URLs
  resolve through the project lifecycle inventory; unknown project namespaces remain
  stable rather than being collapsed to the issue number.
- Rust and Python models expose the optional value as `external_ref: str = ""` for
  compatibility with existing bead field conventions and legacy JSONL rows. SQLite
  stores the empty value as `NULL`; the schema column is nullable and a partial unique
  index covers only non-null, non-empty values.
- Uniqueness is a canonical store invariant, not merely a compatibility-database side
  effect. Create, single update, batch update, event reduction/merge, and legacy JSONL
  import must reject two beads with the same non-empty external ref before writing. The
  error is reported as a conflict and identifies the existing bead/ref; clearing to
  empty remains allowed on any number of beads.
- `changespec_bug_id` constraints stay unchanged. `external_ref` is valid on plans,
  phases, and tasks, and no external-ref CLI path is gated on Patch metadata.
- `_normalize_bug_id` and `find_bug_links` retain their current within-project,
  epic-only behavior for existing callers. New project-qualified normalization and
  lookup APIs are additive.

## Implementation

### 1. Extend the Rust bead domain and persistence contract

In `sase-core`, add the default-empty `external_ref` field to `IssueWire`,
`BeadCreateRequestWire`, `BeadUpdateFieldsWire`, and `BeadIssueUpdateEventFieldsWire`.
Thread it through issue construction, update application and event emission/replay,
JSONL import/export, read/detail results, history diffs, search field metadata/value
rendering, and the narrow Rust `sase bead` create/update/show/search CLI path. Update
all Rust test builders and parity fixtures that construct or serialize `IssueWire`
explicitly.

Enforce the identity invariant while the mutation lock is held:

- Before a create event is appended, reject a non-empty external ref already owned by
  another live bead.
- For single and batch updates, validate the complete planned post-update snapshot so
  conflicts against unchanged beads and within the same batch abort the entire mutation
  byte-identically.
- During event reduction and import, detect duplicate non-empty refs rather than
  accepting an invalid canonical projection. Keep duplicate issue IDs and external-ref
  conflicts diagnostically distinct.

Update every current `issues` table literal/rebuild in
`crates/sase_core/src/bead/schema.rs` with `external_ref TEXT`, preserve it in each
rebuild's insert/select column list, and add:

```sql
CREATE UNIQUE INDEX IF NOT EXISTS idx_issues_external_ref
    ON issues(external_ref)
    WHERE external_ref IS NOT NULL AND external_ref != '';
```

Add `needs_external_ref_migration` and `external_ref_migration_sql` exports following
the existing additive migration helpers. The migration must add the nullable column and
create the partial unique index in the same ordered migration fragment. Expose both
helpers from `sase_core` and `sase_core_py`, register the Python functions, and cover
the binding inventory.

### 2. Thread the field through the Python shell and compatibility mirror

Add `external_ref: str = ""` to `sase.bead.model.Issue`. Decode it in
`sase.core.bead_wire`, include it in Rust mutation-facade create/update payloads, and
include it in Python JSON/detail/list/search wire dictionaries and compatibility JSONL
conversion. Read and write it in the SQLite row mapper and database helpers, mapping
`""` to `NULL` on persistence.

Mirror the nullable column and partial unique index in `_db_schema.py`. Add the
external-ref migration to `_db_migrations.py` after all table-rebuilding migrations so
those legacy rebuilds cannot drop it; delegate migration detection/SQL to the new Rust
bindings. Exercise both a pre-column compatibility database and repeat initialization to
prove the migration is additive, indexed, idempotent, and round-trips an existing row as
`""`.

Add alphabetically placed, documented CLI flags with short aliases per the CLI rules:

- `sase bead create -x/--external-ref VALUE`
- `sase bead update -x/--external-ref VALUE`
- `sase bead update -X/--clear-external-ref`

The two update forms are mutually exclusive. Both the Python handler and Rust fast path
must set/clear the field consistently, support batch updates atomically, and surface
conflict failures without a traceback. Show/detail JSON and human-readable output expose
a non-empty external ref, and search reports `external_ref` as the matched field.

### 3. Add project-qualified normalization and linking

In `src/sase/bug_links.py`, add `normalize_external_ref(value, *, project)` without
widening `_normalize_bug_id`. It should accept issue numbers, `#number`,
`bug:<project>#<number>`, `<project>#<number>`, and issue URLs; resolve the explicit or
supplied project identity to its stable ProjectSpec key; and return `""` for blank or
malformed input.

Add an external-ref link result/helper that:

- Matches the target against `bead.external_ref` and every `bug:` entry in `bead.refs`
  through project-qualified normalization.
- Includes task beads (and any other bead type carrying a matching identity/ref) rather
  than applying the legacy epic/tier filter.
- Matches Patch `BUG:` values using each Patch's stable `project_name` from its
  ProjectSpec path.
- Never treats issue `#42` from two different projects as the same identity.

Keep the helper pure over caller-supplied bead/Patch snapshots after project identity
resolution, export it additively, and retain all legacy aliases and result properties
used by the Bugs pane.

## Tests

Add focused Rust and Python coverage for:

- Fresh schema shape, partial-index behavior, and migration of a store created before
  `external_ref` existed.
- Legacy missing-field defaults plus non-empty JSONL/event/read/wire round trips.
- Create, update, clear, and atomic batch-update behavior across task/phase/plan beads.
- Two creates with the same external ref producing one bead and one reported conflict,
  with no second event or projection row; include update and reducer/import collision
  cases.
- History recording an external-ref set/clear and literal/regex search returning
  `external_ref` in `matched_fields` and rendered snippets.
- Python/Rust CLI parity, help text/short aliases, human and JSON detail output, and
  conflict presentation.
- `bug:sase#42` and `bug:sase-github#42` normalizing to different stable project keys,
  URL/alias/display-name normalization, task-bead participation, `refs` matching, Patch
  matching, blanks, and malformed inputs.
- Existing `_normalize_bug_id` and `find_bug_links` tests remaining unchanged and green.

## Verification

1. Run `cargo fmt --all -- --check`, focused bead tests,
   `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`
   in `sase-core`.
2. Run `just install` in `sase` so the local PyO3 extension includes the Rust changes.
3. Run focused Python suites for bead DB
   migrations/model/JSONL/project/CLI/history/search/core facades and
   `test_bug_links.py`.
4. Run `just check`; because this phase changes the Rust/Python wire boundary and
   storage migration broadening set, finish with `just check-full`.

## Out of scope

- Do not implement the external issue mirror, TUI chips/actions, provider capability
  changes, or Bugs-pane retirement owned by later epic phases.
- Do not relax or repurpose `changespec_bug_id`, infer an external identity from
  arbitrary `refs`, or rewrite existing bead stores with guessed identities.
- Do not edit SASE memory or generated instruction files.
