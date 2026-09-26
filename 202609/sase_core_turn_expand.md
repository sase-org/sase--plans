---
tier: epic
title: sase-core additive sase-turn rename (core-expand)
goal: 'sase-core names the former sase-shell concept with turn and named-proc vocabulary
  in Rust modules, types, functions, constants, tests, comments, and messages, and
  registers the two new pyo3 binding names next to the legacy ones. Every renamed
  input accepts the old and new spellings. Serialized output, schema versions, the
  SQLite gate_shell_id column, and goldens stay byte-identical, so a sase tree pinned
  to the previous core, and a sase workspace rebuilt against this core, still passes
  sase tool run check with no sase source changes.

  '
parent_bead: sase-1ab.1
phases:
- id: scan-wires
  title: Agent-scan wires and gate lookup
  depends_on: []
  size: medium
  description: 'scan-wires: rename the agent-scan shell wires, hand-read meta keys,
    and gate-id lookup, pin legacy serde output, keep the SQLite column, and register
    the new gate lookup binding beside the legacy one.

    '
- id: procs
  title: Named-proc store, launch, and holds
  depends_on:
  - scan-wires
  size: medium
  description: 'procs: rename proc-shell fields, lifecycle helpers, launch-plan names,
    and hold candidates to named-proc vocabulary, accept the new spellings, keep emitted
    values legacy, and register the new name validator beside the legacy binding.

    '
- id: fleet-runtime
  title: Fleet, runner capacity, and gateway
  depends_on:
  - procs
  size: medium
  description: 'fleet-runtime: rename fleet row kinds, locator ids, owner status,
    and runner-slot shell fields, emit the legacy spellings, accept the new ones,
    and leave the fleet golden unchanged.

    '
- id: sweep
  title: Editor text, classification, and cross-repo check
  depends_on:
  - fleet-runtime
  size: small
  description: 'sweep: retarget the editor proc snippet, classify every remaining
    shell hit, and prove a sase workspace rebuilt against this core passes check with
    no sase source changes.'
proposed_by: bbugyi200.athena.sase-1ab.1
create_time: 2026-09-26 00:28:10
status: wip
bead_id: sase-1ab.1.1
---

- **PROMPT:** [prompts/202609/sase_core_turn_expand.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sase_core_turn_expand.md)
- **PARENT:** [202609/sase_turn_rename.md](https://github.com/sase-org/sase--plans/blob/main/202609/sase_turn_rename.md)
- **BEAD:** [sase-1ab.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ab/sase-1ab.1.1.md)

# Plan: sase-core additive sase-turn rename (core-expand)

## Context

This epic implements the `core-expand` phase of the parent epic "Rename sase shell to
sase turn" (`plan:202609/sase_turn_rename.md`, bead `sase-1ab`). That parent plan is the
authority for vocabulary, identifier rules, the meanings of "shell" that must not
change, and the compatibility policy. Read its **Vocabulary**, **Identifier rules**,
**Meanings of "shell" that must not change**, **Compatibility policy**, and **sase-core
additive rename** sections before starting any phase here.

Repo: **sase-core** (linked repo). Open it with `sase repo open sase-core -r "<why>"`,
read its `AGENTS.md`, and work only in the printed path. Every phase lands as a
non-breaking Conventional Commit (`feat(core): ...` or `refactor(core): ...`, never
`feat!` and never a `BREAKING CHANGE:` footer). Do not edit any `version`,
`CHANGELOG.md`, `*_SCHEMA_VERSION` constant, `FLEET_PROTOCOL_VERSION`, SQLite column or
index name, or golden contract. Current values that must stay put:
`AGENT_SCAN_WIRE_SCHEMA_VERSION` 10, `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` 33,
`PROC_WIRE_SCHEMA_VERSION` 3, `FLEET_CONTRACT_SCHEMA_VERSION` 6,
`FLEET_PROTOCOL_VERSION` 2 (both `fleet_contract/error.rs` and
`sase_gateway/src/wire.rs`), `RUNNER_CAPACITY_POLICY_SCHEMA_VERSION` 6,
`AGENT_HOLD_WIRE_SCHEMA_VERSION` 2, `LAUNCH_PLAN_WIRE_SCHEMA_VERSION` 2,
`PROC_DISPATCH_WIRE_SCHEMA_VERSION` 1.

Do not close `sase-1ab` or `sase-1ab.1`. Phase workers record `PROPOSED FOLLOW-UP:`
notes on their own phase bead and do not create beads. The land agent of this child epic
closes `sase-1ab.1` after every phase has landed.

Phases are sequential. `python_wire_parity.rs`, `lib.rs`, and `prelude.rs` are shared,
and later phases rename call sites that earlier phases have already moved to the new
type names.

### Scale

The concept is concentrated, not a repo-wide substring. The files with the most hits are
`fleet_agent_session.rs`, `procs/store.rs`, `agent_scan/index/tests/lineage.rs`,
`agent_scan/wire.rs`, `agent_scan/scanner.rs`, `fleet_owner_facts.rs`,
`fleet_catalog.rs`, `agent_scan/index/storage.rs`, `agent_hold.rs`, and
`agent_launch/plan_resolution.rs`. `turn` is a substring of `return`. Never
blind-replace. Search with a token or with `\bshells?\b`.

### Shared rules for every phase

1. **Rust names.** Follow the parent identifier rules. `shell` inside this concept
   becomes `turn`; a proc-store "shell" becomes named-proc vocabulary (`shell_name` →
   `proc_name`, `shell_kind` on a proc record → `proc_role`, `proc-shell` lifecycle →
   the named-proc constant below). Qualify turn names (`agent_session_turn`,
   `gate_turn`, `turn_kind`). Do not rename `turn_nonce`, provider `*Turn*` types, or
   `num_turns` if any appear. Do not rename `Proc.kind` (`command` / `tui` / `detached`)
   or proc-role values `proc`, `gate`, and `service`. Rename comments, docstrings, error
   messages, log messages, test names, and helpers along with the code, except emitted
   strings covered by rule 2. Do not keep Rust-side `pub use` aliases for old Rust
   names. Update callers in the same phase. Do not add new root exports in
   `crates/sase_core/src/lib.rs` or new `core_*` aliases in
   `crates/sase_core_py/src/prelude.rs`. Rename an existing entry in place. Prefer
   module-path imports in code you touch. Touch only the `lib.rs` and `prelude.rs` lines
   whose names you rename. Do not reformat either file.

2. **Serialized output stays byte-identical.** Every renamed serialized field or enum
   variant gets `#[serde(rename = "<legacy>", alias = "<new>")]` and keeps any alias it
   already has. It emits the legacy spelling and accepts both.
   `#[serde(deny_unknown_fields)]` wires (runner capacity, hold candidates, fleet
   locators) need the alias so a later Python writer can send the new key. Enums that
   use `rename_all = "snake_case"` need an explicit rename on the variant, for example
   `#[serde(rename = "agent_shell", alias = "agent_turn")]`. Hand-written emitters
   (`as_str`, `format!`, `safe_identifier` prefixes, diagnostic codes, conflict `field`
   strings) keep emitting the legacy spelling. Put that spelling in a named constant
   with a `// legacy sase-shell spelling; flips in contract-flip` comment. The matching
   parser accepts the new spelling too.

3. **Two different `shell_kind` fields.** On agent metadata, `shell_kind` becomes
   `turn_kind`. Its legacy value `proc` means a monitor: deserializing
   `turn_kind`/`shell_kind` value `monitor` stores `proc`, so a round trip still emits
   `proc`. `gate` stays `gate`. Comparison helpers treat `proc` and `monitor` as the
   same monitor kind. On a proc record, `shell_kind` becomes `proc_role` and the values
   `proc`, `gate`, and `service` stay as stored and as emitted. Do not map proc-role
   `proc` to `monitor`.

4. **Three spellings for the member object.** `AgentMetaWire` and `DoneMarkerWire`
   already deserialize `agent_session_shell` with `alias = "family_shell"`. After the
   rename the Rust field is `agent_session_turn` and serde is
   `rename = "agent_session_shell", alias = "agent_session_turn", alias = "family_shell"`.
   Hand-read JSON in `scanner.rs` (`agent_session_shell_from_object`, around the
   `agent_session_shell` / `family_shell` lookup, and `data.get("shell_kind")`) reads
   `agent_session_turn`, then `agent_session_shell`, then `family_shell`, and
   `turn_kind` then `shell_kind`, through one helper per key family. Prefer the new key
   when both are present. Do not drop the `family_shell` fallback.

5. **pyo3.** Only two bindings gain a second name. PyO3 allows one `#[pyo3(name)]` per
   function, so write the new `#[pyfunction]` and a thin legacy wrapper that calls the
   same private body. Comment the wrapper
   `// legacy binding name; removed in contract-flip`. Do not use `macro_rules!`.
   Register both in `register_<domain>`. Rename the existing `core_*` prelude alias to
   the new core function. Returned dict keys stay legacy. The pairs are
   `find_gate_turn_by_gate_id` / `find_gate_shell_by_gate_id` and
   `validate_standalone_named_proc_name` / `validate_standalone_proc_shell_name`.

6. **Strings sase already asserts.** Before changing a diagnostic code, conflict
   `field`, error message, or other string that crosses into Python, `git grep` it in
   the sase workspace. If a test or `require_rust_binding` name matches it, keep the
   legacy string. sase must keep passing with no source changes. Record a
   `PROPOSED FOLLOW-UP:` for `contract-flip` or `wire-cutover` naming the string and the
   file.

7. **Unrelated shell.** Leave Unix process execution, shell completion, and UI-chrome
   names alone. In this repo that includes sudo `"shell": false` fixtures and any
   command-execution helper that is not the session-member concept.
   `crates/sase_core/src/tool_run/triage/tests/mod.rs` mentions
   `src/sase/gate_shell/handoff.py`; that path still exists in sase. Leave it.
   `service/status.rs` shell hits are out of scope unless a hit is this concept.

8. **Tests.** Rename shell-named tests and helpers. For each renamed input, add a test
   that the new spelling and the legacy spelling deserialize or parse to the same value,
   and that serialization still emits the legacy JSON. A `serde_json::to_string` or
   `to_value` equality against the legacy key is enough. Keep
   `crates/sase_core/tests/python_wire_parity.rs` key order and JSON fixtures. Update
   only Rust field names in that file (`agent_session_shell`, `shell_name`,
   `shell_kind`) as those fields rename. Keep
   `crates/sase_gateway/contracts/api_fleet_v1/fleet_api_v1.json` byte-identical,
   including its `historical_shell` description. Do not set `UPDATE_FLEET_CONTRACT` or
   `UPDATE_MOBILE_CONTRACT`.

9. **Verify.** Batch edits. Iterate with `just fast` and
   `just test -p <crate> <filter>`. Never run bare `cargo`. Pass `sase tool run check`
   in sase-core before finishing the phase. It takes about 5 minutes; give the tool a
   timeout of 15 minutes or more. Never run `just check-full`.

10. **Notes.** On the phase bead, record the remaining concept hits in the files you
    own, grouped as unrelated meaning, legacy spelling pinned for `contract-flip`, or
    legacy binding name. Record later-phase work as `PROPOSED FOLLOW-UP:` notes. Do not
    create beads.

## Phase scan-wires: Agent-scan wires and gate lookup

Scope (sase-core):

- `crates/sase_core/src/agent_scan/wire.rs`: `AgentSessionShellWire` →
  `AgentSessionTurnWire`, `AgentSessionShellMonitorWire` →
  `AgentSessionTurnMonitorWire`, `AgentSessionShellGateWire` →
  `AgentSessionTurnGateWire`. On `AgentMetaWire` and `DoneMarkerWire`, rename
  `agent_session_shell` per shared rule 4. On `AgentMetaWire` only, rename `shell_kind`
  to `turn_kind` per shared rules 2 and 3 (`DoneMarkerWire` has no `shell_kind` today).
  Do not bump `AGENT_SCAN_WIRE_SCHEMA_VERSION`.
- `agent_scan/scanner.rs`: `agent_session_shell_from_object` →
  `agent_session_turn_from_object`, reading the three object keys and the two kind keys
  through the helpers in shared rule 4. Apply the `monitor` → stored `proc` mapping only
  for this metadata kind. Update the struct literals and the tests around
  `shell_kind: "gate"`.
- `agent_scan/index/`: rename `find_gate_shell_by_gate_id` →
  `find_gate_turn_by_gate_id`, `gate_shell_id_from_record` → `gate_turn_id_from_record`,
  `RecordSummary.gate_shell_id` → `gate_turn_id`, and `LAST_GATE_SHELL_LOOKUP_*` →
  `LAST_GATE_TURN_LOOKUP_*`. The SQL column and index stay `gate_shell_id` /
  `idx_agent_artifacts_gate_shell_id`, including `storage.rs`, `maintenance.rs`,
  `query.rs`, and the v31 migration. Put the column literal in one constant,
  `GATE_TURN_INDEX_COLUMN: &str = "gate_shell_id"`, with the shared-rule-2 comment. Do
  not bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION`. Rename the tests in
  `index/tests/lineage.rs` (`find_gate_shell_by_gate_id_*`,
  `schema_v30_upgrade_adds_and_backfills_gate_shell_id_projection`). The SQL strings
  inside those tests stay.
- Call sites of the renamed types and of the `agent_session_shell` field must compile in
  this phase: `agent_runtime.rs` (`is_real_monitor_member_record`,
  `is_real_gate_member_record`, `is_runner_slot_occupying_record`),
  `agent_stats/runner.rs`, `fleet_agent_session.rs`, `fleet_owner_facts.rs`,
  `fleet_catalog.rs`, `lib.rs` re-exports, and the `agent_session_shell` identifier in
  `python_wire_parity.rs`. In those files, rename only the type and the field. Leave
  fleet variants, `shell_id`, proc fields, and function names such as
  `concrete_agent_session_shell_kind` for later phases.
- Bindings: `crates/sase_core_py/src/agent_scan/mod.rs` and the
  `find_gate_shell_by_gate_id` prelude alias, per shared rule 5. Add a round-trip test
  that both Python names return the same value.
- Tests: dual-spelling input for the member object (three keys) and for metadata
  `turn_kind` (`shell_kind: "proc"`, `turn_kind: "monitor"`, and `shell_kind: "gate"`),
  each asserting the serialized JSON still uses `agent_session_shell` and `shell_kind`
  with value `proc` or `gate`.

Exit: `sase tool run check` in sase-core.

## Phase procs: Named-proc store, launch, and holds

Scope (sase-core). Depends on scan-wires so the member-wire renames are already in the
tree.

- `crates/sase_core/src/procs/wire.rs` and `procs/store.rs`:
  - `shell_name` → `proc_name` and proc-record `shell_kind` → `proc_role` on `ProcWire`,
    `XpromptProcMetaWire`, `ProcReserveWire`, and `ProcUpdateWire`, pinned per shared
    rule 2. `default_shell_kind` → `default_proc_role`; its value stays `Some("proc")`.
  - `PROC_SHELL_LIFECYCLE` and `default_origin` stay emitted as `"proc-shell"`. Name the
    constant `PROC_LIFECYCLE_NAMED_PROC` with that legacy value and the shared-rule-2
    comment. `PROC_LIFECYCLES` keeps `"legacy"` and `"proc-shell"` and also accepts
    `"named-proc"`. `is_proc_shell` → `is_named_proc`, true for both spellings.
    `ensure_proc_shell` → `ensure_named_proc`. `ValidationMode::ProcShellWrite` →
    `NamedProcWrite`.
  - `normalized_conflict_keys` still builds `shell:{project}:{name}` from the name.
    Comparison treats a `named-proc:` prefix as equal to `shell:` for the same project
    and name, including a live row stored with the legacy key against a request that
    sends the new prefix. One named helper, with a test for both directions. The
    conflict error `field` stays `"shell_name"`.
  - Rewrite store messages that say "proc-shell" or "shell name" only when shared rule 6
    says sase does not assert them.
- `agent_launch/`: `ProcUnitWire.shell_name` and `Prepared`/`proc_runtime.rs`
  `shell_name` → `proc_name`, pinned. `validate_standalone_proc_shell_name` →
  `validate_standalone_named_proc_name`, `validate_proc_shell_name` →
  `validate_named_proc_name`, `is_valid_proc_shell_name` → `is_valid_named_proc_name`.
  The diagnostic code stays `"invalid-proc-shell-name"` (shared rule 6; `contract-flip`
  renames it to `invalid-named-proc-name`). Update `plan_resolution.rs`,
  `typed_units.rs`, `condition.rs`, `launch_hold.rs`, `identity.rs`, `admission.rs`, and
  `tests/typed_plan.rs` field uses.
- `agent_hold.rs`: `AgentHoldCandidateWire.proc_shell` → `named_proc` with
  `rename = "proc_shell", alias = "named_proc"`. `deny_unknown_fields` stays. The
  emitted selector-match kind stays `"proc_shell"`; a reader that compares match kinds
  also accepts `"named_proc"`. Update `runner_capacity/holds.rs`, which builds
  `proc_shell: None`. Rename
  `proc_shell_matches_name_and_hood_selectors_without_agent_session_kin` with the
  behavior unchanged. Do not bump `AGENT_HOLD_WIRE_SCHEMA_VERSION`.
- `python_wire_parity.rs`: update the Rust fields `shell_name` / `shell_kind` only. The
  JSON fixtures keep `"lifecycle": "proc-shell"`, `"shell_name"`, and `"shell_kind"`.
- Bindings: `crates/sase_core_py/src/agent_launch/mod.rs` and the prelude alias, per
  shared rule 5. `procs/mod.rs` and `procs/tests.rs` keep sending the legacy keys and
  gain one test that a `proc_name` / `named-proc` payload validates the same as the
  legacy payload. Update binding code that names the renamed core items.
- Do not bump `PROC_WIRE_SCHEMA_VERSION`, `LAUNCH_PLAN_WIRE_SCHEMA_VERSION`, or
  `PROC_DISPATCH_WIRE_SCHEMA_VERSION`.

Exit: `sase tool run check` in sase-core.

## Phase fleet-runtime: Fleet, runner capacity, and gateway

Scope (sase-core). Depends on procs so proc fields and hold candidates already use the
new Rust names.

- `fleet_agent_session.rs`: `ConcreteAgentSessionShellKind` →
  `ConcreteAgentSessionTurnKind`. Variants `Proc`, `Monitor`, `Gate`, `Plan`, `Code`,
  and `Member` keep their names. `concrete_agent_session_shell_kind` →
  `concrete_agent_session_turn_kind`, `record_is_concrete_agent_session_shell` →
  `record_is_concrete_agent_session_turn`, `agent_session_shell` → `agent_session_turn`.
  The metadata kind helper treats stored `proc` and input `monitor` as the monitor kind
  (shared rule 3). `lib.rs` re-exports of these names are renamed in place.
- `fleet_contract/status.rs`: `FleetRowKindWire::AgentShell` → `AgentTurn` and
  `HistoricalShell` → `HistoricalTurn`, with `rename = "agent_shell"` /
  `"historical_shell"` and aliases `agent_turn` / `historical_turn`. `default_row_kind`
  follows. `FleetAgentSessionRoleWire::HistoricalShell` → `HistoricalTurn` the same way.
  Role variants `Root`, `Member`, `Monitor`, `Gate`, and `Proc` stay. Update match sites
  in `projection.rs`, `resolution.rs`, `reads.rs`, `fleet_catalog.rs`
  (`row_kind_for_record`), and `fleet_mutation.rs`.
- Locators (`fleet_contract/locators.rs`): `AgentInstanceLocatorWire.shell_id` →
  `turn_id` with `rename = "shell_id", alias = "turn_id"`.
  `validate_identifier("shell_id", ...)` may keep naming the serialized field.
  `instance_key_unchecked` keeps emitting the `shell` segment. Add one helper, next to
  `logical_key_matches`, that treats a stored `turn` segment as equal to the emitted
  `shell` segment, and use it wherever a stored instance key is compared.
  `safe_identifier(..., "shell")` in `fleet_catalog.rs` keeps emitting `shell-<hex>`.
  Where those ids are compared, `turn-<hex>` and `shell-<hex>` with the same digest are
  equal. One helper, with tests. Do not bump `FLEET_CONTRACT_SCHEMA_VERSION` or
  `FLEET_PROTOCOL_VERSION`.
- `fleet_owner_facts.rs`: `apply_shell_facts` → `apply_turn_facts`.
  `OwnerPresentationFactsWire.shell_start_status` / `shell_stop_status` →
  `turn_start_status` / `turn_stop_status`, pinned to the legacy key names.
  `string_fields_mut` must still list those two fields. The wire is part of the fleet
  contract; emitted keys stay `shell_start_status` and `shell_stop_status`.
- `runner_capacity/wire.rs`: `agent_session_shell_kind` / `_id` / `_state` →
  `agent_session_turn_kind` / `_id` / `_state`, each `rename`d to the current key and
  aliased to the new key. `deny_unknown_fields` stays. Do not bump
  `RUNNER_CAPACITY_POLICY_SCHEMA_VERSION`. Update `records.rs` and the tests under
  `runner_capacity/tests/` that set those fields. The test in `tests/wire.rs` that
  asserts `encoded["agent_session_shell_kind"]` and that `family_shell_kind` is absent
  must still pass without changing the expected keys.
- `agent_runtime.rs`: if scan-wires left a "gate-shell" comment, rename the comment to
  gate turn. Do not change occupancy behavior.
- `crates/sase_gateway/src/fleet_reads/` (`content.rs`, `resolution.rs`,
  `tests/presentation.rs`) and `federation_worker/imp/tests/operations.rs`: rename Rust
  identifiers. `contract.rs` and `contracts/api_fleet_v1/fleet_api_v1.json` stay
  byte-identical. Confirm with `just test -p sase_gateway committed_` and no `UPDATE_*`
  variable.
- `crates/sase_core_py/src/fleet/tests.rs` builds `"shell_id"` and
  `"row_kind": "agent_shell"`. Those JSON keys and values stay. Update only Rust names
  around them. If the test starts sending the new keys, it must also still assert that
  the response uses the legacy keys.
- Tests: dual input for `agent_shell`/`agent_turn`,
  `historical_shell`/`historical_turn`, `shell_id`/`turn_id`, the instance-key segment,
  the `shell-`/`turn-` fallback id, owner status keys, and the three runner-slot keys.
  Each asserts the serialized form is the legacy spelling.

Exit: `sase tool run check` in sase-core.

## Phase sweep: Editor text, classification, and cross-repo check

Depends on fleet-runtime.

- Editor text, still in sase-core: `crates/sase_core/src/editor/wire.rs` snippets
  `%wait(proc=${1:proc-id-or-shell-name})` and the "Wait for a prompt-owned proc by ID
  or shell name." string, and `editor/directive/metadata.rs` "Wait for a proc ID or
  shell name", say proc name instead (`proc-id-or-proc-name`, "Wait for a proc ID or
  proc name"). Update the core tests that snapshot those strings. This is display text,
  not a wire key. If `sase tool run check` in the sase workspace then fails on that
  string, keep the core text change only when the sase assertion can stay untouched;
  otherwise leave the snippet legacy and record a `PROPOSED FOLLOW-UP:` for
  `runtime-cutover`.
- Classify every remaining concept hit. Run
  `git grep -n -E 'shell|Shell|SHELL' -- crates ':!*CHANGELOG*'` and
  `git grep -n -E 'proc-shell|proc_shell|ProcShell' -- crates ':!*CHANGELOG*'`. Classify
  each as unrelated (Unix shell, completion, sudo `shell: false`, the sase `gate_shell`
  path fixture), a legacy serialized spelling or constant pinned for `contract-flip`, a
  legacy binding name, or a legacy-input fixture. Fix small stragglers (identifiers,
  comments, messages, test names) in place. Do not flip a pinned legacy spelling.
- Confirm no `*_SCHEMA_VERSION`, `FLEET_PROTOCOL_VERSION`, SQLite column,
  `fleet_api_v1.json` byte, or `python_wire_parity.rs` JSON key changed (`git diff`
  against the commit before `scan-wires`, restricted to those items).
- Confirm both new binding names and both legacy names are registered. After
  `just install` in the sase workspace, import `sase_core_rs` and read all four
  attributes.
- Cross-repo check: in the sase workspace, `just install` against this core, then
  `sase tool run check`. It must pass with no sase source changes. Give `just install`
  and the check long timeouts (`just install` can take several minutes; the check is the
  full sase gate). Do not run `just check-full`.
- `sase tool run check` in sase-core.
- On this phase bead, record `PROPOSED FOLLOW-UP:` notes listing every legacy serialized
  spelling, constant, and binding that `contract-flip` must flip or remove, by file, and
  every sase reader that `wire-cutover` must switch to the new binding names
  (`find_gate_shell_by_gate_id` in `src/sase/core/agent_scan_facade.py` is one).

Exit: every remaining shell hit is classified, and `sase tool run check` passes in
sase-core and in the sase workspace with no sase source changes.
