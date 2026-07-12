---
create_time: 2026-07-11 17:27:30
status: done
prompt: 202607/prompts/migrate_sdd_prompts_into_plans.md
tier: tale
---
# Plan: Migrate SDD `prompts/` into `plans/<YYYYMM>/prompts/`

## Context

We just unified `tales/` and `epics/` into a single `plans/` directory (see the retired
`src/sase/sdd/_plan_migration.py` machinery and the "retire legacy plan layout" change). The next consolidation step is
to stop keeping prompt snapshots in a parallel top-level `prompts/` tree and instead nest them under the plan month
directory they belong to:

- **Old**: `prompts/<YYYYMM>/<name>.md` paired with `plans/<YYYYMM>/<name>.md`
- **New**: `plans/<YYYYMM>/prompts/<name>.md` paired with `plans/<YYYYMM>/<name>.md`

The months match one-to-one (a prompt in `prompts/202605/` moves to `plans/202605/prompts/`). After migration, the
top-level `prompts/` directory must be removed from the SDD store entirely, and every reference to the old paths must be
updated.

### Inventory of the current SDD store (this project's companion repo)

- 2,141 prompt files across `prompts/202603/` … `prompts/202607/` (no month-less strays, no READMEs, no non-markdown
  files). `plans/` additionally has a `202602/` month that has no prompts counterpart — that is fine; only existing
  prompt months get a nested `prompts/` subdir.
- Plan files carry a `prompt:` frontmatter link in three prefix shapes: `sdd/prompts/...` (~2,021),
  `.sase/sdd/prompts/...` (99), and bare `prompts/...` (2). One plan has an empty `prompt:` value.
- Two `legends/202605/*.md` files also carry `prompt: sdd/prompts/202605/...` links, so link rewriting must cover all
  SDD markdown kinds, not just `plans/`.
- Prompt files carry the reverse `plan:` link. Plans are not moving, so those links stay valid as-is.
- No bead `design` fields, ChangeSpec `.gp` files, or agent metadata reference `prompts/` paths.
- `sase-core` (Rust) treats a plan's `prompt_link` as an opaque display string and enumerates plans non-recursively
  (`plans/<shard>/*.md`), so nested prompt files will not be discovered as plans and **no Rust core changes are
  needed**. Python-side plan discovery (`links.py` globs `*/*.md`, `plan_inventory`) is likewise two-level and safe.

## Design decisions

1. **New canonical prompt location**: `plans/<YYYYMM>/prompts/<name>.md`. "prompts" remains a logical _kind_
   (`sase sdd list --kind prompts` keeps working); only its physical location changes.
2. **Follow the plans-migration precedent**: implement idempotent, collision-safe migration machinery that runs during
   `sase sdd init`, commits the result to the SDD store with `commit_sdd_store_files`, and can be retired in a follow-up
   once all stores are migrated (same lifecycle as `_plan_migration.py`).
3. **Legacy read compatibility**: keep the legacy `prompts/` and `specs/` directories readable through the existing
   kind-root alias mechanism during the transition, so unmigrated SDD stores in other projects still resolve until their
   next `sase sdd init`. Retiring these aliases is an explicit follow-up, not part of this change.
4. **Reference rewriting scope**: rewrite frontmatter `prompt:` links (all three prefix shapes) across every SDD
   markdown kind (`plans/`, `legends/`, `myths/`, `research/`). Prose/body mentions of old prompt paths inside
   historical documents are left unchanged, consistent with the plans migration.
5. **Uniform runtimes**: nothing here is agent-runtime-specific; all changes live in shared SDD path/link/write logic.

## Phased Implementation

### Phase 1: Teach readers and writers the new layout

**Goal**: All code that writes or resolves SDD prompt paths uses `plans/<YYYYMM>/prompts/` as the canonical location,
with legacy locations still readable.

- `src/sase/sdd/_write.py` — `write_sdd_files()` writes the prompt snapshot to `plans/<YYYYMM>/prompts/<name>.md` (same
  month dir as the plan). `sdd_link_path()` needs no change; emitted links become `sdd/plans/<YYYYMM>/prompts/<name>.md`
  / `.sase/sdd/plans/...` automatically.
- `src/sase/sdd/_paths.py` —
  - `find_sdd_file(base, "prompts", name)` must search the nested layout first (`plans/*/prompts/<name>` under each
    plans root) and then the legacy `prompts/`/`specs/` roots (flat and `*/<name>` forms), keeping the existing
    deterministic first-match behavior. Callers (`commit_hooks._infer_prompt_link`,
    `axe/run_agent_exec_plan_sdd.commit_sdd_files_for_exec_plan`) then work unchanged.
  - `SDD_CANONICAL_DIRS`: drop `prompts` from the `sase sdd path <kind>` choices (there is no single prompts directory
    anymore), while keeping `prompts` in the root-detection set used by `looks_like_sdd_root` so unmigrated stores are
    still recognized.
- `src/sase/sdd/links.py` —
  - `list_sdd_files()`: enumerate prompts from `plans/*/prompts/*.md` (kind `prompts`) in addition to legacy
    `prompts|specs/*/*.md`; keep plan enumeration at `plans/*/*.md` so nested prompts are never classified as plans.
  - `_read_sdd_file()`: accept the 4-part nested relpath (`plans/<YYYYMM>/prompts/<file>`), keeping `yyyymm` at part
    index 1 so counterpart inference (same month + name) keeps pairing prompts with plans.
  - Update the two `LEGACY_INVALID_SDD_ERROR_ALLOWLIST` relpaths to their new `plans/202605/prompts/...` locations.
  - `validate_sdd_tree()` / `repair_sdd_links()` logic is kind-based and should keep working once listing and relpath
    parsing understand the new layout; extend tests to prove it.
- `src/sase/prompt/search/sources.py` — `_sdd_prompt_roots()` must yield the nested prompt dirs (glob `plans/*/prompts`
  under each SDD base, including the project-local `.sase/sdd` store) alongside the legacy roots;
  `load_sdd_prompt_hits()`'s rglob-over-roots and de-dup then work unchanged.
- `src/sase/prompt/cli_export.py` — the `--sdd` export target becomes `<sdd_dir>/plans/<YYYYMM>/prompts/<basename>.md`.
- `src/sase/default_config.yml` — the two launch prompts referencing `@sdd/prompts/**/{{ file_base }}.md` must point at
  the new location (e.g. `@sdd/plans/**/prompts/{{ file_base }}.md`); verify the chosen glob resolves the prompt file
  (and not the plan, which shares its basename) through the file-reference expansion machinery.
- `src/sase/ace/tui/models/_diff_badge.py` — verify the bookkeeping classifier still recognizes new-style paths
  (`sdd/plans/...` already matches via the `plans` segment); keep `prompts` in `_SDD_PLAN_DIRS` for legacy diffs and add
  a regression test for the nested path.

**Tests**: update/extend `tests/test_sdd_paths.py` (find*sdd_file nested + legacy precedence), `tests/test_sdd.py`
(write_sdd_files targets, links/validate/repair round-trips in the new layout),
`tests/prompt_command/test_search_sources.py`, `tests/prompt_command/test_export_save.py`,
`tests/ace/tui/models/test_diff_badge.py`, plus the commit-workflow and axe exec-plan tests that fabricate
`prompts/<YYYYMM>/` fixtures (`tests/test_commit_workflow_artifacts.py`, `tests/workflows/test_commit_workflow.py`,
`tests/test_axe_run_agent_exec_plan*\*`).

### Phase 2: Migration machinery

**Goal**: An idempotent migration that relocates prompt files and rewrites references, mirroring the retired
`_plan_migration.py` design.

- New `src/sase/sdd/_prompt_migration.py`:
  - Plan actions: for every `prompts/<YYYYMM>/*.md` (and `specs/` if present), destination
    `plans/<YYYYMM>/prompts/<file>`; month-less strays shard by file mtime (precedent behavior); collision-safe
    `_1`-suffix dedup against existing destinations; unreadable files produce warnings, never abort the run.
  - Move via `git mv` when the store is a git repo, falling back to rename (reuse the precedent `_move`/`_git_root`
    helpers).
  - Build an old→new relpath map and rewrite frontmatter `prompt:` fields across all SDD markdown files (`plans/`,
    `legends/`, `myths/`, `research/`) using the precedent `_rewrite_reference` prefix-variant matching (`""`, `"sdd/"`,
    `".sase/sdd/"`, and suffix matching).
  - Clean up: remove now-empty `prompts/` month dirs and the top-level `prompts/` directory itself.
  - Idempotent: a second run finds nothing to do and performs no writes.
- Wire into `handle_sdd_init` (`src/sase/main/sdd_handler.py`) exactly like the plans migration was: run after
  `materialize_sdd_store`, print warnings, and when the store is not in-tree commit moved+changed paths via
  `commit_sdd_store_files` with a message like "Migrate SDD prompts into plan month directories".

**Tests**: new `tests/test_sdd_prompt_migration.py` covering the move+rewrite happy path (all three link prefix shapes),
legends-link rewriting, collision dedup, month-less shard fallback, prompts-dir removal, idempotent re-run, and the init
wiring/commit behavior (mirror the structure of `tests/test_sdd_plan_tiers.py`'s migration tests).

### Phase 3: Documentation and generated guides

- `src/sase/sdd/_init_files.py` — rewrite `SDD_README_CONTENT`'s Directory Layout, example paths, link examples, and the
  Compatibility section (canonical dirs become `plans/`, `research/`, `beads/`; prompts live in
  `plans/<YYYYMM>/prompts/`; legacy `prompts/`/`specs/` remain readable until retired). Extend the `plans/` directory
  README to document the nested `prompts/` subdirectory.
- `docs/sdd.md` and `docs/prompt.md` — update layout descriptions, example frontmatter, and the
  `sase prompt export --sdd` target path.
- The `assets/sdd-directory-map.png` guide image still shows a top-level `prompts/` directory. It is a manually produced
  asset with no in-repo generation script — update it if feasible, otherwise flag it explicitly in the PR as a known
  follow-up so the generated README and image do not silently disagree.

### Phase 4: Apply the migration to this project's SDD store

**Goal**: The actual data move the user asked for.

- Run `sase sdd init` in the project workspace so the new migration executes against the companion SDD store
  (`.sase/sdd`, backed by `sase-org/sase--sdd`).
- Verify:
  - `prompts/` no longer exists in the store; `plans/<YYYYMM>/prompts/` holds all 2,141 files with months matched.
  - `sase sdd validate` passes (no new errors; the two legacy-allowlisted files still surface as warnings at their new
    paths).
  - `sase sdd list --kind prompts` and `sase sdd links` show the migrated paths with bidirectional links intact.
  - Spot-check: a migrated plan's `prompt:` link resolves (all three original prefix shapes), the two legends links
    resolve, and `sase plan search` output does not list prompt snapshots as plans.
  - The migration committed (and pushed per store policy) to the companion repo as a single migration commit.

## Verification

- `just check` (after `just install`) for the code changes, plus targeted `pytest` runs of the SDD, prompt-command,
  commit-workflow, and axe exec-plan suites.
- End-to-end: create a throwaway plan via the normal plan-approval flow in a scratch store and confirm the prompt
  snapshot lands in `plans/<YYYYMM>/prompts/`, the plan's `prompt:` link points there, and `sase sdd validate` accepts
  the pair.

## Follow-ups (out of scope)

- Retire the legacy `prompts/`/`specs/` read aliases and delete `_prompt_migration.py` once all SDD stores are migrated
  (mirrors the "retire legacy plan layout" change).
- Regenerate the SDD directory-map image if it cannot be updated in this change.
