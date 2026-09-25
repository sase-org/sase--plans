---
tier: epic
title: Allow zero-load agents with %queue(weight=0)
goal: 'A user can author `%queue(weight=0)` or `%q(w=0)` to launch a sase agent that
  adds no weighted load to runner capacity. The directive round-trips through every
  re-authoring path, the agent and its user-authored lineage run at weight 0 end to
  end, and the epic-launch monitor''s host-set zero weight still never gives its successors
  a free ride.

  '
phases:
- id: core-contract
  title: Rust queue contract accepts authored zero weight
  depends_on: []
  size: medium
  description: 'core-contract: in the linked sase-core repo, make the %queue parser
    accept an exactly zero weight literal, make the canonical formatter emit weight=0,
    update editor/LSP metadata, harden the legacy capacity=0 drain for zero-weight
    launches, and pin the existing zero-weight admission rules with tests.'
- id: py-runtime
  title: Python runtime honors explicit zero weight
  depends_on: []
  size: medium
  description: 'py-runtime: in sase, stop Python validators, fallbacks, and inheritance
    paths from rejecting or dropping an explicit 0.0 queue weight, keep epic-launch
    monitor zero from leaking to successors, and render a w0 badge; testable with
    records, without the new core.'
- id: pin-e2e-docs
  title: Core pin bump, end-to-end directive tests, and docs
  depends_on:
  - core-contract
  - py-runtime
  size: small
  description: 'pin-e2e-docs: move sase-core-revision.txt past the core-contract commit,
    add directive-level tests that exercise %q(w=0) through the real Rust binding,
    and update the user docs and config text that say weight must be positive.'
proposed_by: bbugyi200.apollo.1n
create_time: 2026-09-25 09:48:57
status: wip
bead_id: sase-198
---

- **PROMPT:** [prompts/202609/queue_zero_weight.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/queue_zero_weight.md)
- **BEAD:** [sase-198](https://github.com/sase-org/sase--beads/blob/main/pages/sase-198/README.md)

# Plan: Allow zero-load agents with `%queue(weight=0)`

## Background

`%queue` / `%q` takes `weight=` / `w=`, the capacity units a launch claims against
`max_running_agents` or its own `capacity=` budget. Today the Rust directive contract
(`crates/sase_core/src/queue_directive.rs` in the linked `sase-core` repo) rejects every
zero spelling. `%queue(weight=0)`, `%q(w=0)` and `%q(w=0.0)` all fail with code
`invalid-queue-weight` whether `queue_capacity_budget` is on or off. The canonical
formatter also silently drops a zero weight: `format_queue_directive(weight=0.0)`
returns `None`.

Most of the backend already supports zero weight. Commit `e3e926f` in sase-core ("accept
explicit zero-weight capacity records") added an explicit `queue_weight: 0.0` plus
`queue_weight_explicit: true` as a valid non-occupying weight, for the host epic-launch
monitor (`queue_weight_override=0.0` in `src/sase/bead/epic_launch.py` and
`src/sase/monitor/member.py`). Three sase-core checks already accept an explicit zero
and reject an implicit zero:

- runner capacity: `record_weight_is_valid`
- the artifact scanner: `marker_queue_weight_is_valid`
- the fleet contract: `fleet_queue_weight_is_valid`

The Python runner-slot record builders and TUI loaders follow the same rule.

That commit kept user-authored weights strictly positive on purpose. This epic lifts
that restriction.

## Semantics (decisions)

1. **Authorable zero.** `weight=0` / `w=0` is valid whether `queue_capacity_budget` is
   on or off. It is accepted only when the literal denotes exactly zero: `0`, `0.0`,
   `.0`, `0.`, `+0`, `0e5`. Negative spellings (`-0`, `-0.0`) and nonzero literals that
   underflow to zero (`1e-324`) are still rejected, along with NaN, infinity, overflow,
   empty, boolean and nonnumeric values. An accepted zero is stored as `+0.0` and
   formats as `weight=0`.
2. **A zero-weight agent adds no load but still queues.** While it runs it contributes
   `0.0` to occupied capacity, so every other launch sees the same free capacity as if
   it were not running. Admission does not change: the agent keeps its priority/FIFO
   turn, and the existing sase-core rule "Zero-weight waiters still require free runner
   capacity before admission" still applies. It waits only while the fleet is exactly
   full or over-committed. The rejected alternative was to let weight 0 bypass the
   runner queue entirely. That would be a separate opt-in, not a meaning of `weight`.
3. **User-authored zero is inherited.** Successors of a user-authored zero-weight agent
   inherit weight 0 and carry it as **explicit** (`queue_weight_explicit: true`),
   because an implicit zero is invalid by design. The successors are plan, questions and
   pipe follow-ups, `%id(..., session=parent)` session-attach children, gate-shell
   members, and monitor continuations. Zero-weight claims stay non-reusable
   (`claim_is_reusable`), so each successor still takes its own queue turn, at weight 0.
4. **Host-set zero is not inherited.** The epic-launch monitor's `queue_weight_override`
   zero applies to the monitor member only. Successors of that monitor must inherit the
   weight they would have had without the override: the starter's weight and
   explicitness, or the default 1.0. They must never inherit 0.
5. **No feature flag.** This is a small, complete change that only accepts input the
   parser used to reject. It is not a staged beta, and no old branch has to stay
   reachable.

## Phase `core-contract`: Rust queue contract accepts authored zero weight

Work in the linked checkout (`sase repo open sase-core -r "<why>"`). Read its
`AGENTS.md` first and follow its build, verification and commit conventions. This is
additive behavior, so use a `feat:` subject; it is not a breaking change.

1. **Parser** (`crates/sase_core/src/queue_directive.rs`):
   - Change `parse_weight` to accept a literal that denotes exactly zero:
     - Parse with `str::parse::<f64>`.
     - Reject any literal that starts with `-`.
     - When the parsed value is `0.0`, accept it only if every mantissa digit is `0`.
       Strip an optional leading `+`, split at `e`/`E`, and require the mantissa's
       characters to be only `0` and `.` with at least one digit. This keeps the
       underflow-to-zero rejection.
     - Normalize an accepted zero to `+0.0`.
   - Add a public helper for the authored-weight rule, for example
     `authored_queue_weight_is_valid(value) -> bool` (finite and `>= 0.0`). Use it in
     `format_queue_directive`.
   - Do **not** loosen `queue_weight_is_valid`. It is also the positivity check for
     capacity limits in `runner_capacity/capacity_math.rs`, `snapshot.rs`, `waiters.rs`
     and `candidate.rs`, and a zero limit must stay invalid.
   - Optional cleanup: make the three duplicate explicit-zero helpers
     (`record_weight_is_valid`, `marker_queue_weight_is_valid`,
     `fleet_queue_weight_is_valid`) delegate to one shared helper. Their accept/reject
     results must stay exactly the same.
   - Update the `weight_error` message and keep the code `invalid-queue-weight`.
     Suggested text: "%queue(weight=...) requires a non-negative finite base-10 float;
     negative, NaN, infinity, overflow, underflow-to-zero, boolean, empty, and
     nonnumeric values are rejected. Use weight=0 for a launch that adds no runner
     load."
2. **Formatter**: make `format_queue_directive` emit `weight=0` for a zero weight. Check
   that `format_queue_weight(0.0)` and a `-0.0` input both render `0`, never `-0`. Also
   update the `%queue(capacity=N, priority=P, weight=W)` canonical-form doc comment if
   it names positivity.
3. **Budget validation**: `validate_queue_budget_fields` rejects `weight > capacity`, so
   weight 0 already passes. Add a test that `%q(1, w=0)` is accepted with
   `queue_capacity_budget` on.
4. **Legacy persisted `capacity=0` with zero weight**:
   `normalize_persisted_queue_capacity` maps a persisted explicit zero capacity to
   `admission_limit = effective_weight` when the budget flag is on. With
   `effective_weight == 0.0` that limit is `0`, which `waiters.rs` reports as
   `invalid-capacity-limit`. This can happen if `%q(0, w=0)` is authored with the flag
   off and the flag is turned on while the agent waits.
   - Make that case a real drain barrier: block while occupied weighted load is above
     zero, then admit once it is exactly zero.
   - It must never surface as `invalid-capacity-limit`.
   - Keep the result for nonzero weights exactly as it is today, including the
     `legacy-capacity-zero` diagnostic.
   - Also update the pyo3 `normalize_persisted_queue_capacity` binding, the
     `PersistedQueueCapacityNormWire` fields, and the `sase_core_py` tests if the fix
     changes them.
5. **Editor and LSP metadata** (`crates/sase_core/src/editor/directive/metadata.rs`,
   `crates/sase_core/src/editor/wire.rs`):
   - For the `w` / `weight` keyword specs in both `%queue` metadata tables, replace
     "Positive capacity units" with wording that allows zero, for example "Non-negative
     capacity units claimed by this launch; 0 adds no load".
   - Add a `NonNegativeFloat` `DirectiveValueRole` variant, mirroring
     `NonNegativeInt`/`PositiveInt`, and use it for those four specs. Python does not
     read value roles.
   - Add a `0` entry to `QUEUE_WEIGHT_SUGGESTIONS` ("No load: runs without counting
     toward capacity").
   - Update any editor schema or wire fixtures and the `sase_xprompt_lsp` completion
     tests that pin these strings.
6. **Dispatch prompt**: `agent_unit_dispatch_prompt` in
   `crates/sase_core/src/agent_launch/admission.rs` re-emits `%queue` through
   `format_queue_directive` when `queue_weight_explicit` is set. Add a test that an
   explicit `queue_weight: 0.0` round-trips as `%queue(weight=0)`. `typed_units.rs`
   already passes `weight` through with `is_some()` explicitness; add a typed-launch
   test for `%q(w=0)` if the existing tests do not cover it.
7. **Admission rules to pin with tests** (in `runner_capacity`), adding whichever are
   missing:
   - A running explicit-zero record adds `0.0` to occupied capacity, so a weight-1
     waiter still fits a limit the zero agent would otherwise fill.
   - An explicit-zero waiter is admitted when there is free capacity.
   - It is blocked with `insufficient-capacity` when occupied load equals the limit.
   - It respects `queue-order` behind an earlier eligible waiter.
   - A successor of a zero-weight claim does not reuse the claim.
8. **Tests in `queue_directive.rs`**:
   - Move `"0"` out of `rejects_invalid_weight_values`.
   - Keep `"-0"`, `"-0.0"` and `"1e-324"` there, and add `"-0e0"` and `"0e-400x"`-style
     junk.
   - Add an accepts test for the zero spellings in decision 1 with the flag both off and
     on. Assert `fields.weight == Some(0.0)`, that the sign bit is clear, and that the
     formatted value is `%queue(weight=0)`.
   - Add a persisted-zero test for decision 4.
9. Verify with `sase tool run check` in the sase-core checkout, which runs the full
   `just check` gate.

## Phase `py-runtime`: Python runtime honors explicit zero weight

Work in the sase repo. Nothing in this phase needs the new core. Tests should build
agent_meta / waiting records with `queue_weight: 0.0` and `queue_weight_explicit: true`
directly; the pinned core's runner-capacity snapshot and artifact scanner already accept
them. Across all changes, keep the invariant: **explicit zero is valid, implicit zero
stays invalid.**

1. **Validators that reject an explicit zero** (each currently crashes a zero-weight
   launch):
   - `src/sase/axe/run_agent_wait_slot_state.py`:
     - `valid_queue_weight` rejects `weight <= 0`.
     - `marker_queue_weight_state` ignores the explicit flag, both for the
       `waiting.json` weight and for the directive weight. It should accept `0.0` when
       the corresponding explicit flag is true (`waiting_data["queue_weight_explicit"]`,
       or `directive_explicit`).
     - This is on the path every agent takes: `run_agent_runner.py` →
       `wait_for_runner_slot`.
   - `src/sase/axe/run_agent_wait_slot_candidate.py`:
     `candidate_scan_queue_weight_error` calls `valid_queue_weight` on the scanned
     `waiting` / `agent_meta` weight. Accept `0.0` when that record's
     `queue_weight_explicit` is true.
   - `src/sase/axe/run_agent_directive_metadata.py`:
     - `_coerce_queue_weight` and `_existing_queue_weight` raise on 0. They are used by
       `preserved_agent_metadata`, which runs on every extraction including the re-exec
       after a dependency wait, and by `_parent_queue_weight` for session-attach
       children. Accept `0.0` when the same meta has `queue_weight_explicit: true`.
     - Update `QUEUE_WEIGHT_ERROR` wording to match.
   - `src/sase/gate_shell/log.py`: `_positive_finite_weight` and
     `_gate_shell_queue_weight` reject 0. Gate members copy the parent's `queue_weight`
     and `queue_weight_explicit`, so accept an explicit zero and fix the error text.
   - Prefer one small shared Python helper for "valid queue weight given explicitness"
     over four local copies. Its accept/reject results must match the existing
     `src/sase/core/runner_slots/_admission_capacity_records.py` /
     `_is_explicit_zero_weight` behavior.
2. **Truthiness fallbacks that turn 0.0 into 1.0** (use `is None` checks):
   - `src/sase/core/runner_slots/_admission.py`
     (`waiter.get("requested_weight") or 1.0`)
   - `src/sase/ace/tui/models/agent_runner_slots.py` (`... or DEFAULT_QUEUE_WEIGHT` for
     `requested_weight`)
   - `src/sase/integrations/agent_list_entries.py` (`entry.wait.queue_weight or 1.0`,
     which also skews `runner_slot_queue_display_key` parked ordering)
3. **Inheritance keeps zero explicit (decision 3) and never leaks host zero
   (decision 4)**:
   - Add one helper, for example
     `inheritable_queue_weight(meta) -> tuple[float | None, bool]`, that returns the
     weight and explicitness a successor should inherit:
     - user-authored explicit `0.0` returns `(0.0, True)`
     - a positive weight keeps today's behavior: inherited implicitly, so lineage claim
       reuse and weight inheritance still work
     - a host-override zero returns the pre-override values
   - To tell host-override zero apart, change `create_monitor_member_artifacts` in
     `src/sase/monitor/member.py`: when `queue_weight_override` is given, stash the
     inherited pre-override `queue_weight` / `queue_weight_explicit` (or their absence)
     in dedicated monitor fields before overwriting them, and mark the override.
   - Route every inheritance site through the helper:
     - `src/sase/axe/run_agent_helpers_artifacts.py` (follow-up meta currently copies
       `queue_weight` and forces `queue_weight_explicit = False`, which turns 0 into an
       invalid implicit zero)
     - `_parent_queue_weight` and its caller in
       `src/sase/axe/run_agent_directive_metadata.py` (session-attach children)
     - `queue_launch_prefix` and `launch_wire_extra` in
       `src/sase/monitor/continuation_delivery.py` (monitor continuations)
     - any gate/shell member copy in `src/sase/gate_shell/member.py` /
       `src/sase/shells/member.py` that should follow the same rule
   - Check how an epic-launch monitor's continuation (`src/sase/monitor/followup.py`)
     resolves its session-attach parent, so the monitor's zero cannot reach it through
     either the `%queue` prefix or parent-meta inheritance.
4. **TUI display**: `format_queue_weight_badge_value` in
   `src/sase/ace/tui/models/_agent_runner_slot_types.py` hides `weight <= 0.0`. Show a
   `w0` badge for an explicit zero; implicit or invalid values stay hidden. It feeds:
   - the row badge (`_queue_weight_badge.py`)
   - the `Weight:` header line (`prompt_panel/_agent_display_header_metadata.py`)
   - the wait section and queue ladder "needs" text (`_agent_wait_section.py`,
     `_agent_queue_section.py`)

   Make sure no "needs" text divides by or pluralizes the weight oddly at 0.

5. **Tests**:
   - Update `tests/test_run_agent_directive_metadata.py` and
     `tests/test_run_agent_runner_slot_weight_invalid.py`: drop `0` from their
     explicit-invalid parametrizations, and add explicit-zero accept cases plus
     implicit-zero reject cases.
   - Update `tests/test_capacity_snapshot_parity.py` and
     `tests/ace/tui/test_fleet_agents_projection_capacity.py`, which currently assert
     that an explicit zero shows no badge or `Weight:` text.
   - Add tests for each fallback in step 2.
   - Add tests for follow-up and session-attach inheritance of a user-authored zero
     (explicit zero out).
   - Add tests that an epic-launch-style monitor member (`queue_weight_override=0.0`)
     yields successors, including the `queue_launch_prefix` / `launch_wire_extra`
     outputs, whose weight is not 0.
   - Add a gate-shell test with a zero-weight parent.
   - Keep `tests/test_enrich_agent_waiting_queue.py`'s implicit-zero-invalid case
     passing.
6. Follow the sase lint/test memory note for verification before finishing.

## Phase `pin-e2e-docs`: Core pin bump, end-to-end directive tests, and docs

Work in the sase repo after `core-contract` is pushed.

1. **Pin**: move `sase-core-revision.txt` to a sase-core commit that includes the
   `core-contract` change. `just ratchet-core-revision` works when sase-core's remote
   HEAD contains it; see "The CI source revision pin" in `docs/rust_backend.md`.
2. **End-to-end directive tests through the real binding**:
   - `collect_queue_fields` for `%q(w=0)`, `%queue(weight=0.0)` and `%q(1, w=0)`, with
     the flag both off and on. Expect `weight == 0.0` and no errors. `%q(w=-0)` still
     returns `invalid-queue-weight`.
   - `format_queue_directive(weight=0.0)` returns `"%queue(weight=0)"`.
   - Directive extraction (`src/sase/xprompt/_directive_extract.py` →
     `run_agent_directive_metadata.py`) writes `queue_weight: 0.0` and
     `queue_weight_explicit: true` to agent metadata.
   - `set_prompt_queue` / `set_prompt_wait_and_queue` in
     `src/sase/xprompt/_directive_edit_wait.py` preserve `weight=0` when editing a
     prompt that has only `%q(w=0)`, and when it is combined with other fields. Today
     they drop it. Cover the ops persist-directive path in
     `src/sase/ops/commands/_agent_directive.py` too.
   - The typed-launch dispatch prompt (`agent_unit_dispatch_prompt` via
     `src/sase/core/agent_launch_facade.py`) carries `%queue(weight=0)`.
   - Re-run the `py-runtime` epic-monitor successor tests against the new core, since
     the formatter now emits zero. `queue_launch_prefix` must still not emit `weight=0`
     for an epic-launch monitor's continuation.
   - Optionally, add an integration test that a launched `%q(w=0)` agent passes
     `wait_for_runner_slot` and its running record adds 0 to the snapshot's occupied
     capacity.
3. **Docs and config text**. Replace "positive" and "zero cannot be authored" with the
   semantics in decisions 1–4. Explain that a zero-weight agent adds no load but still
   takes its queue turn and needs free capacity to start, and that `%q(w=0)` is the way
   to launch one.
   - `docs/xprompt.md`: the weight paragraph that lists zero as rejected, and the "A
     zero weight cannot be authored" passage.
   - `docs/configuration.md` and `docs/troubleshooting/runner-slots.md`: the "positive
     finite fractional or larger weight" wording. Also revisit the monitor-only
     zero-weight passages in `docs/troubleshooting/runner-slots.md` and
     `docs/monitors.md`.
   - `docs/ace.md`: the badge text that says non-default weights render `wN`; zero now
     renders `w0`.
   - `src/sase/default_config.yml`: the `max_running_agents` comment. Keep the
     `max_running_agents` description in `src/sase/config/sase.schema.json` in sync with
     it.
4. Follow the sase lint/test memory note for verification before finishing.
