---
tier: epic
title: Per-project research sidecar repos
goal: 'Every enabled SASE project declares, resolves, and documents its own <project>--research
  sidecar repo: generated agent instruction files list non-auto-cloned sidecar repos
  again, sase init writes (and can create) the per-project research entry in each
  project''s sase.yml, and the shared sase-org/sase--research pin is removed from
  the global chezmoi-managed config.

  '
phases:
- id: memory-render
  title: Render sidecar repos in generated agent instruction files
  depends_on: []
- id: per-project-research
  title: Per-project research derivation + init-written sidecar entry
  depends_on: []
- id: cutover
  title: Config cutover across chezmoi and all enabled projects
  depends_on:
  - memory-render
  - per-project-research
create_time: 2026-07-15 08:28:04
status: wip
prompt: 202607/prompts/per_project_research_sidecars.md
bead_id: sase-62
---

# Plan: Per-project research sidecar repos

## Context & Problem

The sase-60 epic made sidecar repos config-declared (`repos.sidecar`) and, at cutover (sase-60.5), converted research
into a single shared sidecar: `repo: sase-org/sase--research` declared once in the global chezmoi-managed
`home/dot_config/sase/sase.yml`. Two things about that end state are wrong and must be reversed/fixed:

1. **Regression — sidecar repos vanished from agent instruction files.** Before sase-60, managed projects auto-injected
   a `<project>--research` _linked_ entry into memory generation, so `memory/sase.md` (and the AGENTS.md/CLAUDE.md/…
   shims built from it) contained a bullet like
   ``- `sase--research`: Durable SASE research reports and generated media.`` Commit `8c716fa74` (sase-60.5) deleted
   that injection, and **no code path renders `repos.sidecar` entries into memory files** — the memory pipeline only
   reads `repos.linked` (or its deprecated aliases). Agents therefore no longer learn the research repo exists or that
   `sase repo open` / the `/sase_repo` skill is how to access it. (The follow-up regeneration that dropped the bullet
   from this repo's instruction files is commit `dc7c38387`.)
2. **Wrong design — one shared research repo for all projects.** Each enabled project should use its **own** research
   sidecar in its own GitHub org (`bbugyi200/actstat--research`, `bobs-org/bob-cli--research`,
   `sase-org/sase--research`), created by `sase init` when missing. The declaration must live in each **project repo's**
   `sase.yml` (not chezmoi), so `sase init` generates identical agent instruction files for a project on any machine.

## Current State (key facts, all verified)

- Memory generation: `_load_memory_inputs` (`src/sase/main/init_memory_handler.py:94`) reads the **project-local**
  `sase.yml` for project memory and the **global** config for home memory — strictly separated. Entries come from
  `linked_entries_from_config` (`src/sase/main/init_memory/config.py:163`), which only consults `repos.linked` /
  `linked_repos` / `sibling_repos` (`config.py:146-160`), skips `auto_clone: true` entries, and renders each entry via
  `_linked_repo_list_item` (`src/sase/main/init_memory/root_rendering.py:52`) into the `{{ linked_repo_entries }}` slot
  of `src/sase/main/init_memory/templates/memory-sase.template.md` (heading + "Configured linked repositories for this
  context:" + `/sase_repo` boilerplate). `LinkedRepoMemoryEntry` (`src/sase/main/init_memory/models.py:14`) requires a
  `path`, which sidecar entries do not have. The `project_name`/`primary_root` params of `linked_entries_from_config`
  are vestigial leftovers of the deleted research injection and are available for slug derivation.
- Sidecar config: `repos.sidecar` entries merge global-then-project-local with per-role field override
  (`merged_sidecar_entries_from_config`, `src/sase/_linked_repo_config.py:126`). Schema (`sase.schema.json:35-72`)
  already allows `name`, `repo`, `description`, `auto_clone`, `visibility`, `disabled` — no schema additions needed.
- Slug derivation: when `repo:` is omitted, `_sidecar_repo_identity` (`src/sase/_linked_repo_config.py:200-241`) uses
  `repo = store_repo or _derived_sidecar_repo(primary, role)` — i.e. the SDD store record
  (`{primary}/.sase/sdd-store.json`) **beats** the `<owner>/<project>--<role>` org-convention derivation
  (`_derived_sidecar_repo`, `:392-399`). The actstat and bob-cli store records currently pin
  `research → sase-org/sase--research` (written during the sase-60.5 cutover), so a bare per-project `{name: research}`
  entry would today still resolve to the shared repo. The sase-60 design explicitly intended the research half of the
  store record to become "ignored-but-tolerated" once research is config-declared — the current precedence does not
  implement that intent for explicit entries.
- `sase repo init` (`src/sase/main/repo_init_handler.py`) iterates every enabled configured sidecar entry
  (`_configured_sidecar_specs`, `:384-435`), preflights remotes, and creates missing GitHub repos only after a
  non-bypassable interactive default-No confirmation (`_confirm_sidecar_creation`, `:275-310`; `--yes` cannot grant it).
  It writes exactly one explicit config entry — `{name: plans, auto_clone: true}` — via the hardcoded
  `_explicit_plans_config_update` (`:443-506`), using the comment-preserving ruamel machinery (`set_key`/`dump_yaml`).
  Bare `sase init` runs memory → repo → skills (`src/sase/main/init_registry.py`).
- `sase repo path research --ensure` and `sase repo open` both resolve configured sidecars by role name or slug;
  materialization re-points clones whose origin mismatches the expected remote
  (`_materialize_remote_identified_sidecar`, `src/sase/linked_repos.py:573-618`).
- Rollout inventory: enabled projects are `sase` (org `sase-org`), `actstat` (org `bbugyi200`), `bob-cli` (org
  `bobs-org`). All three `<project>--research` GitHub repos exist and are **not** archived, so cutover is
  adoption/re-pointing, not creation. `DEFAULT_RESEARCH_DESCRIPTION` ("Durable SASE research reports and generated
  media.") still exists at `src/sase/_linked_repo_config.py:25` and matches the description currently in the chezmoi
  global entry.
- Chezmoi global config (`home/dot_config/sase/sase.yml` in the chezmoi repo) declares the shared entry under
  `repos.sidecar` and defines the `#research`, `#research/more`, and `#research_swarm` xprompts in terms of
  `$(sase repo path research --ensure)` — those keep working unchanged once each project declares its own entry.

## Target Design

1. **Instruction files**: generated project memory lists every _enabled, non-auto-cloned_ `repos.sidecar` entry from the
   project-local `sase.yml` alongside the linked-repo bullets (home memory does the same from the global config).
   Bullets render the **slug** (e.g. `sase--research`) plus the entry description, restoring the pre-regression format;
   slugs and role names both already resolve through `sase repo open`. Auto-cloned sidecars (plans) stay unlisted,
   exactly like auto-cloned linked repos, because agents reach them via `sase repo path` without an audited open.
2. **Config**: each managed project's `sase.yml` carries
   `repos.sidecar: [{name: plans, auto_clone: true}, {name: research, description: "Durable SASE research reports and generated media."}]`.
   No `repo:` pin — the org-convention derivation produces `<org>/<project>--research` per project. `sase repo init`
   (and therefore bare `sase init`) writes both entries idempotently for managed projects, so future projects get the
   same wiring automatically, with an identical description sourced from `DEFAULT_RESEARCH_DESCRIPTION`.
3. **Identity precedence fix**: for entries _explicitly declared_ in `repos.sidecar` with `repo:` omitted, the
   org-convention derivation wins over a conflicting store-record repo; the store record only contributes (e.g. its SSH
   remote form) when it agrees with the derived slug — mirroring the existing remote-URL consistency check at
   `_linked_repo_config.py:224-230`. The injected implicit plans fallback keeps its current store-first behavior (the
   store record remains the layout authority for plans/legacy layouts). `sase repo init` re-records the store so it
   converges on the per-project repo.
4. **Global config**: the chezmoi `repos.sidecar` research declaration is deleted; chezmoi no longer defines research
   wiring for projects.

## Phases

### Phase `memory-render` — Render sidecar repos in generated agent instruction files

Scope (sase repo only):

- Extend the memory-input layer (`src/sase/main/init_memory/config.py` + `models.py`) to also read `repos.sidecar` from
  the same config file already used per root (project-local `sase.yml` for project memory; global config for home
  memory). Produce render entries for sidecar items that are not `auto_clone: true` and not `disabled: true`. Reuse or
  generalize `LinkedRepoMemoryEntry` (path must become optional or a sibling model added — only `name` and `description`
  are rendered).
- Slug for the bullet name: basename of a pinned `repo:` when present; otherwise `<project>--<name>` derived from the
  project name the pipeline already resolves (`project_memory_name` / the vestigial `project_name`/`primary_root`
  params). For the home root (no project), fall back to the pinned basename or the role name.
- Validation: rendered sidecar entries require a `description`, with a config error consistent with the linked-repo
  message (`config.py:194-200`) — the init-written entry from phase `per-project-research` always carries one, and
  auto-cloned/disabled entries are exempt because they are skipped before validation.
- Template: update `templates/memory-sase.template.md` so the intro line covers sidecars too (e.g. "Configured linked
  and sidecar repositories for this context:"); keep the `/sase_repo` boilerplate unchanged. Preserve the empty-state
  fallback line semantics.
- Tests (`tests/main/test_init_memory_handler.py`, `tests/main/test_init_memory_markdown_templates.py`): project sidecar
  renders in project memory only (not home), auto-clone and disabled entries are skipped, missing description errors,
  pinned-`repo:` slug vs derived slug, and the restored bullet format
  ``- `sase--research`: Durable SASE research reports and generated media.``
- Docs: `docs/configuration.md` `repos.sidecar` section documents that enabled non-auto-cloned sidecars appear in
  generated agent instruction files.

Acceptance: with a project `sase.yml` declaring the research entry, `sase init` regenerates `memory/sase.md` and the
provider shims with the research bullet and updated intro line; a config with only linked repos renders exactly as today
apart from the intro wording. `just check` passes.

### Phase `per-project-research` — Per-project research derivation + init-written sidecar entry

Scope (sase repo only; independent of `memory-render`):

- **Identity precedence**: in `_sidecar_repo_identity` (`src/sase/_linked_repo_config.py:200-241`), make explicitly
  config-declared entries with `repo:` omitted prefer `_derived_sidecar_repo` over a conflicting `store_repo` (store
  data only refines the result when its basename matches the derived slug). Keep store-first behavior for the injected
  implicit plans fallback (distinguishable via the default-entry marker). Extend `tests/test_linked_repo_resolution.py`
  (poisoned/preserved store-remote tests around `:131-209`) with the stale-store-repo case: bare `{name: research}` +
  store record pointing at a foreign slug must resolve to `<org>/<project>--research`.
- **Init-written research entry**: generalize `_explicit_plans_config_update`
  (`src/sase/main/repo_init_handler.py:443-506`) into a per-role writer that ensures both managed-project entries:
  `{name: plans, auto_clone: true}` and `{name: research, description: DEFAULT_RESEARCH_DESCRIPTION}` (no `repo:` pin,
  `auto_clone` left at its false default so research stays lazy). Idempotency per role name (an existing entry —
  including `disabled: true` — is never rewritten); comment-preserving output; `plan_repo_init` (`--check`/`--diff`) and
  the action summaries (`:104`, `_summarize_repo_actions`) reflect both entries.
- **Store convergence**: `sase repo init` re-records the sidecar store so the research half points at the per-project
  repo once resolution derives it (verify via the existing `initialize_sidecars` store-maintenance path and cover with a
  test).
- **Creation contract unchanged**: missing `<org>/<project>--research` remotes still require the interactive default-No
  confirmation; `--yes` and non-TTY runs still cannot authorize creation. Verify that a non-interactive `sase init -y`
  (the post-commit hook) degrades gracefully — reports the missing remote with guidance to run `sase repo init`
  interactively rather than hard-failing the hook — and adjust if it does not.
- Tests: extend `tests/main/test_repo_init_handler.py` / `test_repo_init_plan.py` (both entries written, idempotency,
  comment preservation, disabled-entry respected, existing-remote adoption without prompt, non-interactive behavior) and
  init onboarding coverage (bare `sase init -M` flow writes both entries).
- Docs: `docs/init.md` (repo step now writes the plans _and_ research entries) and `docs/configuration.md` (per-project
  research derivation, `disabled: true` opt-out). No `sase.schema.json` changes are expected (all fields exist), but
  re-check the schema-sync gotcha before landing.

Acceptance: on a managed project with no explicit research entry, `sase repo init` writes the entry, resolves
`<org>/<project>--research` even when a stale store record points elsewhere, adopts an existing remote without a
creation prompt, and re-records the store; `just check` passes.

### Phase `cutover` — Config cutover across chezmoi and all enabled projects

Scope (chezmoi + sase + actstat + bob-cli configs; depends on both prior phases; requires the installed `sase` CLI to
include them — verify before starting and upgrade/reinstall if the deployed build predates the prior phases):

- **Chezmoi** (open via `/sase_repo`): delete the `repos.sidecar` research entry from `home/dot_config/sase/sase.yml`
  (keep `repos.linked`). Leave the `#research*` xprompts unchanged — `$(sase repo path research --ensure)` now resolves
  per project. Regenerate home-root agent instruction files if the memory template wording changed. Commit, then run
  `chezmoi update -a --force` per that repo's instructions.
- **sase project**: run `sase init` so the research entry is written into this repo's `sase.yml` and `memory/sase.md` +
  provider shims regain the `` `sase--research` `` bullet; commit. (Research remote stays `sase-org/sase--research` — it
  is the sase project's own org-derived repo.)
- **actstat and bob-cli** (open via `/sase_repo`): run `sase init` per project; confirm each project's `sase.yml` gains
  the research entry, instruction files gain the `` `actstat--research` `` / `` `bob-cli--research` `` bullet, and the
  store-record research half re-points to `bbugyi200/actstat--research` / `bobs-org/bob-cli--research`. Commit per
  project. Both remotes already exist and are unarchived, so no creation prompts are expected.
- **Verify end-to-end per enabled project**: `sase repo list` shows `research` (kind sidecar, lazy);
  `sase repo path research --ensure` prints the project-local research clone whose `origin` is the project's own
  `<org>/<project>--research` remote (the origin-identity check re-points clones previously aimed at
  `sase-org/sase--research`); `sase repo open research` and `sase repo open <project>--research` both resolve.
- **Do not** delete, archive, or rewrite `sase-org/sase--research` — it remains the sase project's research repo.
- Flag for manual user follow-up (not automated): research content that actstat/bob-cli agents wrote into
  `sase-org/sase--research` since the sase-60.5 consolidation stays there until the user decides whether to move it back
  into the per-project repos (the old per-project repos were never archived, so duplicated pre-consolidation content may
  also need triage); and the `home` project no longer resolves any research sidecar once the global entry is gone
  (pre-sase-60 status quo) — if `#research` should work from home-context agents, the user must decide where to declare
  that separately.

Acceptance: no `repos.sidecar` research declaration remains in chezmoi; every enabled project's own repo declares (and
its instruction files document) its org-local research sidecar; verification commands above pass on all three projects.

## Testing

- Phases `memory-render` and `per-project-research` each extend the unit suites named in their scope and must pass
  `just check` in the sase repo.
- Phase `cutover` is verified operationally with the per-project command checklist above; it makes no sase-repo code
  changes beyond what `sase init` regenerates (config entry + instruction files), so `just check` applies only if
  sase-repo files change.

## Risks & Notes

- **Memory files require explicit user permission**: the instruction-file changes in this epic are exactly what the user
  requested, and they are produced by running `sase init` (the sanctioned generator) — implementing agents must not
  hand-edit `memory/*.md`, `AGENTS.md`, or provider shims beyond that regeneration.
- **Stale store records are the sharp edge**: without the phase `per-project-research` precedence fix, bare research
  entries on actstat/bob-cli would keep resolving to `sase-org/sase--research` via their store records. The cutover
  phase must confirm store convergence rather than assume it.
- **Dirty clones**: primary-workspace research clones with dirty worktrees are deliberately not replaced on origin
  mismatch (`src/sase/linked_repos.py:587-606`); if verification hits one, surface it to the user instead of forcing.
- **Cross-repo timing**: chezmoi/actstat/bob-cli edits happen only in the cutover phase, after the code phases are
  merged and the installed CLI is upgraded, so `sase init` never runs against code that cannot write or render the new
  entries.
- **Template wording change** touches every generated instruction-file set (home + all projects); each is refreshed by
  its own project's `sase init` run during cutover, and other machines converge the next time their post-commit
  `sase init -y` hook runs.
