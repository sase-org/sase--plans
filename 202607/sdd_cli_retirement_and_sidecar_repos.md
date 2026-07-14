---
create_time: 2026-07-14 09:29:13
bead_id: sase-60
status: wip
prompt: 202607/prompts/sdd_cli_retirement_and_sidecar_repos.md
tier: epic
---
# Retire `sase sdd`, Make Sidecar Repos First-Class, and Generalize Repo Config

## Context & Goals

Recent work generalized SASE repos into primary / sidecar / linked / external kinds, but sidecar repos are still
conceptually welded to their legacy SDD foundations: they are derived at runtime from the SDD store record (or the
`<project>--plans` / `<project>--research` naming convention) rather than declared in configuration, and their CLI
surface lives on the legacy `sase sdd` command group. This plan completes the generalization:

1. Retire the `sase sdd` CLI entirely, migrating its durable functionality onto `sase repo` and `sase plan` and deleting
   its completed one-time migration helpers.
2. Introduce a `repos:` config field with `repos.linked` (successor to `linked_repos`) and a new `repos.sidecar` list so
   users can declare custom sidecar repos.
3. Make the implicit managed-project sidecar wiring explicit: `sase init` writes a `repos.sidecar` plans entry into the
   project's `sase.yml`, and entries support `disabled: true`.
4. Add a `sase repo init` command that creates (after interactive confirmation) all configured sidecar GitHub repos —
   public by default, configurable per repo — and absorbs `sase init workspace`; `sase init` delegates to it via a new
   `sase init repo` alias, mirroring the existing `memory`/`skills` alias pattern.
5. Convert `sase--research` into a custom, non-auto-cloned sidecar shared by **all** enabled projects (declared once in
   the global chezmoi-managed `sase.yml`), replacing the per-project `<project>--research` injection.

Important scoping fact discovered during research: the `sase.sdd` **Python package is a core dependency** of beads, the
commit finalizer, doctor, plan search, axe, and the TUI. Only the CLI surface (`src/sase/main/parser_sdd.py`,
`src/sase/main/sdd_handler.py`, `src/sase/main/sdd_init_config.py`) and completed migration modules are removed; the
package stays.

## Current State (key facts)

- `sase sdd` subcommands: `init`, `links`, `list`, `migrate`, `path`, `repair-links`, `validate`
  (`src/sase/main/parser_sdd.py`, `src/sase/main/sdd_handler.py`).
- Sidecars are injected, not declared: `inject_default_linked_repos()` (`src/sase/_linked_repo_config.py:92`) appends
  `<project>--plans` (`auto_clone: true`) and `<project>--research` (lazy) entries when the project-local `sase.yml` has
  `is_sase_managed: true` and `default_linked_repos` is not `false`. Sidecar identity otherwise comes from the SDD store
  record (`{primary}/.sase/sdd-store.json`) via `_sdd_sidecar_repo_dirnames()` (`src/sase/linked_repos.py:274`).
- Sidecar clones live at `<workspace>/sase/repos/<kind>` where `<kind>` is the logical dirname (`plans`, `research`),
  not the repo slug. Linked clones live under `sase/repos/linked/`, external under `sase/repos/external/`.
- `sase sdd path [beads|plans|research] [-e/--ensure]` prints one absolute path; `--ensure` materializes the backing
  sidecar clone (`ensure_sdd_kind_clone`, `src/sase/sdd/store.py:427`). Consumers include the chezmoi `#research`,
  `#research/more`, and `#research_swarm` xprompts and the generated `sase_beads` skill
  (`src/sase/xprompts/skills/sase_beads.md`).
- GitHub sidecar creation is provider-owned via pluggy hooks (`ws_preflight` / `ws_create_sdd_remote`,
  `src/sase/workspace_provider/_hookspec.py`), driven from `src/sase/sdd/_sidecar_init.py`, with a non-bypassable
  interactive default-No confirmation (`_confirm_sdd_sidecar_creation`, `src/sase/main/sdd_handler.py:208`).
- `sase init` onboarding runs an `InitCommandSpec` registry (memory, sdd, skills, workspace) from
  `src/sase/main/init_registry.py`; explicit `sase init memory|skills` are thin aliases for `sase memory init` /
  `sase skill init`, and `sase init sdd` re-dispatches into `handle_sdd_command`.
- `sase plan search` already supports query-less browsing over SDD-store and machine-local plans, with
  `--kind {tale,epic,research}`.
- All three enabled projects (sase, actstat, bob-cli) are already on the split `sidecar_repos` layout; the legacy-clone
  migration that `sase sdd migrate` performs is complete everywhere.
- Repo/config logic in this area is Python-owned (no `sase_core_rs` involvement); `src/sase/repo_inventory.py`
  explicitly documents itself as the future sase-core migration seam. This work stays in Python and preserves that seam.

## Target Design

### Config model

New top-level `repos` mapping, valid in both merged user config and project-local `sase.yml`:

```yaml
repos:
  linked: # successor to top-level `linked_repos`
    - name: sase-core
      path: ../sase-core
      description: ...
      auto_clone: true
  sidecar:
    - name: plans # role name; doubles as the sase/repos/<name> clone dirname and CLI lookup key
      auto_clone: true # sidecar default is false; the init-written plans entry sets true
    - name: research
      repo: sase-org/sase--research # optional pin; default derivation is <project>--<name> in the project's org
      description: Durable SASE research reports and generated media.
      visibility: public # creation-time visibility; public is the default, private supported per repo
      disabled: false # explicit opt-out that also suppresses the implicit plans fallback
```

Key-resolution chain and compatibility:

- `repos.linked` is canonical; `linked_repos` and `sibling_repos` remain readable deprecated aliases (same precedence
  pattern already used for `sibling_repos`), with doctor drift and Config Center visibility for the new field.
- Explicit `repos.sidecar` entries replace implicit injection when present (matched by name or derived slug). The
  implicit `<project>--plans` injection survives as a compatibility fallback for managed projects with no explicit
  entry, paired with doctor drift nudging `sase init`. The implicit `<project>--research` injection is deleted at
  cutover (Phase 5): research becomes a purely config-declared custom sidecar.
- Sidecar remote derivation: `repo:` accepts `owner/repo` or a bare slug; when omitted, the slug defaults to
  `<project>--<name>` in the project's own org (today's convention, so the init-written `plans` entry needs no extra
  fields). Materialization verifies an existing clone's `origin` against the expected remote and re-materializes (or
  reports) on mismatch — this is what safely re-points `sase/repos/research` clones from `<project>--research` to the
  shared `sase-org/sase--research`.
- The SDD store record remains the layout authority for the _plans/SDD store_ (legacy layouts included); the research
  half of the record becomes ignored-but-tolerated once research is config-declared.

### CLI surface

| Legacy                                 | Destination                                                             |
| -------------------------------------- | ----------------------------------------------------------------------- |
| `sase sdd path [kind] [-e]`            | New `sase repo path REPO [-e/--ensure]` (accepts role names and slugs)  |
| `sase sdd init`                        | New `sase repo init` (also absorbs `sase init workspace`)               |
| `sase init sdd`                        | New `sase init repo` alias → `sase repo init`                           |
| `sase init workspace`                  | Folded into `sase repo init`; subcommand removed                        |
| `sase sdd links/validate/repair-links` | New `sase plan links {list,validate,repair}` nested group (bare → list) |
| `sase sdd list`                        | Folded into query-less `sase plan search` (add a `prompt` kind)         |
| `sase sdd migrate`                     | Deleted (migration complete on every managed project)                   |
| `sase sdd` group itself                | Deleted                                                                 |

`sase repo path` design decisions:

- Read-only by default; `-e/--ensure` clones/synchronizes the backing sidecar (drop-in parity with `sase sdd path`
  semantics so `$(sase repo path research --ensure)` works in xprompts and scripts).
- Resolves **primary and sidecar** repos only. Linked and external repos still require the audited `sase repo open`;
  `path` refuses them with a pointer to `open`. This keeps the repo-open audit boundary intact while giving scripts an
  unaudited resolver for project-owned storage.
- The `beads` kind is not ported: callers use `$(sase repo path plans)/beads` (the `SASE_SDD_BEADS_DIR` env var remains
  the primary interface for agents).

`sase repo init` design:

- For the current project: (a) when `is_sase_managed: true` (or `sase init -M` just wrote it), ensure the project
  `sase.yml` has an explicit `repos.sidecar` plans entry (comment-preserving write; skipped when present or
  `disabled: true`); (b) for every enabled configured sidecar entry, preflight the remote, prompt to create a missing
  GitHub repo (per-repo interactive default-No confirmation naming host, repo, and visibility; `--yes` cannot grant it —
  same non-bypassable property as today), clone/adopt at `sase/repos/<name>`, seed README/assets (existing plans and
  research templates; a generic template for custom sidecars), commit/push, and maintain the plans store record for SDD
  compatibility; (c) ensure the tracked `/sase/repos/` gitignore rule (absorbed from `sase init workspace`), honoring
  `--check`, `--diff`, and `--no-commit`.
- Registered in the `InitCommandSpec` registry as a single `repo` step replacing `sdd` and `workspace` (bare `sase init`
  order becomes memory → repo → skills), following the planner/runner pattern of
  `src/sase/main/init_workspace_handler.py`.

## Phases

Each phase is a separate CL implemented by a distinct agent instance. Phases 2 and 3 both depend on Phase 1 and are
mutually independent; Phase 4 depends on 2 + 3; Phase 5 depends on all prior phases.

### Phase 1 — Config generalization: `repos.linked` + `repos.sidecar`

Scope (this repo only):

- Add the `repos` mapping to `src/sase/config/sase.schema.json` (new `sidecarRepo` definition: required `name`; optional
  `repo`, `description`, `auto_clone` default false, `visibility` enum public/private default public, `disabled` default
  false) and mark `linked_repos`/`sibling_repos` deprecated. Update `src/sase/default_config.yml` if defaults are
  surfaced there.
- Extend `src/sase/_linked_repo_config.py`: canonical-key chain `repos.linked` → `linked_repos` → `sibling_repos` (warn
  on divergent duplicates, as done for `sibling_repos`); new sidecar entry merge (global + project-local);
  explicit-entry override/dedup of the implicit injection; `disabled: true` suppression; keep implicit plans+research
  injection behavior otherwise unchanged in this phase.
- Generalize sidecar identity in `src/sase/linked_repos.py` (dirname/slug/remote derivation from config entries with
  store-record and naming-convention fallbacks; origin-URL identity check on existing clones) and surface
  config-declared sidecars in `src/sase/repo_inventory.py` (records carry both role name and slug; `sase repo open`
  matching accepts either).
- Doctor: deprecated-key drift for `linked_repos`/`sibling_repos`; Config Center picks the new field up from the schema.
- Docs: `docs/configuration.md` (`linked_repos` section becomes `repos`).
- Tests: extend `tests/test_linked_repo_resolution.py`, `tests/test_repo_inventory.py`,
  `tests/main/test_repo_handler.py`, schema/config tests; add coverage for pinned `repo:`, `disabled: true`, visibility
  parsing, and alias precedence.

Acceptance: existing projects behave identically with legacy config; a project declaring `repos.sidecar` entries sees
them in `sase repo list` with kind `sidecar` and can open them by role name or slug.

### Phase 2 — `sase repo path`

Scope (this repo only; depends on Phase 1):

- New `path` subcommand in `src/sase/main/parser_repo.py` + handler in `src/sase/main/repo_handler.py`: positional
  `REPO` (role name, slug, or primary name), `-e/--ensure`, `-p/--project`, `-w/--workspace`; prints exactly one
  absolute path on stdout; refuses linked/external repos with a `sase repo open` pointer. Follow CLI rules (alphabetized
  help, short aliases).
- `--ensure` reuses the generalized sidecar materialization from Phase 1 (today's `ensure_sdd_kind_clone` path).
- Update in-repo references from `sase sdd path` to `sase repo path`: `src/sase/main/parser_memory.py` examples, the
  `src/sase/xprompts/skills/sase_beads.md` skill source (`$(sase repo path plans)` / `$(sase repo path plans)/beads`),
  and `src/sase/sdd/templates/` README templates. Skill regeneration/deployment happens in Phase 5.
- Tests: new `sase repo path` handler tests modeled on `tests/main/test_sdd_parser_path.py` (in-tree, separate_repo,
  sidecar_repos layouts; ensure semantics; linked/external refusal).

Acceptance: `sase repo path research --ensure` and `sase repo path plans` are drop-in replacements for the
`sase sdd path` equivalents in every resolved layout; `sase sdd path` itself still works (removed in Phase 4).

### Phase 3 — `sase repo init` + `sase init` integration

Scope (this repo only; depends on Phase 1):

- New `src/sase/main/repo_init_handler.py` with `plan_repo_init`/`run_repo_init`/`handle_repo_init_command`, porting and
  generalizing `run_sdd_init` + `plan_sdd_init` (`src/sase/main/sdd_handler.py`) and `src/sase/sdd/_sidecar_init.py`
  from the fixed `("plans", "research")` pair to N configured sidecar entries, plus the gitignore-rule logic from
  `src/sase/main/init_workspace_handler.py`. Per-repo creation confirmation preserves the interactive-only, default-No
  contract and honors per-entry `visibility`.
- Writing the explicit plans entry: reuse the comment-preserving YAML write machinery used by `enable_sase_management`
  (`src/sase/project_management.py`) to add `repos.sidecar: [{name: plans, auto_clone: true}]` when managed and absent.
- Wire `init` into `src/sase/main/parser_repo.py` / `repo_handler._HANDLERS`; replace the `sdd` and `workspace` specs in
  `src/sase/main/init_registry.py` with one `repo` spec; remove the `sdd` and `workspace` subparsers from
  `src/sase/main/parser_init.py`, add the `repo` alias, and update dispatch in `src/sase/main/entry.py`. `sase sdd init`
  keeps working until Phase 4 by delegating to the new runner.
- Docs: `docs/init.md` command tables and flow description.
- Tests: port `tests/main/test_sdd_init_handler.py`, `tests/main/test_init_sdd_plan.py`, and
  `tests/main/test_init_workspace_handler.py` to the repo-init surface; update init onboarding tests (registry order,
  aliases, `-M` flow now also producing the sidecar entry); add coverage for custom-sidecar creation with pinned `repo:`
  and private visibility.

Acceptance: on a managed project, bare `sase init` plans/applies memory → repo → skills; `sase repo init` writes the
explicit plans entry, creates/adopts every configured sidecar after confirmation, and maintains the gitignore rule;
`sase init workspace` and `sase init sdd` are gone from help while bare onboarding covers both behaviors.

### Phase 4 — `sase plan links`, search folding, and `sase sdd` deletion

Scope (this repo only; depends on Phases 2 + 3):

- Add a nested `links` group to `src/sase/main/parser_plan.py` with `list` (default), `validate`, and `repair`
  subcommands routed through `src/sase/main/plan_command_handler.py`, reusing `sase.sdd.links` functions
  (`collect_sdd_links`, `validate_sdd_tree`, `repair_sdd_links`) with the same flags/exit codes as the sdd equivalents.
  The central default-`list` machinery gives bare `sase plan links` the list behavior.
- Extend `sase plan search --kind` with `prompt` so `sase sdd list`'s prompt inventory is not lost; tier browsing is
  already covered by query-less search.
- Delete the `sase sdd` CLI: `src/sase/main/parser_sdd.py`, `src/sase/main/sdd_handler.py`,
  `src/sase/main/sdd_init_config.py`, registration in `src/sase/main/parser.py`, dispatch in `src/sase/main/entry.py`.
  Delete completed migration machinery: `src/sase/sdd/migrate.py` and `src/sase/sdd/_prompt_migration.py` (and their
  call sites in the new repo-init runner). Keep the rest of the `sase.sdd` package.
- Sweep stale references: doctor next-step messages (`src/sase/doctor/checks_config_sdd.py`), living docs
  (`docs/cli.md`, `docs/sdd.md`, `docs/sdd_storage.md`, `docs/beads.md`, `docs/init.md`); leave published blog posts
  under `docs/blog/` as historical record.
- Tests: remove/port `tests/main/test_sdd_*` handler tests and the migrate coverage in
  `tests/sdd_store/test_split_init_and_migrate.py`; add `sase plan links` handler tests.

Acceptance: `sase sdd` no longer parses; every durable behavior is reachable via `sase repo path`, `sase repo init`,
`sase plan links`, and `sase plan search`; `just check` passes with no references to removed commands outside
`docs/blog/`.

### Phase 5 — Cutover: shared research sidecar + external config rollout

Scope (this repo + chezmoi + actstat + bob-cli project configs; depends on all prior phases):

- Code (this repo): remove the implicit `<project>--research` injection from `src/sase/_linked_repo_config.py` and the
  research entry injection in `src/sase/main/init_memory/config.py`; keep the implicit plans fallback + doctor drift;
  research paths/env (`SASE_SDD_RESEARCH_DIR`) now resolve through the config-declared `research` sidecar when one
  exists.
- Chezmoi (open via `/sase_repo`): in the global `home/dot_config/sase/sase.yml`, rename `linked_repos:` →
  `repos.linked` and add the shared research sidecar declaration
  (`repos.sidecar: [{name: research, repo: sase-org/sase--research, auto_clone: false, description: ...}]`) so every
  enabled project inherits it; update the `research` and `research/more` xprompts and
  `home/dot_xprompts/research_swarm.md` to `$(sase repo path research --ensure)`; drop the `alias sdd='sase sdd'` line
  from `home/dot_config/aliases.sh`. Commit and run `chezmoi update -a --force` per that repo's instructions.
- Project configs: rename `linked_repos` → `repos.linked` in this repo's `sase.yml`; open actstat and bob-cli via
  `/sase_repo` and do the same there; run `sase init` per project so the explicit plans entries get written and sidecar
  state is verified. The origin-identity check from Phase 1 re-points any stale `sase/repos/research` clones (previously
  `<project>--research`) at the shared remote.
- Regenerate and deploy skills (`sase skill init --force`, then chezmoi apply) so the deployed `sase_beads` skill uses
  `sase repo path`.
- Verify end-to-end: from each enabled project, `sase repo path research --ensure` prints the shared research clone and
  `sase repo list` shows `research` (kind sidecar, lazy) plus the explicit plans entry.
- Flag for manual user follow-up (not automated): archiving the now-orphaned `bbugyi200/actstat--research` and
  `bobs-org/bob-cli--research` GitHub repos after consolidating any content into `sase-org/sase--research`.

## Risks & Notes

- **Memory files**: some Tier-1/Tier-2 memory content mentions SDD paths (e.g. the bead-change exception references the
  `sdd/beads/` directory). Memory files must not be edited without explicit user permission — implementing agents should
  surface needed memory updates to the user instead of applying them.
- **Config schema sync** is the top repo gotcha: every phase touching config must update
  `src/sase/config/sase.schema.json` in the same CL.
- **Generated skills**: `src/sase/xprompts/skills/` sources change in Phase 2, but regeneration/deployment is deferred
  to Phase 5 so deployed skills never reference commands the installed `sase` doesn't have yet.
- **Cross-repo timing**: chezmoi/actstat/bob-cli edits land only in Phase 5, after the new CLI is merged, so the
  `#research*` xprompts never point at a command that doesn't exist.
- **Compatibility windows**: `linked_repos`/`sibling_repos` aliases and the implicit plans fallback stay until a later
  cleanup; `default_linked_repos: false` remains honored as a legacy spelling of disabling injected entries.
- The `sase.sdd` package keeps its name for now; renaming it (e.g. to a plans-store module) is deliberately out of scope
  to keep these CLs reviewable.
