---
tier: epic
title: '%queue capacity multiplier (<M>x)'
goal: 'The `%queue` / `%q` capacity argument accepts a multiplier `<M>x` (at most
  two decimal places). Admission resolves it as M times the machine''s effective `max_running_agents`
  budget. Every `#research_swarm` agent authors `%q(1.5x, w=0.25)`, so on a machine
  whose effective budget is 5 each research agent gets a capacity of 7.5.

  '
phases:
- id: core-parse
  title: Rust multiplier syntax, formatting, launch wires, and editor metadata
  depends_on: []
  size: medium
  description: 'core-parse: in sase-core, parse and validate `<M>x` wherever `%queue`
    accepts capacity, carry it as `queue_capacity_multiplier` through QueueFieldsWire
    and the typed agent/proc launch wires, format it canonically, add helper bindings,
    and update the completion metadata.

    '
- id: core-admission
  title: Rust admission resolution, scan records, and fleet contract
  depends_on:
  - core-parse
  size: medium
  description: 'core-admission: in sase-core, resolve a persisted multiplier against
    the request''s effective_limit when computing the admission limit, add the field
    to runner-capacity records and waiters, agent-scan meta and waiting wires (with
    an index schema bump), and the fleet summary contract, and extend the normalize
    binding.

    '
- id: sase-plumbing
  title: sase launch, persistence, admission, and continuation plumbing
  depends_on:
  - core-admission
  size: medium
  description: 'sase-plumbing: bump the sase-core pin. Then carry `queue_capacity_multiplier`
    from directive extraction through agent_meta and waiting markers, runner-slot
    admission records, typed-unit and proc admission, and continuation reauthoring.
    Update the xprompt and runner-slot docs.

    '
- id: sase-surfaces
  title: TUI and CLI display plus capacity editing surfaces
  depends_on:
  - sase-plumbing
  size: medium
  description: 'sase-surfaces: render multiplier capacities as `c1.5x` badges with
    resolved units in the TUI detail, wait lane, and queue ladder, and expose them
    in agent-list JSON. The wait modal and the agent directive edit commands accept
    `<M>x` and keep it when other queue fields are edited.

    '
- id: research-swarm
  title: Research swarm authors a 1.5x capacity multiplier
  depends_on:
  - sase-plumbing
  size: small
  description: 'research-swarm: in sase-research-artifacts, render `%q(1.5x, w=0.25)`
    in every swarm segment, with the optional `runners` input replacing the multiplier
    with an absolute budget. Update the tests, wheel and publish smokes, docs, and
    dependency floors where a published core supports it.'
proposed_by: bbugyi200.apollo.1o
create_time: 2026-09-25 12:24:30
status: wip
bead_id: sase-19f
---

- **PROMPT:** [prompts/202609/queue_capacity_multiplier.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/queue_capacity_multiplier.md)
- **BEAD:** [sase-19f](https://github.com/sase-org/sase--beads/blob/main/pages/sase-19f/README.md)

# Plan: `%queue` capacity multiplier (`<M>x`)

## Background

- Today `%queue` capacity must be a positive integer at every layer. In sase-core,
  `crates/sase_core/src/queue_directive.rs` defines
  `QueueFieldsWire.queue_capacity: Option<u32>`, and `parse_capacity` accepts ASCII
  digits only. sase persists the value as `agent_meta.queue_capacity` plus
  `queue_capacity_explicit: true`, and the waiting marker carries the same keys.
- The admission limit is computed at exactly one place:
  `crates/sase_core/src/runner_capacity/waiters.rs::waiter_admission_limit`. It calls
  `normalize_persisted_queue_capacity(capacity, explicit, weight, request.effective_limit, capacity_budget)`,
  and the candidate path inherits the result.
  `RunnerCapacityRequestWire.effective_limit` is an `f64` supplied by Python as
  `float(get_max_running_agents())`. Its callers include
  `src/sase/axe/run_agent_wait_slots.py`, `src/sase/agent/proc_capacity_admission.py`,
  the TUI loaders, and `src/sase/integrations/agent_list_entries.py`.
  `get_max_running_agents()` (`src/sase/config/core.py`) returns an active
  `~/.sase/max_running_agents_override.json` limit when one exists, and the configured
  `max_running_agents` otherwise. On the requesting user's machine the configured value
  is 10, but a non-expiring ACE override sets 5. The "configured capacity of 5" is
  therefore this effective value.
- The `queue_capacity_budget` sunset flag defaults to on. When it is on, an explicit
  capacity replaces the global limit as the launch's admission limit. When it is off
  (the legacy branch), an integer capacity is only an occupied-load threshold and the
  admission limit stays global.
- Rust core boundary: parsing, validation, formatting, persisted-value precedence, and
  resolution are shared backend behavior and belong in sase-core. sase calls them
  through `sase_core_rs` bindings.

## Design Decisions

1. **The multiplier is a separate field, resolved at admission. Capacity does not become
   a float.** A new optional key, `queue_capacity_multiplier` (a JSON number such as
   `1.5`), lives alongside the unchanged integer `queue_capacity`. The only place that
   knows the machine's budget is admission, where `effective_limit` is already an `f64`.
   Resolving there has several advantages:
   - A multiplier tracks the live budget, including override changes while an agent is
     queued.
   - Reauthoring (continuations, dispatch prompt rebuilds, TUI edits) keeps the authored
     `1.5x` intent.
   - No persisted integer changes type.
   - Older readers that ignore the new key fall back to the global budget.

   Alternative: resolve at launch into a fractional capacity. It was rejected because it
   would turn every `u32`/`int` capacity path in both repos into a float, freeze the
   launch-time budget, and lose the multiplier on reauthoring.

2. **Syntax.** `<M>x` is accepted wherever capacity is: `%q:1.5x`, `%q(1.5x, ...)`,
   `%q(capacity=1.5x)`, and the `%queue` spellings.
   - `M` is either one or more digits with an optional `.` and 1–2 fractional digits, or
     `.` followed by 1–2 digits. It is followed by a lowercase `x`.
   - Accepted examples: `1x`, `2x`, `0.5x`, `.5x`, `1.25x`, `1.50x`.
   - Rejected with an actionable message:
     - more than two decimal places (`1.125x`); the message states the two-decimal limit
     - zero (`0x`, `0.00x`), in both flag states
     - signs, exponents, `inf`/`nan`, uppercase `X`, a bare `x`, and `1.x`
     - a value whose hundredths overflow `u32`
   - New error codes are `invalid-queue-capacity-multiplier` and
     `queue-overflow-capacity-multiplier`. The existing non-integer capacity message
     also points to the `<M>x` form (for example, when the user writes `1.5` without
     `x`).
3. **Capacity is still one field.** Integer and multiplier forms are mutually exclusive.
   Supplying both, whether in one occurrence or across occurrences, raises the existing
   duplicate-`capacity` error.
4. **Weight check.** The parse-time `queue-weight-exceeds-capacity` rejection applies
   only to integer capacities. A multiplier cannot be checked when the prompt is parsed.
   If `weight > M × budget`, admission's existing `weight-exceeds-limit` blocker keeps
   the launch QUEUED, and it is admitted if the budget later grows.
5. **Precision.** The multiplier is held internally as integer hundredths and stored as
   `hundredths / 100` in JSON.
   - Resolution: `resolved = round_2dp(hundredths × effective_limit / 100)`. For
     example, `1.5x` × 5 → 7.5 and `1.15x` × 3 → 3.45, with no float noise.
   - Canonical formatting trims trailing zeros: `1.5x`, `2x`, `0.25x`, `1.05x`.
   - Resolved capacities display with at most two decimals (`7.5`, not `7.50`).
6. **Persistence precedence.**
   - Writers set exactly one form and remove the other key:
     - an integer: `queue_capacity` plus `queue_capacity_explicit: true`
     - a multiplier: `queue_capacity_multiplier` only; its presence means explicit, and
       writers do not set `queue_capacity_explicit` for it
   - Readers take the capacity/multiplier pair from the waiting marker when it carries
     either key, and from `agent_meta` otherwise. This mirrors today's waiting-over-meta
     rule.
   - If both keys are present in one source, the integer wins, because only an older
     integer-only writer could have left both.
   - An invalid persisted multiplier (not a number, a bool, ≤ 0, NaN, infinite, or more
     than two decimals after rounding tolerance) is ignored, exactly like an invalid
     integer capacity.
7. **Flag behavior.**
   - `queue_capacity_budget` **on**: the resolved multiplier replaces the global limit
     as this launch's admission limit, exactly like an integer capacity.
   - **Off** (legacy): the multiplier still parses, persists, and reauthors, but has no
     admission effect. The admission limit stays global, and the integer threshold gate
     ignores the multiplier.
   - Tests cover both states.
   - No new feature flag is needed. `<M>x` was previously a parse error, so no existing
     behavior changes, and each landed phase leaves a coherent state: the syntax works
     end to end once `sase-plumbing` lands, and `sase-surfaces` only adds display and
     editing.
8. **Display.**
   - Row badge: `c1.5x`, with the existing over-limit style when `M > 1`.
   - Detail header: `Capacity: 1.5x budget (7.5 capacity units)`, or `1.5x budget` when
     no effective limit is known.
   - The wait lane and the queue ladder use the same two strings.
9. **Out of scope** (these stay integer-only):
   - plan-approval option capacity (`approve_options_modal.py`, `_plan_gate_shared.py`)
   - `sase bead work` capacity flags (`bead/work_queue_capacity.py`)
   - lumberjack/chop `wait_runners` config

## Phase `core-parse`: Rust multiplier syntax, formatting, launch wires, and editor metadata

Work in sase-core. Open it with `sase repo open sase-core -r "<reason>"`, read its
`AGENTS.md`, and follow its conventions:

- free functions over `*Wire` structs
- no `macro_rules!`
- import by module path; do not add root `pub use` names
- never edit versions or changelogs

Keep every existing public Rust and binding signature source-compatible and make the
commit an additive `feat(...)`, not a breaking change.

1. `crates/sase_core/src/queue_directive.rs`
   - Add `queue_capacity_multiplier: Option<f64>` to `QueueFieldsWire` with
     `#[serde(default, alias = "capacity_multiplier", skip_serializing_if = "Option::is_none")]`.
   - Add an internal capacity-value enum (absolute `u32`, or multiplier hundredths
     `u32`). The capacity parse path detects a trailing `x` and routes it to a
     multiplier parser that implements Design Decision 2.
   - `parse_colon_occurrence`, `assign_capacity`, and `merge_queue_part` treat either
     form as the single capacity field and raise the existing duplicate error.
   - The parenthesized empty-queue check counts the multiplier.
   - `validate_queue_budget_fields` skips the weight check when the capacity is a
     multiplier.
   - `format_queue_directive` writes `capacity=<M>x` in the capacity slot.
   - New public helpers:
     - `queue_capacity_multiplier_is_valid(f64) -> bool`: finite, > 0, within two
       decimals under a small tolerance, and hundredths ≤ `u32::MAX`
     - `format_queue_capacity_multiplier(f64) -> Option<String>`: `"1.5x"`
     - `resolve_queue_capacity_multiplier(multiplier: f64, effective_limit: f64) -> Option<f64>`:
       the rounded two-decimal product, or `None` for invalid inputs
     - `parse_queue_capacity_value[_with_flags](raw, flags) -> Result<QueueCapacityValueWire, QueueParseErrorWire>`,
       where
       `QueueCapacityValueWire { queue_capacity: Option<u32>, queue_capacity_multiplier: Option<f64> }`
       lets edit surfaces validate either form
   - Leave `parse_queue_capacity` integer-only.
   - Update the integer-capacity error text to mention `<M>x`.
2. `crates/sase_core/src/agent_launch/`
   - `wires.rs`: `AgentUnitWire` and `ProcUnitWire` gain
     `queue_capacity_multiplier: Option<f64>` (serde default/skip), and
     `has_authored_queue_fields` counts it.
   - `typed_units.rs`: copy it into agent and proc units. The rule that gives a
     no-weight proc a weight of `0.0` treats a multiplier like a capacity.
   - `admission.rs::agent_unit_dispatch_prompt_with_flags`: rebuild `%queue(...)`
     including the multiplier.
   - `plan_resolution.rs::proc_queue_preview`: render `capacity=1.5x`.
3. Editor metadata (`crates/sase_core/src/editor/directive/metadata.rs`)
   - `QUEUE_DIRECTIVE_ON`'s `argument_hint`, description, and `capacity` keyword
     description mention `<M>x` ("multiplier of this machine's max_running_agents
     budget").
   - Add a `1.5x` suggested value ("1.5× this machine's effective max_running_agents
     budget") to `WAIT_CAPACITY_BUDGET_SUGGESTIONS`. The `QUEUE_DIRECTIVE_OFF` metadata
     is unchanged.
   - Keep `DirectiveValueRole::PositiveInt`: the role is descriptive only and no
     validator reads it.
   - Update `editor/directive/tests.rs`,
     `crates/sase_xprompt_lsp/src/server/tests/completion.rs` (flag-on expectations gain
     `1.5x`), and `crates/sase_core_py/src/editor_completion/tests/surfaces.rs` as
     needed.
4. Bindings (`crates/sase_core_py/src/agent_launch/mod.rs`)
   - Register `parse_queue_capacity_value(raw, enabled_feature_flags=None) -> dict`,
     `format_queue_capacity_multiplier(value) -> Optional[str]`, and
     `resolve_queue_capacity_multiplier(multiplier, effective_limit) -> Optional[float]`.
   - Confirm that `collect_queue_fields` and `format_queue_directive` round-trip the new
     field.
5. Tests. Cover:
   - accept, reject, and canonical format for every Design Decision 2 case, in both flag
     states
   - colon, positional, and keyword forms
   - duplicates between the integer and multiplier forms, within one occurrence and
     across occurrences
   - the weight-check skip
   - typed-plan agent and proc units carrying the multiplier
   - the dispatch-prompt rebuild and the proc preview
   - the binding shapes
6. Verify with `sase tool run check` in sase-core (about 5 minutes; use a tool timeout
   of at least 10 minutes).

## Phase `core-admission`: Rust admission resolution, scan records, and fleet contract

Work in sase-core, opened the same way.

1. `queue_directive.rs`
   - Add
     `normalize_persisted_queue_capacity_with_multiplier(queue_capacity, queue_capacity_explicit, queue_capacity_multiplier, effective_weight, global_limit, capacity_budget)`,
     and make the existing function delegate to it with `None`.
   - `PersistedQueueCapacityNormWire` gains `authored_multiplier` and
     `reauthor_multiplier` (`Option<f64>`, skipped when `None`).
   - Flag on, with an explicit integer: unchanged, and the integer wins over a
     multiplier.
   - Flag on, with a multiplier only:
     `admission_limit = resolve_queue_capacity_multiplier(m, global_limit)` and
     `reauthor_multiplier = m`.
   - Flag off: `admission_limit = global_limit`, and the multiplier is still reported as
     `reauthor_multiplier` so continuations preserve the authored intent.
   - Add `queue_capacity_multiplier_from_map(&Map) -> Option<f64>`, which validates with
     `queue_capacity_multiplier_is_valid`.
2. `runner_capacity/`
   - `RunnerCapacityRecordWire` gains `queue_capacity_multiplier: Option<f64>`. It must
     be declared because the struct is `deny_unknown_fields`.
   - `records.rs`: add `explicit_queue_capacity_multiplier(record)`, which returns
     `None` when an explicit integer capacity exists.
   - `waiters.rs::waiter_admission_limit`: use the new normalize function.
   - `RunnerCapacityWaiterWire` gains a display field `queue_capacity_multiplier`; its
     existing `admission_limit` then reports the resolved value, such as 7.5.
   - The legacy flag-off threshold path (`waiters.rs` and `capacity_math.rs` `u32`
     threshold) is unchanged.
   - Bump `RUNNER_CAPACITY_POLICY_SCHEMA_VERSION` only if this module's compatibility
     rules require it for an additive field. If it bumps, update the tests that pin 5.
     The sase and plugin smokes only assert `>= 4`.
3. `agent_scan/`
   - `AgentMetaWire` and `WaitingMarkerWire` gain `queue_capacity_multiplier`.
     `scanner.rs` reads it at both the agent_meta and waiting read sites.
   - Bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` (currently 31) so cached `record_json`
     rows refresh. Follow the v29 `queue_capacity` precedent in
     `agent_scan/index/storage.rs`.
4. Fleet contract
   - `fleet_contract/resolution.rs::ResolvedAgentSummaryWire` gains
     `queue_capacity_multiplier: Option<f64>`.
   - `projection.rs` applies the Design Decision 6 precedence next to
     `queue_capacity_for_record`.
   - Handle `FLEET_CONTRACT_SCHEMA_VERSION` according to this module's compatibility
     rules for additive fields.
   - Add the key to `crates/sase_gateway/src/contract.rs` and regenerate
     `contracts/api_fleet_v1/fleet_api_v1.json` with `UPDATE_FLEET_CONTRACT=1`.
5. Bindings
   - The `normalize_persisted_queue_capacity` binding gains an optional keyword,
     `queue_capacity_multiplier=None`. Existing positional callers are unaffected.
   - `runner_capacity_snapshot` accepts the new record and candidate field.
6. Tests. Cover:
   - `1.5` × `effective_limit` 5 → `admission_limit` 7.5, admitting only while occupied
     load plus weight ≤ 7.5
   - `0.5x` × 5 → 2.5
   - `1.15x` × 3 → 3.45
   - integer wins when both are present
   - the flag-off branch ignores the multiplier
   - the `weight-exceeds-limit` blocker when weight exceeds the resolved limit
   - scanner reads from meta and from the waiting marker, with marker precedence
   - the index version bump
   - fleet projection
   - the binding kwarg
7. Verify with `sase tool run check` in sase-core.

## Phase `sase-plumbing`: sase launch, persistence, admission, and continuation plumbing

Work in the sase repo. Read the "Changing sase-core from a sase workspace" and "The CI
source revision pin" sections of `docs/rust_backend.md`.

1. **Pin.** Bump `sase-core-revision.txt` to the pushed sase-core commit that contains
   both core phases, using `just ratchet-core-revision` or the SHA. Do not touch the
   `sase-core-rs` window in `pyproject.toml`; the release process owns it. Rebuild the
   local core so the new bindings are importable.
2. **Adapter** (`src/sase/xprompt/queue_directive.py`)
   - `format_queue_directive(..., capacity_multiplier: float | None = None)` passes
     `queue_capacity_multiplier`.
   - Add `resolve_authored_queue_capacity_multiplier(data) -> float | None`, following
     Design Decision 6 and delegating validity to the Rust helpers.
   - `reauthor_capacity_for_prefix` gains a multiplier input or companion and returns
     the binding's `reauthor_multiplier`.
   - Add thin wrappers for `parse_queue_capacity_value`,
     `format_queue_capacity_multiplier`, and `resolve_queue_capacity_multiplier`.
   - `validate_queue_capacity` stays integer-only for the out-of-scope callers.
3. **Extraction.**
   - `src/sase/xprompt/_directive_types.py` gains
     `queue_capacity_multiplier: float | None = None`.
   - `_directive_extract.py` reads `queue_capacity_multiplier` from the Rust fields and
     never applies `int(...)` to it.
4. **Persistence and runner.**
   - `src/sase/axe/run_agent_directive_metadata.py` writes
     `agent_meta["queue_capacity_multiplier"]` per Design Decision 6. Mirror any
     existing `queue_capacity` preserve/rewrite handling.
   - Carry the multiplier through all of:
     - `run_agent_directives.py` (`AgentInfo`)
     - `run_agent_runner.py`, into `wait_for_runner_slot(...)`
     - `run_agent_wait_slots.py`
     - `run_agent_wait_slot_candidate.py`
     - `run_agent_wait_markers.py` (waiting marker key)
     - `run_agent_wait_slot_state.py` (legacy flag-off state ignores it)
5. **Admission records.**
   - `src/sase/core/runner_slots/_admission_capacity_records.py` adds a record reader
     with the Design Decision 6 precedence and includes `queue_capacity_multiplier` in
     scan-projected and synthetic records.
   - `runner_slot_candidate_record(..., queue_capacity_multiplier=None)`.
   - `_admission.py` and `_admission_types.py`: `RunnerSlotWaiter` exposes the waiter's
     `queue_capacity_multiplier`.
6. **Wires.**
   - Python `AgentMetaWire` and `WaitingMarkerWire` in
     `src/sase/core/agent_scan_wire_markers.py`, plus `agent_scan_wire_conversion.py`.
   - Typed units: `agent_launch_wire_records.py` (`AgentUnitWire`, `ProcUnitWire`, and
     the authored-queue-fields property), `agent_launch_wire_from_dict.py`, and
     `agent_launch_wire_conversion.py`.
   - `src/sase/agent/launch_admission_store.py` and
     `src/sase/agent/proc_capacity_admission.py`.
7. **Continuations.**
   - `src/sase/monitor/continuation_delivery.py`: `queue_launch_prefix` emits
     `capacity=1.5x`, and `launch_wire_extra` carries the multiplier.
   - Update `continuation_admission.py` if it reads capacity.
8. **Docs.**
   - `docs/xprompt.md`:
     - the directive table and completion row, which now suggests `1.5x`
     - examples: add `%q:1.5x` and `%q(1.5x, w=0.25)`
     - the capacity semantics section: `<M>x` = M × effective `max_running_agents`,
       override-aware; two-decimal rule; worked example 1.5x × 5 = 7.5; re-resolved
       while queued; weight above the resolved budget stays QUEUED; no effect when the
       flag is off
   - `docs/troubleshooting/runner-slots.md`, and the `max_running_agents` section of
     `docs/configuration.md`: multipliers scale off the effective value.
9. **Memory.** Do not edit `sase/memory/xprompts.md`. Record a `PROPOSED FOLLOW-UP:`
   note on this phase bead: that note's `%queue` paragraph ("authored positive integer
   `N`") should mention `<M>x` multipliers.
10. **Tests.**
    - Parse, format, and extract: `tests/test_queue_directive.py`,
      `tests/test_directives_wait.py`, and `tests/test_agent_names_extract_metadata.py`.
    - Admission: add a runner-slot test in which a `queue_capacity_multiplier: 1.5`
      record with `effective_limit` 5 yields `admission_limit == 7.5`. Also cover
      `tests/test_capacity_snapshot_parity.py`,
      `tests/core/test_agent_launch_wire_contract.py`,
      `tests/test_core_agent_scan_wire_agent_meta.py`,
      `tests/monitor/test_continuation_delivery.py`, and
      `tests/test_launch_approval_queue_capacity.py`.
    - Completion parity expectations that now include `1.5x`:
      `tests/test_xprompt_directive_completion_parity.py`,
      `tests/_xprompt_directive_completion_parity_lsp.py`, and
      `tests/ace/tui/widgets/test_directive_arg_completion.py`.
11. Verify with `sase tool run check`.

## Phase `sase-surfaces`: TUI and CLI display plus capacity editing surfaces

Work in the sase repo. Read `sase/memory/tui.md` with `/sase_memory_read` before
changing the TUI. Use the Rust-backed wrappers from `sase-plumbing` for all formatting
and resolution.

1. **Model and loaders.**
   - `src/sase/ace/tui/models/_agent_state.py` gains `queue_capacity_multiplier`. The
     capacity setter keeps the integer/multiplier pair consistent: setting one clears
     the other.
   - Populate it in `_loaders/_meta_enrichment_wire.py`,
     `_meta_enrichment_filesystem.py`, `_meta_enrichment_identity.py`, and
     `_fleet_agents_rows.py` (fleet summary field).
   - Update `_dedup.py`, `_agent_clan_sections.py`, and `_member_roster_digest.py` where
     they copy capacity.
2. **Display.**
   - `widgets/_queue_weight_badge.py` renders `c1.5x`, with the over-limit style when
     `M > 1`.
   - `prompt_panel/_agent_display_header_metadata.py` shows
     `Capacity: 1.5x budget (7.5 capacity units)`.
   - `prompt_panel/_agent_wait_section.py` shows `capacity budget 1.5x (7.5)`.
   - `prompt_panel/_agent_queue_section.py` ladder shows `c1.5x`.
   - Also update `_agent_list_render_agent_status.py`, `_agent_list_render_cache.py`
     (cache keys include the multiplier), `models/_agent_runner_slot_capacity.py`, and
     `models/agent_runner_slots.py`.
3. **CLI/list.** `src/sase/integrations/_agent_list_entry_builder.py`,
   `_agent_list_entry_models.py`, and `src/sase/agents/cli_list.py` expose
   `queue_capacity_multiplier` in entries and JSON.
4. **Editing.**
   - Surfaces:
     - the wait modal (`modals/wait_modal_values.py`, `wait_modal_completion.py`)
     - `actions/agents/_wait_actions.py`
     - `actions/agents/_directive_persistence.py`
     - `src/sase/ops/commands/_agent_directive.py` (`_capacity_from_payload`)
     - `src/sase/xprompt/_directive_edit_wait.py` (`PromptWaitDirective`,
       `set_prompt_queue`, `set_prompt_wait_and_queue`)
   - These surfaces:
     - accept `<M>x` through `parse_queue_capacity_value`
     - prefill `1.5x` for multiplier agents
     - persist the multiplier and clear the integer, and vice versa
     - reauthor the prompt as `capacity=1.5x`
     - preserve an existing multiplier when only priority or weight is edited
   - The out-of-scope surfaces in Design Decision 9 stay integer-only.
5. **Tests.** Update or add widget and model tests:
   - `tests/ace/tui/widgets/test_agent_queue_section.py`, `test_agent_wait_section.py`,
     and `test_agent_list_runner_slot_status.py`
   - `tests/ace/tui/test_agent_runner_slots_capacity.py`,
     `test_fleet_agents_projection_capacity.py`, and `test_wait_modal.py`
   - `tests/ace/tui/actions/test_agent_directive_persistence.py`
   - `tests/test_agent_list_entries.py` and `tests/test_directive_edit.py`

   Do not run `just check-full`. If a PNG golden legitimately changes, follow the tui
   memory's snapshot guidance.

6. Verify with `sase tool run check`.

## Phase `research-swarm`: Research swarm authors a 1.5x capacity multiplier

Work in sase-research-artifacts. Open it with
`sase repo open sase-research-artifacts -r "<reason>"` and read its `AGENTS.md`. Its
Justfile tests against the linked sase and sase-core sources.

1. **Template** (`src/sase_research_artifacts/xprompts/research_swarm.md`). Replace all
   8 queue directives:

   ```
   %q(w=0.25{% if runners is not none %}, capacity={{ runners }}{% endif %}{% if priority is not none %}, priority={{ priority }}{% endif %})
   ```

   with:

   ```
   %q({% if runners is not none %}{{ runners }}{% else %}1.5x{% endif %}, w=0.25{% if priority is not none %}, priority={{ priority }}{% endif %})
   ```

   - The default renders `%q(1.5x, w=0.25)`.
   - `runners=N` renders `%q(N, w=0.25)`: an explicit absolute budget replaces the
     multiplier.
   - `runners=0` still renders and is still rejected at launch.
   - Update the `runners` input description to say it replaces the default `1.5x`
     multiplier (1.5 × this machine's effective `max_running_agents` budget).

2. **Tests and smokes.**
   - `tests/test_xprompt_loading.py`: `_WEIGHTED_QUEUE_TEMPLATE` and the
     `_assert_each_segment_has_one_queue` marker. Payload assertions: by default,
     `queue_capacity is None` and `queue_capacity_multiplier == 1.5`; with `runners`,
     the integer is set and there is no multiplier.
   - `tests/test_wheel_contract.py`.
   - Both smokes in `.github/workflows/publish.yml`: the `%q(1.5x, w=0.25` count is 8,
     `capacity=` expectations become positional, and the payload assertions match the
     tests above.
3. **Docs.** `docs/xprompts.md`: every segment authors `%q(1.5x, w=0.25)`. Explain the
   budget (1.5 × the effective budget, which is 7.5 when the budget is 5) and that
   `runners` replaces it.
4. **Floors.**
   - The published-floor smoke pins `sase-core-rs==0.34.23`, which cannot parse `1.5x`.
     Check PyPI for the first published `sase-core-rs` release that contains the
     `core-parse` work.
   - If one exists, raise the `sase-core-rs` floor in `pyproject.toml`, the smoke pin
     and its assertion, and the `AGENTS.md` dependency sentence.
   - If none has been published yet, keep the floors and record a `PROPOSED FOLLOW-UP:`
     note on this phase bead to raise them before the next plugin release.
   - Keep `sase>=0.17.2`: that release is not yet published and will carry the feature.
     Raise it only if PyPI already has a 0.17.2+ release without the feature.
5. Verify with `sase tool run check` in sase-research-artifacts.

## Acceptance Criteria

- `%q(1.5x, w=0.25)`, `%q:1.5x`, and `%q(capacity=.5x)` parse. `1.125x`, `0x`, `-1x`,
  `1.5X`, and `%q(2, capacity=1.5x)` are rejected with clear messages.
- The canonical form round-trips as `capacity=1.5x`.
- With `queue_capacity_budget` on and an effective budget of 5, an agent launched with
  `%q(1.5x, w=0.25)` persists `queue_capacity_multiplier: 1.5`. Its runner waiter
  reports `admission_limit` 7.5, and it is admitted only while occupied load + 0.25 ≤
  7.5.
- Changing the effective budget (for example, removing the override so it becomes 10)
  re-resolves a queued multiplier agent's limit to 15. With the flag off, the multiplier
  has no admission effect.
- Serial continuations, dispatch prompt rebuilds, and TUI edits preserve `1.5x`. The TUI
  shows `c1.5x` and "1.5x budget (7.5 capacity units)".
- Rendered `#research_swarm` segments each contain exactly one `%q(1.5x, w=0.25)` by
  default.
- `sase tool run check` passes in every touched repo.
