---
tier: tale
title: Hidden agents sidecar foundation
goal: Every SASE-managed project has a configurable hidden agents sidecar that is
  explicitly accessible at one stable machine-level path without entering agent memory
  or workspace clones.
bead: sase-8k.4
parent: sase/repos/plans/202607/agents_sidecar_repo.md
create_time: 2026-07-22 10:58:48
status: done
---

- **PROMPT:** [202607/prompts/hidden_agents_sidecar_foundation.md](prompts/hidden_agents_sidecar_foundation.md)

# Plan: Hidden agents sidecar foundation

## Goal

Introduce `agents` as an intrinsically hidden sidecar role for every SASE-managed project. The role must be configurable
through the existing `repos.sidecar` contract, visible in repository inventory and repository CLI commands, absent from
agent-facing memory and workspace materialization, and anchored at one stable machine-level clone path per project:
`~/.sase/projects/<project_key>/repos/agents`.

This plan implements bead `sase-8k.4` only. It establishes the role, configuration, path, inventory, and visibility
semantics needed by later phases; it does not create or seed the remote repository, synchronize agent data, add machine
agent hoods, or close the parent epic.

## Current behavior and constraints

- `src/sase/_linked_repo_config.py` normalizes configured sidecars and injects only the managed-project `plans`
  fallback. Its default sidecar path points below a checkout's `sase/repos/` tree.
- `src/sase/linked_repos.py` treats every configured sidecar as a workspace-scoped repository and can map configured
  roles/slugs into `sase/repos/<role>` for each numbered workspace.
- `src/sase/main/init_memory/config.py` includes lazy, enabled sidecars in generated repository instructions.
- `src/sase/repo_inventory.py` and `src/sase/_repo_inventory_workspaces.py` expose sidecars with a clone matrix derived
  from each registered workspace. `sase repo path` and `sase repo open` consume that inventory and clone matrix.
- The SDD compatibility store is deliberately limited to `plans` and `research`; `agents` must remain config-driven and
  must not change the SDD store record shape.
- Hidden means excluded from agent workflows, not secret: `agents` must still appear as kind `sidecar` in
  `sase repo list`, and explicit CLI access must remain available.

## Implementation

### 1. Define and inject the hidden role

In `src/sase/_linked_repo_config.py`:

- Add exported constants for the `agents` role, its default description
  (`Hidden sidecar that stores commit-associated sase agent data for this project.`), and
  `HIDDEN_SIDECAR_ROLES = frozenset({"agents"})`.
- Extend `inject_default_linked_repos` so a project whose local config sets `is_sase_managed: true` receives both the
  existing `plans` default and an implicit `<project>--agents` sidecar with `auto_clone: false` and public visibility.
- Preserve the existing suppression and merge rules: a project-local `repos.sidecar` entry for role `agents` replaces
  the implicit entry; `disabled: true` removes it from active behavior; `visibility: private` keeps it enabled with the
  configured visibility and derived remote identity. `default_linked_repos: false` continues to suppress all implicit
  managed-project sidecars.
- Keep remote derivation and canonical SSH normalization on the existing sidecar identity path so the new role does not
  introduce separate remote-policy logic.

### 2. Enforce hiddenness at every agent-facing boundary

- In `src/sase/main/init_memory/config.py`, recognize hidden roles before description validation and omit them from
  generated repository entries whether implicit or explicit. A minimal explicit `agents` override must therefore not
  cause a missing-description error. Do not edit generated memory files, `AGENTS.md`, or provider instruction shims.
- In `src/sase/linked_repos.py`, exclude hidden roles and their slugs from sidecar clone-directory mapping and from
  `resolve_linked_repos_for_project`, even if configuration attempts to set `auto_clone: true`. This makes intrinsic
  hiddenness stronger than user-supplied materialization settings and prevents hidden entries from reaching launch
  metadata or linked-repository environment variables.
- Add an explicit defensive filter in `src/sase/axe/run_agent_runner_setup.py` so manually constructed or stale hidden
  sidecar resolutions are never materialized during agent launch.

### 3. Add the stable machine-level clone path and inventory representation

- Add an exported `hidden_sidecar_clone_dir(project_key, role)` helper in `src/sase/linked_repos.py`. It must construct
  the normalized path below `sase_projects_dir() / <project_key> / repos / <role>`, reject unsafe path components, and
  be independent of any primary or numbered workspace checkout.
- Update `src/sase/repo_inventory.py` so an enabled hidden sidecar is represented once as kind `sidecar`, using its role
  (`agents`) as the primary CLI name, the derived or pinned repository slug as its alternate lookup name, the standard
  description and remote metadata, and the machine-level path computed from the canonical project key. Disabled entries
  must be absent; unmanaged projects must not gain an implicit row.
- Update `src/sase/_repo_inventory_workspaces.py` so hidden sidecars do not acquire per-workspace `sase/repos/agents`
  paths. Their clone records should resolve every registered workspace number to the same machine-level path, keeping
  existing workspace-context validation while guaranteeing one clone per machine and project.
- Document the project-scoped `repos/<role>` state layout in `src/sase/core/paths.py`; do not add `agents` to the rigid
  SDD store record or compatibility writer.

### 4. Preserve explicit repository CLI access without workspace cloning

- Make `sase repo path agents` and lookup by the derived/pinned agents repository slug return the stable machine-level
  path for any registered workspace context. With `--ensure`, reuse the existing remote-identified sidecar
  materialization path at that same target so no checkout-local directory is created.
- Make `sase repo open agents -r <reason>` use the same stable path and existing repository-open audit behavior. Carry
  an explicit machine-scoped/hidden-sidecar signal through the repository target context or equivalent resolution seam
  so numbered workspace selection cannot redirect the clone into `sase/repos/agents`.
- Leave ordinary linked repositories and `plans`/`research` sidecars byte-for-byte compatible in path, clone-matrix,
  `--ensure`, and open behavior.

### 5. Update schema and user documentation

- Update `src/sase/config/sase.schema.json` descriptions for `default_linked_repos`, `repos.sidecar`, `name`,
  `auto_clone`, `disabled`, and `visibility` where needed to explain the intrinsic `agents` role, its machine-level
  location, and the existing project-local opt-out/private controls. No new user-facing configuration key is added.
- Update the repository configuration sections in `docs/configuration.md` to describe automatic `agents` injection for
  managed projects, hiddenness semantics, the stable clone location, inventory/CLI visibility, `disabled: true`, and
  `visibility: private`. Clearly distinguish this foundation from the later repo-init consent and sync behavior.

## Tests

Add or extend focused tests in the existing sidecar, memory, inventory, runner-setup, and repository-handler suites:

- Managed versus unmanaged default injection includes `agents` only for managed projects and preserves the existing
  `plans` behavior.
- Explicit agents overrides suppress the implicit default, with `disabled: true` removing it and `visibility: private`
  surviving normalization and remote derivation.
- Hidden roles never appear in memory entries, workspace sidecar-dir mapping, launch resolution, launch preparation,
  launch metadata, or checkout-local clone paths, including a hostile `auto_clone: true` override.
- The machine-level path helper uses `SASE_HOME`, the canonical project key, and a safe role; invalid path components
  are rejected.
- Repository inventory exposes one `agents` sidecar row with the expected role, slug, description, remote, stable path,
  and stable clone matrix; disabled and unmanaged cases expose none.
- `sase repo path agents`, slug lookup, `--ensure`, and `sase repo open agents` all use the stable path in workspace
  zero and numbered-workspace contexts without regressing ordinary sidecars.
- Schema/documentation assertions that already pin implicit sidecar wording are updated to the new contract.

Run focused tests while iterating, then run the repository-required validation sequence after all changes:

1. `just install`
2. `just check`

If `just check` changes formatting, inspect the diff and rerun `just check` until it passes on the final tree.

## Acceptance criteria

- Every SASE-managed project has an enabled implicit public `agents` sidecar unless project-local config disables or
  overrides it.
- The sidecar is visible and explicitly accessible through repository inventory/path/open commands.
- It is never emitted into generated agent instructions, launch metadata, environment variables, or any workspace's
  `sase/repos/` tree.
- Every workspace context for one project resolves the sidecar to exactly `~/.sase/projects/<project_key>/repos/agents`.
- Existing linked repositories, SDD store compatibility, and `plans`/`research` sidecar behavior remain intact.
- Focused tests and `just check` pass, then bead `sase-8k.4` is closed without changing parent epic `sase-8k`.

## Out of scope

- Creating, confirming, seeding, or pushing the agents sidecar during `sase repo init` (`sase-8k.5`).
- Agent bundle formats, export/import, sync status, sync CLI, periodic checks, or TUI indicators.
- Machine-name configuration or machine agent-hood qualification.
- Expanding `SddStoreRecord` beyond its `plans` and `research` slots.
- Editing SASE memory files or generated instruction shims.
- Closing the parent epic or creating additional beads.
