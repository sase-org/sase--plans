---
tier: epic
title: sase-core additive agent-session rename (core-expand)
goal: 'sase-core names the former agent-family concept "agent session" in every Rust
  module, type, function, constant, enum variant, test, comment, and message, and
  exposes the new pyo3 binding names next to the legacy ones. Every input accepts both
  spellings. Serialized output, schema versions, SQLite columns, and goldens stay
  byte-identical, so a sase tree pinned to the previous core, and every sase workspace
  that rebuilds against the new core, keeps passing `sase tool run check`.

  '
phases:
  - id: identity-directives
    title: Identity, launch, holds, and directive/editor surfaces
    depends_on: []
    size: medium
    description:
      "identity-directives: rename agent_family.rs to agent_session.rs and the
      agent_identity, artifact_link, agent_launch, hold, editor, and LSP family concept.
      Add the parse_agent_session_name and resolve_agent_session_parent bindings, accept
      %id session=, reserve session/sessions, and flip editor completion to session=
      with companion sase tests that tolerate both core shapes."
  - id: scan-runtime
    title: Scan, runtime, lifecycle, runner, and stats wires
    depends_on:
      - identity-directives
    size: medium
    description:
      "scan-runtime: rename the family concept in agent_scan, agent_runtime,
      agent_clan_record, agent_cleanup, agent_ownership, agent_group_archive,
      agent_stats, runner_capacity, gate_followup, and their neighbours. Pin legacy
      serde spellings, read new-then-legacy keys in hand-read JSON, and add the
      reconcile_agent_artifact_index_dismissed_agent_session_members binding."
  - id: fleet
    title: Fleet core and gateway
    depends_on:
      - scan-runtime
    size: medium
    description:
      "fleet: rename fleet_family.rs to fleet_agent_session.rs and the family concept in
      fleet_*, fleet_contract, and sase_gateway. Add the
      fleet_followed_batch_agent_session_promotions binding, accept session: logical-key
      segments and session-<hex> fallback ids on input, and keep fleet_api_v1.json and
      emitted keys unchanged."
  - id: sweep
    title: Classification sweep and cross-repo verification
    depends_on:
      - fleet
    size: small
    description:
      "sweep: classify every remaining famil hit in sase-core, fix stragglers, confirm
      byte-identical output and unchanged schema versions, and verify that a sase
      workspace built against the final core passes sase tool run check. Record
      follow-ups for wire-cutover and core-contract on the phase bead."
proposed_by: bbugyi200.athena.sase-17m.2
parent_bead: sase-17m.2
create_time: 2026-09-23 22:54:46
status: wip
---

- **PROMPT:**
  [prompts/202609/agent_session_core_expand.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_session_core_expand.md)
- **PARENT:**
  [202609/agent_session_rename.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_session_rename.md)

# Plan: sase-core additive agent-session rename (core-expand)

## Context

This epic implements the `core-expand` phase of the parent epic "Rename agent family to
sase agent session" (`plan:202609/agent_session_rename.md`). That parent plan is the
authority for vocabulary, identifier rules, the meanings of "family" that must not
change, and the compatibility policy. Read its **Vocabulary**, **Identifier rules**,
**Meanings of "family" that must not change**, **Compatibility policy**, and **sase-core
additive rename** sections before starting any phase here.

Repo: **sase-core** (linked repo). Open it with `/sase_repo`
(`sase repo open sase-core -r "<why>"`), read its `AGENTS.md`, and work only in the
printed path. Every phase lands as non-breaking Conventional Commits (`feat(core): ...`
or `refactor(core): ...`, never `feat!`). Do not edit any `version`, `CHANGELOG.md`,
`*_SCHEMA_VERSION` constant, SQLite column or index name, or golden contract
(`crates/sase_gateway/contracts/api_fleet_v1/fleet_api_v1.json`,
`crates/sase_core/tests/python_wire_parity.rs` key order). Those change in the parent
epic's later `core-contract` phase.

### Scale

Before this epic, `git grep -i famil -- crates` finds about 1,650 hits in 141 files. The
largest are `agent_runtime.rs`, `fleet_family.rs`, `agent_identity/identity.rs`,
`fleet_follow_promotion.rs`, `agent_hold.rs`, `fleet_presentation.rs`,
`agent_cleanup/planner.rs`, `fleet_catalog.rs`, `agent_ownership/planner.rs`,
`agent_family.rs`, and `agent_scan/scanner.rs`. Many hits are unrelated meanings that
must stay: `provider_usage/` (`model_family`, `family:3p`, AGY families), `query/` (the
ChangeSpec `__N` revert family, tokenizer comments, and the generic `family` field in
query tests), `glossary.rs` (the `Family`→`Families` pluralization test),
`source_language.rs`, `vcs_log/`, and `machine_setup/` (OS/VCS families). Check each hit
before renaming it.

### Shared rules for every phase

1. **Rust names.** Follow the parent plan's identifier rules. `agent_family` →
   `agent_session`, `AgentFamily` → `AgentSession`, `AGENT_FAMILY` → `AGENT_SESSION`. A
   bare `family`/`Family`/`FAMILY` inside a longer identifier becomes
   `agent_session`/`AgentSession`/`AGENT_SESSION` (for example `family_shell` →
   `agent_session_shell`, `FamilyShellWire` → `AgentSessionShellWire`, `family_id` →
   `agent_session_id`). Never use a bare `session` in an identifier. Enum variants that
   stand for the kind value (for example `AgentContainerKind::Family`) become `Session`.
   Rename comments, docstrings, error messages, log messages, test names, and test
   helpers along with the code. Do not keep Rust-side `pub use` aliases for old Rust
   names. Callers inside the workspace are renamed in the same phase, and the compiler
   finds them.
2. **Serialized output stays byte-identical.** Every renamed serialized field or enum
   variant gets `#[serde(rename = "<legacy>", alias = "<new>")]`. It emits the legacy
   spelling and accepts both. This includes structs with `deny_unknown_fields`, where
   `alias` still works. Where an enum uses `rename_all`, give the renamed variant an
   explicit `#[serde(rename = "family", alias = "session")]`. Where a value is produced
   by a hand-written `as_str`/`match` (for example `Self::Family => "family"`,
   `ConvertFamily => "convert_family"`, `"serial_family"`, `RESERVATION_KIND_FAMILY`,
   `CONTAINER_KIND_FAMILY`, link target kind `"family"`, path `families/`), keep the
   emitted string. Put it in a clearly named constant, for example
   `LEGACY_AGENT_SESSION_KIND: &str = "family"`, with a
   `// legacy agent-family spelling; flips in core-contract` comment. Make the matching
   parser (`FromStr`, `match`, or `parse_*`) accept the new value (`"session"`,
   `"convert_session"`, `"serial_session"`, `sessions/`) as well.
3. **Hand-read JSON** (`data.get("agent_family")` and similar) reads the new key first
   (`agent_session`, `agent_session_role`, `agent_session_shell`, and so on) and falls
   back to the legacy key. Route this through one small named helper per module, for
   example `agent_session_str(data)`, not scattered literals. `agent_family_parallel` is
   the retired parallel marker. It stays a legacy input key only. Rename the in-memory
   field and pin its serialized name. Do not invent a new `agent_session_parallel` key
   unless a current writer needs one.
4. **pyo3 bindings.** Register the new Python name and keep the legacy name registered
   until `core-contract`. PyO3 allows one `#[pyo3(name)]` per function, so write the new
   `#[pyfunction] #[pyo3(name = "<new>")] fn py_<new>` and a thin legacy
   `#[pyfunction] #[pyo3(name = "<legacy>")] fn py_<legacy>` that calls a shared private
   body. Give the legacy wrapper a `// legacy binding name; removed in core-contract`
   comment. Do not use `macro_rules!`. Register both in `register_<domain>`, rename the
   `core_*` alias in `crates/sase_core_py/src/prelude.rs` to the new name, and add a
   round-trip test in the domain's `tests.rs` that calls both names and gets equal
   results. Returned dict keys stay legacy in this epic.
5. **Tests.** Rename family-named tests and helpers. For each renamed input, add a test
   that the new spelling and the legacy spelling deserialize or parse to the same value.
   Also assert that serialization still emits exactly the legacy JSON. A
   `serde_json::to_string` equality against a literal legacy fixture is enough. Keep
   existing legacy-shaped fixtures; they are now the proof that legacy input still
   loads.
6. **Crate root and prelude.** When you rename an item that
   `crates/sase_core/src/lib.rs` re-exports at the root, rename the entry in place. Do
   not add new root names or new `core_*` prelude aliases beyond renaming existing ones
   (per `AGENTS.md`, both lists are being retired). Prefer module-path imports in code
   you touch.
7. **Verify.** Batch edits, iterate with `just fast` and
   `just test -p <crate> <filter>`, then pass `sase tool run check` in sase-core before
   finishing the phase. It takes about 5 minutes; give it a tool timeout of 15 minutes
   or more. Never run bare `cargo`. Never run `just check-full`.
8. **Phase exit record.** Each phase records on its own bead the remaining `famil` hits
   in the files it owns, grouped as "unrelated meaning", "legacy spelling pinned for
   core-contract", or "legacy binding name". Record work for the later sase phases as
   `PROPOSED FOLLOW-UP:` notes.

## Phase identity-directives: Identity, launch, holds, and directive/editor surfaces

Scope (sase-core):

- `crates/sase_core/src/agent_family.rs` → `agent_session.rs` (update `lib.rs`
  `pub mod`). Rename `resolve_agent_family_parent`, the `AgentFamily*` resolution wires
  (`AgentFamilyDismissedIdentityWire`, `AgentFamilyParentCandidateWire`,
  `AgentFamilyParentResolutionRequestWire`, `AgentFamilyParentResolutionWire`), and
  `AGENT_FAMILY_RESOLUTION_WIRE_SCHEMA_VERSION` → `AGENT_SESSION_RESOLUTION_...` with
  the value unchanged.
- `agent_identity/` (`identity.rs`, `relationships.rs`, `mod.rs`):
  - `parse_agent_family_name` → `parse_agent_session_name`
  - `AgentFamilyNameWire` → `AgentSessionNameWire`, with `family_name` →
    `agent_session_name` pinned to `"family_name"`
  - `InvalidFamilyName` → `InvalidAgentSessionName`. The error text becomes "invalid
    agent session name ...". Search sase for tests that match the old message text
    (`git grep -n "invalid family name"` in the sase workspace). If any exist, keep the
    message compatible or make the companion sase test change described below.
  - `historical_family_scope`, `ancestors_for_family_name`,
    `parse_normalized_family_name`, `AgentContainerKind::Family`, and
    `FamilyContainerMemberMismatch`
  - the relationship role/kind `Family` (`relationships.rs` emits `"family"` and
    relationship keys `family:<global>`). Emitted strings stay legacy. Parsers also
    accept `session` and `session:<global>`.
  - the link target from `agent_link_target`: kind stays `"family"` and path stays
    `families/<global>.md` (emitted). Any code that parses an agent link path or kind
    also accepts `sessions/<global>.md` and `"session"`.
  - reserved names: add `session` and `sessions` next to `families`/`family` in the
    reserved list (`identity.rs` line ~10). First check that no existing agent uses
    those names: search `~/.sase` agent artifacts and names registry, for example
    `rg -l '"(name|agent_name)": "sessions?"' ~/.sase/projects` and
    `sase agent list -j`. If one does, reject them only for new names (at the creation
    and validation points). Leave historical parsing alone. Record which case applied on
    the bead.
- `artifact_link/path.rs` and `artifact_link_eligibility/`: rename the concept, and keep
  emitting `families/<global>.md`. Any parser of agent page paths also accepts
  `sessions/`.
- `agent_launch/` (`identity.rs`, `typed_units.rs`, `plan_resolution.rs`,
  `launch_hold.rs`, `admission.rs`, `wires.rs`):
  - `%id` parsing accepts `session=<parent>` and `family=<parent>` as equal spellings.
    Both together is an error (`invalid-id-session`, with the message naming the
    conflict). Keep the existing `invalid-id-family` diagnostic code for errors raised
    on a `family=` argument, and add `invalid-id-session` for `session=`. Check that
    sase does not match on these codes (`git grep -n invalid-id-family` in the sase
    workspace).
  - `DirectiveValueRole::Family` → `Session` (serde-pinned), and the `"family"`
    positional keyword membership list gains `"session"`.
  - The launch wires' family fields and values follow shared rule 2.
- Holds: `hold_directive.rs`, `agent_hold.rs`, and `agent_hold_deadlock.rs`. The hold
  store's `families` selector list and `"family"` match kind follow shared rules 2
  and 3. Hold normalization reads `sessions`/`session` first, then the legacy spelling,
  and still writes the legacy spelling.
- Editor and LSP: `editor/directive/metadata.rs`, `editor/completion/*`,
  `editor/diagnostics.rs`, `editor/wire.rs`, `editor/directive/tests.rs`, and
  `crates/sase_xprompt_lsp/` (`lsp_convert.rs`, `server/tests/*`,
  `tests/jsonrpc_stdio.rs`):
  - The `id` directive metadata gets keyword `session`. Update the `conflicts_with`
    lists of `clan` and `tribe` to name `session` as well as `family`.
  - `directive_contract()` lists `session` as an additional keyword and keeps `family`,
    so an older sase still reads the contract.
  - Completion, snippets, and hover offer only `session=`. `family=` stays accepted by
    diagnostics (no unknown-keyword warning) but is never suggested.
  - The existing test that `%family`/`%f` are unknown directives stays.
- Bindings (`crates/sase_core_py/src/agent_identity/`): add `parse_agent_session_name`
  and `resolve_agent_session_parent`, and keep `parse_agent_family_name` and
  `resolve_agent_family_parent` as legacy registrations (shared rule 4). Update
  `agent_launch`, `agent_holds`, `agent_custody`, and `editor_completion` binding code
  that names the renamed core items.

Companion sase change (sase workspace, primary checkout). The editor-surface flip is the
one part of this epic that changes behavior sase tests pin exactly. Every sase workspace
rebuilds `sase_core_rs` from its linked sase-core checkout, while sase CI builds the
core pinned in `sase-core-revision.txt`. Any sase test change must therefore pass
against **both** the pinned core and this phase's core:

1. In the sase workspace, run `just install` (it builds `sase_core_rs` from the linked
   sase-core checkout), then run `sase tool run check`. Known exact pins to expect:
   - `tests/test_xprompt_directive_contract.py` asserts the `id` keyword tuple
     `("bead", "clan", "family", "tribe")`
   - `tests/test_xprompt_directive_completion_parity.py` covers `%id(worker, fa`
   - `src/sase/ace/tui/widgets/_directive_completion_tokens.py`
     `_looks_like_id_keyword_prefix` hardcodes the keyword list
2. For each failure caused by this phase, make the smallest test change that accepts
   both core shapes. For example, assert the `id` keywords equal either the legacy tuple
   or the legacy tuple plus `session`. For the ACE keyword-prefix helper, add
   `"session"` to its tuple (safe with either core). Leave every other sase
   family-concept rename to the later sase phases.
3. Commit the sase change and the sase-core change in the same turn. Record
   `PROPOSED FOLLOW-UP: wire-cutover tightens the dual-shape directive tests to the new shape after the core pin bump`
   on this phase's bead.
4. If a failure cannot be made dual-shape safe, keep that single surface on its legacy
   behavior in core for this epic. Record it as a `PROPOSED FOLLOW-UP` for
   `core-contract` instead of breaking sase.

Exit: `sase tool run check` passes in sase-core. The sase workspace, rebuilt against
this core, passes `sase tool run check`.

## Phase scan-runtime: Scan, runtime, lifecycle, runner, and stats wires

Scope (sase-core):

- `agent_scan/` (`scanner.rs`, `wire.rs`, `mod.rs`,
  `index/{lineage,dismissal, candidates,storage,machine_projection}.rs`, and their
  tests):
  - `FamilyShellWire`, `FamilyShellGateWire`, and `FamilyShellMonitorWire` →
    `AgentSessionShell*Wire`, with the field `family_shell` → `agent_session_shell`
    pinned
  - the `agent_family`, `agent_family_role`, and `agent_family_parallel` fields on the
    meta and done wires, pinned per shared rule 2
  - `resolve_family_dismissal_lineage` and `FamilyDismissalLineage*Wire` →
    `resolve_agent_session_dismissal_lineage` and `AgentSessionDismissalLineage*Wire`
  - `reconcile_agent_artifact_index_dismissed_family_members` →
    `reconcile_agent_artifact_index_dismissed_agent_session_members`
  - scanner hand-read keys (around `scanner.rs:1106–1120` and `:1271`) go through a
    named new-then-legacy helper. `family_shell_from_object` becomes
    `agent_session_shell_from_object` and reads `agent_session_shell` first, then
    `family_shell`. `gate_next_fork` values accept `session` as well as `family`; the
    stored value is not rewritten.
  - The SQLite column `agent_family` and its index keep their names, and so does the
    `"agent_family"` column literal in `candidates.rs`/`storage.rs`. Put that literal in
    one named constant (`AGENT_SESSION_INDEX_COLUMN = "agent_family"`, with the
    shared-rule-2 comment). Rename the Rust variables around it.
- `agent_runtime.rs`, `agent_clan_record.rs`, `agent_tribe.rs`, `machine_hood.rs`,
  `artifact_file/`, `artifact_ref/`, and `axe_chop/`: rename the family concept, and
  route hand-read keys through the helper.
- `agent_cleanup/` (`planner.rs` and its wires), `agent_group_archive/`, and
  `gate_followup/`: rename the family concept, and pin the serialized fields.
- `agent_ownership/` (`planner.rs`, `wire.rs`):
  - `ConvertFamily` → `ConvertSession`, emitting `"convert_family"` and parsing
    `"convert_session"` too
  - `RESERVATION_KIND_FAMILY` and `CONTAINER_KIND_FAMILY` → `..._AGENT_SESSION`
    constants whose value stays `"family"`. The matching comparisons accept `"session"`
    too.
  - diagnostic codes such as `missing_family_member_name` and
    `family_conversion_upsert`: keep the emitted code strings if sase matches on them
    (`git grep` the sase workspace first). Otherwise rename them to
    `missing_agent_session_member_name`/`agent_session_conversion_upsert`, and record
    the rename as a `PROPOSED FOLLOW-UP` for wire-cutover.
- `agent_stats/` (`runner.rs`, `run/*`): `AgentStatsRuntimeGroupByWire::Family` →
  `Session`, emitting `"family"` and accepting `"session"`. The hand-read meta keys go
  through the helper.
- `runner_capacity/` (`claims.rs`, `records.rs`, `wire.rs`, and tests): the claim kind
  `"serial_family"` stays emitted, and `"serial_session"` is accepted wherever claim
  kinds are parsed or compared. Rename the runner-slot family helpers.
- Tests: `crates/sase_core/tests/agent_scan_parity.rs` and `python_wire_parity.rs`.
  Rename Rust identifiers only. Key-order assertions and Python fixtures stay unchanged.
  Add dual-spelling input tests for the meta, done, and `family_shell` objects, the
  hold-free runner-slot records, the cleanup and ownership wires, and the stats
  group-by.
- Bindings (`crates/sase_core_py/src/agent_scan/`): add
  `reconcile_agent_artifact_index_dismissed_agent_session_members` next to the legacy
  name (shared rule 4). Update `agent_scan`, `provider_policy`, and other binding code
  that names renamed items.

Exit: `sase tool run check` in sase-core.

## Phase fleet: Fleet core and gateway

Scope (sase-core):

- `fleet_family.rs` → `fleet_agent_session.rs` (update `lib.rs`):
  - `concrete_family_shell_kind`, `family_id_for_record`, `family_key_for_record`,
    `family_shell`, `record_is_concrete_family_shell`, and `ConcreteFamilyShellKind` →
    agent-session names
- `fleet_follow_promotion.rs`: `followed_batch_family_promotions` and
  `FollowedBatchFamilyPromotion{Request,Result}Wire` → agent-session names.
  `FollowFamilyPromotionWire` → `FollowAgentSessionPromotionWire`.
- `fleet_contract/` (`locators.rs`, `projection.rs`, `resolution.rs`, `follows.rs`, and
  tests):
  - `LogicalAgentLocatorWire.family_id` → `agent_session_id`, pinned to `"family_id"`.
    Validation error labels may keep naming the serialized field.
  - `family_role` and `family_label` projection fields are pinned the same way.
  - `logical_key_unchecked` keeps emitting the `family` segment. Add one
    `logical_key_matches(stored, locator)` helper that accepts a stored key built with
    either a `family:` or a `session:` segment. Use it wherever a stored `logical_key`
    is validated against its locator (`follows.rs` around line 286) or compared.
  - fallback ids: `safe_identifier(..., "family")` keeps emitting `family-<hex>`. Where
    stored agent-session ids are compared (follow matching and promotion), treat
    `session-<hex>` and `family-<hex>` with the same digest as equal. Put this in one
    named helper with tests.
- `fleet_catalog.rs`, `fleet_presentation.rs`, `fleet_owner_facts.rs`,
  `fleet_mutation.rs`, and `fleet_attention.rs`: rename the concept, and keep the wire
  spellings.
- `crates/sase_gateway/` (`fleet_reads/*`, `federation_worker/*`, `fleet_launch.rs`,
  `routes/*`, `contract.rs`): rename the Rust identifiers. `contract.rs` and
  `contracts/api_fleet_v1/fleet_api_v1.json` must stay byte-identical, including the
  `"family"` description entry. Confirm with `just test -p sase_gateway committed_`, run
  without `UPDATE_*` variables.
- Bindings (`crates/sase_core_py/src/fleet/`): add
  `fleet_followed_batch_agent_session_promotions` next to
  `fleet_followed_batch_family_promotions` (shared rule 4).
- Tests: rename family-named fleet tests and helpers (for example the `"family-1"`
  fixture ids may stay as data, but rename the variables). Add tests for
  `logical_key_matches` with both segments, fallback-id equivalence, and
  `family_id`/`agent_session_id` dual input on the locator.

Exit: `sase tool run check` in sase-core.

## Phase sweep: Classification sweep and cross-repo verification

- In sase-core, run `git grep -n -i famil -- crates ':!*CHANGELOG*'` and
  `git grep -n -E 'Family|FAMILY'`. Classify every remaining hit as one of:
  - an unrelated meaning (see the parent plan's list)
  - a legacy serialized spelling, constant, or alias pinned for `core-contract`
  - a legacy binding name
  - a legacy-input fixture or test

  Fix small stragglers (identifiers, comments, messages, and test names) in place.

- Confirm that no `*_SCHEMA_VERSION` value, SQLite schema, golden, or
  `python_wire_parity.rs` key-order assertion changed across the whole epic
  (`git diff <pre-epic-commit>..HEAD` restricted to those items).
- Confirm that all four new binding names and all four legacy names are registered. Use
  a quick `python -c "import sase_core_rs as m; ..."` after `just install` in the sase
  workspace.
- Cross-repo check: in the sase workspace, run `just install` against the final core,
  then `sase tool run check`. It must pass with no sase changes beyond the
  identity-directives companion tests.
- `sase tool run check` in sase-core.
- On this phase's own bead, record `PROPOSED FOLLOW-UP:` notes listing:
  - every legacy serialized spelling, constant, and binding that `core-contract` must
    flip or remove (by file)
  - every sase-side test or reader that `wire-cutover` must tighten or switch to the new
    binding names

Exit: every remaining `famil` hit is classified, and `sase tool run check` passes in
sase-core and in the sase workspace.
