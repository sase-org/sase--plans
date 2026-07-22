---
tier: epic
title: Hidden agents sidecar repo with machine agent hoods
goal: 'Every sase-managed project gains a hidden `<project>--agents` sidecar repo
  that publicly (or privately, by config) stores the data needed to reconstruct every
  sase agent that created a commit for that project; agent names become globally unique
  via required machine agent hoods; syncing is wired into `sase repo init`, the `,U`
  comprehensive update, a cheap periodic per-project check, and a distinct top-right
  TUI indicator.

  '
phases:
- id: machine-config
  title: machine_name config and sase config init
  depends_on: []
  size: medium
  description: '''machine_name config and sase config init'' section: add the required
    machine_name schema field, machine-local identity selection for sase_<machine>.yml
    overlays, the interactive `sase config init` command (chezmoi-aware writes), and
    `sase init`/doctor wiring.

    '
- id: core-hood-helpers
  title: Rust core machine-hood helpers
  depends_on: []
  size: small
  description: '''Rust core machine-hood helpers'' section: add qualify/strip/validate
    machine-hood functions to sase_core, expose them through pyo3 bindings, and cover
    them with unit and parity tests.

    '
- id: machine-hoods
  title: Machine agent hoods end to end
  depends_on:
  - machine-config
  - core-hood-helpers
  size: large
  description: '''Machine agent hoods end to end'' section: qualify new agent names
    with the local machine hood when written to disk, enforce hood-aware uniqueness
    with legacy-name equivalence, strip the local hood across every display/prompt/lookup
    surface, and record machine-aware commit footers.

    '
- id: sidecar-foundation
  title: Hidden agents sidecar foundation
  depends_on: []
  size: medium
  description: '''Hidden agents sidecar foundation'' section: introduce the intrinsically
    hidden `agents` sidecar role, default injection for sase-managed projects, the
    machine-level clone location under ~/.sase, and project-local disable/visibility
    controls.

    '
- id: repo-init
  title: sase repo init integration
  depends_on:
  - sidecar-foundation
  size: medium
  description: '''sase repo init integration'' section: create, seed, and clone the
    agents sidecar during `sase repo init` behind a loud, explicit confirmation that
    all commit-associated agent chats will be pushed to the new repo.

    '
- id: sync-engine
  title: Agents sync engine and CLI
  depends_on:
  - machine-hoods
  - repo-init
  size: large
  description: '''Agents sync engine and CLI'' section: implement the per-agent bundle
    format, the pull -> integrate -> export -> commit -> push sync flow with locking
    and retries, historical backfill from commit footers, and the `sase agent sync`
    command.

    '
- id: tui-sync
  title: TUI sync checks, indicator, and update keymap leg
  depends_on:
  - sync-engine
  size: medium
  description: '''TUI sync checks, indicator, and update keymap leg'' section: add
    the cheap periodic per-project sync-status check, the distinct top-right sync
    indicator widget, and the agents leg of the `,U` comprehensive update.

    '
- id: smoke
  title: End-to-end exercises and docs
  depends_on:
  - tui-sync
  size: small
  description: '''End-to-end exercises and docs'' section: exercise the full flow
    end to end across two simulated machines and finish user-facing documentation.

    '
create_time: 2026-07-22 10:53:31
status: wip
---

# Plan: Hidden agents sidecar repo with machine agent hoods

## Overview and design principles

This epic adds a new, hidden sidecar repo — `<project>--agents` — for every sase-managed project. It stores everything
needed to reconstruct the sase agents that created commits for that project (chat transcripts, agent metadata, commit
associations), so anyone working on the project through sase shares the same historical agent record. To keep agent
names globally unique across machines, every agent gains a top-level _machine agent hood_ derived from a new required
`machine_name` config field.

Design principles that shape every phase:

1. **Background work is read-only; data flows on explicit actions.** Periodic checks only _detect_ that a sync is needed
   and light an indicator. Actual pull/integrate/export/push happens on `,U` (comprehensive update), `sase agent sync`,
   an indicator click, or `sase repo init`. No silent background pushes of chat data, ever.
2. **Local disk stays the source of truth.** The sidecar is a transport/replica. Importing integrates foreign agents
   into the existing local representation (artifact dirs, chats, name registry); it never mutates or overwrites agents
   owned by the local machine.
3. **Hidden means invisible to agent workflows, not secret.** The agents sidecar never appears in generated memory files
   and is never cloned into any workspace's `sase/repos/`. It _does_ appear in `sase repo list` (kind `sidecar`) so the
   user can always see and audit it.
4. **Reuse existing shapes.** The sidecar rides the config-driven `repos.sidecar` machinery (with its existing
   `disabled` and `visibility` fields), the sync check mirrors the update-status engine (snapshot cache + cheap
   revalidation + long-cadence recompute), and the indicator mirrors `UpdatesAvailableIndicator`.
5. **Only completed, commit-associated agents are exported.** An agent qualifies for export when it has a terminal
   marker (`done.json`) and at least one commit in the project's primary repo carries its `SASE_AGENT` footer tag.
   Agents that never committed are never published.

### Key pre-explored facts (verified in this repo)

- Sidecar config entries: `src/sase/_linked_repo_config.py` — `merged_sidecar_entries_from_config` (~line 136,
  normalizes `disabled`, defaults `visibility` to `"public"` at ~line 183, defaults `auto_clone` to False at ~line 182),
  `inject_default_linked_repos` (~line 271, injects the default plans sidecar only when `is_sase_managed` is true),
  sidecar slug derivation `_derived_sidecar_repo` (~line 424, builds `{project_slug}--{role}` and `owner/slug`).
- The SDD store record is rigidly two-slot (`plans`, `research`): `src/sase/sdd/_store_types.py`
  `SddStoreRecord`/`sidecar_for_kind` (~lines 37-61) and `SPLIT_SIDECAR_KINDS = ("plans", "research")` in
  `src/sase/sdd/_sidecar_init.py` (~line 39). The `agents` role must therefore be config-driven only and skipped by
  `_write_compatibility_store`.
- `sase repo init`: `src/sase/main/repo_init_handler.py` — `run_repo_init` (~line 51), the `is_sase_managed` gate (~line
  77), `_explicit_sidecar_config_update` (~line 466, ruamel writes into project `sase/sase.yml`),
  `_configured_sidecar_specs` (~line 407), `_confirm_sidecar_creation` (~line 298, TTY-gated y/N). Creation/clone/seed:
  `src/sase/sdd/_sidecar_init.py` `initialize_sidecars` (~line 108), visibility flows to the provider as
  `options["sdd_visibility"]` (~line 378).
- Memory "Repositories" section: `src/sase/main/init_memory/config.py` `linked_entries_from_config` — the sidecar loop
  already skips `auto_clone or disabled` (~line 296); a hidden role must be added to that skip.
- Workspace auto-clone driver: `src/sase/axe/run_agent_runner_setup.py` `prepare_linked_repo_workspaces_if_needed`
  (~lines 82-145) clones only `auto_clone` repos.
- Config layering: `src/sase/config/core.py` `load_merged_config` (~line 410) merges default -> plugin ->
  `~/.config/sase/sase.yml` -> all `~/.config/sase/sase_*.yml` overlays (sorted; `_get_overlay_paths` ~line 324) ->
  project-local `sase/sase.yml`. Config cache token: `current_config_token` (~line 158); programmatic edits go through
  `src/sase/config/edit.py` / `_edit_plan.py` (`set_key` splices, chezmoi remap via `resolve_write_path` in
  `src/sase/config/targets.py`, `overlay_config_path` ~line 56). Schema: `src/sase/config/sase.schema.json` (no
  top-level `required` array today).
- Init registry: `src/sase/main/init_registry.py` `InitCommandSpec`/`iter_init_command_specs` (~lines 12-52) — adding a
  spec wires both bare `sase init` and the doctor `config.init` check automatically. Parser aliases:
  `src/sase/main/parser_init.py` `register_init_parser`. `config` CLI group: `src/sase/main/parser_commands.py`
  `register_config_parser` (~line 29), dispatch in `src/sase/main/config_handler.py`.
- Agent naming: no single AgentName type; names are `str`. Hood parsing lives in
  `src/sase/ace/tui/models/agent_hoods.py` (`agent_hood` ~line 44, `agent_name_key` ~line 20, `_agent_hood_chain` ~line
  379); family `--` grammar in `src/sase/plan_chain.py` (`AGENT_FAMILY_SEPARATOR` line 9); user-name validation in
  `src/sase/agent/launch_validation.py` (`validate_user_agent_name` ~line 107); durable uniqueness in
  `src/sase/agent/names/_registry.py` (`~/.sase/agent_name_registry.json`) with namespace collision logic in
  `src/sase/agent/names/_templates.py` (`AgentNameNamespaceReservationIndex.candidate_available` ~line 91). The
  `add_dismissed_prefix`/`strip_dismissed_prefix` pair in `src/sase/agent/names/_common.py` (~lines 111-134) is the
  exact precedent for a stored-but-stripped name prefix.
- Agent display: `Agent.refresh_presented_agent_name()` in `src/sase/ace/tui/models/agent.py` (~line 100) computes
  `presented_agent_name`, rendered by `src/sase/ace/tui/widgets/_agent_list_render_agent.py` (~line 401) and the
  prompt-panel display helpers.
- Commit <-> agent linkage: `src/sase/workflows/commit/runtime_tags.py` writes `SASE_AGENT=<name>` /
  `SASE_MACHINE=<hostname>` footers into every commit (`RUNTIME_COMMIT_TAG_KEYS` line 18, `_resolve_machine_name` ~line
  73 currently uses `socket.gethostname()`); parse side is Rust-backed via `src/sase/core/commit_footer_facade.py`. Chat
  transcripts: `~/.sase/chats/YYYYMM/*.md` (`src/sase/history/chat_storage.py`, agent name embedded in filename and
  header). Agent artifact dirs: `~/.sase/projects/<key>/artifacts/ace-run/YYYYMM/DD/<timestamp>/` with
  `agent_meta.json` + `done.json` (Rust-backed paths via `src/sase/core/agent_artifact_paths.py`).
- Update checks + indicator (the template for phase tui-sync): `src/sase/ace/tui/actions/update_toast.py` (10-min
  `set_interval`, thread workers, `revalidate_only` ticks vs 60-min network recompute), engine in `src/sase/updates/`
  (snapshot cache `~/.sase/updates/status_snapshot.json`), widget `src/sase/ace/tui/widgets/updates_indicator.py`,
  top-bar composition in `src/sase/ace/tui/app.py` (~lines 436-443), `,U` flow entry `action_update_sase_shortcut` in
  `src/sase/ace/tui/actions/base.py` (~line 144) into the comprehensive-update mixins under `src/sase/ace/tui/modals/`.
- Rust core (open it with the `/sase_repo` skill: `sase repo open sase-core -r "<reason>"`): no AgentName type and no
  machine concept exist there. Relevant existing pieces: `crates/sase_core/src/agent_name_template.rs` (template
  grammar, `namespace_template`), `crates/sase_core/src/axe_chop/validation.rs` `validate_dotted_agent_name` (~line
  362), hood prefix matcher in `crates/sase_core/src/axe_chop/decision.rs` (~lines 35-53),
  `crates/sase_core/src/commit_footer.rs`, pyo3 module `crates/sase_core_py/src/lib.rs` (`#[pymodule] sase_core_rs`
  ~line 4739). Python binding loader: `src/sase/core/rust.py` `require_rust_binding`.

### Out of scope (do not do in any phase)

- No mass rename/migration of pre-existing on-disk agent names. Legacy unqualified names are treated as belonging to the
  local machine via equivalence logic (see machine-hoods phase).
- No edits to `sase/memory/*.md`, `AGENTS.md`, or provider instruction shims. New glossary terms (Machine Agent Hood,
  Agents Sidecar) need explicit user permission in a live conversation; note the suggestion in the final phase summary
  instead.
- No reconciliation of the sase_gateway `$HOSTNAME` host label with `machine_name`.
- No import of _running_ foreign agents; only terminal agents are exported/imported.

---

## machine_name config and sase config init

Add the machine identity layer everything else builds on.

**Schema + accessor**

- Add `machine_name` to `src/sase/config/sase.schema.json`: type string, pattern `^[a-z_]+$`, description explaining it
  names this machine's top-level agent hood. Add a top-level `"required": ["machine_name"]` array so the Rust-backed
  config inventory and the Config Center surface a missing value as a diagnostic. (Runtime `load_merged_config` never
  enforces `required`, so existing setups keep working; verified.)
- Add a `get_machine_name() -> str | None` accessor in `src/sase/config/core.py` (merged config read) plus a
  `require_machine_name()` helper that raises a clear, actionable error ("run `sase config init`") for callers that
  hard-require it (sync/export paths).
- Do NOT add `machine_name` to `src/sase/default_config.yml` — there is no sane default; it must be explicitly
  configured per machine.

**Machine-local identity selection (multi-machine overlay safety)**

Problem: the overlay glob merges _all_ `~/.config/sase/sase_*.yml` files, and chezmoi users sync their config dir across
machines — so every machine would see every other machine's `sase_<name>.yml` and the alphabetically-last `machine_name`
would win. Fix:

- Introduce a machine-local selector file `~/.sase/machine_name` (plain text, single line; `~/.sase` is machine-local
  state and never chezmoi-synced). Document it in the `src/sase/core/paths.py` header list.
- In config loading (`src/sase/config/core.py`), classify any `sase_*.yml` overlay whose top-level mapping contains
  `machine_name` as a _machine overlay_. Include a machine overlay in the merge only when its `machine_name` equals the
  selector file's content. Non-machine overlays keep today's behavior exactly. When the selector file is missing,
  exclude all machine overlays (so `machine_name` resolves unset and init prompting kicks in).
- Add the selector file's stat to `_compute_current_config_token` so cache invalidation stays correct (mind gotcha:
  `current_config_token()` is the render-path cache key; keep the added work to one stat).

**`sase config init` command**

- Parser: add an `init` subparser to `register_config_parser` in `src/sase/main/parser_commands.py` (keep subcommands
  alphabetically sorted; give every public long option a short alias; the `config` group keeps its bare-default behavior
  via `_default_list_subcommands()` untouched unless a `list` child exists). Dispatch in
  `src/sase/main/config_handler.py`.
- Behavior (idempotent):
  1. If `machine_name` already resolves and the selector file matches: report "already configured" and exit 0.
  2. Scan `~/.config/sase/sase_*.yml` for existing machine overlays. If any exist, offer their names as choices ("Found
     machine config for: athena, zeus") before prompting for a new name — picking an existing name writes only the
     selector file (minimal change).
  3. Otherwise prompt for a machine name (TTY-gated, default suggestion: sanitized lowercase hostname mapped through
     `[^a-z_] -> _`). Validate against `^[a-z_]+$`; re-prompt on invalid input. Warn (and require confirmation) if the
     chosen name collides with an existing reserved top-level agent name/hood in the name registry
     (`get_reserved_agent_names` in `src/sase/agent/names/_registry.py`).
  4. Write `machine_name: <name>` into `~/.config/sase/sase_<name>.yml` — create the file if missing; otherwise make the
     minimal splice via the existing config-edit machinery (`set_key` in `src/sase/config/_edit_yaml.py`, target
     resolution through `overlay_config_path` + `resolve_write_path` so `use_chezmoi: true` remaps the write to the
     chezmoi source tree and the existing deploy pipeline (`src/sase/main/_init_chezmoi_deploy.py`)
     commits/pushes/applies).
  5. Write the selector file, then `clear_config_cache()`.
- Follow the interactive-prompt precedent `_prompt_for_plan` / `_confirm_sidecar_creation` (thread
  `args._init_input_func` through for tests; refuse to prompt on non-TTY with a clear message and non-zero exit).

**`sase init` and doctor wiring**

- Add a `config` `InitCommandSpec` to `iter_init_command_specs` in `src/sase/main/init_registry.py`, ordered _first_
  (memory/repo flows may depend on machine identity later). Provide `plan_config_init(args)` (reports whether
  `machine_name` is configured; check-mode friendly) and `run_config_init(args)`.
- Add a `config` alias subparser to `register_init_parser` in `src/sase/main/parser_init.py` (so `sase init config`
  works like `init memory`/`init repo`), and dispatch in `src/sase/main/entry.py`. The doctor `config.init` check
  (`src/sase/doctor/checks_config_init.py`) picks the new spec up automatically — verify.

**Docs + tests**

- Document `machine_name`, machine overlays, and the selector file in `docs/configuration.md` (overlay section ~lines
  54-62 and a new subsection).
- Tests: overlay filtering (foreign machine overlays excluded; selector missing excludes all; non-machine overlays
  unaffected), config-token invalidation, `sase config init`
  create/minimal-edit/existing-choice/validation/collision-warning paths (use `_init_input_func` injection), chezmoi
  write remapping (mock deploy), `sase init` ordering and doctor check. Follow existing config tests (e.g.
  `tests/test_config_inventory.py`) for conventions.

## Rust core machine-hood helpers

Machine-hood canonicalization is domain logic any frontend must reproduce, so the canonical helpers live in the Rust
core. Open the repo with the `/sase_repo` skill (`sase repo open sase-core -r "..."`) and work in that checkout; changes
land as a separate sase-core PR that must merge (and the `sase-core-rs` dependency rebuild) before the dependent
machine-hoods phase runs.

- New module `crates/sase_core/src/machine_hood.rs` with:
  - `validate_machine_name(name) -> Result<...>`: non-empty, `^[a-z_]+$`.
  - `qualify_machine_agent_name(name, machine_name) -> String`: return `name` unchanged when it already starts with
    `<machine_name>.`; otherwise prepend `<machine_name>.`. Idempotent.
  - `strip_machine_agent_name(name, machine_name) -> String`: remove a leading `<machine_name>.` if present (the
    inverse; also idempotent). Never strips mid-name segments and never strips when the remainder would be empty.
  - `machine_hood_of(name, known_machines) -> Option<String>`: return the leading hood segment when it names a known
    machine (used by import/display to classify foreign agents).
  - Reuse the boundary semantics already proven in `crates/sase_core/src/axe_chop/decision.rs` (~lines 35-53): a name
    belongs to hood X iff it equals X or starts with `X.`.
- Expose all four through the pyo3 module in `crates/sase_core_py/src/lib.rs` (register in the `#[pymodule]` block ~line
  4739; follow the existing wrapper style, doc-comment index ~lines 72-108).
- Tests: inline `#[cfg(test)]` unit tests in `machine_hood.rs` (idempotence, family `--` names, nested hoods,
  empty-remainder guard, validation rejects uppercase/digits/dots), and a small binding test in
  `crates/sase_core_py/src/lib.rs` tests (~line 4993). No wire structs change in this phase, so no schema-version bumps
  or `python_wire_parity.rs` fixtures are needed.
- Python facade in this repo: new `src/sase/core/machine_hood_facade.py` wrapping `require_rust_binding` (mirror
  `src/sase/core/commit_footer_facade.py`), so later phases import from one place. If the rebuilt binding is not yet
  published when this phase's Python facade is authored, gate facade tests on binding availability the same way other
  core facades do.

## Machine agent hoods end to end

Qualify agent names with the local machine hood on every write path, and strip it on every local read/display path.
Foreign hoods (other machines) always display in full. All hood math goes through
`src/sase/core/machine_hood_facade.py`.

**Semantics (the contract):**

- On-disk / durable form: `<machine_name>.<local-name>`, e.g. `athena.foo`, `athena.foo--code`, `athena.chop.x.y`. This
  applies to: the name registry key, `agent_meta.json` `name`/`workflow_name`, `SASE_AGENT_NAME` env, chat
  filenames/headers, and commit footers.
- Display / prompt form on the owning machine: the bare local name (`foo`). Prompts typed on athena referring to `foo`
  and `athena.foo` mean the same agent; both resolve.
- When `machine_name` is not configured: behavior is exactly today's (unqualified). Nothing breaks; `sase config init`
  is the opt-in gate.
- Legacy equivalence: an existing on-disk unqualified name is treated as local. Uniqueness and lookups must compare on
  _qualified_ form: candidate `athena.foo` collides with reserved legacy `foo`, and candidate `foo` collides with
  reserved `athena.foo`.
- Explicitly typing a _different_ known machine's hood as a new agent name prefix is rejected by launch validation with
  a clear error (foreign agents are imported, never launched locally).

**Write-path qualification (find the narrowest choke points):**

- Name allocation/validation at launch: `src/sase/agent/launch_validation.py` (`validate_user_agent_name`,
  `validate_launch_name_requests`) and auto-allocation in `src/sase/agent/names/_auto.py` / `_templates.py`
  (`allocate_agent_name_template`). Auto names allocate the bare token, then qualify before reservation, so the visible
  sequence (`0`, `1`, ... ) is unaffected.
- Registry claims (`claim_registered_name` in `src/sase/agent/names/_registry.py`) store the qualified name; extend
  `AgentNameNamespaceReservationIndex.candidate_available` (`src/sase/agent/names/_templates.py` ~line 91) with
  qualified/legacy equivalence so both collision directions are caught.
- Launch env + meta: wherever `SASE_AGENT_NAME` and `agent_meta.json` names are written (agent launch prep under
  `src/sase/axe/`), the qualified name flows through.
- Clan composition (`%clan`) and family child allocation (`src/sase/plan_chain.py` `_allocate_agent_family_child_name`)
  must qualify the _root_ once, not every segment — `athena.myclan.member`, `athena.family--code`.

**Read-path stripping (audit every user-visible surface):**

- `Agent.refresh_presented_agent_name()` in `src/sase/ace/tui/models/agent.py` (~line 100): strip the local hood for
  `presented_agent_name`; keep `agent_name` raw. Foreign machine hoods pass through untouched, so imported agents
  naturally render as `zeus.bar` and group under a `zeus` hood in the Agents tab.
- Hood grouping: `src/sase/ace/tui/models/agent_hoods.py` (`agent_hood`, `_agent_hood_chain`) must skip the local
  machine hood so it never appears as a kinship group for local agents.
- Prompt-side resolution: name lookups and `@name` references (`src/sase/agent/names/_lookup*.py`, template reference
  resolution, completion candidates in `src/sase/ace/tui/widgets/_agent_completion_candidates.py`) accept bare names and
  resolve them against qualified registry entries; completions display stripped names.
- Chat display: `src/sase/history/chat_catalog.py` header/agent parsing keeps working with qualified names; `sase chats`
  listings show stripped local names, full foreign names.
- Grep-audit for other render sites (`presented_agent_name`, `display_name`, `humanize_cl_name` call sites) and
  normalize each deliberately; when in doubt, strip only at the final render layer, never in stored data.

**Commit footers:**

- `src/sase/workflows/commit/runtime_tags.py`: `SASE_AGENT` records the qualified name; `_resolve_machine_name()`
  switches from `socket.gethostname()` to `get_machine_name()` with hostname fallback, so `SASE_MACHINE` equals the
  machine hood going forward. Legacy commits (unqualified agent + hostname machine) remain parseable — the sync engine's
  backfill treats unqualified footer names as local.

**Tests:**

- Unit: qualification idempotence at every choke point; legacy-equivalence collisions both directions; foreign-hood
  launch rejection; clan/family composition shapes; footer tags.
- TUI: presented-name stripping, hood grouping skip, completion display, with a foreign (`zeus.`) agent fixture
  rendering fully qualified. Follow existing loader/display test conventions under `tests/ace/tui/`.
- The machine name in tests is injected via config fixtures (no reliance on real hostname).

## Hidden agents sidecar foundation

Introduce the `agents` sidecar role and its "hidden" nature as an intrinsic property of the role (no new user-facing
config key).

- New role constants in `src/sase/_linked_repo_config.py`: role `"agents"`, `DEFAULT_AGENTS_DESCRIPTION` ("Hidden
  sidecar that stores commit-associated sase agent data for this project."), and
  `HIDDEN_SIDECAR_ROLES = frozenset({"agents"})` exported for the other integration points.
- Default injection: extend `inject_default_linked_repos` (~line 271) so sase-managed projects get an implicit `agents`
  entry (`{project}--agents`, `auto_clone: False`, `visibility: "public"`), exactly parallel to the implicit plans
  entry. The existing suppression contract holds: an explicit project-local `repos.sidecar` entry for `agents` with
  `disabled: true` fully disables it; `visibility: private` keeps it enabled but makes the GitHub repo private. Both
  fields already exist in the schema (`sidecarRepo` definition) — verify their descriptions mention the agents role and
  update `docs/configuration.md`.
- Hidden-ness enforcement (each site keyed off `HIDDEN_SIDECAR_ROLES`):
  - Memory generation: skip hidden roles in the sidecar loop of `linked_entries_from_config`
    (`src/sase/main/init_memory/config.py` ~line 296) so the repo never appears in `sase/memory/sase.md` / `AGENTS.md`.
    (Do not edit any memory files in this phase; the next `sase memory init` run by the user will simply not list it.)
  - Workspace materialization: hidden roles are never cloned into any checkout's `sase/repos/` — excluded from
    `_sdd_sidecar_repo_dirnames`/clone-dir resolution in `src/sase/linked_repos.py` and from launch-workspace prep
    (already implied by `auto_clone: False` in `src/sase/axe/run_agent_runner_setup.py`; make it explicit and tested).
- Machine-level clone location: hidden sidecars clone once per machine per project at
  `~/.sase/projects/<project_key>/repos/<role>` (new helper in `src/sase/linked_repos.py`, e.g.
  `hidden_sidecar_clone_dir(project_key, role)`; document the layout in `src/sase/core/paths.py`). This keeps the repo
  out of every ephemeral workspace while giving sync a stable path.
- Inventory: `src/sase/repo_inventory.py` lists the agents sidecar with kind `sidecar` and its machine-level path so
  `sase repo list` shows it (principle 3). `sase repo path agents --ensure` (`src/sase/main/repo_handler_path.py`)
  resolves/clones the machine-level location. `sase repo open agents` works like any sidecar.
- Remote policy: the derived remote uses the existing SSH canonicalization and `enforce_sidecar_remote_policy`
  (`src/sase/_git_remote.py` ~line 128) unchanged.
- Tests: injection defaults for managed vs unmanaged projects; disable/private overrides; memory-entry exclusion;
  workspace-clone exclusion; inventory/path/open resolution to the machine-level dir.

## sase repo init integration

Teach `sase repo init` (and therefore bare `sase init`) to create the agents sidecar with an unmissable consent step.

- `_explicit_sidecar_config_update` (`src/sase/main/repo_init_handler.py` ~line 466): also write an explicit `agents`
  entry (name + description) into the project's `sase/sase.yml`, mirroring the plans/research entries, unless an entry
  for the role already exists (including `disabled: true` — never resurrect a disabled entry).
- `_configured_sidecar_specs` (~line 407) includes the agents spec (skipping disabled), so preflight
  (`preflight_sidecars`) and creation (`initialize_sidecars` in `src/sase/sdd/_sidecar_init.py`) flow through the
  generic path with `options["sdd_visibility"]` honoring the configured visibility. Required adaptations:
  - Clone target: hidden roles clone to the machine-level dir (previous phase's helper), not
    `<checkout>/sase/repos/<role>`.
  - `_write_compatibility_store` must not attempt to record the agents role (the store record only has plans/research
    slots).
- Confirmation must be loud and specific. For the agents role, replace the generic `_confirm_sidecar_creation` prompt
  with an explicit consent block, e.g.:

  ```
  The agents sidecar will PUBLISH sase agent data for this project: the full chat
  transcripts (prompts and responses), agent metadata, and commit associations of every
  sase agent that creates a commit here — from this machine and any other machine that
  works on this project. Repo visibility: PUBLIC.
  (Set repos.sidecar[agents].visibility: private in sase/sase.yml for a private repo, or
  disabled: true to opt out.)

  Create public agents sidecar repository <owner>/<project>--agents on <host>? [y/N]
  ```

  Default is No; non-TTY refuses creation (and reports how to re-run). Declining the agents repo must not abort the rest
  of repo init.

- Seeding: seed the new repo with a `README.md` that explains what the repo is, the bundle layout, the privacy
  implications, and how consumers sync (`sase agent sync`), plus an empty `agents/` tree and a `manifest.json` (schema
  in the sync-engine phase). Follow the `_seed_sidecars` pattern; commit via `commit_sdd_files` and push.
- `--check`/`--diff` plan output (`plan_repo_init`) must describe the pending agents-repo creation, its visibility, and
  the machine-level clone path.
- Tests: init plan/diff rendering; explicit-config idempotence (second run makes no file change); disabled entry
  suppresses creation; visibility private flows to the provider; compatibility store untouched; confirmation gate
  (default-no, non-TTY refusal, decline continues other sidecars) via `_init_input_func` injection; seed contents.

## Agents sync engine and CLI

The heart of the feature: a new `src/sase/agents_sync/` package implementing bundles, export, import, sync, backfill,
and status — plus the `sase agent sync` CLI.

**Bundle format (in the sidecar repo):**

```
README.md
manifest.json                    # {"schema_version": 1, "agents": {"<qualified-name>": {"digest": "...", "machine": "...", "updated_at": "..."}}}
agents/<qualified-name>/
  meta.json                      # portable agent record (schema_version, qualified name, machine,
                                 #   workflow, model, llm_provider, changespec/cl name, run timestamps,
                                 #   artifact timestamp + layout version for re-materialization)
  chat.md                        # the chat transcript (verbatim copy)
  commits.json                   # [{"sha": ..., "subject": ..., "committed_at": ...}]
```

- Directory names are the qualified agent name verbatim (dots and `--` are filesystem-safe). Machine hoods make them
  globally unique.
- `meta.json` is a _portable projection_ of `agent_meta.json` — never copy machine-local fields (pids, workspace paths,
  absolute paths). Define the projection explicitly with a schema_version.
- The per-agent `digest` (stable hash over the bundle payload) makes both export ("is my local copy already up there?")
  and status checks cheap — no tree diffing.

**Export (`export_project_agents`):**

- Enumerate candidates: completed local agents (terminal `done.json`) whose qualified name appears in a `SASE_AGENT`
  footer of the project's primary-repo history. Two sources, merged: (a) incremental — commit result markers/changespec
  records written by `src/sase/workflows/commit/commit_tracking.py`; (b) backfill — `git log` over the primary repo
  parsed with `src/sase/core/commit_footer_facade.py` (unqualified legacy footer names are treated as local and
  qualified before matching; entries whose local artifacts/chats no longer exist are skipped with a log line, not an
  error).
- For each candidate missing from or stale in the manifest: write the bundle from the local artifact dir
  (`agent_meta.json`) + chat storage (`src/sase/history/chat_storage.py`) + commit metadata. Update `manifest.json`.
- Requires `machine_name` (via `require_machine_name()`); refuses with the actionable error otherwise.

**Import (`integrate_foreign_agents`):**

- For each manifest entry whose `machine` differs from the local machine and whose digest is not yet integrated locally:
  materialize the local representation —
  - artifact dir via `src/sase/core/agent_artifact_paths.py` (canonical layout, using the bundle's recorded timestamp;
    on a same-second collision, probe forward one second at a time until free), writing a reconstructed
    `agent_meta.json` (qualified name, plus provenance keys `imported_from_machine` and `imported_digest`) and a
    terminal `done.json`;
  - chat file into `~/.sase/chats/YYYYMM/` using the standard filename format so `sase chats` and the TUI pick it up;
  - name-registry reservation (`claim_registered_name`) so local naming can never collide.
- Verify the Rust agent-scan path tolerates the two extra provenance keys in `agent_meta.json` (the scan wire is a
  projection; if it rejects unknown keys, coordinate a minimal sase-core change via `/sase_repo` — but expect none to be
  needed).
- Idempotent by digest: re-running sync never duplicates. Never touches bundles whose machine equals the local machine
  (local disk is authoritative for our own agents).

**Sync (`sync_project_agents_repo`):** per project: acquire a per-project lock file (bounded wait, mirror the
materialization-lock pattern) → ensure the machine-level clone exists (skip with a status note if the sidecar is
disabled or not yet created) → `git pull --rebase` (network timeout via the existing `network_git_timeout`;
`GIT_TERMINAL_PROMPT=0` and ssh BatchMode so a credential prompt can never hang a caller) → `integrate_foreign_agents` →
`export_project_agents` → if dirty: commit (`chore(agents): sync from <machine>`) and push; on push rejection, pull
--rebase and retry once → write the status snapshot (below). Every step reports into a structured per-project
`SyncOutcome` so callers (CLI, `,U` leg) can render truthful summaries.

**Status (`compute_agents_sync_status`):** cheap by construction —

- Snapshot cache at `~/.sase/agents_sync/status_snapshot.json` (schema_version, per-project: behind/ahead counts,
  unexported-agent count, last fetch time; atomic `os.replace` writes; mirror `src/sase/updates/cache.py`).
- Revalidate tick (no network): compare local clone HEAD vs its already-fetched upstream ref, and recheck the unexported
  count from the incremental markers.
- Recompute (network) only at the long cadence or on `--refresh`: `git fetch` per project, staggered so N projects never
  fetch in the same burst.

**CLI: `sase agent sync`** (new subcommand of the existing `sase agent` group; keep the group's subcommand listing
alphabetized; every long option gets a short alias):

- Default: sync all enabled projects (enumerate via the project lifecycle facade); report a colored per-project table
  (pulled / integrated / exported / pushed / skipped-why).
- Options: `-p/--project <name>` (repeatable), `-c/--check` (status only, no mutation, uses the snapshot + revalidation;
  `-r/--refresh` forces the network recompute), `-j/--json` (machine-readable outcomes/status).
- Excellent `-h` output per the CLI rules; document the group's bare-`list` default behavior remains unchanged.
- Update `docs/` (new `docs/agents_sidecar.md`: what the repo is, layout, privacy model, disable/private config, sync
  semantics, machine hoods interplay) and cross-link from `docs/configuration.md`.

**Tests:** bundle round-trip (export → wipe → import reproduces artifacts/chat/registry); digest idempotence;
legacy-footer backfill; foreign-vs-local partitioning; timestamp collision probing; lock contention; push-rejection
retry; status snapshot/revalidate semantics; CLI table/json/check modes. Use tmp `SASE_HOME` + local bare git remotes to
simulate the sidecar remote (no network).

## TUI sync checks, indicator, and update keymap leg

Surface sync state in `sase ace` without hurting performance, and fold syncing into `,U`.

**Periodic check** (mirror `src/sase/ace/tui/actions/update_toast.py` exactly in shape):

- New mixin (e.g. `src/sase/ace/tui/actions/agents_sync_check.py`): one `set_interval` timer registered from the
  post-mount startup loads; the timer callback is thin/sync and only schedules; the body runs in a
  `run_worker(..., thread=True)` with an in-flight guard. Per tui_perf rule 10: ticks call `compute_agents_sync_status`
  in revalidate mode (no network); the network fetch happens only at the longer recompute cadence, staggered across
  projects inside the engine. No stats/globs on render paths; results land on the UI thread via `call_from_thread`.
- Config knobs in `src/sase/default_config.yml` + schema, under `ace.agents_sync`: `check_interval_minutes: 10`,
  `recompute_interval_minutes: 30`, `indicator: true`. (Gotcha memory: default_config.yml must be updated alongside any
  new config values.)
- State attrs initialized in `src/sase/ace/tui/actions/_state_init.py` (timer handle, in-flight flag, last status).

**Indicator** (top-right, distinct from every existing badge):

- New widget `src/sase/ace/tui/widgets/agents_sync_indicator.py` (`AgentsSyncIndicator(Static)`, id
  `agents-sync-indicator`), modeled on `updates_indicator.py`: hidden at zero; otherwise renders a compact `⇅ N` segment
  where N = number of enabled projects needing attention (behind, ahead, or with unexported agents). Accent: a green
  distinct from the existing violet/orange/cyan/gold family — suggest `#5FD787` with dark text, matching the established
  `bold #1a1a1a on <accent>` segment style. Idempotent `set_status(...)` setter; truthful tooltip listing each pending
  project and its reason (e.g. `sase: 2 behind · 1 agent to publish`) plus the `,U` hint; `on_click` triggers the sync
  (via the tracked-task path below).
- Compose in the top bar (`src/sase/ace/tui/app.py` ~lines 436-443, ordered next to `UpdatesAvailableIndicator`), export
  from `src/sase/ace/tui/widgets/__init__.py`, add the `#agents-sync-indicator` CSS rule in
  `src/sase/ace/tui/styles.tcss` mirroring `#updates-indicator`.
- Add a PNG visual snapshot test for the indicator states in the existing suite (`just test-visual`, goldens under
  `tests/ace/tui/visual/snapshots/png/`).

**`,U` comprehensive update leg:**

- Add an agents-sync leg to the comprehensive update flow
  (`src/sase/ace/tui/modals/plugins_browser_comprehensive_update*.py`): after the agent-CLI and SASE legs, run
  `sync_project_agents_repo` for every enabled project inside the same worker; aggregate a third outcome onto
  `ComprehensiveUpdateResult` (`src/sase/ace/comprehensive_update.py`) and the summary line (e.g.
  `Agents repos: 2 synced, 1 pushed, 1 skipped`). Failures are per-project and non-fatal to the update; the leg
  completes before any post-update restart. Preview shows the pending per-project sync state so the confirmation is
  truthful.
- Manual trigger parity: the indicator click and a leader/command entry route through the tracked-task machinery
  (`_submit_tracked_task` in `src/sase/ace/tui/actions/task_actions.py`) so syncs appear in the Task Queue, dedupe, and
  count at quit. If a leader key is added, update the keymap in `src/sase/default_config.yml` and the command
  catalog/help modals consistently.
- After a sync completes, refresh the indicator through the same revalidation scheduling (no new refresh code paths).

**Tests:** timer/worker guard behavior (no overlap, thin callbacks), revalidate-vs-recompute cadence (assert no network
calls on plain ticks), indicator set/clear/tooltip, `,U` leg aggregation + summary rendering, click-to-tracked-task
wiring. Respect the perf memory: verify with `SASE_TUI_PERF=1` locally that j/k p95 stays unaffected.

## End-to-end exercises and docs

Exercise the whole feature the way a user will hit it, and finish the docs.

- Two-machine simulation test (integration-style, tmp `SASE_HOME`s + a shared local bare "GitHub" remote): machine A
  runs `sase config init` (name `alpha`), launches/records an agent that commits, runs `sase repo init` (agents repo
  consented) and `sase agent sync`; machine B (`beta`) syncs and asserts: the imported `alpha.*` agent appears in its
  artifact dirs, chats, name registry, and `sase chats`; names display stripped on A and qualified on B's data; B's own
  commit-associated agent round-trips back to A.
- Negative-path exercises: `disabled: true` project never creates/clones/syncs; `visibility: private` reaches the
  provider; missing `machine_name` produces the actionable error and the doctor/init surfaces; declining the repo-init
  consent leaves everything untouched.
- Docs pass: `docs/agents_sidecar.md` + `docs/configuration.md` sections are complete, accurate, cross-linked; CLI `-h`
  output for `sase config init` and `sase agent sync` meets the CLI rules (sorted, short aliases, colored).
- Final summary must remind the user (do not do it): glossary/memory entries for "Machine Agent Hood" and the agents
  sidecar would be a natural follow-up but require their explicit approval, and existing machines need a one-time
  `sase config init` run.
