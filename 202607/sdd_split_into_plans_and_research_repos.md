---
create_time: 2026-07-11 19:07:12
status: done
prompt: 202607/prompts/sdd_split_into_plans_and_research_repos.md
bead_id: sase-5q
tier: epic
---
# Plan: Split the SDD companion repo into `--plans` and `--research` linked repos

## Product context / goal

Today each SASE project stores durable planning context in a single SDD companion GitHub repo (`<owner>/<repo>--sdd`,
e.g. `sase-org/sase--sdd`) cloned per-checkout at `.sase/sdd/`, containing `plans/<YYYYMM>/` (with nested
`plans/<YYYYMM>/prompts/` snapshots), `research/<YYYYMM>/`, `beads/`, a generated `README.md`, and
`assets/sdd-directory-map.png`.

This plan retires that single companion in favor of **two public linked repos**, generalizing the linked-repo concept to
support them:

- **`<repo>--plans`** (e.g. `sase-org/sase--plans`): the flattened contents of `plans/` **plus the `beads/` directory**.
  "Flattened" means the `plans/` kind directory disappears — the monthly `<YYYYMM>/` directories (each still containing
  its plan files and nested `prompts/` snapshots) move to the repo root, next to `beads/`, `assets/`, and the generated
  `README.md`. Configured with the new `auto_clone: true`, so it is cloned/synced automatically during workspace
  preparation before every agent launch.
- **`<repo>--research`** (e.g. `sase-org/sase--research`): the flattened contents of `research/` — monthly `<YYYYMM>/`
  directories at the repo root, next to `assets/` and the generated `README.md`. `auto_clone` defaults to `false`; the
  clone materializes lazily.

Alongside the migration, the linked-repo concept gains:

1. A new per-entry **`auto_clone` config field** (default `false`) on `linked_repos` entries: when `true`, the linked
   repo is cloned/synced while preparing the workspace directory before launching an agent.
2. **Agent-file exclusion**: linked repos with `auto_clone: true` are omitted from the generated "Linked Repositories"
   section in agent instruction files (`AGENTS.md` and provider shims) — they are always present in the workspace, so
   agents never need the `sase workspace open` dance for them.
3. **Default companion entries**: any project whose project-local `sase.yml` has `is_sase_managed: true` gets both
   companion linked repos injected by default (explicit same-name entries override; a new project-local
   `default_linked_repos: false` opts out entirely).
4. **`sase init` generates the GitHub repos**: initialization creates the `--plans`/`--research` companion repos when
   missing — **public**, like today's `--sdd` creation — and stamps each with a generated `README.md` that embeds a
   **GPT-image infographic** (packaged static asset, drift-tracked, produced by a dedicated image phase).

User-confirmed constraints from review of the previous draft:

- Companion repos are **public** (the existing creation path already passes `--public`; keep it that way).
- The legacy `tales/`, `legends/`, `myths/` tier directories are dead and are **not migrated** (a few tracked straggler
  files remain in `sase--sdd`; they stay behind in the archived repo).
- Flattening removes only the `plans/`/`research/` kind directory — the `<YYYYMM>/` monthly directories are **kept** and
  move to the companion repo root.

Interfaces that must keep working: `sase plan propose/approve` flows, `sase bead` commands, the `#research*` xprompts
and `#research_swarm` (defined in the chezmoi repo), `sase sdd path/list/validate/links/repair-links`,
`sase plan list/search`, and the `sase validate` drift gates.

---

## How it works today (verified grounding)

**Linked repos** (`src/sase/linked_repos.py`):

- Config: `linked_repos` list entries with required `name`/`path`/`description` and `additionalProperties: false`
  (`src/sase/config/sase.schema.json:8-29`, `$ref`s at 979/987; defaults `src/sase/default_config.yml:7`). No
  `auto_clone` field exists yet. Project-local entries live in the repo's `sase.yml`; global entries in
  `~/.config/sase/sase.yml` (currently `chezmoi`); merge via `_resolution_config`/`_merged_entries_from_config`
  (`linked_repos.py:415,446`; legacy `sibling_repos` alias).
- Resolution: `resolve_linked_repos_for_project` (`linked_repos.py:311`) → `_resolve_linked_repos` (`:339`) producing
  `_ResolvedLinkedRepo` (`:50`; name/env_name/primary_dir/workspace_dir/workspace_num) and `LinkedRepoResolution`
  (`:70`).
- Clone location: `<host_checkout>/sase/repos/<name>` (`linked_repo_clone_dir`, `:556`; `LINKED_REPO_CLONES_SUBDIR`,
  `:43`), legacy `.sase/workspaces/<name>` fallback + rename migration (`:564-602`). `/sase/repos/` is protected via
  `.git/info/exclude` plus the tracked root `.gitignore` rule (`src/sase/main/init_workspace_handler.py`).
- Launch prep **eagerly materializes every resolved linked repo** with a distinct workspace dir on every launch:
  `prepare_linked_repo_workspaces_if_needed` (`src/sase/axe/run_agent_runner_setup.py:80-123`), called from
  `src/sase/axe/run_agent_runner.py:245,353`. Env vars `SASE_LINKED_REPO_<NAME>_DIR` (`LINKED_REPO_ENV_PREFIX`,
  `linked_repos.py:25`) point agents/build tooling at the clones — this repo's `Justfile:17` resolves `sase_core_dir`
  from them with a `../sase-core` fallback, and `just install/test` (the default VCS hooks) depend on that resolution
  succeeding.
- Lazy access: `sase workspace open -p <name> -r <reason> <num>` (`src/sase/main/workspace_handler_list.py`) with reason
  auditing via `record_opened_linked_repo`.
- Agent-file rendering: `_extend_linked_repository_section` (`src/sase/main/init_memory/root_rendering.py:58`, bullets
  via `_linked_repo_list_item:54`) from `_linked_repos_raw` (`src/sase/main/init_memory/config.py:142`, project-local
  `sase.yml` only).

**SDD store** (`src/sase/sdd/`):

- Single-root model: `SddStoreRecord` (`_store_types.py:27`;
  schema_version/storage/provider/host/repo/remote_url/discovery/probed_at) and `SddStore` (`:41`), recorded at
  `.sase/sdd-store.json` (`_store_records.py:96`). `SDD_CANONICAL_DIRS = ("beads", "plans", "research")`
  (`_paths.py:7-11`). `SASE_SDD_DIR` exported to agents (`env.py:11,28`).
- Writers hardcode `plans/<YYYYMM>/` with nested `prompts/`: `_write.py:42-48`, `plan_approval_actions.py:479`,
  `workflows/commit/commit_hooks.py:227-228`, `axe/run_agent_exec_plan_sdd.py:102-104` (reference strings
  `.sase/sdd/plans/<yyyymm>/<name>.md`). Prompt discovery globs `plans/*/prompts` plus legacy roots (`_paths.py:44-53`);
  `find_sdd_file` (`:64`).
- Plan frontmatter links are workspace-relative and include the kind prefix (e.g.
  `prompt: .sase/sdd/plans/202607/prompts/<name>.md`, older files use `sdd/plans/...`);
  `sase sdd validate/links/repair-links` audit them.
- Beads always resolve as `<sdd_root>/beads`: `sdd/beads.py:27`, `bead/cli_common.py:34-35`, `bead/workspace.py`,
  `doctor/checks_beads.py:34`, `ace/tui/actions/event_refresh/_sdd_paths.py:9-21`. `beads/beads.db*` gitignore
  management in `sdd/_bead_ignore.py:8-14`. The Rust bead facades receive a resolved `beads_dir` and are path-agnostic.
- Commit/push is single-repo: `commit_sdd_store_files` commits `store.repo_root` and pushes per `sdd.push_after_commit`
  (`sdd/_commit.py:274-300,394-408`). Workspace clones sync via `_store_link.py` (origin = real remote `:306`,
  `pull --rebase` `:179-188`, stale-clone replacement `:88-119`).
- README/skeleton generation: `sdd/_init_files.py` (`SDD_README_CONTENT:17` embeds `assets/sdd-directory-map.png:24`;
  `plan_sdd_init_actions:126`; packaged asset `src/sase/sdd/assets/sdd-directory-map.png` read via `importlib.resources`
  `:185-187`).
- Companion repo creation: provider hooks `ws_materialize_sdd_store` / `ws_preflight_sdd_companion` /
  `ws_create_sdd_remote` (`workspace_provider/_hookspec.py:208,225,240`, all firstresult) implemented in the
  **sase-github** plugin: `_companion_sdd_candidates` (`workspace_plugin.py:618`, honors the `sdd.repo.name` override),
  `_create_github_sdd_repo` (`:770`) — already `gh repo create ... --public` (`:787`) with an "SDD companion repository
  for X" description — plus `_require_sdd_creation_authorization` and `_ensure_github_sdd_label`.
- `sase init` framework: specs registered in order memory → sdd → skills → workspace (`main/init_registry.py:22-59`);
  `InitCommandSpec` is `name`/`plan`/`run` (`:12-21`). Companion creation is interactively confirmed with a typed yes on
  a TTY (`main/sdd_handler.py:150-184`); `--yes` cannot authorize it (`main/init_onboarding.py:155-174`,
  `main/parser_init.py:113`).
- `sase sdd` CLI today: `init/links/list/path/repair-links/validate` (`main/sdd_handler.py:28-41`);
  `sase sdd path <kind>` restricted to `SDD_CANONICAL_DIRS` (`main/parser_sdd.py:91`). No `--ensure` flag and no
  `migrate` subcommand exist yet.
- `sase plan list/search` index repo plans via `resolve_sdd_dir` (`plan_search/facade.py:30-47`).
- `is_sase_managed`: existing project-local field (`project_management.py:14,34,77`); this repo's `sase.yml` sets it
  `true` and configures four linked repos (sase-core, sase-github, sase-telegram, sase-nvim).

**Research xprompts** (all in the chezmoi repo):

- `home/dot_config/sase/sase.yml`: `research` writes under `$(sase sdd path research)/$(date +%Y%m)/`; `research/more`
  follows `$(sase sdd path research)/README.md` conventions; `research/image` asks for a GPT-image infographic;
  `research/prompt` wraps `#research`. Model aliases `research_a/b/lead` + bucket.
- `home/dot_xprompts/research_swarm.md`: two researchers + consolidator + image agent (`%model:codex/gpt-5.6-sol` +
  `#research/image`); the consolidator resolves the "effective research directory" via `$(sase sdd path research)` and
  builds `<name>/{<name>__a.md,<name>__b.md,<name>.md}` there.
- GPT image has **no first-party integration**: it works only through the Codex CLI runtime's built-in `image_gen` tool
  on a Codex model. The draft `sase_plan_codex_disable_image_gen_tool.md` (repo root) proposes disabling that tool; it
  is NOT implemented yet (no `imagegenext` reference in `src/sase/llm_provider/`) — see Risks.
- Generated skill templates hardcode SDD paths: `src/sase/xprompts/skills/sase_beads.md` uses
  `${SASE_SDD_DIR}/plans/{YYYYMM}/...`; skills are rendered by `sase skill init` and deployed via chezmoi.

**Current `sase--sdd` contents** (for the migration phase): `plans/<YYYYMM>/{*.md,prompts/*.md}` for 202602-202607,
`research/<YYYYMM>/*.md` (+ `*_infographic.png`), `beads/{issues.jsonl,events,config.json, metadata.json,beads.db*}`,
`README.md`, `assets/sdd-directory-map.png`, and six residual tracked legacy-tier files (`legends/README.md`,
`legends/.gitkeep`, two `legends/202605/*.md`, `myths/README.md`, plus untracked `tales/` leftovers) that are
intentionally left behind.

**Rust core boundary**: linked-repo resolution, config merge, init machinery, and SDD storage policy are pure Python;
Rust only receives fully-resolved paths (bead facades). **No `../sase-core` changes are required.**

---

## Design decisions

1. **Two public companions replace `--sdd` for migrated projects.** Naming: `<owner>/<repo>--plans` and
   `<owner>/<repo>--research`, created **public** in the origin owner's org (the existing `--public` creation path is
   kept; per-suffix descriptions). The old `<owner>/<repo>--sdd` repo is archived (not deleted) after verification.
   Unmigrated projects keep the legacy single-root behavior indefinitely.
2. **Layout keeps monthly directories.** Plans repo root: `<YYYYMM>/` dirs (plan files + nested `prompts/` snapshots),
   `beads/`, `assets/`, generated `README.md`. Research repo root: `<YYYYMM>/` dirs (research files,
   `*_infographic.png`, and research-swarm `<name>/` output dirs exactly as they land today), `assets/`, generated
   `README.md`. Because monthly dirs move verbatim, there are **no basename collisions** to resolve; migration only
   rewrites `prompt:`/`plan:` frontmatter links (dropping the `[.sase/]sdd/plans/` prefix to the canonical post-split
   form the readers/validators understand).
3. **`auto_clone` defaults to `false`, and launch prep honors it.** This intentionally changes behavior: today every
   configured linked repo is materialized on every launch; afterwards only `auto_clone: true` entries are (env var
   exports follow materialization). All other linked repos materialize lazily via `sase workspace open`. To preserve
   today's build behavior, this repo's `sase.yml` sets `auto_clone: true` on **sase-core** (the `Justfile`/hook commands
   depend on its clone); the three plugin repos and chezmoi stay lazy.
4. **`auto_clone: true` repos are omitted from agent files.** The generated "Linked Repositories" section lists only
   repos that require `sase workspace open`. For managed projects the injected `--research` entry (auto_clone false) IS
   listed — deterministically, derived only from repo contents (`is_sase_managed`), never from clone/remote existence,
   to keep the memory drift gate stable across machines.
5. **Default companion injection is config-derived; prep is runtime-aware.** When the project-local `sase.yml` has
   `is_sase_managed: true` and `default_linked_repos` is not `false`, resolution injects `<repo>--plans` (auto_clone
   true) and `<repo>--research` with machinery-provided path/description (explicit same-name entries win via the
   existing dedupe). Launch prep quietly skips injected companions whose backing repo/record is not materialized yet
   (pre-migration projects), so the injection is safe to ship before any project migrates.
6. **Migrated-ness authority is the SDD store record.** A new record shape (schema_version bump) records companion
   storage with per-kind repos/remotes. Legacy records keep the single-root behavior. `sase doctor` flags
   mixed/half-migrated states. Rendering never consults the record (see decision 4).
7. **`sase sdd path <kind>` stays the agent-facing resolver.** For migrated projects it maps `plans` → the plans clone
   root, `beads` → `<plans clone>/beads`, and `research` → the research clone root; otherwise legacy roots. A new
   `--ensure`/`-e` flag materializes the backing clone on demand (research's lazy path). Bare `sase sdd path` prints the
   plans clone root for compatibility. Per-kind env vars (`SASE_SDD_PLANS_DIR`, `SASE_SDD_RESEARCH_DIR`,
   `SASE_SDD_BEADS_DIR`) are exported alongside a backward-compatible `SASE_SDD_DIR` (plans clone root for migrated
   projects). Because monthly dirs are kept, `$(sase sdd path research)/$(date +%Y%m)/` keeps working unchanged in the
   research xprompts.
8. **Companion clones ride the linked-repo clone location with SDD-style sync.** Clones live at
   `<host_checkout>/sase/repos/<repo>--plans|--research` (already ignored + excluded). Unlike plain linked clones,
   companion clones set `origin` to the real remote and reuse the `_store_link.py` pull-rebase/stale-replacement
   semantics, because workspaces must push plan/bead/research commits directly.
   `sase workspace open -p <repo>--research ...` and `sase sdd path research --ensure` resolve to the same clone.
9. **Infographics are static packaged assets** (mirroring `sdd-directory-map.png`): committed under
   `src/sase/sdd/assets/` with regenerable `.prompt.md` sidecars, stamped into each companion's `assets/` by init,
   drift-tracked as bytes. Image generation never runs at init time; a dedicated phase produces the PNGs with GPT image
   on a Codex model.
10. **Repo creation stays explicitly confirmed.** Companion creation rides the existing provider-hook +
    typed-confirmation pattern (`--yes` still cannot authorize creating GitHub repos).

---

## Phase 1: Linked-repo generalization — `auto_clone`, agent-file exclusion, default injection

Owner: distinct agent instance. This repo only.

Objective: generalize linked-repo config and launch behavior so companion repos can be expressed, without any SDD
awareness yet.

Scope:

- Schema (`src/sase/config/sase.schema.json:8-29`): add `auto_clone` (boolean, default `false`) to the `linkedRepo`
  definition; add the project-local `default_linked_repos` boolean (default `true`). Update
  `src/sase/default_config.yml` comments and `docs/configuration.md`. (Schema sync is this repo's most common gotcha —
  see the `config_schema_sync` mentor focus in `sase.yml`.)
- Resolution (`src/sase/linked_repos.py`): thread `auto_clone` through `_merged_entries_from_config` →
  `_resolve_linked_repos` → `_ResolvedLinkedRepo`/`LinkedRepoResolution`. Inject default companion entries
  (`<project>--plans` auto_clone true, `<project>--research`) per Design Decision 5, with machinery-provided
  path/description; explicit same-name entries win via the existing dedupe; injected companions without a backing
  repo/record are skipped quietly at prep time.
- Launch prep (`src/sase/axe/run_agent_runner_setup.py:80-123`): filter eager materialization to `auto_clone: true`
  entries; gate `SASE_LINKED_REPO_<NAME>_DIR` env exports to materialized entries so build tooling never receives
  dangling paths. Verify the other launch entry points only resolve (materialize=False) and need no change.
- This repo's `sase.yml`: set `auto_clone: true` on the `sase-core` entry (build dependency — `Justfile:17` and the
  `just lint`/`just test` hooks); leave sase-github/sase-telegram/sase-nvim lazy.
- Agent files (`src/sase/main/init_memory/config.py`, `root_rendering.py:58-94`): exclude `auto_clone: true` entries
  from the "Linked Repositories" section; list the injected `--research` companion for managed projects (config-derived
  only). Regenerate this repo's memory via `sase memory init` so the drift gate stays green (never hand-edit
  `AGENTS.md`/shims/`memory/*.md`).
- Read `memory/cli_rules.md` via `sase memory read` before touching any CLI surface.
- Tests: schema validation, resolution threading, injection/override/opt-out, launch-prep gating and env-export gating,
  quiet-skip of unmaterialized companions, memory rendering exclusion + managed-project research listing,
  `sase memory init --check` idempotency.

Exit criteria: `auto_clone`, exclusion, and injection behave per Design Decisions 3-5; launches no longer eagerly prep
non-auto_clone linked repos; builds still find sase-core; `just check` and `sase validate` pass.

## Phase 2: SDD two-root storage model + kind-root resolution

Owner: distinct agent instance. Depends on Phase 1. This repo only.

Objective: teach the SDD subsystem that a migrated project's kinds live in two companion clones (monthly dirs at each
root), while preserving legacy single-root behavior everywhere else.

Scope:

- Store records (`sdd/_store_types.py`, `_store_records.py`): new companion-storage record shape (schema_version bump)
  carrying per-kind repo/remote mapping; loaders accept both shapes; `SddStore` (or a successor view) exposes per-kind
  roots. Migrated-ness = record says companion storage (Design Decision 6).
- Kind-root resolver in `src/sase/sdd/`: `plans`/`beads` → plans clone, `research` → research clone for migrated
  projects; legacy single-root fallback otherwise (`resolve_sdd_dir`/`resolve_sdd_store` callers keep working).
- Path layout: for migrated projects, writers drop the kind prefix but keep monthly dirs —
  `<plans_root>/<YYYYMM>/<name>.md` + `<plans_root>/<YYYYMM>/prompts/<name>.md` (`sdd/_write.py:42-48`,
  `plan_approval_actions.py:479`, `workflows/commit/commit_hooks.py:227-228`) and
  `axe/run_agent_exec_plan_sdd.py:102-104` reference strings. Readers/validators/globs handle a root whose monthly dirs
  sit next to `README.md`/`assets/`/`beads/`: `sdd/_paths.py` (`sdd_prompt_roots`, `find_sdd_file`,
  `looks_like_sdd_root`), `sdd/links.py`, `sase sdd list/validate/links/repair-links`, and `plan_search/facade.py`
  (index the plans clone for migrated projects).
- `sase sdd path`: route through the kind resolver; add `--ensure`/`-e` to materialize the backing companion clone on
  demand; keep bare `sase sdd path` printing the effective root (plans clone). Alphabetical help/options per
  `memory/cli_rules.md` (read it first).
- Env: export `SASE_SDD_PLANS_DIR`/`SASE_SDD_RESEARCH_DIR`/`SASE_SDD_BEADS_DIR` alongside `SASE_SDD_DIR` (`sdd/env.py`,
  consumed at agent launch).
- Companion clone sync: adapt `sdd/_store_link.py` (origin = remote, pull --rebase, stale replacement) to the companion
  clone locations; the plans companion's auto_clone launch prep uses it (replacing `ensure_workspace_sdd_clone`'s role
  for migrated projects).
- Docs: `docs/sdd.md`, `docs/sdd_storage.md`, `docs/configuration.md` describe the two-repo layout + legacy mode.
- Tests: record round-trip both shapes, kind-root routing (migrated vs legacy), writer paths (monthly dirs kept, kind
  prefix dropped), reader/validator behavior on companion roots, `--ensure`, env exports, plan search over the plans
  clone.

Exit criteria: with companion clones present, all plan/research reads and writes land in the right clone under
`<YYYYMM>/`; legacy projects are byte-for-byte unaffected; `just check` passes.

## Phase 3: Beads relocation, commit routing, doctor, and skill templates

Owner: distinct agent instance. Depends on Phase 2. This repo only.

Objective: move bead storage under the plans companion and make every SDD commit/push route to the owning repo.

Scope:

- Beads resolve under the plans root for migrated projects: `sdd/beads.py`, `bead/cli_common.py`, `bead/workspace.py`,
  `doctor/checks_beads.py`, `ace/tui/actions/event_refresh/_sdd_paths.py`. The `beads/beads.db*` gitignore management
  (`sdd/_bead_ignore.py`) targets the plans repo root. Rust bead facades are untouched (they receive the resolved
  `beads_dir`).
- Commit/push routing: `commit_sdd_store_files` (and the auto-commit machinery funneling through it) commits and pushes
  the repo that owns the written path — plan approvals, prompt snapshots, and bead mutations to the plans repo; research
  writes to the research repo. Honor `sdd.push_after_commit` per repo.
- `sase doctor`: SDD checks understand the two-companion layout and flag half-migrated states (companion record without
  clones, legacy `.sase/sdd` lingering after migration, etc.).
- Generated skill templates (`src/sase/xprompts/skills/`): update `sase_beads.md` (drop
  `${SASE_SDD_DIR}/plans/{YYYYMM}/...` in favor of `sase sdd path plans` / the new env vars with monthly-dir paths) and
  audit `sase_plan.md`/`sase_agents_status.md` for stale path prose. Read `memory/generated_skills.md` via
  `sase memory read` first; regenerate with `sase skill init` (chezmoi deployment happens in Phase 7).
- Tests: bead CLI + TUI-refresh paths against a plans-companion layout, commit routing per owning repo, doctor states,
  legacy fallbacks.

Exit criteria: `sase bead` and plan/bead auto-commits work end to end against a plans companion clone; research commits
route to the research repo; `just check` passes.

## Phase 4: `sase init` creates the companion repos; split-migration command

Owner: distinct agent instance. Depends on Phase 2 (can overlap Phase 3). This repo + the **sase-github** linked plugin
(open it with `sase workspace open -p sase-github -r "<reason>" <workspace_num>`).

Objective: make initialization create/adopt both **public** companion GitHub repos and stamp each with a generated,
drift-tracked README embedding a packaged infographic; provide the migration command Phase 6 runs.

Scope:

- Provider hooks (sase-github plugin): generalize the `--sdd` companion discover/create hooks to arbitrary companion
  suffixes (`--plans`, `--research`) with the same probe/create/adopt/error-classification behavior and **public**
  creation (`gh repo create <owner>/<repo>--plans --public --description ...`, per-suffix descriptions). Keep `--sdd`
  discovery working for legacy projects.
- Init specs (this repo): evolve the `sdd` init spec (`main/init_registry.py`, `main/sdd_handler.py`) so that on a
  managed GitHub project it plans/creates/materializes **both** companions (typed interactive confirmation, `--yes`
  cannot authorize creation, check/diff support, config/record written only after success — mirror the existing
  create-first/write-record-last reliability rules), writing the Phase 2 companion store record.
- Generated content (`sdd/_init_files.py` pattern): per-repo README constants (purpose, monthly-dir layout, key
  commands, embedded infographic) + packaged assets `src/sase/sdd/assets/plans-directory-map.png` and
  `research-directory-map.png` with `.prompt.md` sidecars. Commit **valid placeholder PNGs** now, drift-tracked as
  bytes; tests must not pin exact bytes so Phase 5 can swap them. READMEs are deterministic (no timestamps/paths) so
  init `--check` stays a no-op after apply. The research README documents the research conventions that `#research/more`
  references.
- Migration command (exact surface per `memory/cli_rules.md` — read it first; e.g. `sase sdd migrate`): move
  `plans/<YYYYMM>/` → plans-repo root, `research/<YYYYMM>/` → research-repo root, copy `beads/` (excluding `beads.db*`,
  re-applying ignore rules), rewrite `prompt:`/`plan:` frontmatter links to the post-split form, **leave
  `tales/`/`legends/`/`myths/` stragglers and the old README/assets behind**, push both repos, write the companion store
  record, retire the legacy `.sase/sdd` clone; `--check`/`--diff` previews; idempotent/re-runnable; takes the existing
  materialization lock.
- Tests: plugin candidate naming/creation/adoption per suffix (public), init planning/confirmation/apply for both repos,
  README/asset drift idempotency, migration dry-run + link rewriting + record writing. `just check` in both repos.

Exit criteria: on a managed GitHub project, `sase init` (confirmed) creates both public companions with README +
placeholder infographic and green drift checks; the migration command passes its dry-run tests; `just check` passes in
both repos.

## Phase 5: GPT-image infographics for both repos

Owner: distinct agent instance. **Model: `codex/gpt-5.6-sol`** (GPT image runs only through the Codex runtime's built-in
image_gen tool). Depends on Phase 4. Asset-only — no Python changes.

Scope:

- Follow the house style (`docs/images/infographic-style-brief.md`, `docs/agent_images.md`): clean ~16:9 (~1600x900)
  light-background architecture infographics; generate a **text-free structural base** with GPT image, then add short
  exact labels deterministically (SVG/ImageMagick, pinned font); no model-rendered text in the final raster.
- `plans-directory-map.png`: the plans-repo story — `sase plan propose` → review/approval → `<YYYYMM>/` plan file +
  nested `prompts/` snapshot + bead state in `beads/`, auto-cloned into every workspace (`auto_clone: true`) under
  `sase/repos/`, committed/pushed by the SDD machinery.
- `research-directory-map.png`: the research story — research swarm fan-out → consolidated report + infographic in
  `<YYYYMM>/` → lazily-materialized clone via `sase sdd path research --ensure`.
- Replace the Phase 4 placeholders at the packaged paths; finalize both `.prompt.md` sidecars (exact prompt, alt text,
  post-processing steps) so the images are regenerable without chat history.
- Verify legibility at GitHub Markdown width and terminology accuracy (`auto_clone`, `sase sdd path`, repo names).

Exit criteria: both final PNGs committed (packaged paths + sidecars), byte-stable against the init drift check; labels
accurate; no Python diffs.

## Phase 6: Execute the migration for the sase project

Owner: distinct agent instance. Depends on Phases 3, 4, 5. Runs against the live `sase-org` GitHub org — every
repo-creating/archiving step uses the interactive confirmations built in Phase 4.

Scope:

- Pick a quiet window (no running agents writing beads/plans). Run the new init flow to create `sase-org/sase--plans`
  and `sase-org/sase--research` (public) with final READMEs + infographics.
- Run the migration command for the sase project: verify `--check`/`--diff` first, then apply. Confirm results: monthly
  dirs intact at each repo root, `prompt:`/`plan:` links valid (`sase sdd validate`), beads intact (`sase bead list`, DB
  rebuilt from `issues.jsonl`), research files + swarm dirs + `*_infographic.png` carried over, legacy-tier stragglers
  left behind in `sase--sdd`.
- Sweep stragglers: stale references to `.sase/sdd/...` or `sdd/plans/...` paths in live docs/docstrings — update the
  live ones, leave purely historical citations.
- Archive `sase-org/sase--sdd` on GitHub (archive, NOT delete) only after end-to-end verification passes.
- End-to-end verification in a fresh numbered workspace: launch prep auto-clones the plans companion;
  `sase sdd path plans|beads|research` resolve into the clones; propose + approve a throwaway plan and confirm it lands
  in `sase--plans` under the current `<YYYYMM>/` with its prompt snapshot; create/close a throwaway bead and confirm
  push; `sase sdd path research --ensure` + write a test research file; `sase validate` + `sase doctor` green.
- `just check` in this repo after any code/doc tweaks.

Exit criteria: both companions live and verified, old `--sdd` archived, no agent-facing flow regressed.

## Phase 7: Chezmoi research xprompts, skills deployment, memory regeneration

Owner: distinct agent instance. Depends on Phase 6. Chezmoi linked repo (open with
`sase workspace open -p chezmoi -r "<reason>" <workspace_num>`) + this repo for drift reconciliation.

Scope:

- Read `memory/xprompts.md` via `sase memory read` before editing xprompts.
- `home/dot_config/sase/sase.yml` xprompts: `research` keeps its `$(date +%Y%m)/` segment but resolves the root with
  `$(sase sdd path research --ensure)` so the lazy clone materializes; `research/more` likewise switches its
  README-conventions lookup to `--ensure`; audit `research/image` and `research/prompt` wording (no path changes
  expected).
- `home/dot_xprompts/research_swarm.md`: the consolidator's "effective research directory" resolves via
  `$(sase sdd path research --ensure)`; the `<name>/` output layout is unchanged. Update `old_research_swarm.md` only if
  still expected to work; otherwise note it as legacy.
- Deploy the Phase 3 generated-skill updates through chezmoi (per `memory/generated_skills.md`).
- Regenerate agent instruction files wherever the Phase 1 renderer changed output: `sase memory init` for this repo (if
  not already clean) and the chezmoi/home context — never hand-edit `AGENTS.md`, provider shims, or `memory/*.md`.
  Reconcile the `sase validate` drift gates in both contexts.
- Per the chezmoi repo's own instructions, run `chezmoi update -a --force` after committing there.
- Smoke test: run `#research` and `#research_swarm` on a scratch topic; confirm files land in `sase--research` under the
  current `<YYYYMM>/` and the swarm's image agent still generates an infographic.

Exit criteria: research xprompts write to the research companion (monthly layout preserved), skills deployed, memory
drift gates green in both contexts, swarm smoke test passes.

---

## Risks and mitigations

- **Codex image_gen may get disabled.** The draft `sase_plan_codex_disable_image_gen_tool.md` (not yet implemented —
  verified) proposes disabling the tool that Phase 5 depends on. Mitigation: run Phase 5 before that plan lands, or make
  the disable conditional/opt-out first.
- **Public exposure.** The companion repos are public by explicit user decision; the migration phase should still
  eyeball the flattened content once before pushing (no secrets/tokens in plans/research/beads).
- **Interim window for managed projects.** Between Phase 1 landing and a project's migration, injected companions exist
  in config but not on GitHub. Prep quiet-skips them and `sase sdd` stays on the legacy root (record-gated), so nothing
  breaks; agent files list `--research` slightly early. Other managed projects (e.g. actstat, bob-cli) need a routine
  `sase memory init` refresh after upgrading; their migrations are out of scope but supported by the same command.
- **Launch behavior change (Design Decision 3).** Repos that stop being eagerly cloned could break tooling that assumed
  their presence. Mitigation: sase-core is pinned `auto_clone: true` here; env exports are gated to materialized clones
  so fallbacks (`../sase-core`) engage cleanly; instruction files already mandate `sase workspace open` for the rest.
- **Drift-gate flapping.** Generated READMEs, memory renders, and injected listings must be pure functions of repo
  contents (never clone existence or machine state). Every phase touching generated content proves init → `--check` is a
  no-op.
- **Beads concurrency during migration.** Bead writes from live agents during Phase 6 could be lost. Mitigation: quiet
  window + materialization lock + a final `issues.jsonl` diff against the old repo before archiving.
- **Link rot.** `prompt:`/`plan:` frontmatter paths embed the old `sdd/plans/` prefix. Mitigation: migration rewrites
  them and `sase sdd validate` gates the cutover before archiving.
- **Older sase versions on other machines.** A machine running pre-split sase against a migrated project would look for
  `.sase/sdd`. Mitigation: land Phases 1-4 in releases before running Phase 6; legacy fallback + `sase doctor` guidance
  cover the transition; the archived `--sdd` repo remains readable.

## Out of scope / notes

- No Rust core (`../sase-core`) changes: linked repos, config, init, and SDD policy are Python; bead facades receive
  resolved paths (boundary litmus test satisfied).
- Migrating other projects' `--sdd` companions (actstat, bob-cli, the home context) — the machinery and the migration
  command support doing it later; legacy behavior is preserved meanwhile.
- No changes to plan tier semantics (`tier:` frontmatter), the plan-approval TUI, or the bead schema.
- `tales/`, `legends/`, `myths/` are dead: never migrated, no code awareness added; their stragglers live on only in the
  archived `sase--sdd` history.
