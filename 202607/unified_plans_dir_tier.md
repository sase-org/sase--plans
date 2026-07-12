---
create_time: 2026-07-11 13:39:17
status: done
prompt: 202607/prompts/unified_plans_dir_tier.md
tier: tale
---
# Plan: Unify SDD `tales/` and `epics/` into a single `plans/` directory keyed by `tier` frontmatter

## Context

Today, SDD plan artifacts are classified by their storage directory: task-level plans live in
`<sdd_root>/tales/<YYYYMM>/*.md` and larger multi-phase plans in `<sdd_root>/epics/<YYYYMM>/*.md` (with `research/` as a
third, unrelated kind). This directory-based classification is duplicated everywhere: the Rust core's plan discovery
(`crates/sase_core/src/plan/read.rs` `REPO_PLAN_KINDS`), Python link validation (`sase/sdd/links.py`), path aliasing
(`sase/sdd/_paths.py`), the write paths (`sase/sdd/_write.py`, `sase/plan_approval_actions.py`,
`sase/workflows/commit/commit_hooks.py`, `sase/axe/run_agent_exec_plan_sdd.py` / `run_agent_exec_plan_accept.py`, the
ACE TUI approval modals), diff badges, the commit finalizer, generated READMEs, skill templates, and bead examples.

We want a single canonical `plans/` directory where the `tier` YAML frontmatter field (`tale` | `epic`) is the source of
truth for classification. `sase sdd init` migrates existing stores, readers keep legacy compatibility for unmigrated
stores, all writers set `tier` going forward, and `sase plan list` gets first-class tier display and filtering.

Facts that shape the design (from this project's SDD store and the code):

- The store currently holds ~2,365 sharded plan files (~2,097 tales, ~268 epics). 133 files already carry `tier: epic`
  (132 under `epics/`, **1 under `tales/`** — evidence that frontmatter must win over directory). Many old files have
  **no frontmatter at all**, and a few have **unparseable frontmatter** (validation has a legacy allowlist for these).
- History is inverted today: `plans/` is currently a _legacy alias for tales_ (`_SDD_PLAN_KIND_ALIASES`,
  `LEGACY_PLAN_KINDS`, and `_resolve_link_path`'s `plans/ → tales/` fallback). This plan reverses that relationship, and
  must keep old `plans/…` links (that meant tales) resolving correctly.
- The Rust core owns plan discovery/filter/rank for `sase plan search` (per the core-backend boundary, tier
  classification for discovery belongs there). Nothing in the core reads a `tier` key today; frontmatter is captured
  generically into `PlanWire.frontmatter`.
- Bead records persist plan paths in their `design` field (project-root-relative, e.g. `.sase/sdd/epics/…`), and
  prompts/plans cross-link via `plan:`/`prompt:` frontmatter — both break on a rename-only migration.
- The `tier` name collides with **bead** tiers (`plan`/`epic`), a separate concept. Docs and help text must disambiguate
  ("plan-file tier" vs "bead tier"); no bead schema changes are in scope.

## Design decisions

1. **Tier vocabulary**: `tier: tale` | `tier: epic`. `research/` is _not_ a tier and its directory is unchanged. Values
   are normalized on read (trim + lowercase). A plan file in `plans/` with a missing or unrecognized tier reads as
   `tale`, but `sase sdd validate` flags it (see Phase 5).
2. **Frontmatter wins everywhere**: a valid `tier` field overrides the containing directory, including for files still
   sitting in legacy `tales/`/`epics/` (fixes the misfiled tale that already says `tier: epic`). The directory is only a
   fallback for legacy files without the field.
3. **Canonical layout**: `plans/<YYYYMM>/<name>.md`. `tales/` and `epics/` become read-only legacy locations that
   discovery, link resolution, and `find_sdd_file` still understand until stores are migrated.
4. **Approval choices keep their names**: the "Tale"/"Epic" gate choices (and `sase plan approve --kind tale|epic`) now
   select the _tier value written into the plan_, not the destination directory. The user's approval choice overrides
   any tier the planner already wrote into the file.

## Phase 1 — sase-core: tier-aware plan discovery

In the sibling Rust core repo (`sase-core`, `crates/sase_core/src/plan/`):

- `read.rs`: additionally scan `<sdd_root>/plans/<YYYYMM>/*.md`. Kind for these files = normalized `tier` frontmatter
  (default `tale`). Keep the legacy `tales/`/`epics/` scans, but apply the same frontmatter-wins rule there. The `kinds`
  filter must now be applied _post-parse_ for tier-classified files (it is currently a per-directory read-time filter).
  Keep deterministic ordering (document where `plans/` sorts relative to legacy kinds).
- No wire/schema change required: `PlanWire.kind` keeps its `tale`/`epic`/`research`/`local` vocabulary and the
  `plan_search` pyo3 binding signature is untouched (the Python facade already passes roots + kinds). The raw `tier`
  value remains visible via the generic `frontmatter` map.
- Tests: `plans/` discovery; tier classification incl. default-to-tale, unknown values, malformed frontmatter;
  frontmatter-wins inside legacy dirs; `kinds` filtering across a mixed migrated/legacy corpus; ordering determinism.

## Phase 2 — sase: read-path compatibility + tier-setting writers

Read/alias compatibility:

- `sdd/_paths.py`: add `plans` to `SDD_CANONICAL_DIRS` (keep `tales`/`epics` listed so `looks_like_sdd_root` still
  recognizes unmigrated stores). Rework `sdd_kind_roots` aliases canonical-first: kinds `plans`/`tales` → (`plans`,
  `tales`); `epics` → (`plans`, `epics`).
- `sdd/links.py`: `list_sdd_files` scans `plans/` and classifies each file's kind from tier (plus legacy dirs with
  frontmatter-wins); validation's plan/prompt link-kind checks accept `plans/` files; `_resolve_link_path` gains two-way
  aliasing (legacy `tales/…`/`epics/…` links resolve into `plans/…` after migration, and old `plans/…` links keep
  resolving into `tales/…` for unmigrated stores); `repair-links` emits `plans/`-based link values when the file lives
  there.
- Introduce one small shared helper module for "read a plan file's tier" and "legacy↔canonical path alias candidates" so
  plan list, links, migration, and validation don't each reimplement it. (Discovery classification for search stays in
  the Rust core; these helpers are Python-side glue.)

Writers — every path that lands a plan in the SDD store writes to `plans/` and sets `tier`:

- `sdd/_write.py write_sdd_files`: write `plans/<YYYYMM>/`; accept the existing `plan_kind` values (`tales`, `epics`,
  `plans`) as aliases that map to a tier, and set the `tier` frontmatter field alongside the existing
  `prompt`/`create_time` handling.
- `plan_approval_actions._archive_plan_for_approval`: destination `plans/<YYYYMM>/`; set tier from the persisted
  approval action (`epic` → `epic`, else `tale`), overriding any pre-existing tier.
- `workflows/commit/commit_hooks.py` (plan copy safety net): destination `plans/`; preserve an existing valid tier, else
  set `tale`.
- `axe/run_agent_exec_plan_sdd.py` + `run_agent_exec_plan_accept.py` + ACE TUI `actions/agents/_notification_modals.py`:
  `plan_kind_for_action` and the plan-ref builders emit `sdd/plans/…` / `.sase/sdd/plans/…` paths; the epic-creation
  xprompt receives a `plans/`-based ref; the action→tier mapping travels alongside so the written file still gets the
  right tier.
- Path classifiers: `ace/tui/models/_diff_badge.py` `_SDD_PLAN_DIRS` += `plans`; `llm_provider/commit_finalizer_git.py`
  `_SDD_PLAN_DIR_PREFIXES` += `sdd/plans/` (keep legacy prefixes).
- `main/plan_search_render.py`: `_display_path` strips a leading `plans/` like it strips kind dirs today; kind styles
  unchanged (tale/epic labels now come from tier via the core).

## Phase 3 — `sase sdd init` migration

Add a migration step to `plan_sdd_init` / `run_sdd_init`, surfaced through the existing `--check` (report) and `--diff`
(preview) machinery as `InitAction`s, and ordered _after_ companion materialization and legacy-artifact import so
imported files are migrated in the same run:

For every sharded `.md` under `tales/` and `epics/` (excluding generated `README.md`s):

1. **Move** to `plans/<same shard>/<name>.md` — `git mv` when the store is git-backed (preserve history), plain rename
   otherwise. On a tales/epics name collision within a shard, dedup with a `_<n>` suffix and record the rename for link
   rewriting.
2. **Frontmatter**: keep an existing valid `tier`; otherwise set `tale` (from `tales/`) or `epic` (from `epics/`). Files
   with no frontmatter get a new block. When `create_time` is absent, set it from the file's pre-move mtime so recency
   ordering doesn't drift after the move touches the file. Files with **unparseable frontmatter** are moved but
   content-untouched (`set_frontmatter_fields` would prepend a second block and corrupt them) and reported as warnings.
3. **Link rewrite**: update prompt files' `plan:` fields for every legacy spelling (`tales/…`, `epics/…`, `sdd/…`,
   `.sase/sdd/…` prefixes), including dedup-renamed targets. Moved plans keep their `prompt:` links (targets didn't
   move).
4. **Bead design paths**: rewrite bead `design` fields that reference moved files, going through the bead mutation
   machinery so JSONL and the DB cache stay consistent.
5. **Cleanup**: delete the generated `tales/README.md` / `epics/README.md`, remove now-empty legacy directories, and
   shard any stray flat `.md` directly under `tales/`/`epics/` into `plans/<mtime YYYYMM>/`.
6. **Commit**: separate-repo stores commit via `commit_sdd_store_files` with a clear migration message; bare-git in-tree
   stores use the existing init commit path; plain in-tree stores leave changes for the normal commit flow and print
   what moved.

Properties: idempotent (re-run with nothing to migrate is a no-op — required since `ensure_sdd_initialized` runs from
many flows), per-file independent so a partial failure is completed by re-running, and skipped files (non-UTF-8,
unreadable) are reported rather than fatal.

Generated guides refresh (same phase): rewrite `SDD_README_CONTENT`'s layout + compatibility sections around `plans/`
and tier (documenting `tales/`, `epics/`, `specs/`, and old-style `plans/`-as-tales as legacy); replace the
`tales`/`epics` entries in `SDD_DIRECTORY_README_CONTENT` with a `plans` README that explains the tier field; teach init
to delete the superseded generated directory READMEs; regenerate the `sdd-directory-map.png` asset to show `plans/`
(same approach as the prior directory-map redesign work).

## Phase 4 — CLI: `sase plan list` tier support + command alignment

`sase plan list` (the proposal/approval pipeline dashboard):

- Every row (proposed / approved / rejected) gains a `tier` value parsed best-effort from the plan file's frontmatter;
  rows whose recorded path is stale after migration retry through the legacy↔canonical alias helper; unknown → `-`
  (typical for freshly proposed plans, which have no tier until approval).
- New repeatable `-t/--tier` filter (choices `tale`, `epic`) applied to all three sections; `--json` rows include
  `tier`; summary counts show a per-tier breakdown when a filter is active.
- Rich rendering: a compact Tier column, color-coded to match the approval modal (tale green, epic magenta).
- Follow the CLI rules memory: alphabetized options, short aliases, polished `-h` text with examples
  (`sase plan list --tier epic`).

Alignment elsewhere:

- `sase plan search`: no new flag — `--kind tale|epic|research` is now tier-driven via the core; update help text to say
  so.
- `sase sdd list`: `--kind` keeps `prompts|tales|epics|all` as tier-based filters over `plans/` + legacy dirs, and gains
  `plans` (all plan files regardless of tier); output kind labels come from tier.
- `sase sdd path`: add `plans` to the kind choices; `tales`/`epics` remain accepted but resolve to the `plans/`
  directory with a one-line deprecation note on stderr.
- Help/consequence strings mentioning directories: `plan approve --kind` help, `plan_approval_choices`
  `consequence_text` ("Commit to sdd/plans (tier: tale)" / "(tier: epic); launch bd/new_epic"), `bead/cli_admin.py`
  examples.

## Phase 5 — Templates, prompts, docs, and future enforcement

- `sase sdd validate`: plans-dir files with a missing or invalid `tier` are errors (post-migration everything valid has
  one; unparseable-frontmatter files already error, subject to the legacy allowlist). Files still living under
  `tales/`/`epics/` produce a warning nudging `sase sdd init` migration. This is the standing enforcement that the tier
  property "is always added" going forward, backstopping the writers from Phase 2.
- Skill templates (`src/sase/xprompts/skills/sase_beads.md`): update `${SASE_SDD_DIR}/tales/…` / `epics/…` examples to
  `${SASE_SDD_DIR}/plans/{YYYYMM}/…` and disambiguate plan-file tier vs bead tier. Follow the generated-skills process:
  `sase skill init --force`, then `chezmoi apply`.
- `src/sase/default_config.yml` bundled prompts: the `bd` epic-close prompt ("should be in the sdd/epics/ directory")
  and the `bd/review/plan` prompt's `@sdd/tales/**` globs → `plans/` (keep legacy globs alongside during the
  transition). Implementer should read `memory/xprompts.md` first per the long-term-memory rule.
- Out-of-repo follow-up to flag for the user in release notes: private chezmoi xprompts that write `tier: epic` or
  reference `sdd/epics/` (e.g. the epic-planning flow) deserve a pass; unmigrated other projects keep working via legacy
  read compat until `sase sdd init` is run there.

## Edge cases covered

- Plan files with no frontmatter (common pre-2026-03) → new block with tier + backfilled `create_time` from mtime.
- Unparseable frontmatter → move-only, never rewrite (avoids corruption); surfaced by validate.
- `tier` already present but contradicting the directory → frontmatter wins (observed in the wild).
- Unknown tier values (`tier: foo`) → read as tale, flagged by validate.
- tales/epics filename collision within the same `YYYYMM` shard → dedup suffix + link rewrite follows the rename.
- Old `plans/…` frontmatter links that historically meant `tales/` → still resolve (two-way alias).
- Prompt `plan:` links in all four prefix spellings; bead `design` paths; agent-meta `plan_path`s (read-side alias
  fallback in `sase plan list` since historical meta files can't be rewritten).
- Flat stray files directly under `tales/`/`epics/`; non-UTF-8/unreadable files (skip + report).
- All three store shapes: in-tree, `.sase/sdd` local, and separate-repo companion (including the
  legacy-import-then-migrate composition, and stores with a stale `sdd.stale-backup/` sibling which is never touched).
- Idempotency and partial-failure re-runs of `sase sdd init`; `--check`/`--diff` preview before any write.
- `~/.sase/plans/` machine-local archive is a different concept and is untouched (kind `local`); `research/` unchanged.
- Bead tier (`plan`/`epic`) vs plan-file tier naming collision handled in docs/help only.

## Test plan

- **sase-core**: unit tests per Phase 1 (cargo test).
- **sase**: unit tests for each touched module — alias/kind-root resolution, links list/validate/repair over migrated,
  unmigrated, and mixed stores; writer tests asserting `plans/` destinations + tier fields for approval archive,
  commit-hook copy, `write_sdd_files`, and axe/TUI plan refs; migration end-to-end tests (frontmatter variants,
  collisions, link + bead rewrites, README cleanup, idempotency, `--check`/`--diff` output, git-history preservation via
  `git mv`); `sase plan list` tier parsing/filter/JSON/rendering; `sase sdd list|path` behavior. Update the existing
  fixture-heavy suites that build `sdd/tales`//`sdd/epics` trees (test_sdd*, sdd_store/*, exec-plan approval tests,
  commit finalizer/workflow tests, plan search tests, diff-badge tests), while keeping a dedicated legacy-layout
  regression suite so unmigrated stores stay first-class.
- `just check` in sase (after `just install`); the Rust core repo's own checks for the core change.

## Sequencing

1. `sase-core` CL: additive `plans/` + tier discovery (safe to land first; nothing writes `plans/` yet).
2. `sase` CL(s), in order: (a) read compat + shared tier/alias helpers + writers, (b) `sase sdd init` migration +
   generated guides/asset, (c) `sase plan list` tier UX + CLI/help/docs/skill-template updates + validate enforcement.
3. User runs `sase sdd init` per project to migrate each store; unmigrated stores keep working throughout.
