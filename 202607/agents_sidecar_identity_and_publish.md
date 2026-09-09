---
tier: epic
title: Per-user machine identities and full-family agents-sidecar publishing
goal: "Evolve the hidden `<project>--agents` sidecar so that (1) machine agent hoods are
  stripped from local disk and applied only at the sidecar/commit boundary as part of a
  new three-level `<username>.<machine_name>.<local>` identity, (2) a single commit
  publishes the committing agent's entire top-level hood and family with enough data to
  revive whole families on a remote machine, plus a beautiful per-family overview
  markdown, (3) `SASE_MACHINE` is removed and `SASE_AGENT` carries the full identity as
  a hyperlink to that overview, (4) `sase commit` auto-pushes the new agent data, and
  (5) the TUI indicator lights only on incoming changes, `,U` integrates only
  already-detected changes without fetching, and a new Updates-tab `a` keymap performs
  the heavy network sync on demand.

  "
phases:
  - id: config-identity
    title: Per-user + per-machine identity config
    depends_on: []
    size: medium
    description: "Group machine identity under `id.machine_name` (renamed from top-level
      `machine_name`) and add a required `id.username`; teach the overlay classifier,
      accessors, and `sase config init` about both, with legacy fallback and migration.

      "
  - id: core-identity-helpers
    title: Rust core three-level identity helpers + facade
    depends_on:
      - config-identity
    size: medium
    description: "Add `validate_username`, `qualify_global_agent_name`, and
      `strip_to_local_agent_name` to the sibling Rust core and expose them through the
      identity facade, keeping the existing machine-only helpers for legacy read-path
      stripping.

      "
  - id: local-dequalification
    title: Stop persisting the machine hood on local disk
    depends_on:
      - core-identity-helpers
    size: large
    description: "Neutralize on-disk machine-hood qualification at every write choke
      point so this machine's own agents persist bare local names; keep the read-path
      strip (for legacy and foreign names) and registry legacy-equivalence intact.

      "
  - id: sidecar-export-and-overview
    title: Publish whole hoods + families, revival data, and overviews
    depends_on:
      - core-identity-helpers
      - local-dequalification
    size: large
    description: "Expand export from the lone committing agent to its entire top-level
      hood + family, qualify to the global name, carry the extra data needed to revive
      families, strip imports to the correct local level, and generate a beautiful
      per-family overview markdown plus a manifest hole index.

      "
  - id: commit-tags-and-autopush
    title: SASE_AGENT hyperlink, remove SASE_MACHINE, auto-push on commit
    depends_on:
      - sidecar-export-and-overview
    size: large
    description: "Remove the `SASE_MACHINE` tag, make `SASE_AGENT` the full identity
      rendered as a hyperlink to the family overview, and add a `handle_sase_agent`
      commit hook that exports + pushes the committing agent's hood to the agents
      sidecar after each commit.

      "
  - id: tui-indicator-update-keymap
    title: Incoming-only indicator, snapshot-gated ,U, and the Updates `a` keymap
    depends_on:
      - sidecar-export-and-overview
    size: medium
    description: "Light the sync indicator only on incoming changes, gate the `,U`
      agents leg to already-detected changes with no per-project fetch, and add an
      Updates-tab `a` keymap that performs the full network sync of every enabled
      project's agents repo.

      "
  - id: chezmoi-migration-and-e2e
    title: Config migration, two-machine e2e, and docs
    depends_on:
      - local-dequalification
      - sidecar-export-and-overview
      - commit-tags-and-autopush
      - tui-indicator-update-keymap
    size: medium
    description: "Migrate the athena chezmoi overlay to the new `id` shape, exercise the
      full flow across two simulated machines (including remote family revival), and
      finish the docs.

      "
create_time: 2026-09-09 19:52:59
status: wip
---

- **PROMPT:**
  [prompts/202607/agents_sidecar_identity_and_publish.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/agents_sidecar_identity_and_publish.md)

# Plan: Per-user machine identities and full-family agents-sidecar publishing

## Overview and design principles

The hidden `<project>--agents` sidecar (epic `sase-8k`) already exists: it stores
portable bundles for completed, commit-associated agents and syncs them between
machines. This epic corrects and substantially extends it based on five decisions the
design owner made after the initial landing. The through-line is a clean separation
between the **local representation** of an agent (what lives in `~/.sase` on one
machine) and its **global representation** (what is published to the sidecar, embedded
in commit footers, and linked from GitHub).

### The identity model (the spine of this epic)

Every machine has an identity of two parts, both config-driven:

- `id.username` — globally unique across all SASE users, the same across all of one
  user's machines (recommend the user's GitHub username). This is net-new.
- `id.machine_name` — this machine's hood token (renamed from the current top-level
  `machine_name`).

From these we define three names for the same agent:

| Name                      | Form                                                                     | Where it lives                                                                            |
| ------------------------- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| **Global name**           | `<username>.<machine>.<local>` e.g. `bbugyi200.athena.foo.bar.baz--code` | agents sidecar bundle dirs, `manifest.json`, `SASE_AGENT` footer, GitHub overview links   |
| **Local name (owner)**    | bare `<local>` e.g. `foo.bar.baz--code`                                  | this machine's own agents on disk (registry, `agent_meta.json`, chats, `SASE_AGENT_NAME`) |
| **Local name (imported)** | the global name with only the _matching_ identity prefix stripped        | foreign agents materialized on this machine                                               |

Import stripping is symmetric with the identity: strip the **longest** leading identity
prefix that matches this machine.

- Same user **and** same machine (our own bundle round-tripping): strip
  `<username>.<machine>.` → bare `foo`.
- Same user, different machine: strip `<username>.` → `<machine>.foo` (e.g. `zeus.foo`).
- Different user: strip nothing → `<username>.<machine>.foo` (e.g. `alice.zeus.foo`).

This is exactly what the design owner asked for ("remove the `<username>.` prefix … from
any machine configured to use that same `id.username`… strip the entire
`<username>.<machine_name>` when both … match"). It also makes the TUI's existing
dotted-prefix hood grouping do the right thing for free: our own agents show bare (no
identity hood), a peer machine's agents group under a `<machine>` hood, and another
user's agents group under a `<username>` → `<username>.<machine>` hood — **the machine
and username hoods only ever materialize for foreign agents on disk, and in the sidecar;
never for our own agents on disk.** That is the correction requested in bullet 1.

### Design principles

1. **The machine/username hoods exist only at the boundary.** Locally, our own agents
   are bare. Qualification to the global name happens in exactly two writers: the
   sidecar export path and the commit-footer builder. This reverses the current behavior
   where `qualify_local_agent_name` runs at ~20 on-disk write choke points.
2. **A commit publishes a constellation, not a single agent.** When any agent in a
   top-level hood commits, the whole hood (recursively) and the committer's family are
   published, so the sidecar always holds coherent, revivable groups — never an orphaned
   member.
3. **Publishing is an explicit, consented action, and now includes `sase commit`.** The
   repo-init consent already warns that all commit-associated agent data is published;
   auto-pushing on `sase commit` is the natural fulfillment of that consent, mirroring
   how `handle_sase_plan` already commits and pushes the plans sidecar during a commit.
   No _silent_ background pushes are added — pushes ride explicit user actions
   (`sase commit`, `sase agent sync`, the `a` keymap).
4. **Reads stay cheap and stay put.** Background checks only detect incoming work and
   light an indicator; integration and fetching happen on explicit actions. `,U` never
   fetches per project; it integrates what a prior periodic check already detected. The
   heavier network sync is opt-in via a dedicated `a` keymap.
5. **Beautiful, truthful surfaces.** Each published family gets a hand-designed overview
   markdown; the `SASE_AGENT` footer links straight to it; the indicator tooltip and
   `,U`/`a` summaries state exactly what happened per project.
6. **Reuse existing shapes.** Mirror `handle_sase_plan` for the `SASE_AGENT` hyperlink
   and commit-time push; mirror the snapshot-gated `,U` provider pattern for the agents
   leg; extend the existing `agents_sync` bundle/manifest/status engine rather than
   replacing it; keep the identity math in the Rust core per the backend-boundary rule.

### Key pre-explored facts (verified in this repo and the linked sase-core checkout)

- **Config (all in `src/sase/config/core.py`):** the raw merged key `"machine_name"` is
  read in exactly four places — `get_machine_name` (~262), `discover_machine_names`
  (~398), and the overlay classifier `_get_selected_overlay_paths` (~416 presence check,
  ~419 equality check). Selector file `~/.sase/machine_name` via `machine_name_path()`
  (`src/sase/core/paths.py` ~81); its stat is already in `_compute_current_config_token`
  (~132). Schema property at `src/sase/config/sase.schema.json` ~347-351 with root
  `additionalProperties: false` and **no** top-level `required` array. `_deep_merge`
  (~306) already merges nested dicts, so an `id:` mapping merges cleanly across
  overlays.
- **`sase config init`:** `src/sase/main/config_init_handler.py` — `plan_config_init`
  (~38), `_prompt_machine_name` (~74), `_write_new_machine_overlay` (~160, uses
  `set_key(text, ("machine_name",), value)` at ~166), `_write_machine_selector` (~172),
  `run_config_init` (~196). `set_key` (`src/sase/config/_edit_yaml.py` ~52) already
  accepts a nested tuple key path. Parser: `register_config_parser`
  (`src/sase/main/parser_commands.py` ~29, `init` child ~40); `sase init config` alias
  in `src/sase/main/parser_init.py` ~124; dispatch `src/sase/main/config_handler.py` ~11
  and `src/sase/main/entry.py` ~196/212; onboarding `src/sase/main/init_registry.py`
  ~22.
- **Identity facade:** `src/sase/core/machine_hood_facade.py` — `MachineHoodIdentity`
  (machine_name + known_machines), `qualify_local_agent_name` (~68),
  `strip_local_agent_name` (~88), `canonical_local_agent_name_key` (~116),
  `local_agent_name_lookup_candidates` (~124). Rust primitives `validate_machine_name`,
  `qualify_machine_agent_name`, `strip_machine_agent_name`, `machine_hood_of` live in
  the sibling Rust core (open with `/sase_repo`: `sase repo open sase-core`) at
  `crates/sase_core/src/machine_hood.rs`, bound in `crates/sase_core_py/src/lib.rs`.
- **On-disk write choke points (all call `qualify_local_agent_name`):** launch identity
  `src/sase/axe/run_agent_directive_identity.py` (~185, then `SASE_AGENT_NAME` at ~301
  and registry claims at ~287/296); meta `src/sase/axe/run_agent_directive_metadata.py`
  `_qualify_agent_identity_metadata` (~238-257) written by
  `src/sase/axe/runner_artifacts.py` (~46/80); directives
  `src/sase/axe/run_agent_directives.py` (~222); registry mutations
  `src/sase/agent/names/_registry_mutations.py` (~61 and
  138/172/220/259/260/309/354/431); registry rebuild
  `src/sase/agent/names/_registry_scan.py` (~180); template alloc
  `src/sase/agent/names/_templates.py` (~271); multi-prompt allocators
  `src/sase/agent/multi_prompt_launch_plan.py` (~241/243/330),
  `src/sase/agent/multi_prompt_reference_allocation.py` (~54/58),
  `.../multi_prompt_reference_allocator.py` (~72/210/252); clan/family
  `src/sase/agent/clan_membership.py` (~136), `src/sase/agent/_family_promotion.py`
  (~87), `src/sase/agent/_family_attach_resolution.py` (~87/168); TUI writes
  `src/sase/ace/tui/actions/rename.py` (~317), `.../agents/_revive_artifacts.py`
  (~363/393/402), `.../agents/_directive_persistence.py` (~191).
  `src/sase/plan_chain.py` does **not** qualify (composes `<base>--<suffix>` from an
  already-resolved base).
- **Read path (unchanged by this epic):** `Agent.refresh_presented_agent_name`
  (`src/sase/ace/tui/models/agent.py` ~142) strips only the local machine hood; foreign
  hoods pass through and group naturally via `agent_name_key`
  (`src/sase/ace/tui/models/agent_hoods.py` ~20). Lookups accept bare/legacy via
  `local_agent_name_lookup_candidates`.
- **Sync engine (`src/sase/agents_sync/`):** export enumeration `bundles.py`
  `_build_local_bundles` (~146); the decisive export gate is `if not commits: continue`
  at ~216; export already re-qualifies the raw meta name at ~178/427; import
  `integrate_foreign_bundles` (~114) keys foreign vs local purely on
  `entry.machine == machine` (~123) and stores imported names verbatim via
  `claim_imported_registered_name`; `count_unexported_local_agents` (~239, markers
  only); bundle IO + `PORTABLE_META_FIELDS` allowlist +
  `validate_qualified_name(value, machine)` in `io.py` (~31/153); on-disk layout
  `agents/<qualified-name>/{meta.json,chat.md,commits.json}` + top-level
  `manifest.json`; git transaction `git_sync.py` `sync_agents(projects=...)` (~32,
  already project-filterable); status snapshot + revalidate/recompute `status.py`
  (~33/130). Historical backfill
  `_historical_commit_associations`/`_historical_local_agent_name` (`bundles.py`
  ~363-427) consumes `SASE_AGENT`/`SASE_MACHINE` footers. Targets `targets.py` (~104
  agents-sidecar record; remote at ~169). Seed scaffold `src/sase/sdd/_init_files.py`
  (~62 `expected_sdd_sidecar_files` agents branch) and template
  `src/sase/sdd/templates/sidecar-agents-README.md`.
- **Commit footers:** `src/sase/workflows/commit/runtime_tags.py` —
  `RUNTIME_COMMIT_TAG_KEYS = {"AGENT","MACHINE"}` (~18); `_resolve_agent_name` already
  qualifies (~57/74); `_resolve_machine_name` (~80) is the only `SASE_MACHINE` producer;
  `apply_auto_commit_tags_with_runtime` (~115) is a second producer of both. Applied at
  `src/sase/workflows/commit/workflow.py` (~170-176). Hyperlink precedent:
  `handle_sase_plan` (`src/sase/workflows/commit/commit_hooks.py` ~281, called from
  `workflow.py` ~116) builds `LinkedCommitTagValue(ref, target)` where
  `target = format_sase_plan_link(ref, store=store)`
  (`src/sase/workflows/commit/plan_paths.py` ~85 → `github_blob_url` in
  `src/sase/_git_remote.py` ~63) and then commits + pushes the plans sidecar (~430).
  Footer rendering is Rust (`crates/sase_core/src/commit_footer.rs`); tests there
  reference `SASE_MACHINE` (~487/526-531). Parse facade
  `src/sase/core/commit_footer_facade.py`.
- **Family/hood enumeration:** `agent_family_base(name)` and `_split_agent_family_name`
  (`src/sase/plan_chain.py` ~307/327; separator `--` at ~9);
  `_agent_hood_chain(name)[0]` = top-level hood, `is_agent_descendant(name, ancestor)`
  matches `ancestor.`/`ancestor--`, `_descendant_ranges`
  (`src/sase/ace/tui/models/agent_hoods.py` ~33/359/381); disk-based family scan
  `_iter_family_members`/`find_agent_family` (`src/sase/agent/names/_lookup_groups.py`
  ~121/308).
- **Revive (`R`):** `default_config.yml` `start_rewind: "R"` (~219) →
  `src/sase/ace/tui/actions/agents/_revive_flow.py` (~36) →
  `SavedAgentGroupRevivalModal`. Revive rebuilds on-disk markers so `load_all_agents()`
  rediscovers a family; the authoritative reconstructable field sets are
  `_build_agent_meta_data` and `_build_done_json_data`
  (`src/sase/ace/tui/actions/agents/_revive_artifacts.py` ~377/326). Faithful _relaunch_
  of exact family members (commit `330c25856`) additionally needs `raw_xprompt.md`,
  `model`/`llm_provider`/`role_suffix`/`agent_family`/`phase_bead_id`, and
  `embedded_workflows.json` (`src/sase/ace/tui/models/artifact_files.py` ~166).
- **TUI:** indicator `src/sase/ace/tui/widgets/agents_sync_indicator.py` (glyph `⇅`,
  accent `#5FD787`); the shared "needs attention" predicate
  `agents_sync_status_needs_attention` (`src/sase/ace/tui/agents_sync_format.py` ~10)
  currently lights on behind/ahead/unexported and any non-ready state, and is also
  consumed by the `,U` preview; periodic mixin `src/sase/ace/tui/actions/agents_sync.py`
  (revalidate every 10 min, recompute/fetch every 30 min); config `default_config.yml`
  ~105-108 `ace.agents_sync`. `,U` preview is already network-free
  (`plugins_browser_comprehensive_update.py` ~131
  `get_agents_sync_status(revalidate_only=True)`); the execution leg
  `_execute_agents_leg` (`plugins_browser_comprehensive_update_execution.py` ~236) calls
  `sync_agents()` with **no** project filter (this is the per-project fetch to gate);
  `agents_runnable` (`plugins_browser_comprehensive_update_models.py` ~66) treats every
  enabled project as runnable. Admin Center = `ConfigCenterModal` ("SASE Admin Center");
  the Updates tab pane is `PluginsBrowserPane` whose single-key bindings are **hardcoded
  in `BINDINGS`** (`src/sase/ace/tui/modals/plugins_browser_pane.py` ~199-222), not in
  `default_config.yml`. Tracked-task helper `_submit_tracked_task`
  (`src/sase/ace/tui/actions/task_actions.py` ~122); existing indicator-click sync
  `action_sync_agents` (`src/sase/ace/tui/actions/agents_sync.py` ~176,
  `dedup_key="agents-sync"`, `exclusive_scopes=("agents-sync",)`).

### Out of scope (do not do in any phase)

- No mass rename/migration of existing on-disk agent names. Names that the _current_
  implementation already qualified as `<machine>.foo` remain valid; they resolve and
  display via the unchanged read-path strip and registry legacy-equivalence. New own
  agents are written bare.
- No editing of `sase/memory/*.md`, `AGENTS.md`, or provider instruction shims. New
  glossary terms ("Username Agent Hood", the identity model) require the design owner's
  explicit permission in a live conversation; the final phase only _reminds_.
- No change to how plans/research sidecars work, to non-agents commit tags (`TYPE`,
  `PLAN`), or to the Rust footer grammar itself (only the removal of one produced tag
  and its tests).
- No reconciliation of the sase_gateway `$HOSTNAME` host label with `id.machine_name`.
- No import of _running_ foreign agents; only terminal agents are exported/imported.

---

## Per-user + per-machine identity config

Establish the two-part identity that everything else builds on. Backwards-compatible:
unconfigured or legacy setups keep working; `sase config init` is the opt-in/migration
gate.

**Schema (`src/sase/config/sase.schema.json`):**

- Add an `id` object property with its own `additionalProperties: false` and two
  sub-properties:
  - `machine_name`: string, pattern `^[a-z_]+$`, description "This machine's agent-hood
    token; only applied to agent names when publishing to the agents sidecar or writing
    commit footers."
  - `username`: string, GitHub-compatible pattern (lowercase; alphanumeric with single
    internal hyphens; no dots and no `--` so it is a safe hood segment) — suggested
    `^[a-z0-9](?:-?[a-z0-9])*$`, max length 39; description must state it MUST be unique
    across all SASE users and SHOULD be identical across this user's machines (recommend
    the GitHub username).
- Keep the legacy top-level `machine_name` property in the schema (still `^[a-z_]+$`)
  marked deprecated in its description, so existing overlays validate until migrated.
  (Root stays `additionalProperties: false`.)

**Accessors + overlay classification (`src/sase/config/core.py`):**

- Read the nested key with a legacy fallback: `get_machine_name()` resolves
  `id.machine_name` and falls back to the legacy top-level `machine_name`; add
  `get_username()`/`require_username()` mirroring
  `get_machine_name`/`require_machine_name` (username has no selector file — it rides
  whichever machine overlay the selector picks).
- `discover_machine_names()` reads `data["id"]["machine_name"]` with legacy
  `data["machine_name"]` fallback.
- The overlay classifier `_get_selected_overlay_paths` treats an overlay as
  machine-specific when it declares `id.machine_name` **or** legacy `machine_name`; a
  machine overlay participates only when that value equals the selector. (An overlay
  carrying only `id.username` and no machine name must not be misclassified — guard the
  presence check on the machine token specifically.)
- Config token: the selector stat already participates; no new stat needed (username
  lives inside overlays already stat'd).
- Add `require_identity()` returning `(username, machine_name)` and raising the
  actionable "run `sase config init`" error when either is missing — the single
  hard-require used by export/footer paths.

**`sase config init` (`src/sase/main/config_init_handler.py`):**

- Add `_prompt_username` mirroring `_prompt_machine_name` (TTY-gated,
  `args._init_input_func` injectable, re-prompt on invalid input against the username
  pattern). Its prompt text must make the uniqueness contract unmissable, e.g.: "Your
  SASE username must be globally unique across all SASE users and should be the same on
  every machine you own. We recommend your GitHub username." Offer no silent default
  (unlike machine name); require an explicit value.
- Extend `run_config_init` to prompt for username after machine name, and write **both**
  under `id` into the same machine overlay file:
  `set_key(text, ("id","machine_name"), machine)` and
  `set_key(text, ("id","username"), username)`. When an overlay (or the base config)
  already carries a legacy top-level `machine_name`, migrate it: write `id.machine_name`
  and remove the legacy key in the same splice (via `set_key`/a delete helper), so the
  file ends with the new shape only. Keep the chezmoi remap + deploy path unchanged.
- "Already configured" / `--check` / `plan_config_init` must require **both**
  `id.machine_name` and `id.username` (and treat a legacy-only `machine_name` as "needs
  migration", surfaced as an actionable diagnostic, not silently accepted).
- The selector file continues to hold only the machine name and is unchanged.

**`sase init` and doctor:** the existing `config` `InitCommandSpec` and doctor
`config.init` check pick up the stricter "configured" definition automatically; verify
the doctor message names both fields and the migration case.

**Docs:** update the overlay/identity section of `docs/configuration.md` for the `id`
grouping, `id.username`, the uniqueness contract, and legacy migration.

**Tests:** nested-key resolution + legacy fallback; overlay classification with
`id.machine_name`, legacy `machine_name`, and username-only overlays;
`get_username`/`require_username`/`require_identity`; `sase config init`
prompt/create/migrate/existing-choice/validation paths (via `_init_input_func`); chezmoi
write remap (mocked deploy); `--check`/plan/doctor "needs migration" surfacing. Follow
`tests/test_config_inventory.py` conventions.

## Rust core three-level identity helpers + facade

Identity canonicalization is shared backend domain logic (any frontend that renders or
links agents must reproduce it), so the primitives live in the Rust core per the
backend-boundary rule. Open the sibling repo with `/sase_repo`
(`sase repo open sase-core -r "..."`); this lands as a separate sase-core PR that must
merge and its `sase_core_rs` binding be rebuilt before the dependent phases run.

**Rust (`crates/sase_core/src/machine_hood.rs`, or a sibling `identity.rs`):**

- `validate_username(name) -> Result<...>`: enforce the GitHub-compatible pattern above
  (never contains `.` or `--`).
- `qualify_global_agent_name(name, username, machine) -> String`: idempotent and
  legacy-safe. If `name` already starts with `<username>.<machine>.`, return it
  unchanged. Otherwise strip a leading `<machine>.` (legacy machine-qualified) and a
  leading `<username>.` if present, then prepend `<username>.<machine>.`. Preserve any
  leading `NNNNNN.` dismissed prefix outside the identity prefix (mirror the facade's
  existing dismissed-prefix handling).
- `strip_to_local_agent_name(name, username, machine) -> String`: strip
  `<username>.<machine>.` when both match; else strip a leading `<username>.` when only
  the username matches; else return unchanged. Never strip to empty; never touch
  mid-name segments; preserve a dismissed prefix.
- Keep `qualify_machine_agent_name`/`strip_machine_agent_name`/`machine_hood_of` as-is
  for legacy read-path stripping.
- Inline `#[cfg(test)]` coverage: idempotence, family `--` names, all three strip
  levels, legacy machine-only input, already-global input, dismissed prefixes, username
  validation rejects dots/`--`/uppercase/leading-hyphen. No wire struct changes → no
  schema-version bump. Add a binding smoke test in `crates/sase_core_py/src/lib.rs`
  tests.

**Facade (`src/sase/core/machine_hood_facade.py`):**

- Extend `MachineHoodIdentity` into an identity snapshot that also carries `username`
  (source from `get_username()` in `.current()`); keep `known_machines` for the existing
  machine helpers. Add `AgentIdentity` as the clearer public alias if desired, but keep
  the old name importable to limit churn.
- Add `qualify_global_agent_name(name, identity=None)` and
  `strip_to_local_agent_name(name, identity=None)` wrapping the new Rust helpers; when
  `username`/`machine_name` are unconfigured, both are strict no-ops (today's behavior).
- Keep
  `qualify_local_agent_name`/`strip_local_agent_name`/`canonical_local_agent_name_key`/`local_agent_name_lookup_candidates`
  for the read path and legacy equivalence.
- Gate facade tests on binding availability the same way the other core facades do, in
  case the rebuilt binding is not yet published when the Python side is authored.

## Stop persisting the machine hood on local disk

Reverse the current behavior so this machine's **own** agents are written to disk with
bare local names. This is broad but mechanical: every write choke point already funnels
through `qualify_local_agent_name`, and the read path already tolerates bare names (it
strips only the local hood, so bare names pass through unchanged while legacy
`<machine>.` names and foreign hoods keep working).

**Write-path changes:** at each choke point listed in "Key pre-explored facts", stop
qualifying the durable name — write the bare local name instead. Concretely, neutralize
the `qualify_local_agent_name` call (or replace with identity resolution that is a no-op
for local names) at: launch identity `run_agent_directive_identity.py` (~185, so
`SASE_AGENT_NAME`, the registry claim, and meta all receive the bare name); meta
qualification `run_agent_directive_metadata.py` `_qualify_agent_identity_metadata`
(~238-257) — the simplest correct move is to make this a no-op for local identities
while still leaving already-foreign names untouched; directives
`run_agent_directives.py` (~222); registry mutations `_registry_mutations.py` (all
local-claim sites) and rebuild `_registry_scan.py` (~180); template/multi-prompt
allocators; clan/family composition (`clan_membership.py`, `_family_promotion.py`,
`_family_attach_resolution.py`); and the TUI rename/revive/directive-persistence
writers.

**Keep intact:**

- `claim_imported_registered_name` (foreign names still stored verbatim — Phase
  `sidecar-export-and-overview` refines the _level_ they are stripped to at import
  time).
- The read path (`refresh_presented_agent_name`, `agent_name_key`, chat display) —
  unchanged; bare names render bare, legacy `<machine>.` own names still strip, foreign
  hoods still show.
- Registry legacy-equivalence:
  `canonical_local_agent_name_key`/`local_agent_name_lookup_candidates`/`candidate_available`
  must keep treating bare `foo` and legacy `<machine>.foo` as the same agent so lookups
  and collision checks span both spellings during the transition.

**Foreign-hood launch policy:** with local names bare, a user typing a name that happens
to start with a peer machine's token no longer needs a special "foreign hood" launch
rejection — uniqueness is enforced by the registry against any imported reservation.
Simplify `launch_validation.py` accordingly (compare on the canonical key; reject only
true registry collisions), and document this as an intentional behavior change.

**Commit footers (interim):** in this phase, leave `runtime_tags.py` resolving
`SASE_AGENT` via the existing helper; the switch to the full global name + hyperlink is
owned by Phase `commit-tags-and-autopush`. (After this phase, the on-disk meta name is
bare, which the global-qualify helper handles.)

**Tests:** new own agents persist bare names in registry/meta/env/chats; legacy
`<machine>.` on-disk names still resolve, display, and collide-equivalently with their
bare spelling; clan/family composition produces bare roots; a foreign (`zeus.*`) fixture
is untouched by write paths and still rejected from local launch only when it collides
with a real reservation. Follow the naming/registry test suites under `tests/agent/` and
the TUI loader tests under `tests/ace/tui/`.

## Publish whole hoods + families, revival data, and overviews

Turn the sidecar from "one committing agent" into "the committer's entire top-level
hood + family, revivable, with a beautiful overview". All work is centered in
`src/sase/agents_sync/` plus a new overview generator.

**Global names in the sidecar:** the bundle/manifest/footer scheme becomes the
**global** name `<username>.<machine>.<local>`.

- Export qualification: replace the `qualify_local_agent_name` at `bundles.py` ~178/427
  with `qualify_global_agent_name` (username+machine, legacy-safe). Bundle directory
  names, `manifest.json` keys, and `ManifestEntry.name` all use the global name. Extend
  `validate_qualified_name` in `io.py` (~153) to require the `<username>.<machine>.`
  prefix and store the source `(username, machine)` for validation. Carry both
  `username` and `machine` on `ManifestEntry`/`PortableAgentMetadata`.

**Expanded export enumeration (the core change, `bundles.py` `_build_local_bundles`
~146-236):**

- First collect the set of **committing** names as today (per-artifact commit markers +
  git-log footer backfill).
- Then **expand**: for each committing name `A`, add (a) all members of `A`'s family
  (`agent_family_base(A)` then the disk family scan, or a name-prefix match on
  `<base>--`), and (b) every local agent in `A`'s **top-level hood** recursively —
  `top = _agent_hood_chain(family_base(A))[0]` (or the family base itself when there is
  no dot), then every local completed-agent name matching `name == top` or
  `name.startswith((f"{top}.", f"{top}--"))`. Union across all committers.
- Export every artifact in the union that is completed (`done.json` outcome `completed`,
  has `agent_meta.json`, not imported), **regardless of whether that individual member
  committed** — i.e. remove the `if not commits: continue` gate for members of the
  expanded set (members with no commits get an empty `commits.json`, which the
  wire/import layer already accepts). Add a name-based top-level-hood enumerator helper
  (thin; built on the `agent_hoods` descendant semantics but operating on
  `iter_agent_artifact_dirs` metadata, not `Agent` rows).
- Update `count_unexported_local_agents` (`bundles.py` ~239) to compute against the
  **same expanded set** so `sase agent sync --check` and the status snapshot stay
  truthful.

**Data needed to revive whole families on a remote (bullet 2):**

- Extend `PORTABLE_META_FIELDS` (`io.py` ~31) with the family-structure and revival
  fields currently missing: `parent_agent_name` (stable, machine-independent — add it to
  the portable projection derived from `parent_timestamp` during export), and the
  revival-relevant subset of `_build_agent_meta_data` not already present (e.g.
  `plan_path` handling, `epic_plan_ref`, `retry_*` as appropriate). Keep the projection
  strictly free of machine-local values (pids, absolute paths, workspace dirs).
- Add a `prompt.md` file to each bundle (the agent's `raw_xprompt.md` when present) so a
  synced remote can not only _revive_ (rediscover) but faithfully _relaunch_ exact
  family members via the existing `R`/kill-edit path. Include `embedded_workflows.json`
  bytes when present (as a bundle sibling) for rollover-ref fidelity. Guard each as
  optional (older/absent artifacts export without them).
- Import reconstruction (`integrate_foreign_bundles`/`_imported_markers`, `bundles.py`
  ~114/516): after materializing a family's members, build a
  `parent_agent_name -> newly-assigned local artifact timestamp` map for that family and
  rewrite each member's reconstructed `parent_timestamp` to point at the freshly
  imported parent, so the family tree is internally consistent on the importing machine
  and `load_all_agents()` groups/revives it correctly. Materialize
  `raw_xprompt.md`/`embedded_workflows.json` into the imported artifact dir when the
  bundle carries them.

**Import stripping to the correct local level (bullets 1 + 4):** replace the verbatim
import naming with `strip_to_local_agent_name(entry.name, identity)`:

- own bundle round-tripping (username+machine match) → bare local name; treat as
  already-owned and skip re-materializing our own agents (local disk is authoritative —
  mirror the current `entry.machine == machine` skip, but on the full identity so a peer
  machine sharing our username is _not_ skipped).
- same-user, different machine → store `<machine>.<local>`.
- different user → store `<username>.<machine>.<local>`.
- `claim_imported_registered_name` claims the stripped on-disk name; provenance keys
  `imported_from_machine`/`imported_from_username`/`imported_digest` record the source.

**Beautiful per-family overview markdown (bullet 2, "beautiful agent family
overview"):**

- New generator (e.g. `src/sase/agents_sync/overview.py`) producing, per **agent hole**
  (a family, or a solo agent) in the exported set, a deterministic markdown at
  `overviews/<global-hole-name>.md` in the sidecar. Design the document to be genuinely
  readable on GitHub: a title with the hole's global name and tribe/clan; a summary line
  (member count, roles present, overall status, models/providers, first/last activity);
  a **members table** (role, presented name, status, model, provider, associated commits
  with short SHAs, chat link to `agents/<global-name>/chat.md`); the plan/goal and
  top-level prompt; and a "Revive" note explaining that `sase agent sync` on another
  machine plus `R` on the Agents tab reconstructs this family. Each member gets a stable
  heading/anchor (e.g. `## <role_suffix or name>`) so the `SASE_AGENT` footer can
  deep-link to the exact member.
- Also generate a top-level-hood index `overviews/<top-hood>/README.md` linking every
  hole overview in that hood (the "constellation" view), and record hole→overview-path +
  member list in `manifest.json` under a new `holes` section so the index and links stay
  consistent and importable.
- Generation is deterministic (no timestamps beyond recorded artifact times; stable
  ordering) so re-export is a no-op when nothing changed — keeping the digest/dirty
  checks meaningful.

**Seed/README + privacy copy:** update `src/sase/sdd/templates/sidecar-agents-README.md`
and the repo-init consent copy to state that publishing now includes the committing
agent's **entire top-level hood and family** (not just the single committer), that names
are globally qualified as `<username>.<machine>.<local>`, and that `overviews/` holds
human-readable family pages. Keep the visibility/`disabled` controls prominent.

**Tests (`tests/agents_sync/`):** expanded-set enumeration from a single committer
across a multi-family hood (matching the design owner's `foo.bar.baz--code` example);
commit-less members export with empty `commits.json`; global-name round-trip; the three
import strip levels (own skipped, same-user machine-qualified, foreign fully-qualified)
via injected identity fixtures; family reconstruction — export a family, wipe, import on
a second identity, assert artifact dirs + chats + `raw_xprompt.md` + parent linkage
reproduce a revivable/relaunchable family; overview markdown golden(s) and the manifest
`holes` index; `count_unexported` parity with the expanded export set. Use tmp
`SASE_HOME` + local bare git remotes.

## SASE_AGENT hyperlink, remove SASE_MACHINE, auto-push on commit

Make commit provenance carry the full identity as a live link, drop the redundant
machine tag, and publish on commit.

**Remove `SASE_MACHINE`:** drop `"MACHINE"` from `RUNTIME_COMMIT_TAG_KEYS`
(`runtime_tags.py` ~18); remove the `MACHINE` producers in
`_resolve_runtime_commit_tags` (~46) and `apply_auto_commit_tags_with_runtime` (~131)
and delete `_resolve_machine_name`. Keep parse tolerance: legacy commits carrying
`SASE_MACHINE` must still parse (the tag is simply ignored). Rewrite the sidecar
backfill `_historical_commit_associations`/`_historical_local_agent_name` (`bundles.py`
~363-427) to derive locality from the now-fully-qualified `SASE_AGENT` value alone (no
`MACHINE` cross-check); legacy unqualified/machine-only footer names are treated as
local via the qualify helper. Update the Rust footer tests that reference `SASE_MACHINE`
(`commit_footer.rs` ~487/526-531) to the new expectation.

**`SASE_AGENT` = full identity + hyperlink (mirror `handle_sase_plan`):**

- Resolve the value with `qualify_global_agent_name` so `SASE_AGENT` is
  `<username>.<machine>.<local>`.
- Add a `handle_sase_agent(payload, cwd)` hook in `commit_hooks.py`, structured like
  `handle_sase_plan`, and call it from the commit workflow (`workflow.py`, alongside the
  existing plan hook at ~116 and before the create-commit at ~176):
  1. Resolve the current agent's global name and the current project's **agents
     sidecar** target (remote URL + default branch + machine-level clone) via the
     `agents_sync` targets/inventory (not an `SddStore`).
  2. Compute the deterministic overview path + anchor for the committer's hole
     (`overviews/<global-hole>.md#<member>`) and build
     `target = github_blob_url(remote_url, provider="github", branch, path)`. Set
     `SASE_AGENT = LinkedCommitTagValue(global_name, target)` when a target resolves,
     else the plain global name (best-effort, exactly like `format_sase_plan_link`).
  3. **After** the commit is created (so the new SHA is captured), export the
     committer's expanded hood + regenerate the overviews (Phase
     `sidecar-export-and-overview`), then commit + push the agents sidecar — the
     auto-push in bullet 5. A disabled/uncreated sidecar, a missing identity, or a
     network/push failure must log and continue without failing or unwinding the code
     commit (the link is best-effort; the next `sase agent sync`/`a` reconciles).
- Because the hyperlink only needs the deterministic _path_ (not a completed push), step
  2 runs before the commit and step 3 after, so a push failure never blocks committing.
- Remove `SASE_AGENT` from the generic `apply_runtime_commit_tags` runtime resolution
  now that the hook owns it for real commits; keep a plain (unlinked) full-name
  `SASE_AGENT` for raw SDD auto-commits (`apply_auto_commit_tags_with_runtime`) where no
  agents sidecar/link applies.

**Tests:** `SASE_AGENT` renders as a Markdown reference link to the correct
`overviews/...` blob URL with the full identity label; no `SASE_MACHINE` is emitted;
legacy footers still parse; backfill classifies locality from the qualified name;
`handle_sase_agent` exports+pushes on commit and degrades gracefully when the sidecar is
disabled/absent or push fails (commit still succeeds); auto-commit path emits an
unlinked full name. Use local bare remotes; assert the pushed sidecar contains the
expanded hood + overview.

## Incoming-only indicator, snapshot-gated ,U, and the Updates `a` keymap

Make the TUI reflect the new publish-on-commit reality: the badge means "others changed
something you care about", `,U` integrates cheaply, and a dedicated key does the heavy
network sync.

**Indicator lights only on incoming changes (bullet 5):**

- Add a new indicator-only predicate (e.g. `agents_sync_status_has_incoming(status)` in
  `agents_sync_format.py`): true only when `state == "ready"` and `behind > 0`. Do
  **not** light on `ahead`/`unexported` (auto-push keeps those ~0) nor on
  `not_created`/`missing_upstream`/`error` (those are configuration, not "incoming from
  other machines"). Use it in `AgentsSyncIndicator.set_status` only; leave the shared
  `agents_sync_status_needs_attention` (used by the `,U` preview) untouched. Keep
  counting **projects** (one per enabled project with incoming changes), the `⇅ N`
  glyph, and the `#5FD787` accent. Tooltip lists each project and its incoming detail
  plus the `a`/`,U` hints.

**Snapshot-gated `,U` agents leg (bullet 5):**

- The preview is already network-free. Gate the execution leg `_execute_agents_leg` to
  sync **only** the projects a prior periodic check already flagged: pass
  `sync_agents(projects=[s.project_key for s in preview.agents_status.projects if s.behind])`
  (the captured `revalidate_only` snapshot). No fresh fetch is added — `sync_agents`
  already accepts a project filter.
- Tighten `agents_runnable` (`plugins_browser_comprehensive_update_models.py` ~66) to
  require at least one project with `behind > 0`, so `,U` is a truthful no-op when
  nothing incoming has been detected; update the preview summary wording to "Integrates
  N enabled agents repositories with already-detected incoming changes (no fetch)".
- Keep the leg per-project non-fatal and folded into the single tracked comprehensive
  task and its aggregate summary.

**New Updates-tab `a` keymap (bullet 5):**

- Add `Binding("a", "sync_agents_repos", "Sync agents")` to
  `PluginsBrowserPane.BINDINGS` (the Updates pane; keys are hardcoded there, so no
  `default_config.yml` change is needed to match convention). Add
  `action_sync_agents_repos` that submits a tracked task via
  `self.app._submit_tracked_task(...)` reusing `dedup_key="agents-sync"` /
  `exclusive_scopes=("agents-sync",)` and
  `on_complete=self.app._schedule_agents_sync_indicator_revalidation`. The task runs the
  **full** `sync_agents()` (fetch + pull + integrate across every enabled project,
  updating this machine's on-disk agent state) — the network compensation for `,U` no
  longer fetching. Gate visibility in `check_action` so `a` works from every Updates
  sub-tab; add the key to the Core/Plugins/Agent-CLI footer hints. Because it shares the
  `agents-sync` scope with the indicator click and the `,U` agents leg, it cannot
  overlap them.

**Tests:** indicator lights on `behind>0` only and stays dark for
ahead/unexported/error/not_created; `,U` execution passes only behind-project keys and
performs no fetch (assert no network in the leg); `agents_runnable` no-op path; the
Updates `a` binding submits the full-sync tracked task, dedupes against the indicator
click and `,U`, and refreshes the indicator on completion; footer hints include `a`.
Add/refresh the indicator PNG visual snapshot for the incoming-only states
(`just test-visual`).

## Config migration, two-machine e2e, and docs

Land the flow the way a user hits it, migrate the design owner's real config, and finish
the docs.

- **chezmoi migration (bullet 4):** open the chezmoi linked repo with `/sase_repo` and
  edit the `sase_athena.yml` overlay to the new shape — replace `machine_name: athena`
  with an `id:` mapping holding `machine_name: athena` and `username: bbugyi200`. Follow
  the repo's normal chezmoi apply/commit flow. (This is the only real-config change in
  the epic; do it here, after the code that reads the new shape has landed.)
- **Two-machine simulation (integration-style, tmp `SASE_HOME`s + a shared local bare
  "GitHub" remote):** machine A (`bbugyi200`/`alpha`) runs `sase config init`,
  launches/records a multi-member family inside a top-level hood where one member
  commits; asserts local disk holds **bare** names, the agents sidecar holds **global**
  names for the whole hood + family + a rendered `overviews/` page, and the commit's
  `SASE_AGENT` is a link to it with no `SASE_MACHINE`. Machine B (`bbugyi200`/`beta`,
  same user) syncs via the `a`-equivalent full sync and asserts alpha's family imports
  as `alpha.<...>` (username stripped, machine kept), appears in artifact
  dirs/chats/registry, and can be **revived** (`R` path reconstruction) and relaunched
  (raw prompt present). A third identity (`alice`/`gamma`) imports the same family as
  `alice.alpha.<...>` (nothing stripped) — verifying the three strip levels. Round-trip
  B's own committing family back to A. Negative paths: `disabled: true` never
  creates/pushes/syncs; `visibility: private` reaches the provider; missing
  `id.username` produces the actionable error at export/footer time and in doctor/init;
  auto-push failure leaves the code commit intact.
- **Indicator/`,U`/`a` exercise:** a project with detected incoming changes lights the
  indicator (incoming-only); `,U` integrates without fetching; `a` performs the full
  network sync; ahead/unexported never lights the badge.
- **Docs:** finish `docs/agents_sidecar.md` (global identity scheme, whole-hood/family
  publishing, `overviews/` layout, the `SASE_AGENT` link, auto-push on commit,
  indicator/`,U`/`a` semantics, revival) and the `docs/configuration.md` `id` section;
  verify `sase config init` and `sase agent sync -h` meet the CLI rules.
- **Final summary reminder (do not do it):** glossary/memory entries for the "Username
  Agent Hood" and the revised identity model would be natural follow-ups but require the
  design owner's explicit approval; and every existing machine needs a one-time
  `sase config init` run to add `id.username` and migrate `machine_name`.
