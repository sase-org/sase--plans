---
tier: epic
title: Nested-clone context safety and legacy SDD storage retirement
goal: >
  sase commands derive project context from the repository they are actually run in (never from a host workspace
  surrounding a nested clone), sase init refuses loudly when run from the wrong directory or without the project's
  workspace-provider plugin instead of silently writing legacy layouts, and the legacy local and separate_repo SDD
  storage modes — including their adoption/migration machinery and every downstream branch — are removed from the
  codebase.
phases:
  - id: context-boundaries
    title: Repo-boundary-aware context derivation and env-pin hygiene
    depends_on: []
  - id: init-guards
    title: sase init wrong-context guardrails and loud provider failures
    depends_on: [context-boundaries]
  - id: sidecar-policy
    title: First-class sidecar_repos provider policy with no silent legacy fallback
    depends_on: [init-guards]
  - id: legacy-state-cutover
    title: Audit and converge remaining on-disk legacy SDD state
    depends_on: [sidecar-policy]
  - id: legacy-removal
    title: Remove local/separate_repo storage modes and adoption machinery
    depends_on: [legacy-state-cutover]
---

# Plan: Nested-clone context safety and legacy SDD storage retirement

## Context & Problem

During the sase-62 cutover, the executing agent ran `sase init` (memory and repo steps) from repo clones that
`sase repo open` had materialized under the host workspace's `sase/repos/linked/` and `sase/repos/external/`
directories. Four distinct failures surfaced, all reproducible from code inspection:

1. **Project-name misderivation.** Memory generation derived the HOST project's name (`sase`) for foreign nested clones,
   because checkout-marker discovery walks every ancestor directory to the filesystem root and the marker check
   short-circuits before the clone's own git remote is ever consulted.
2. **Silent legacy-layout fallback.** `sase repo init` run in a checkout lacking its project's local, gitignored SDD
   store record (`.sase/sdd-store.json` lives only in the registered primary checkout) silently initialized a legacy
   in-tree `sdd/` layout instead of failing or converging on the sidecar-repos layout.
3. **Provider-plugin misclassification.** In a venv without the `sase-github` plugin, GitHub clones were classified as
   `bare_git`, whose `in_tree` policy again produced legacy layouts; store maintenance took a "compatibility-only" path
   that left stale records in place.
4. **Launch-env leakage.** `SASE_ACTIVE_PROJECT_DIR` (set by agent launch to the launch workspace) leaked into
   cross-project subcommands and had to be unset by hand to avoid contaminating generated output.

Separately, the user wants all remaining **legacy SDD storage logic** removed. A full inventory (below) shows the legacy
surface is the `local` storage mode, the old single-root `separate_repo` record layout (`.sase/sdd` clones), their
silent fallbacks, and the adoption/migration engine — roughly 42 source files and 40 test files carry branches for them.
**`in_tree` is NOT legacy**: it is the declared, current policy of the built-in bare-git workspace provider
(`src/sase/workspace_provider/plugins/bare_git_workspace.py:87`, `docs/sdd_storage.md`), so `in_tree` support and its
`is_in_tree` branches stay. If bare-git projects should also migrate someday, that is a separate epic.

## Current State (key facts, verified)

- **Marker walk is unbounded.** `find_marker_from_cwd` (`src/sase/workspace_provider/marker.py:137`) returns the nearest
  ancestor `.sase/checkout.json`, crossing git-repo boundaries. Clones under `<host>/sase/repos/{linked,external}/`
  carry no marker of their own, so the host workspace's marker wins.
- **Name derivation prefers the marker.** `project_memory_name` (`src/sase/main/init_memory/config.py:96`) checks the
  marker first and only falls through to `remote.origin.url` / git-toplevel derivation when no marker is found.
  `primary_workspace_root_for_memory` (`config.py:135`) and `get_primary_workspace_dir` (`src/sase/sdd/_paths.py:189`,
  marker branch at `:256`) leak the same way, and `_inherited_sdd_record_owner_anchor` (`src/sase/sdd/store.py:484`)
  deliberately inherits the host's store record through the ancestor marker.
- **Init has no wrong-directory guards.** The only pre-flights are "not a project directory" / "not sase-managed"
  (`src/sase/main/repo_init_handler.py:23-24,69-78`). Nothing detects cwd under a host's `sase/repos/` tree, even though
  the path constants exist (`src/sase/linked_repos.py:71-73`) — opened-repo records are written to the agent's artifacts
  dir, not into the clones, so init cannot use those.
- **Legacy fallbacks are silent.** `_resolve_sdd_storage` (`src/sase/sdd/store.py:468-481`) resolves missing-record +
  missing/unknown policy to `SDD_STORAGE_LOCAL`; `_materialize_sdd_store` (`store.py:296`) short-circuits to that
  resolution without any provider call; `sase repo init` dispatch (`repo_init_handler.py:110-117`) reaches
  `_run_legacy_store_init` whenever the record is unreadable, and `_project_provider_sdd_policy` (`:577-588`) only
  recognizes `{in_tree, local, separate_repo}` — the sidecar era is represented indirectly (GitHub's plugin still
  declares `sdd_storage_policy="separate_repo"` as its pre-init policy).
- **Env pins.** `SASE_ACTIVE_PROJECT_DIR` is one of `PROVIDER_PROJECT_DIR_ENV_VARS` (`src/sase/env_contracts.py`);
  consumers include commit tracking (`src/sase/workflows/commit/commit_tracking.py:375`), the commit finalizer config,
  and provider CLIs. `scrub_linked_repo_env` (`src/sase/_linked_repo_env.py:93`) scrubs only linked-repo vars.
- **Legacy inventory (for the removal phases).** Modes in `src/sase/sdd/_store_types.py`: `in_tree` (current, bare-git),
  `sidecar_repos` (current, provider-managed), `separate_repo` (legacy record layout at `<workspace>/.sase/sdd`, but
  also still the GitHub plugin's declared policy value), `local` (pure legacy fallback at `<primary>/.sase/sdd`).
  Adoption engine: `src/sase/sdd/_store_adoption.py` (imports loose legacy `sdd/` and `.sase/sdd` trees into sidecars).
  Legacy record spellings: `companion_repos` storage value and `companions` sidecar key
  (`src/sase/sdd/_store_records.py`). Legacy branches span the bead layer (`bead/cli_common.py`
  `_resolve_legacy_beads_location`, `bead/workspace.py`), doctor (`doctor/checks_config_sdd.py`,
  `doctor/checks_beads.py`), commit workflow (`workflows/commit/plan_paths.py`, `commit_tracking.py:406`), commit
  finalizers, axe reference builders (`axe/run_agent_exec_plan_sdd.py`), plan/prompt search `.sase/sdd` roots
  (`prompt/search/sources.py`), TUI attribution (`ace/tui/widgets/prompt_panel/_agent_commits.py:281`,
  `ace/tui/models/agent_plan_goal.py:305`), repo inventory, and `sdd.repo.name` in `src/sase/default_config.yml`
  (support for unmigrated stores). Beads live in the plans sidecar (`kind_root("beads")`) for sidecar storage and
  in-tree `sdd/beads/` only for bare-git projects — both current.
- **Rust core boundary is unaffected.** No SDD storage mode or resolved SDD path crosses the Python↔Rust wire
  (`src/sase/core/*wire*.py` mention sdd only in docstrings; `agent_artifact_helpers.py` consumes an opaque display
  path). No `sase-core` changes are expected anywhere in this epic.

## Target Design

1. **Boundary rule for context derivation.** A checkout marker (and the project identity, primary workspace, and SDD
   store record it anchors) applies only to paths inside the git checkout that carries it. From a nested clone, marker
   discovery finds nothing and identity derivation falls through to the clone's own git remote — which is already
   correct. One deliberate carve-out: host-scoped _sidecar_ clones (`<host>/sase/repos/plans`, `.../research`) belong to
   the host project and keep inheriting host context (bead resolution and store maintenance rely on this). Linked
   (`sase/repos/linked/`) and external (`sase/repos/external/`) clones never inherit host identity.
2. **Init refuses wrong contexts.** Every `sase init` entry point (bare, `memory`, `repo`, `skills`) pre-flights the
   cwd: inside any host-scoped `sase/repos/` clone it aborts with the host project, the clone's own identity, and the
   remediation (run from the project's own registered checkout, e.g. via `sase repo open <project>`); when the derived
   project name contradicts the cwd repo's origin-derived name it aborts likewise. No bypass flag.
3. **Provider-managed storage is explicit and loud.** `sidecar_repos` becomes a first-class provider policy declared by
   the GitHub plugin. For provider-managed projects, a missing store record triggers provider preflight/adoption
   (re-recording against the project's _registered_ primary from the project spec, not a cwd walk-up) or an actionable
   error; an unloadable provider plugin is an error naming the plugin — never a `bare_git`/`in_tree` or `local`
   reinterpretation of the project.
4. **Legacy modes cease to exist.** After a one-time audited convergence of on-disk state across all registered
   projects, `local` and `separate_repo` (mode, record layout, adoption engine, and every downstream branch) are
   deleted. Encountering a legacy record afterwards is a hard error telling the user to run `sase repo init`.
5. **Launch-env pins are scoped.** `SASE_ACTIVE_PROJECT_DIR` (and sibling provider project-dir pins) are honored only
   when the consuming command's cwd lies inside the pinned directory; cross-project subprocess launches scrub them the
   same way linked-repo env vars are scrubbed today.

## Phases

### Phase `context-boundaries` — Repo-boundary-aware context derivation and env-pin hygiene

Scope (sase repo only):

- Introduce a bounded marker resolver in `src/sase/workspace_provider/marker.py` (or a thin wrapper module): walk upward
  from the start dir but stop at the start dir's own git toplevel; when the start dir is not inside any git repository,
  preserve today's full walk. Implement the sidecar carve-out: when the bounded search finds no marker and the enclosing
  git toplevel is exactly a host-scoped sidecar clone (`SIDECAR_REPO_CLONES_SUBDIR` shape,
  `src/sase/linked_repos.py:71-73`) whose host carries a marker, return the host's marker; linked/external clone roots
  return none.
- Migrate every `find_marker_from_cwd` caller to the bounded resolver: `src/sase/main/init_memory/config.py:41`,
  `src/sase/sdd/_paths.py:269`, `src/sase/sdd/store.py:499` (`_inherited_sdd_record_owner_anchor` at `:484` keeps
  host-record inheritance only via the sidecar carve-out), `src/sase/main/repo_handler.py:245,551`, and the bead modules
  that consult markers. Audit remaining direct `read_marker` walk-ups.
- `project_memory_name` / `primary_workspace_root_for_memory` need no structural change once the resolver is bounded —
  foreign clones fall through to git-remote derivation; verify the fallthrough ordering stays marker → remote → toplevel
  → basename.
- Env-pin hygiene: add a scoped accessor for `SASE_ACTIVE_PROJECT_DIR`/`PROVIDER_PROJECT_DIR_ENV_VARS` that yields the
  pin only when cwd is inside the pinned directory, and a scrub helper (sibling of `scrub_linked_repo_env`,
  `src/sase/_linked_repo_env.py:93`) applied where sase launches subprocesses targeting a different project's checkout.
  Update consumers: `workflows/commit/commit_tracking.py:375`, `llm_provider/commit_finalizer_config.py`,
  `llm_provider/codex.py:120`, `llm_provider/agy.py`. The agent-launch overwrite path
  (`src/sase/agent/launch_spawn.py:74-84`) keeps its current behavior.
- Tests: name/primary/record-owner derivation from (a) a linked clone, (b) an external clone, (c) a plans/research
  sidecar clone (host inheritance preserved), (d) a subdirectory of a numbered workspace, (e) the registered primary
  checkout, (f) a non-git directory; env-pin honored/ignored/scrubbed cases. Extend
  `tests/main/test_init_memory_handler.py` and the workspace-provider marker tests.

Acceptance: from a linked or external clone nested under a host workspace, memory-name derivation reports the clone's
own project name and the host's SDD store record is not inherited; sidecar clones still resolve host context; bead and
commit flows from sidecar clones behave as today. `just check` passes.

### Phase `init-guards` — sase init wrong-context guardrails and loud provider failures

Scope (sase repo only):

- Shared init pre-flight (wired through `src/sase/main/init_registry.py` / `init_onboarding.py` so bare `sase init` and
  each subcommand hit it once): abort when cwd is inside any host-scoped `sase/repos/` clone (sidecar, linked, or
  external — init must only run in a project's own registered checkout or numbered workspace). The error names the host
  project, the clone (kind and identity), and the remediation command. Add the identity cross-check: derived project
  name vs. the cwd repo's origin-derived name; mismatch aborts with both names shown.
- Loud provider failures: when a project's VCS/provider resolution indicates a provider whose plugin is not importable
  (e.g. GitHub remotes without `sase-github` installed), `sase init repo` and store materialization
  (`src/sase/sdd/store.py` provider dispatch, `src/sase/workspace_provider/_registry.py:111` classification path) fail
  with an install/remediation message instead of reclassifying the project as `bare_git`. Detection design is part of
  this phase (e.g. the registry consults the project spec's provider, or providers register remote-shape claims); the
  constraint is that a provider-managed project must never silently downgrade to `in_tree`/`local` behavior.
- Respect the CLI conventions from `memory/cli_rules.md` for any new options or help-text changes (alphabetical
  ordering, short aliases, excellent help).
- Tests: guard triggers for each clone kind, mismatch abort, plugin-missing errors for repo init and materialization,
  and non-regression for legitimate contexts (primary checkout, numbered workspace, `--all` inventory runs from the host
  workspace).

Acceptance: `sase init`, `sase init memory`, and `sase init repo` each exit non-zero with the guidance message — and
write nothing — when run inside `<host>/sase/repos/{linked,external,plans,research}/...`; with the GitHub plugin absent,
repo init for a GitHub project errors instead of creating any legacy layout. `just check` passes.

### Phase `sidecar-policy` — First-class sidecar_repos provider policy with no silent legacy fallback

Scope (sase repo + sase-github plugin repo; open `sase-github` via the `/sase_repo` skill):

- Host: extend the provider policy contract (`WorkflowMetadata.sdd_storage_policy`,
  `src/sase/workspace_provider/_hookspec.py:131-144`) to include `sidecar_repos`; keep accepting `separate_repo` as a
  transitional alias for provider-managed storage until phase `legacy-removal`. Update `_project_provider_sdd_policy`
  (`src/sase/main/repo_init_handler.py:577-588`) and repo-init dispatch (`:110-117`, `:198-205` for `--check`/`--diff`)
  to key on provider-managed vs `in_tree` explicitly: provider-managed projects run the configured-sidecars path; a
  missing/unreadable store record triggers provider preflight and record convergence (`preflight_sdd_sidecar` /
  `_sidecar_init`) instead of `_run_legacy_store_init`; `in_tree` projects keep today's behavior.
- `_resolve_sdd_storage` (`src/sase/sdd/store.py:468-481`): missing record + provider-managed policy resolves to sidecar
  storage; missing record + missing/unknown policy becomes an actionable error (never `SDD_STORAGE_LOCAL`).
  `_materialize_sdd_store` (`store.py:296`) loses its silent legacy short-circuit for provider-managed projects.
- Registered-primary record resolution: store-record reads/writes for init and maintenance resolve the primary checkout
  from the project spec (`~/.sase/projects/<name>` WORKSPACE_DIR) when available, so `sase repo init` run from any
  legitimate checkout of the project converges the same record instead of minting divergent per-checkout state.
- sase-github: declare `sdd_storage_policy="sidecar_repos"`; update plugin tests and any policy-string assertions.
- Tests (sase): policy dispatch matrix (`sidecar_repos`, transitional `separate_repo`, `in_tree`, missing-policy error),
  missing-record preflight/convergence, registered-primary resolution; extend `tests/main/test_repo_init_handler.py`,
  `test_repo_init_plan.py`, `tests/sdd_store/test_resolution.py`, `test_materialize.py`.

Acceptance: with both repos' changes installed, a GitHub project checkout with no local store record converges to the
sidecar layout (record re-written against the registered primary) via `sase repo init`, and no code path reaches
`_run_legacy_store_init` or `local` storage for provider-managed projects. `just check` passes in sase; the sase-github
test suite passes.

### Phase `legacy-state-cutover` — Audit and converge remaining on-disk legacy SDD state

Scope (operational; requires the deployed `sase` CLI and sase-github plugin to include all prior phases — verify first
and reinstall if needed):

- Enumerate every registered project — enabled, disabled, and the hidden `home` project — via `sase project list` (JSON)
  and each project spec. For each primary checkout and numbered workspace, audit: `.sase/sdd-store.json` records with
  `storage: separate_repo`, `companion_repos`, or a `companions` key; materialized `.sase/sdd` clones (primary or
  workspace level); stray in-tree `sdd/` directories in provider-managed projects (bare-git `in_tree` projects are
  exempt); and stray `.sase/sdd/` directories inside host-scoped linked/external clones.
- Converge findings with the still-present adoption path (`sase repo init` per project) so durable artifacts are
  imported into sidecars and records re-written as `sidecar_repos`. Dirty or ambiguous trees (uncommitted content, merge
  conflicts) are reported to the user for a decision — never force-deleted.
- Remove confirmed-stale leftovers (empty or fully-adopted `.sase/sdd` clones, stray directories) only after adoption
  verifies their content is preserved in the sidecar.
- Strip retired config keys (`sdd.storage`, `sdd.version_controlled`) and any `sdd.repo.name` overrides from real
  project `sase.yml` files encountered during the audit, so phase `legacy-removal` can delete their compatibility
  handling.
- Re-run `sase doctor` and `sase init --check` per enabled project; commits (only where registered repos changed) go
  through the sanctioned sase commit workflow. Generated memory/instruction files are only ever changed by running
  `sase init` — no hand edits.

Acceptance: the audit reports zero legacy records, clones, spellings, or retired keys across all registered projects;
enabled projects pass `sase doctor` and `sase init --check`. `just check` applies only if sase-repo files changed.

### Phase `legacy-removal` — Remove local/separate_repo storage modes and adoption machinery

Scope (sase repo only; `in_tree` and `is_in_tree` branches stay untouched throughout):

- Core store: drop `SDD_STORAGE_LOCAL` and `SDD_STORAGE_SEPARATE_REPO` from `src/sase/sdd/_store_types.py` and all
  resolution in `store.py` (missing record + provider-managed policy already errors or preflights after phase
  `sidecar-policy`; a record bearing an unknown/legacy storage value becomes a hard error naming `sase repo init`).
  Delete `src/sase/sdd/_store_adoption.py` and its call sites; delete the workspace-level `.sase/sdd` clone logic in
  `_store_link.py`; drop `companion_repos`/`companions` compatibility from `_store_records.py`; simplify
  `_commit.py`/`_commit_store.py` gating to sidecar-only; remove the `local` fallback in `beads.py` and the `.sase/sdd`
  exception fallback in `env.py`; remove `.sase/sdd` link fallbacks from `links.py`/`_write.py` and the legacy roots
  from `_paths.py` (`sdd_kind_roots` keeps in-tree `sdd/<kind>` for bare-git).
- Consumers: `bead/cli_common.py` (`_resolve_legacy_beads_location`, `local`/`separate_repo` branches),
  `bead/workspace.py` legacy dir; doctor legacy checks replaced by the hard-error path (`doctor/checks_config_sdd.py`,
  `checks_beads.py` legacy candidates); commit workflow `.sase/sdd` handling (`workflows/commit/plan_paths.py`,
  `commit_tracking.py:406`); commit finalizer legacy probes (`llm_provider/commit_finalizer.py:426-451`,
  `commit_finalizer_state.py`); axe `.sase/sdd` reference forms (`axe/run_agent_exec_plan_sdd.py` and the `sdd_in_tree`
  plumbing that only distinguished legacy forms — keep the in-tree form); plan/prompt search `.sase/sdd` roots
  (`prompt/search/sources.py`, `main/plan_search_render.py`); TUI legacy attribution and prefix stripping
  (`ace/tui/widgets/prompt_panel/_agent_commits.py`, `ace/tui/models/agent_plan_goal.py`); repo inventory legacy path
  (`repo_inventory.py`); `repo_init_handler.py` `_run_legacy_store_init`/`_plan_legacy_store_actions`; bead-path
  detection in `integrations/_mobile_helper_beads.py` and `agent/bead_display.py`.
- Contract cleanup: drop `separate_repo` from the accepted provider policy values and hookspec docs (the plugin migrated
  in phase `sidecar-policy`); remove `sdd.repo.name` and retired-key (`sdd.storage`, `sdd.version_controlled`) handling
  from `src/sase/default_config.yml`, config loading, and doctor (phase `legacy-state-cutover` guaranteed no real config
  still carries them).
- Do not touch the unrelated same-named "legacy" helpers: `workspace_handler_list.py` repo-open audit spellings and
  `is_legacy_static_linked_repo_record` (linked-repo records, not SDD storage).
- Tests: delete or rewrite legacy-mode suites (`tests/sdd_store/`, `tests/sdd_policy_helpers.py` policy stamps,
  `tests/test_bead/` legacy resolution cases, doctor suites, repo-init legacy plan/run cases, commit/axe/plan-approval
  in-tree-vs-store cases reduce to sidecar + in-tree). Add the hard-error coverage for legacy records.
- Docs: update `docs/sdd_storage.md` (mode table shrinks to `in_tree` + `sidecar_repos`, legacy sections replaced by a
  short "historical layouts" note), plus touched references in `docs/sdd.md`, `docs/beads.md`, `docs/configuration.md`,
  `docs/init.md`, `docs/workspace.md`, `docs/architecture.md`, `docs/commit_workflows.md`.
- Note for the implementer: this phase removes many public symbols — review `memory/symvision.md` (via
  `sase memory read`) before addressing Symvision lint fallout.

Acceptance: no references to `SDD_STORAGE_LOCAL`, `SDD_STORAGE_SEPARATE_REPO`, `companion_repos`, `companions`, or
runtime `.sase/sdd` paths remain in `src/` (docs may retain historical notes); encountering a legacy record at runtime
produces the hard error; `just check` passes.

## Testing

- Phases `context-boundaries`, `init-guards`, `sidecar-policy`, and `legacy-removal` each extend the unit suites named
  in their scope and must pass `just check` in the sase repo (plus the sase-github suite for `sidecar-policy`).
- Phase `legacy-state-cutover` is verified operationally with the per-project audit checklist and `sase doctor` /
  `sase init --check` runs; sase-repo `just check` applies only if sase-repo files change.
- End-to-end regression for the original incident (covered across phases): from a linked-repo clone under a host
  workspace, `sase init` aborts with guidance; from a project's registered checkout, `sase init` converges the store and
  derives the correct project name with no `SASE_ACTIVE_PROJECT_DIR` contamination.

## Risks & Notes

- **`in_tree` is explicitly out of scope for removal** — it is the current bare-git provider policy. If the user wants
  bare-git projects migrated to sidecars too, that is a follow-up epic; this plan keeps every `is_in_tree` branch.
- **Ordering is load-bearing.** The adoption/migration engine must survive until phase `legacy-state-cutover` has
  converged all on-disk state; `depends_on` encodes this. Do not land `legacy-removal` early.
- **Cross-repo coordination.** Phase `sidecar-policy` changes the sase-github plugin (open via `/sase_repo`; commit
  through the sanctioned commit workflow). The host keeps accepting `separate_repo` transitionally, so host-first
  deployment is safe; the cutover phase re-verifies the deployed CLI + plugin pairing before touching real state.
- **Marker-walk change has wide blast radius.** Every marker consumer (bead resolution, repo handlers, store
  maintenance) changes behavior for nested-clone cwds. The sidecar carve-out preserves the intended host inheritance;
  phase `context-boundaries` tests must cover bead and commit flows from sidecar clones explicitly.
- **Provider-detection design risk.** "Fail loudly when the plugin is missing" requires distinguishing a GitHub project
  without its plugin from a genuine bare-git project. The init-guards phase owns this design; acceptable approaches
  include consulting the project spec's provider or remote-shape claims, but silent downgrade must end.
- **Memory-file rule.** No phase hand-edits `memory/*.md`, `AGENTS.md`, or provider shims; any instruction-file drift is
  produced only by running `sase init` in the affected project, per repository policy.
- **Rust core boundary.** Verified untouched: no storage modes or SDD paths cross the Python↔Rust wire. If an
  implementing agent discovers otherwise, follow the `rust_core_backend_boundary` memory (update `sase-core` first).
- **Home project.** The hidden `home` project has no research sidecar and its commands often run from non-git
  directories, where the bounded walk intentionally preserves today's behavior; the cutover audit includes it so any
  legacy record it holds is converged or surfaced.
