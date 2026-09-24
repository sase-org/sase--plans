---
tier: epic
title: Python persistence and wire cutover to agent session (wire-cutover)
goal: 'sase is pinned to the landed core-expand sase-core commit, calls only the new
  agent-session binding names, and names the former agent-family concept "agent session"
  in its canonical metadata keys, Python wire mirrors, Agent model fields, durable
  Python-owned JSON, and the agent name registry. New data is written only with agent-session
  keys and values, every reader still loads pre-rename data through named legacy helpers,
  and `sase tool run check` passes.

  '
phases:
- id: pin-bindings
  title: Core pin bump and new binding names
  depends_on: []
  size: medium
  description: 'pin-bindings: ratchet sase-core-revision.txt to the landed core-expand
    commit, switch every caller to parse_agent_session_name, resolve_agent_session_parent,
    reconcile_agent_artifact_index_dismissed_agent_session_members, and fleet_followed_batch_agent_session_promotions,
    rename the Python facade wrappers for them, update tools/validate_sase_core_rs
    and the demo seed, and tighten the dual-shape directive tests to session-only.'
- id: canonical-keys
  title: Canonical agent-session metadata keys and shared accessor
  depends_on:
  - pin-bindings
  size: medium
  description: 'canonical-keys: make src/sase/plan_chain.py own AGENT_SESSION_* keys,
    the separator, and LEGACY_AGENT_FAMILY_* constants read only by one shared accessor.
    Route every agent_meta.json / done.json reader through it and make every writer
    emit only agent_session, agent_session_role, and agent_session_shell, dropping
    legacy keys on rewrite.'
- id: wire-mirrors
  title: Python wire mirrors hydrate either spelling
  depends_on:
  - canonical-keys
  size: medium
  description: 'wire-mirrors: rename the agent-session fields and types in the src/sase/core
    wire mirrors (scan markers and conversion, agent_scan_wire_family_shell.py to
    agent_scan_wire_agent_session_shell.py, launch, cleanup, group-archive, runner-slot,
    hold, gate hand-off, monitor follow-up, wait-dependency index) and the fleet nodes,
    rows, promotion, and follow store. Each hydrates from either spelling and sends
    new spellings to core.'
- id: agent-model
  title: Agent model fields
  depends_on:
  - wire-mirrors
  size: medium
  description: 'agent-model: rename the family-concept fields of the Agent dataclass
    (src/sase/ace/tui/models/_agent_state.py) to agent_session* and update every src
    and tests reference mechanically, keeping ACE module, label, and row names for
    ace-cutover. Dismissed agent bundles still load the old field names.'
- id: durable-json
  title: Durable Python-owned JSON surfaces
  depends_on:
  - agent-model
  size: medium
  description: 'durable-json: saved dismissed groups (canonical_global_agent_session),
    wait_for_fork_sources and chat-fork source kind session, gate descriptors and
    gate_next_fork session, notification action_data agent_session_root_suffix, ops
    revert requests, launch-request agent_session_type, and stats runtime_group_by.
    Writers emit only new spellings; named legacy readers load pre-rename files.'
- id: name-registry
  title: Agent name registry session kinds and schema v3
  depends_on:
  - agent-model
  size: small
  description: 'name-registry: agent_name_registry.json reservation_kind and container_kind
    become session, readers accept family, and SCHEMA_VERSION goes 2 to 3 through
    the existing legacy-upgrade and stale-cache rebuild path without moving a rebuild
    onto ACE startup or the UI thread.'
- id: verify
  title: Classification sweep and phase verification
  depends_on:
  - durable-json
  - name-registry
  size: small
  description: 'verify: sweep the wire-cutover surfaces for remaining agent-family
    keys, confirm every durable surface has a legacy-input test and a no-legacy-emitted
    test, run sase tool run check, and record hand-offs for runtime-cutover, ace-cutover,
    and core-contract on the sase-17m.3 phase bead.'
proposed_by: bbugyi200.athena.sase-17m.3
parent_bead: sase-17m.3
create_time: 2026-09-24 02:56:40
status: wip
bead_id: sase-17m.3.1
---

- **PROMPT:** [prompts/202609/agent_session_wire_cutover.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_session_wire_cutover.md)
- **PARENT:** [202609/agent_session_rename.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_session_rename.md)
- **BEAD:** [sase-17m.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17m/sase-17m.3.1.md)

# Plan: Python persistence and wire cutover to agent session (wire-cutover)

## Context

This epic implements the `wire-cutover` phase (bead `sase-17m.3`) of the parent epic
"Rename agent family to sase agent session" (`plan:202609/agent_session_rename.md`, epic
`sase-17m`). That parent plan is the authority for vocabulary, identifier rules, the
meanings of "family" that must not change, and the compatibility policy. Before starting
any phase here, read its **Vocabulary**, **Identifier rules**, **Meanings of "family"
that must not change**, **Compatibility policy**, and **Python persistence and wire
cutover** sections.

Repo: **sase** only. Do not edit sase-core. If a phase finds that core does not accept a
new spelling that this plan wants to send, keep sending the legacy spelling through one
named boundary helper. Then record a `PROPOSED FOLLOW-UP:` note for `core-contract` on
your phase bead.

### State of the world at planning time

- `sase-core-revision.txt` pins `1a2a752ff499015642215888ecf3b11f2c1c0c34`.
- The `core-expand` work has landed on sase-core `origin/master` (head `ae9dbf6`,
  "refactor(core): sweep remaining agent-family spellings to agent session").
- What core-expand guarantees:
  - Every renamed serde field and variant is serialized with its legacy name and accepts
    the new name as an alias. For example, `agent_scan/wire.rs` has
    `rename = "agent_family", alias = "agent_session"` and
    `rename = "family_shell", alias = "agent_session_shell"`, and the
    relationship/stats/editor kinds have `rename = "family", alias = "session"`.
  - The hand-read agent-meta scanner (`agent_scan/scanner.rs`) reads `agent_session`,
    `agent_session_role`, and `agent_session_shell` first, then falls back to the legacy
    keys.
  - The ownership planner compares container kinds against both `"family"` and
    `"session"`.
  - Every dict that core returns, and every value it emits, still uses the **legacy**
    spelling. Python readers must therefore accept both spellings throughout this epic.
- New pyo3 binding names are registered next to the legacy names, which stay until
  `core-contract`:
  - `parse_agent_session_name`
  - `resolve_agent_session_parent`
  - `reconcile_agent_artifact_index_dismissed_agent_session_members`
  - `fleet_followed_batch_agent_session_promotions`
- Scale: about 208 src files and 656 test files mention the concept. For example, the
  Agent-field kwargs `agent_family=` and `agent_family_role=` appear about 1,240 times
  in tests.
- No current writer emits `agent_family_parallel`; it has readers only. Per the parent
  policy, it stays a legacy input key. The in-memory and wire fields become
  `agent_session_parallel`, and `agent_session_parallel` is also read as the new key.

### Scope boundary with later sibling phases

This epic renames **keys, values, fields, wire types, and the binding surface**. It does
not rename runtime modules or user syntax:

- `runtime-cutover` (`sase-17m.4`) owns:
  - non-ACE module renames (`agent/_family_attach_*.py`, `_family_promotion.py`,
    `agent_family_plan_preview.py`, `history/chat_fork/family.py`, and so on)
  - the remaining local identifiers, comments, and messages in those packages
  - the `%id(..., session=)` / `session:` / `--next-fork session` user syntax
  - the `legacy_agent_family_syntax` sunset flag
  - CLI help and JSON output keys (`sase agent list -j`, `search -j`, `index`, `wait`,
    and the editor bridge)
- `ace-cutover` (`sase-17m.5`) owns ACE module names, classes, row kinds, labels, CSS
  ids, visible copy, and PNG goldens.
- `core-contract` owns the emitted-value flip, the artifact-index SQLite column and its
  schema bump, and Rust schema versions.
- `session-pages` owns the agents-sidecar `families/` → `sessions/` publication paths.

Inside a file you already touch, you may rename a local variable or private helper that
exists only to carry a renamed key or field when that keeps the file coherent. Do not
start module renames, user-syntax changes, or CLI output-key changes. In particular, the
CLI and gate-spec input `--next-fork family` / `"fork": "family"` keep working unchanged
here. Only the internally stored value is normalized, in `durable-json`.

### Rules for every phase

- Compatibility helpers:
  - Durable readers prefer the new spelling and fall back to the legacy one.
  - Put every fallback in an explicitly named helper or constant, such as
    `LEGACY_AGENT_FAMILY_KEY` or a `legacy_*` / `*_with_legacy_fallback` function, never
    in scattered literals.
  - These readers are permanent: mark each one with a short
    `# legacy agent-family spelling` comment.
- Writers emit only new spellings. Any read-modify-write drops the legacy key.
- Payloads that Python builds and sends to core use the new spelling, which core accepts
  as an alias. Values that Python reads back from core arrive in the legacy spelling
  until `core-contract`. Mirrors must hydrate from either.
- Tests:
  - Keep existing legacy-shaped fixtures that prove migration, and rename them so their
    names say they are legacy.
  - Add new-shape fixtures next to them.
  - For each durable surface a phase touches, add:
    - a legacy-input test that loads a realistic pre-rename file
    - a write test proving that no legacy key or value is emitted
- If you rename a test file, update its test IDs in `tests/shard_timings.json` and
  `tests/reproducible_flake_baseline.txt`.
- Leave every unrelated meaning of "family" alone: `model_family`, provider-usage
  `family:` scopes, `vcs_family`, the Patch `RelationKind.FAMILY`/`.family` operator,
  CSS `font-family`, and so on.
- Verification:
  - Run `just install` first in a fresh workspace.
  - Verify each phase with `sase tool run check`. Never run `just check-full` unless
    explicitly told to.
  - Read `sase/memory/lint_and_test.md` before finishing. Read `sase/memory/tui.md`
    before editing ACE files for anything beyond a mechanical rename.
- Record out-of-scope discoveries as `PROPOSED FOLLOW-UP:` notes on your own phase bead.
  Do not create beads.

## Phase: pin-bindings — Core pin bump and new binding names

1. Run `just ratchet-core-revision` to move `sase-core-revision.txt` to sase-core
   `origin/master`. It must include the core-expand commits, `ae9dbf6` or later. Rebuild
   the local binding (`just install` / `just rust-install`, as `docs/rust_backend.md`
   describes) so tests run against the new core.
2. Switch every binding call to the new names, and rename the Python facade wrappers to
   match:
   - `src/sase/core/agent_identity_facade.py`:
     - `parse_agent_family_name` → `parse_agent_session_name`, calling the
       `parse_agent_session_name` binding
     - its result types, such as `AgentFamilyName*` → `AgentSessionName*`, and the
       `.family_name` attribute → `.agent_session_name`
     - the `AgentFamilyNameKind` enum → `AgentSessionNameKind`. Its `FAMILY` member and
       value stay until `core-contract`, because core still emits `"family"`. Accept
       `"session"` too.
     - the hydration must read the new result key, then the legacy key
     - update `__all__`
   - `src/sase/core/agent_scan_facade.py` and every caller:
     `reconcile_agent_artifact_index_dismissed_family_members` →
     `reconcile_agent_artifact_index_dismissed_agent_session_members`. Callers include
     `core/agent_artifact_index_lifecycle.py` and `agents/cli_index.py`.
   - `src/sase/agent/_family_attach_candidates.py`: `resolve_agent_family_parent` →
     `resolve_agent_session_parent`. Keep the module name; `runtime-cutover` renames it.
   - `src/sase/ace/tui/models/_fleet_agents_promotion.py`:
     `fleet_followed_batch_agent_session_promotions`
   - All `parse_agent_family_name` callers (`git grep -n parse_agent_family_name`):
     - `agent/names/_generation_guard.py`, `_lookup_groups.py`, `_lookup_resolution.py`
     - `agents_sync/inventory*.py`, `publication_snapshot.py`,
       `rendering_family_page.py`
     - `sase_agent.py`, `scripts/_agent_chat_from_name_common.py`, `sdd/hosted_links.py`
3. Update `tools/validate_sase_core_rs`:
   - its `agent_family*`, `family_shell_*`, and `family_id` checks must accept the
     new-or-legacy key, preferring the new one
   - it must call the new binding names
   - also update `demos/scripts/seed_sase_ace_demo`
4. Tighten the dual-shape directive tests from companion commit `a764a76d4` to require
   the `session` `%id` keyword now that the pin includes core-expand:
   - the id keyword tuple in `tests/test_xprompt_directive_contract.py`
   - `tests/test_xprompt_directive_completion_parity.py` and
     `tests/ace/tui/widgets/test_directive_completion_candidates.py`
   - the keyword tuple in `src/sase/ace/tui/widgets/_directive_completion_tokens.py`

   Completion and snippets should offer `session=`, as core's contract now does.
   `family=` must still parse.

5. Confirm that sase never matches on core's old diagnostic codes
   `missing_family_member_name` / `family_conversion_upsert`
   (`git grep -n 'missing_family_member_name\|family_conversion_upsert'`). If a match
   exists, accept both codes.
6. Tests: route the existing binding round-trip tests through the new names, and add one
   test per renamed binding that asserts the new name is registered in `sase_core_rs`.
7. Exit:
   `git grep -nE 'parse_agent_family_name|resolve_agent_family_parent|dismissed_family_members|batch_family_promotions' -- src tests tools demos`
   finds nothing, and `sase tool run check` passes.

## Phase: canonical-keys — Canonical metadata keys and shared accessor

1. `src/sase/plan_chain.py` becomes the owner of the canonical keys:
   - Constants:
     - `AGENT_SESSION_KEY = "agent_session"`
     - `AGENT_SESSION_ROLE_KEY = "agent_session_role"`
     - `AGENT_SESSION_PARALLEL_KEY = "agent_session_parallel"`
     - `AGENT_SESSION_SHELL_KEY = "agent_session_shell"`
     - `AGENT_SESSION_SEPARATOR = "--"`
   - Legacy constants: `LEGACY_AGENT_FAMILY_KEY`, `LEGACY_AGENT_FAMILY_ROLE_KEY`,
     `LEGACY_AGENT_FAMILY_PARALLEL_KEY`, and `LEGACY_AGENT_FAMILY_SHELL_KEY`
     (`"family_shell"`). These replace `AGENT_FAMILY_FIELD`, `AGENT_FAMILY_ROLE_FIELD`,
     `AGENT_FAMILY_PARALLEL_FIELD`, and `AGENT_FAMILY_SEPARATOR`.
   - One shared accessor family, for example `agent_session_value(meta)`,
     `agent_session_role_value(meta)`, `agent_session_parallel_value(meta)`, and
     `agent_session_shell_value(meta)`. Each reads the new key, then the legacy key.
     These are the **only** places that name the legacy constants.
   - A writer helper, for example `set_agent_session_fields(meta, ...)` /
     `strip_legacy_agent_family_keys(meta)`, that writes the new keys and removes the
     legacy ones.
   - Rename the family-concept public helpers in this module to agent-session names, and
     update all callers in src and tests:
     - `agent_family_phase_name`, `agent_family_base`, `agent_family_suffix_token`
     - `is_agent_family_member`, `agent_family_role_for_suffix`
     - `allocate_agent_family_child_suffix`
     - the private `_split_agent_family_name` / `_reserved_*` / `_allocate_*` helpers
       and `_EXPLICIT_FAMILY_ROLES`

     The helpers are used about 250 times.

2. Route every raw reader of these keys through the accessor. Find them with
   `git grep -nE '"agent_family(_role|_parallel)?"|"family_shell"|AGENT_FAMILY_(FIELD|ROLE_FIELD|PARALLEL_FIELD)' -- src`,
   which returns about 110 hits in about 60 files. That includes:
   - agent names lookup (`_lookup_groups.py`, `_lookup_resolution.py`,
     `_registry_scan_payloads.py`), `wait_watch/_resolve.py`, `monitor/start_lane.py`,
     and `monitor/followup.py`
   - `gate_shell/transaction.py`, `reclaim.py`, `log.py`, `handoff_launch.py`, and
     `followup.py`; `plan_shell/create.py`, `question_shell/create.py`, and
     `shells/followup.py`
   - `bead/epic_launch.py`, `core/agent_hold_facade.py`, `core/agent_hold_pending.py`,
     `agent/_restart_planning.py`, `_restart_preview.py`, `running.py`,
     `_running_listing_common.py`, and `fork_waits.py`
   - `agents_sync/inventory.py`, `inventory_sources.py`, `v2_validation.py`, and
     `publication_snapshot.py`; `agents/cli_list.py`; and
     `completion/candidates/catalog_agents.py`
   - `core/wait_dependency_resolution/_artifact_state.py` and `_index.py`;
     `core/runner_slots/_admission_capacity_records.py`; and
     `core/agent_cleanup_targets.py`
   - `scripts/_agent_chat_from_name_*.py`, `xprompt/workflow_loader_definition.py`, and
     `integrations/_agent_list_entry_builder.py`
   - the ACE loaders (`_meta_enrichment_filesystem.py`, `_meta_enrichment_identity.py`,
     `_workflow_loaders.py`), `_confirmation_sase_agents.py`, `_entry_relaunch.py`,
     `_availability_agents.py`, and `ace/revert_agent_resolution.py`. This is a key
     access change only; ACE identifiers stay for `ace-cutover`.
3. Writers emit only the new keys and drop legacy keys on rewrite:
   - `axe/run_agent_directive_metadata.py`, `axe/run_agent_helpers_artifacts.py`,
     `axe/run_agent_successor.py`, `axe/run_agent_runner_finalize.py`,
     `axe/run_agent_exec_questions.py`, and `axe/run_agent_wait_slot*.py`
   - `agent/_family_promotion.py` and `agent/_family_attach_launch.py`
   - the launch-request planning, continuation, and follow-up paths
     (`agent/launch_request_planning.py`, `launch_request_continuation.py`,
     `launch_request_followup.py`)
   - the monitor and gate-shell writers of the nested shell object in `agent_meta.json`
     and `done.json`, which move from `family_shell` to `agent_session_shell`
4. Tests:
   - legacy-input tests: a realistic pre-rename `agent_meta.json` and `done.json`,
     including a nested `family_shell` and an `agent_family_parallel: true` marker,
     still resolve through the accessor, the name lookup, and a core scan
   - write tests: a directive launch, a promotion, and a monitor/gate shell write
     produce no `agent_family*` or `family_shell` key
   - a mixed file that holds both spellings resolves to the new value
5. Exit: outside `plan_chain.py`, no raw `"agent_family"`, `"agent_family_role"`,
   `"agent_family_parallel"`, or `"family_shell"` key literal remains in src, except
   inside explicitly named legacy helpers that the next phases own.
   `sase tool run check` passes.

## Phase: wire-mirrors — Python wire mirrors hydrate either spelling

1. Rename the scan wire mirror:
   - `core/agent_scan_wire_markers.py`: the `agent_family`, `agent_family_role`,
     `agent_family_parallel`, and `family_shell` fields become `agent_session`,
     `agent_session_role`, `agent_session_parallel`, and `agent_session_shell`.
   - `core/agent_scan_wire_conversion.py` and `agent_scan_wire_records.py` hydrate them
     from the new key, then the legacy key.
   - `git mv src/sase/core/agent_scan_wire_family_shell.py src/sase/core/agent_scan_wire_agent_session_shell.py`,
     then rename inside it:
     - `FamilyShellWire` / `FamilyShellMonitorWire` / `FamilyShellGateWire` →
       `AgentSessionShell*Wire`
     - `family_shell_from_mapping` → `agent_session_shell_from_mapping`
     - the private role constants
   - Keep the flat-keys compatibility projection. Update every importer
     (`gate_shell/store.py`, ACE loaders, and so on).
2. Rename the agent-session fields in the other Python mirrors and records. Hydrate each
   from either spelling, and send only new spellings to core:
   - launch records (`agent_launch_wire_records.py`, `_from_dict.py`, `_conversion.py`)
   - cleanup (`agent_cleanup_wire.py`, `agent_cleanup_python.py`,
     `agent_cleanup_targets.py`)
   - group archive (`agent_group_archive_wire.py`: `canonical_global_family` →
     `canonical_global_agent_session`, reading the legacy key)
   - runtime (`agent_runtime_wire.py`) and alias history (`agent_alias_history_wire.py`)
   - runner-slot records (`core/runner_slots/*`)
   - hold identity (`agent_hold_facade.py`, `agent_hold_pending.py`,
     `agent_hold_liveness.py`), whose hold payload key `family` becomes `session` when
     sent to core. Core accepts `session` as an alias.
   - the wait-dependency index (`core/wait_dependency_resolution/_index*.py`,
     `_artifact_state.py`, `_types.py`, `_submitted_plans.py`)
   - artifact relations (`artifact_relation_layout.py`, `artifact_relations.py`); the
     relation-kind value stays for `ace-cutover`
   - gate hand-off evidence and monitor follow-up kwargs (`monitor/followup.py`,
     `gate_shell/followup.py`, `shells/followup.py`)
3. Rename the fleet Python mirrors:
   - `ace/tui/models/_fleet_agents_nodes.py`, `_fleet_agents_rows.py`,
     `_fleet_agents_identity.py`, and `_fleet_agents_promotion.py`
   - `dispatch/follow_store.py`: `promote_family_follow` →
     `promote_agent_session_follow`, and its `family_locator` argument; update the
     callers in `ace/tui/actions/agents/_fleet_follow.py`

   Core still emits `family_id` / `family_role` / `family_label` and `family:` logical
   keys. Read new-then-legacy, and store `follows.json` logical keys exactly as core
   returns them. Core-expand already parses both `family:` and `session:` segments.

4. Tests:
   - wire mirrors built from legacy-shaped core dicts and from new-shaped dicts hydrate
     identically
   - payloads sent to core carry only new spellings, and core accepts them in a real
     round trip through `sase_core_rs`
   - rename the shell-wire tests to match the module
5. Exit: no family-concept field or type name remains in `src/sase/core/` except named
   legacy readers and the `AgentSessionNameKind.FAMILY` value that `core-contract`
   flips. `sase tool run check` passes.

## Phase: agent-model — Agent model fields

1. In `src/sase/ace/tui/models/_agent_state.py`, rename the family-concept fields of the
   `Agent` dataclass:
   - `agent_family` → `agent_session`
   - `agent_family_role` → `agent_session_role`
   - `agent_family_parallel` → `agent_session_parallel`
   - `is_imported_family_container` → `is_imported_agent_session_container`
   - `is_remote_family_container` → `is_remote_agent_session_container`
   - `family_container` → `agent_session_container`
   - `imported_family_parent_synthetic` → `imported_agent_session_parent_synthetic`
   - `derived_plan_family_root` → `derived_plan_agent_session_root`

   Also update the field comments.

2. Update every reference mechanically, including `getattr(agent, "agent_family…")`
   strings. Use
   `git grep -nP '\b(agent_family(_role|_parallel)?|is_(imported|remote)_family_container|family_container|imported_family_parent_synthetic|derived_plan_family_root)\b' -- src tests`.
   There are about 200 src references and about 1,600 test references, mostly
   `Agent(agent_family=..., agent_family_role=...)` kwargs. Leave ACE module, class,
   label, and row-kind names, such as `_agent_status_family_core.py` and
   `agent_family_members.py`, for `ace-cutover`.
3. Dismissed agent bundles (`ace/tui/models/agent_bundle.py`, `_dismiss_persistence.py`,
   `_dismiss_memory.py`):
   - bundles are written with the new field names
   - loading maps the old field names through one named legacy-field table
   - add a test that loads a realistic pre-rename dismissed bundle, and one that proves
     a written bundle holds no legacy field
4. Watch the performance contract: this is a rename only. Add no filesystem work or
   awaits to navigation or render paths.
5. Exit: no reference to the old `Agent` field names remains in src or tests outside the
   bundle legacy-field table. `sase tool run check` passes.

## Phase: durable-json — Durable Python-owned JSON surfaces

For each surface below:

- writers emit only the new spelling
- one named legacy reader accepts the pre-rename key or value
- a legacy-input test and a no-legacy-emitted test cover it

Surfaces:

1. Saved dismissed groups (`ace/dismissed_agent_groups.py` and its archive wire):
   `canonical_global_family` → `canonical_global_agent_session`.
2. `wait_for_fork_sources` and chat-fork sources: source kind `"family"` → `"session"`.
   - Files: `agent/fork_waits.py`, `history/chat_fork/common.py`, `build.py`,
     `continuation/replay.py`, and
     `core/wait_dependency_resolution/_index_fork_queries.py`.
   - `continuation_baseline.py` and `scripts/sase_chop_wait_checks.py` must accept both
     kinds.
   - Stored `"family"` kinds still load and resolve.
3. Gate descriptors and `gate_next_fork`: store `"session"` instead of `"family"`.
   - Files: `notification_gates/model_shell.py` (`GATE_SHELL_NEXT_FORKS`), the gate
     descriptor, `agent/launch_request.py`, and the gate-shell store.
   - Normalize the user-supplied `family` value to `session` at the input boundary. The
     CLI choices and the gate-spec input stay as they are; `runtime-cutover` owns that
     syntax.
   - A stored `"family"` still loads.
4. Notification `action_data`: add `agent_session_root_suffix` as the key written by
   `gate_shell/transaction.py`, and add it first in the existing multi-key lookup
   (`ace/tui/actions/agents/_notification_*.py`). `family_root_suffix` stays a named
   legacy key.
5. Ops revert requests (`ops/commands/_agent_revert.py`, `ace/revert_agent_*.py`):
   - `family_base` → `agent_session_base`
   - revert scope `"family"` → `"session"`
   - queued requests with the old spelling still execute
6. Launch requests: `agent_family_type` → `agent_session_type` and its context keys in
   `agent/relaunch_prompt.py`, `agent/launch_request*.py`, and the bead epic-launch
   hand-off payloads (`bead/epic_launch_handoff_*.py`, `epic_launch.py`).
7. Stats `runtime_group_by`: `stats/query.py`, `stats/_view_builders.py`, and the ACE
   statistics pane data.
   - The group-by value `"family"` → `"session"`, sent to core, which accepts the alias.
   - Stored or configured `"family"` still loads.
   - Core returns `"family"` until `core-contract`, so readers accept both.
8. Fleet `follows.json`: confirm that Python only stores the logical keys core returns
   and never parses the `family:` segment. Add a test that a `follows.json` with
   `family:` keys and one with `session:` keys both load and promote. If Python does
   parse the segment anywhere, accept both.
9. Exit: every surface listed above has both tests, and `sase tool run check` passes.

## Phase: name-registry — Agent name registry session kinds and schema v3

1. In `src/sase/agent/names/`, store `reservation_kind` and `container_kind` as
   `"session"` instead of `"family"`:
   - writers: `_registry_group_mutations.py`, `_registry_scan_entries.py`,
     `_forced_reuse.py`
   - readers: `_registry_batch.py`, `_registry_queries.py`
   - the typed `Literal["family", "clan"]` parameters become
     `Literal["session", "clan"]`, including in `bead/cli_work_cleanup_apply.py`,
     `cli_work_cleanup_targets.py`, and `cli_work_legacy_preview.py`
   - `agents/catalog/_derive.py` and the `agents_sync` container kinds accept both
     values; the published sidecar kinds stay for `session-pages`

   Readers accept `"family"` through one named helper, for example
   `is_agent_session_container_kind(value)`. Core ownership-planner results still carry
   `"family"`; normalize them at the Python apply boundary.

2. Bump `SCHEMA_VERSION` in `_registry_store.py` from 2 to 3. Make v2 a legacy version:
   - read it and upgrade its entries' kinds in memory
   - mark it `_needs_rebuild`, as the existing v1 path does
3. Confirm where a stale registry rebuild is triggered. It must stay on the existing
   off-UI-thread / lazy path and must not move an archive-sized rebuild onto ACE startup
   or the UI thread. If it would, gate it the way the v1 upgrade is gated.
4. Tests:
   - a realistic v2 registry file with `"family"` kinds loads, answers container queries
     correctly, and is rebuilt/rewritten as v3 with only `"session"`
   - a v3 write emits no `"family"` kind
   - the forced-reuse and bead work-cleanup paths still release stale session containers
5. Exit: `sase tool run check` passes.

## Phase: verify — Classification sweep and phase verification

1. Sweep the surfaces this epic owns:
   - `git grep -nE '"agent_family|"family_shell"|canonical_global_family|family_root_suffix|agent_family_type|family_base' -- src tests tools demos`
   - `git grep -nP '\bagent_family(_role|_parallel)?\b' -- src`

   Classify every hit as one of:
   - a named legacy reader, constant, or fixture
   - out of scope for this epic, belonging to `runtime-cutover`, `ace-cutover`,
     `session-pages`, or `core-contract`
   - an unrelated meaning

   Fix any in-scope straggler in place.

2. Confirm:
   - every durable surface in the parent plan's **Python persistence and wire cutover**
     section has a legacy-input test and a no-legacy-emitted test
   - the core pin includes core-expand
   - no legacy binding name is called
3. Record hand-offs as `PROPOSED FOLLOW-UP:` or evidence notes on `sase-17m.3`:
   - for `core-contract`: the list of Python readers that can drop their
     legacy-emitted-value branches, and any spelling that core was found not to accept
   - for `runtime-cutover`: the module renames this epic deliberately left, and any
     remaining raw family identifiers in non-ACE packages
4. Exit: `sase tool run check` passes on the final tree.
