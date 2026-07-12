---
create_time: 2026-07-10 17:08:51
status: done
prompt: 202607/prompts/linked_repo_local_workspaces.md
tier: tale
---
# Plan: Retire `linked_repos[].workspace.strategy` — Local Per-Workspace Linked Repo Clones

## Problem

Linked repos currently support a `workspace.strategy` config field with two values:

- `suffix` (default): numbered linked workspaces are materialized under a **shared, project-key-derived root** (e.g.
  `~/.local/state/sase/workspaces/<org>/<linked-repo>/<linked-repo>_<N>`). The path is keyed only by (linked repo,
  workspace number) — it is completely independent of the _host_ project. Two agents working on **different main
  projects** (or the linked repo's own project) that share a workspace number resolve to the **same checkout**, and both
  agent launch (`prepare_linked_repo_workspaces_if_needed`) and `sase workspace open` run `prepare_workspace()` on it,
  wiping the other agent's in-progress changes.
- `none`: the linked repo's primary directory is always used directly. This exists **only** to protect the chezmoi repo
  (shared by multiple main sase projects) from the collision above, and it has grown a large amount of special-case
  machinery: the commit finalizer's "advisory" dirty lane, static-repo branches in memory-init rendering, delta-panel
  exclusions, and revert exclusions.

Goal: **remove the `workspace.strategy` field entirely** and give _all_ linked repos one uniform strategy that cannot
collide across projects: clone linked repo workspaces **locally inside the host workspace directory**, under the
git-ignored `.sase/workspaces/` directory.

## Target Design

### New unified resolution rule (replaces both `suffix` and `none`)

For a host project resolved at `workspace_num`:

- `workspace_num <= 1` (primary checkout): the linked repo resolves to its **primary directory**, exactly as both
  strategies behave today. No clone is created.
- `workspace_num > 1`: the linked repo resolves to `<host_workspace_checkout>/.sase/workspaces/<linked_repo_name>/`,
  where `<host_workspace_checkout>` is the host project's numbered checkout (pure resolution via
  `WorkspaceStore(host_primary).resolve(workspace_num)` — no I/O needed for path computation, so TUI/metadata paths stay
  cheap).

Because the clone lives _inside_ the host workspace, the path is naturally namespaced by (host project, workspace
number, linked repo) and the cross-project collision disappears. The `.sase/` directory is already added to each
workspace's `.git/info/exclude` during runner setup (`src/sase/axe/run_agent_runner_setup.py`), so nested clones stay
invisible to the host repo's git and survive `git clean -fd` during host workspace preparation.

Materialization reuses the existing local-clone helper (`_ensure_git_clone_at` in
`src/sase/workspace_provider/utils.py`): clone from the linked primary (hardlinked objects, cheap), then re-point
`origin` at the primary's real remote URL and fetch — identical semantics to today's suffix clones, just a different
target path. After cloning, run the same `ensure_workspace_sdd_clone` step suffix clones get today so the linked repo's
SDD companion works. Nested clones are **not** recorded in the shared workspace registry and get no checkout marker —
they are owned by (and live/die with) the host checkout, not `sase workspace list/cleanup`.

### Consequences of removing `none`

- chezmoi becomes an ordinary linked repo: agents in numbered workspaces get their own private chezmoi clone via
  `sase workspace open`, dirty chezmoi clones **block** the commit finalizer like any other linked repo (instead of the
  current advisory lane), and chezmoi deltas/reverts participate in ACE like other linked repos. Agent changes reach the
  real chezmoi checkout through the git remote (commit + push from the clone), never by writing to
  `~/.local/share/chezmoi` directly.
- The entire finalizer "advisory static sibling" lane, the memory-init "static linked repo" rendering branch, and the
  `suffix`-only filters in deltas/revert code become dead and are deleted.

## Implementation

### Phase 1 — Core resolution (`src/sase/linked_repos.py`)

- Delete `_VALID_WORKSPACE_STRATEGIES` and all strategy parsing/validation in `_resolve_linked_repos`. A leftover
  `workspace:` mapping (or `workspace.strategy`) in a config entry is **ignored with a non-fatal deprecation warning** —
  the repo still resolves (today an invalid strategy skips the repo; a stale `none` must not silently drop chezmoi from
  runs during the transition).
- Rewrite `_resolve_workspace_dir` to the rule above: primary for `workspace_num <= 1`; otherwise
  `<host_ws>/.sase/workspaces/<name>`. It needs the host primary dir, host workspace checkout resolution, and the repo
  name — all already available in `_resolve_linked_repos`. `materialize=False` stays pure; `materialize=True` clones via
  `_ensure_git_clone_at` (plus SDD ensure), creating parent dirs as needed.
- Drop `workspace_strategy` from `_ResolvedLinkedRepo`, `to_json_dict()`, and `_json_safe_entry` (entry
  dedup/equivalence keys shrink to name+path). The env JSON (`SASE_LINKED_REPOS_JSON` and the sibling alias) stops
  emitting the key; `workspace_num` and `primary_dir` stay.
- Add one small shared helper for readers of **old recorded metadata/env** (see Compatibility):
  `is_legacy_static_linked_repo_record(mapping) -> bool` — true when a record carries `workspace_strategy == "none"`.
- Materialization ordering: launch-time resolution in `src/sase/agent/launch_spawn.py` must not clone into a host
  checkout that may not exist yet. Resolve with `materialize=False` there (env paths are deterministic strings), and let
  the runner materialize after host workspace prep (Phase 2). Verify the other `materialize=True` callers run after the
  host checkout exists (the revert flow already calls `ensure_workspace_checkout` on the host before resolving linked
  repos).

### Phase 2 — Launch/runner flow (`src/sase/axe/`)

- `prepare_linked_repo_workspaces_if_needed` (`run_agent_runner_setup.py`): runs after host `prepare_workspace`. Replace
  the `workspace_strategy == "suffix"` filter with `workspace_dir != primary_dir`, and make it **materialize the nested
  clone first** (it now owns clone creation), then `prepare_workspace` it to the default revision as today.
- `refresh_linked_repos_for_workspace` keeps working unchanged apart from the resolution rule (claim changes happen
  after the new host checkout is materialized).

### Phase 3 — `sase workspace open` / `path` routing (`src/sase/main/workspace_handler_*.py`)

- When the `-p` target is a configured linked repo of the **current context** (env metadata or config — the
  `_is_configured_linked_repo` / `_configured_linked_repo` machinery already resolves this), `open`/`path` with
  `workspace_num > 1` must resolve the **host** project context (from cwd/env, as `_current_project_context` does) and
  return `<host_ws_num_checkout>/.sase/workspaces/<name>`, materializing
  - `prepare_workspace`-ing it on `open`. The CLI shape and agent instructions
    (`sase workspace open -p <linked_repo> -r "<reason>" <workspace_num>`) are unchanged — the number still identifies
    the host workspace.
- `workspace_num <= 1` keeps returning the linked primary.
- If no host project context can be inferred (human shell outside any project workspace), fail with a clear error
  explaining linked workspaces are now host-scoped — the old shared fallback must not silently return.
- A linked repo that is _also_ its own sase project (e.g. sase-core) only takes this path when it is a linked repo of
  the current context; operating on the project directly (from its own checkouts) is unchanged.
- Opened-marker recording (`record_opened_linked_repo`) is unchanged — it records the nested path.

### Phase 4 — Downstream consumers of `workspace_strategy`

- **Config schema** (`src/sase/config/sase.schema.json`): delete the `workspace` property from the `linkedRepo`
  definition (strategy is its only key). Doctor/config-layer validation will then flag stale usage; runtime parsing
  still warns-and-ignores (Phase 1).
- **TUI models** (`src/sase/ace/tui/models/agent.py`, `_loaders/_meta_enrichment_common.py`): drop
  `LinkedRepoMetadata.workspace_strategy`; `parse_linked_repos` skips legacy-static records (helper from Phase 1) and
  otherwise accepts records without the key.
- **Deltas** (`src/sase/ace/tui/widgets/file_panel/_linked_deltas.py`): remove the suffix-only filter and the
  `_static_none_workspace_names` lane; eligibility is "has a recorded workspace_dir" (legacy-static records already
  filtered at parse time).
- **Revert** (`src/sase/ace/revert_agent_resolution.py`, `revert_agent_workspace.py`): remove the
  `workspace_strategy == "suffix"` filters — all resolved linked repos with a non-primary workspace_dir participate.
  `_resolve_linked_checkouts`'s doc/filter simplifies to the same `workspace_dir != primary_dir` test.
- **Commit finalizer** (`src/sase/llm_provider/commit_finalizer_types.py`, `_state.py`, `_prompting.py`,
  `commit_finalizer.py`): delete the `_WorkspaceStrategy` type, `SiblingTarget.workspace_strategy`, the advisory
  candidate/dirty/prompting lane, and the `workspace.strategy: none` prompt text. Legacy-static records from old agent
  metadata/env are dropped at the target-building boundary (never blocking, never advisory). Dirty linked clones are
  always blocking.
- **Memory init** (`src/sase/main/init_memory/config.py`, `models.py`, `root_rendering.py`): remove
  `LinkedRepoMemoryEntry.workspace_strategy`/`static_path` and the static-repo rendering branch — every linked repo now
  renders with the `sase workspace open` instructions.

### Phase 5 — Config, docs, generated memory

- **chezmoi config** (`~/.local/share/chezmoi/home/dot_config/sase/sase.yml`, committed in the chezmoi repo via its main
  directory): delete the `workspace: { strategy: none }` block from the chezmoi linked repo and rewrite its
  `description` — the "You should NOT use `sase workspace open` for this linked repo" guidance inverts to normal
  linked-repo behavior.
- **Docs**: update `README.md` (linked-repo section) and `docs/init.md`, `docs/commit_workflows.md`, `docs/llms.md`,
  `docs/ace.md`, `docs/configuration.md` — remove `workspace.strategy: none` / advisory / static-repo language and
  describe the `.sase/workspaces/` layout.
- **Generated memory regeneration**: the memory-init renderer change makes `memory/sase.md` (this repo), the provider
  shims, and the chezmoi-home global memory stale, and `sase validate` (run by `just check`) gates on that drift.
  Regenerate via the standard memory-init flow. ⚠️ Memory files must not be modified without explicit user permission
  granted **in the implementing conversation** — the implementing agent must ask before regenerating (this plan does not
  itself constitute permission).

### Phase 6 — Tests

- Update/extend `tests/test_linked_repos.py` and `tests/test_sibling_repos.py`: new path-derivation tests (primary for
  `<=1`; `<host_ws>/.sase/workspaces/<name>` otherwise), legacy `workspace.strategy` config ignored-with-warning (repo
  still resolves), env JSON no longer contains the key.
- Runner setup tests (`tests/test_run_agent_runner_setup.py`, deferred-workspace tests): clone-then-prepare behavior,
  launch-time resolution no longer materializes.
- Workspace handler tests (`tests/main/test_workspace_handler_*.py`): linked `open`/`path` host-scoped routing,
  no-host-context error, primary passthrough.
- Finalizer tests: fold `test_commit_finalizer_siblings_advisory.py` into blocking-behavior tests plus a
  legacy-static-record (dropped, non-blocking) case; update `siblings`/`linked_repos_compat` suites.
- Revert/delta/TUI tests and PNG snapshot fixtures: strategy fields removed from fixtures; legacy-static records
  excluded. `tests/test_core_agent_scan_wire.py` fixtures may keep `workspace_strategy` in one record to pin passthrough
  tolerance of old metadata.
- Memory-init tests (`tests/main/test_init_memory_*.py`): all repos render workspace-open instructions.

## Compatibility & Migration

- **Old recorded metadata / long-running agents**: `agent_meta.json` and env from pre-change runs still carry
  `workspace_strategy`. Records with `"suffix"` keep working (absolute `workspace_dir` paths still exist); records with
  `"none"` point at shared primaries and are **dropped by readers** (never prepared, reverted, delta'd, or
  commit-blocked) via the Phase-1 helper. New records simply omit the key.
- **Rust core boundary**: verified `sase-core` carries `agent_meta.linked_repos` as untyped JSON maps
  (`Vec<Map<String, Value>>` passthrough) and its config layer treats `linked_repos` opaquely — **no sase-core wire/API
  change is required**. Parity fixtures there keep passing as passthrough data.
- **Existing shared suffix checkouts** (e.g. `<state-root>/<org>/<linked>/<linked>_<N>`): simply stop being used by
  linked flows; where the linked repo is also a project, they remain that project's own workspaces. No
  migration/deletion step — existing registry TTL cleanup handles staleness.
- **Config rollout**: code tolerates (warns on) the stale `workspace:` block, so the sase change and the chezmoi config
  edit do not need to land atomically.

## Non-Goals

- No change to how the _host_ project's numbered workspaces are resolved (WorkspaceStore policies, registry, cleanup,
  migration commands).
- No linked-repos-of-linked-repos recursion inside nested clones.
- No removal of the deprecated `sibling_repos` alias or the sibling env-var compatibility layer — only the
  `workspace_strategy` key disappears from the payloads both layers emit.

## Verification

- `just check` (note: memory regeneration in Phase 5 is required for the `sase validate` freshness gate to pass after
  the renderer change).
- Targeted suites: linked repos, workspace handler, runner setup, finalizer, revert, linked-delta/TUI (+ PNG snapshots
  if fixture rendering changed).
- Manual smoke: from a numbered workspace of one project, `sase workspace open -p <linked> -r test <N>` → path is
  `<host_ws>/.sase/workspaces/<linked>`; repeat from a second project at the same `<N>` and confirm the first clone's
  dirty state is untouched; confirm `git status` in the host workspace shows no `.sase/workspaces/` noise.
