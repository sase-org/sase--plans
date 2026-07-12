---
create_time: 2026-07-11 16:07:35
status: done
prompt: 202607/prompts/retire_legacy_plan_layout.md
tier: tale
---
# Plan: Retire legacy SDD plan directories and migration compatibility

## Context

The unified plan layout has completed its rollout: durable SDD plans now live only under `<sdd_root>/plans/<YYYYMM>/`
and carry `tier: tale|epic` frontmatter. Every project store on every machine has been updated, so the compatibility
window described by the original unification plan can close.

Three kinds of cleanup remain:

1. The SASE companion SDD repository still has 34 tracked files under `tales/` and one under `epics/`. These are not
   undiscovered plans: `tales/202604/perf_artifacts/` contains the Phase 7 performance artifact corpus (including its
   sole remaining Markdown README), and `epics/202605/structured_episodic_memory_mvp_infographic.png` is an image that
   belongs beside its now-canonical plan. The original migration intentionally moved only plan Markdown files, leaving
   these support assets behind.
2. `sase sdd init` still plans, previews, executes, and separately commits the one-time `tales/`/`epics/` migration,
   including collision resolution, frontmatter backfill, prompt-link rewrites, bead design rewrites, README deletion,
   and directory cleanup.
3. Python and Rust readers still scan legacy directories or retry legacy aliases, and active help, generated guides,
   prompts, docs, path classifiers, and tests still describe or protect the transition.

The current bead projection is already clean: `beads/issues.jsonl` has 82 `design` values under `sdd/plans/` and none
under `sdd/tales/` or `sdd/epics/`. Append-only bead event streams and historical plan/research prose still contain old
paths as historical evidence; they are not live references and should not be rewritten wholesale.

This cleanup concerns the unified-plan transition only. Unrelated compatibility contracts—such as old `specs/` prompt
lookup, local `~/.sase/plans/` archives, or compatibility for other SASE subsystems—remain out of scope.

## Product and compatibility decisions

- `plans/` becomes the only repository plan directory understood by runtime discovery, link validation, file lookup,
  diff/commit classifiers, and generated guidance. `research/` remains separate.
- `tale` and `epic` remain first-class **plan-file tier** values and user-visible approval choices. Tier-oriented CLI
  filters such as `sase plan ... --tier tale|epic` and the existing `sase sdd list --kind tales|epics` semantics are
  product vocabulary, not filesystem compatibility, and remain available while being documented as tier filters over
  `plans/`.
- Filesystem aliases do not remain. In particular, `sase sdd path tales|epics`, stale agent-meta path retries,
  `find_sdd_file(..., "tales"|"epics", ...)`, old link spellings, and `sdd/tales|epics` auto-commit classifications stop
  working. A stale external reference will fail normally instead of being silently redirected.
- Writers should express intent as a singular tier (`tale` or `epic`), not as the old plural destination directory. The
  physical destination remains unconditionally `plans/`.
- Historical prose, commit messages, and append-only event payloads remain historically accurate. Current operational
  references, generated files, examples, fixtures, and live projections are canonicalized.

## Phase 1 — Finish the companion-repository data move

In the SDD companion repository:

- Move the complete `tales/202604/perf_artifacts/` subtree to `plans/202604/perf_artifacts/` with history-preserving
  renames. Moving the whole subtree keeps its README adjacent to the JSON/JSONL evidence it documents and eliminates the
  legacy directory rather than relocating only the Markdown file.
- Move the two `tales/202605/perf_artifacts/*.json` files to `plans/202605/perf_artifacts/` for the same reason.
- Move `epics/202605/structured_episodic_memory_mvp_infographic.png` to
  `plans/202605/structured_episodic_memory_mvp_infographic.png`, beside its plan.
- Update current companion content that acts as an operational reference to those assets, including the performance
  README's path examples and the specific plan references to the infographic. Do not mass-rewrite historical mentions of
  `tales/` or `epics/` elsewhere in the SDD corpus.
- Refresh the companion's generated top-level README from the new canonical guide content, then verify there are no
  tracked files or directories left under `tales/` or `epics/`, no legacy prompt `plan:` links, and no legacy live bead
  `design` values. Historical event-stream payloads are explicitly excluded from that last textual audit.

Update every active main-repository consumer of the moved performance paths at the same time: the Justfile benchmark
targets/comments, `.gitignore`, CI artifact upload path, performance harness constants and fixtures, baseline metadata,
and `docs/rust_backend.md`. Runtime outputs should now land under `sdd/plans/<YYYYMM>/perf_artifacts/`, so future runs
cannot recreate a legacy tree.

## Phase 2 — Remove the `sase sdd init` migration lifecycle

- Delete `src/sase/sdd/_plan_migration.py` in full. Its action/result models, collision naming, `git mv` fallback,
  frontmatter/create-time repair, prompt and bead rewriting, README cleanup, warning handling, and repeated database
  refresh optimization are all one-shot migration machinery with no remaining caller.
- Simplify `run_sdd_init` to materialize the configured store and refresh generated files only. Remove migration
  warnings and the special separate-companion commit path/message used solely for moved plans; retain the ordinary
  generated-file/provider initialization behavior.
- Simplify `plan_sdd_init` so `--check` and `--diff` report provider materialization/import work and generated guide
  drift only. Remove synthetic move/delete actions and migration warnings while preserving the command's existing
  read-only preview guarantees.
- Delete migration-specific tests, including collision, malformed-frontmatter, flat-file sharding, prompt/bead rewrite,
  README cleanup, idempotency, and migration database-refresh cases. Replace them with focused init tests asserting that
  init refreshes canonical generated files and neither scans nor mutates arbitrary plan content.
- Update `sase sdd init -h`, command documentation, and generated README command descriptions so init is presented as
  store materialization plus generated-guide refresh, with no migration promise.

## Phase 3 — Make Python plan handling canonical-only

Consolidate the Python plan-tier helpers around the surviving responsibility: normalize/read a `tale|epic` tier from a
canonical plan file.

- Remove `PLAN_DIRS`, canonical↔legacy path candidate generation, stale-path existence retries, link alias iteration,
  and directory-derived tier fallback from `sdd/plan_tiers.py`. Classification of a canonical plan should come from
  frontmatter, with the existing best-effort tale fallback only where a display/search path must tolerate invalid
  content; validation continues to reject a missing or invalid tier.
- In `sdd/links.py`, scan only `plans/*/*.md` for plan artifacts. Resolve only canonical prompt/plan links after
  normalizing supported SDD-root prefixes; remove legacy-directory warnings and two-way alias resolution. Keep tier
  classification and tier-based list filtering.
- In `sdd/_paths.py`, remove `tales` and `epics` from recognized SDD directory names and remove their lookup aliases.
  Preserve the path-shape logic required for in-tree and companion SDD roots and the unrelated `prompts`/`specs`
  behavior.
- Remove `tales`/`epics` as `sase sdd path` choices and delete its deprecation redirect. Keep `plans` as the only plan
  child path. Polish list/path help and examples so logical tale/epic filters are clearly distinguished from physical
  directories.
- Replace plural `plan_kind` plumbing in `write_sdd_files`, exec-plan helpers, and their callers with an explicit
  singular plan tier. Remove acceptance of `plans`, `tales`, and `epics` as interchangeable writer inputs; map approval
  actions directly to `tale|epic` and always write under `plans/`.
- Remove old directory entries from the ACE diff badge and git commit-finalizer path classifiers, and remove bundled
  xprompt globs or examples that retain `@sdd/tales/**` / `@sdd/epics/**` fallbacks.
- Stop `sase plan list` from retrying stale legacy plan paths when deriving tier metadata. Current canonical paths keep
  their tier; irrecoverably stale historical metadata naturally displays `-`.
- Rewrite tests and fixtures that exercise current plan behavior to use `plans/` plus explicit tier. Delete dedicated
  legacy-layout regression cases. Opaque historical-path parsing tests may keep old strings only when the path is
  deliberately treated as data and no compatibility behavior is under test.

## Phase 4 — Make Rust discovery canonical-only

In the linked `sase-core` repository:

- Reduce repository plan discovery to `plans/` and `research/`. Remove `tales/` and `epics/` from `REPO_PLAN_KINDS`,
  related ordering documentation, and directory-fallback classification.
- Continue parsing `tier` after frontmatter is read and applying kind filters after classification. Canonical plans
  still default best-effort reads to tale for malformed/missing/unknown tier values so the core can return a result;
  Python validation remains the enforcement layer.
- Convert all discovery and public-search fixtures from legacy directories to `plans/` with explicit tier where the test
  distinguishes tales from epics. Retain coverage for tier normalization, missing/unknown/malformed tier fallback, kind
  filtering, deterministic ordering, title/time parsing, local archive behavior, and research isolation.
- Verify the pyo3 wire/API remains unchanged: `PlanWire.kind` continues to expose `tale`, `epic`, `research`, or
  `local`, so no binding or frontend schema migration is required.

## Phase 5 — Remove transition-era guidance and lock in the final contract

- Rewrite generated `SDD_README_CONTENT` to state the canonical directories and tier requirement directly. Remove the
  `tales/`/`epics/` compatibility and `sase sdd init` migration prose while retaining only unrelated compatibility notes
  that still apply. Regenerate/deploy any affected generated skill files through `sase skill init --force` and
  `chezmoi apply` if source skill templates change.
- Audit active README/docs/help/default-config text for claims that plans are stored in `tales/` or `epics/`, including
  SDD, ACE, commit-workflow, configuration, xprompt, and bead examples. Describe paths as `plans/` and classification as
  `tier: tale|epic`. Do not rewrite archived SDD plans/research solely to erase historically correct terminology.
- Add a narrow canonical-layout regression audit: source/config/current docs and operational tests must not contain
  plan-directory globs or write targets under `sdd/tales/` or `sdd/epics/`; the companion must not track either legacy
  directory. Exclude intentional historical corpora and append-only event logs from this check.

## Verification

- Companion repository:
  - confirm `git ls-files 'tales/**' 'epics/**'` is empty;
  - confirm all moved performance artifacts and the infographic exist under `plans/`;
  - confirm current prompt frontmatter and `beads/issues.jsonl` contain no live legacy paths;
  - run `sase sdd validate` against the companion store.
- Main SASE repository:
  - run focused SDD init, link/validation, plan inventory/list/search, writer/approval, diff-badge, commit-finalizer,
    and Phase 7 performance-helper tests;
  - run `just install` followed by the required `just check`;
  - run `just phase7-perf-check` (or its focused metadata/path tests if environment-sensitive timing makes the full
    benchmark unsuitable) to prove reports are emitted at the canonical path.
- Rust core:
  - run `cargo fmt --all -- --check`;
  - run clippy across the workspace with warnings denied;
  - run the full Rust workspace test suite.
- Final audit all three repositories for clean status and for transition-only symbols/path literals. Remaining matches
  must be either unrelated compatibility or explicitly historical data, not executable plan-layout behavior.

## Sequencing and landing

1. Move the companion support assets and update main-repository operational paths so no workflow recreates legacy
   directories.
2. Remove Python migration and read/write compatibility, update generated guidance, and make its tests canonical-only.
3. Remove Rust legacy discovery and convert core fixtures.
4. Run cross-repository verification, then commit each repository through the normal SASE commit finalizer workflow.

The companion move should land before or together with compatibility removal. Once the cleanup lands, rollback means
reverting the relevant repository commits; `sase sdd init` will no longer offer an automatic path back from legacy
directories.
