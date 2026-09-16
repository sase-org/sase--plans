---
tier: epic
title: Arm %hold at launch submission
goal: 'A launch that carries `%hold` arms a durable hold as soon as it is submitted.
  For plain agent launches the runner arms the hold before any dependency wait. For
  typed plans the hold is armed before any unit dispatches. The hold then follows
  each unit to its runner or proc without losing its original timing. It is released
  when a unit never dispatches, and it survives every hand-off between processes.
  The armer never holds its own kin, and a hold-carrying launch gets an implied, non-authored
  priority boost.

  '
phases:
- id: hold-core
  title: Rust hold store, launch armer, and wire support
  depends_on: []
  size: medium
  description: 'hold-core: add the `launch` armer kind, arm-time kin rejection, and
    armer rebind to the Rust hold store. Add a pure launch-unit armer builder and
    key helper, a `future` edge in the typed hold-cycle check, and a waiting-marker
    wire that keeps "absent" separate from "false" for `wait_priority_explicit`. Add
    pyo3 bindings and bump the core pin.'
- id: hold-facade
  title: Python hold facade and launch-hold primitives
  depends_on:
  - hold-core
  size: medium
  description: 'hold-facade: extend the facade with an explicit armer and selectors,
    rebind, a shared TTL resolver, and `launch`-kind validation and liveness (a receipt
    only counts once it is complete). Add `launch_hold.py` with the key, armer, arm,
    pre-arm, rebind, re-anchor, and release primitives, plus `HOLD_ARMER_WAIT_PRIORITY`.
    The CLI reuses the TTL resolver.'
- id: typed-arm
  title: Pre-arm typed plans and follow units to dispatch
  depends_on:
  - hold-facade
  size: medium
  description: 'typed-arm: pre-arm hold-carrying units under the admission lock, with
    idempotent per-unit markers and rollback on failure. The coordinator re-anchors
    the holds before it acks startup. Agent dispatch passes the key and re-anchors
    the hold to the spawned runner; proc dispatch rebinds the hold to the proc. Units
    that never dispatch release their hold, and proc candidates and proc capacity
    admission use the key and the implied priority.'
- id: bootstrap-arm
  title: Arm or rebind in the agent runner bootstrap
  depends_on:
  - hold-facade
  size: medium
  description: 'bootstrap-arm: carry the parsed hold on AgentInfo. Arm a fresh agent
    hold, or rebind the SASE_LAUNCH_HOLD_KEY pre-arm, right after directive extraction;
    skip refresh passes and retry handoffs, and scrub the env var. Thread an implied
    hold priority through runner-slot admission as non-explicit, and make the TUI
    wire enrichment honor an explicit false.'
proposed_by: bbugyi200.athena.sase-11l.5.1.2
parent_bead: sase-11l.5.1.2
create_time: 2026-09-16 16:01:33
status: wip
bead_id: sase-11l.5.1.2.1
---

- **PROMPT:** [prompts/202609/hold_launch_arming.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/hold_launch_arming.md)
- **PARENT:** [202609/hold_directive_surface.md](https://github.com/sase-org/sase--plans/blob/main/202609/hold_directive_surface.md)
- **BEAD:** [sase-11l.5.1.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11l/sase-11l.5.1.2.1.md)

# Plan: arm `%hold` at launch submission

## 1. Context

This epic delivers phase **arm-runtime** (bead **sase-11l.5.1.2**). It is a phase of the
epic **sase-11l.5.1**, "The %hold prompt directive", which is itself part of phase
sase-11l.5 of the epic **sase-11l**. Several pieces have already landed:

- **Phase directive-surface (sase-11l.5.1.1):**
  - the `agent_holds` beta flag;
  - the Rust `hold_directive.rs` collector, formatter, and `hold_fields_to_selectors`,
    with their bindings;
  - `%hold` parsing into `PromptDirectives.hold` (a mapping of Rust `HoldFieldsWire`
    fields) and into `AgentUnitWire.hold` / `ProcUnitWire.hold`;
  - the dispatch-prompt re-emit of the canonical `%hold(...)`;
  - the plan-time diagnostics `hold-self`, `hold-cycle`, and the `%repeat` / `%dispatch`
    rejections;
  - the Python adapter `src/sase/xprompt/hold_directive.py` (`HoldFields`,
    `hold_fields_to_selectors`, `agent_holds_enabled`).
- **sase-11l.2, .3, and .4:**
  - the Rust store `crates/sase_core/src/agent_hold.rs`;
  - the `hold-barrier` runner-slot blocker;
  - the `sase agent hold` CLI (`src/sase/agents/cli_hold.py`);
  - the facade `src/sase/core/agent_hold_facade.py`.
- **sase-11l.7:** `AdmissionEngine._hold_blocks` / `_proc_hold_candidate` in
  `src/sase/agent/launch_admission_engine.py` evaluate holds for pending proc units.
  Their candidates do not set `armer_key` yet; this epic adds it (§6.6).
- **sase-11l.8:** hold visibility (`held_by`, the Holds pane, and the doctor check). It
  renders armer kinds generically.

### Required reading for every worker

- The parent plans: `sase bead show sase-11l.5.1` (the section "Phase arm-runtime" is
  the source spec this epic refines) and `sase bead show sase-11l`.
- Via `/sase_memory_read`: `xprompts.md`, `sase_flags.md`, `lint_and_test.md`, and
  `tui_perf.md` (for the enrichment edit).

### Rules for every phase

- **Rust first.** Cross-repo work changes sase-core first. Open it with
  `sase repo open sase-core -r "<why>"`, following the `/sase_repo` skill.
- **Bindings.** Add every new binding to the module doc index in
  `crates/sase_core_py/src/lib.rs`, and keep `tools/check_sase_core_rs_bindings` and
  `tools/validate_sase_core_rs` green.
- **Pin.** Bump `sase-core-revision.txt` per `tools/ratchet_core_revision` once the
  sase-core commit is published. If it is still unpublished, use
  `just rust-dev-install .venv` locally and record that fact in the closing note.
- **Backend boundary.** Selector, kin, identity, and key semantics live in Rust. Python
  does I/O, liveness facts, process plumbing, and presentation.
- **Flag gating.** With `agent_holds` off, nothing in this epic may write to the hold
  store. A non-bare `%hold` already fails to parse, and a bare `%hold` is inert. Keep
  the Off branch explicit, because sase-11l.10 will delete it.
- **Verification.** Run `just check` before finishing, and run the `cargo test` filters
  named in each phase.
- **No beads, no memory edits.** Record discovered work with
  `sase bead note sase-11l.5.1.2 'PROPOSED FOLLOW-UP: …'`. Workers close only their own
  phase bead.
- **Concurrent siblings.** sase-11l.5.1.3 (preview-confirm) appends
  `preview_pending_capture` to the end of the facade. Keep facade edits additive and
  rebase with care.

## 2. Refinements to the parent spec (all phases follow these)

While studying the code, I found three places where the parent spec, taken literally,
drops a hold:

1. **`receipt.json` is not a completion marker.** `AdmissionEngine._progress()` writes
   `receipt.json` on every return, including `complete: false` when the engine is
   blocked. A `launch` armer whose `done_marker_path` is the receipt must therefore
   count as "done" only when the receipt parses and has `complete: true`. §5.1 defines
   this.
2. **The pre-arm must follow the runner.** An inline typed plan that completes (for
   example, one agent unit with no waits) spawns no coordinator, and the inline process
   exits right away. At that point the pre-arm's pid is dead and the receipt is
   complete, so liveness would prune the record before the runner bootstrap can rebind
   it. After a successful local agent dispatch, the engine therefore re-anchors the
   unit's record to the spawned runner. The record keeps kind `launch`, the same key,
   and the same timing, but gets `pid = spawned[0].pid` and
   `done_marker_path = <spawned[0].artifacts_dir>/done.json` (§6.3). The runner later
   rebinds that record to its agent armer.
   - Both rebinds are keyed operations under the store lock. If the runner rebinds
     first, the engine's re-anchor finds no old key and does nothing.
3. **`SASE_LAUNCH_HOLD_KEY` must not leak.** Agents inherit their runner's environment.
   A child launch with its own `%hold` would otherwise see a stale key, find no record,
   and skip arming. The runner pops the variable before doing anything else with it
   (§7.2).

It also adds four guards:

4. **In-plan `future` cycles.**
   - All plan units are created after the plan's pre-arm, so a `future` hold on unit H
     fences every non-kin sibling. If H waits (directly or transitively) on a sibling,
     the plan deadlocks. The same happens when two siblings both carry `future`.
   - The typed hold-cycle check therefore treats `future` as matching every non-kin
     sibling (§4.4).
5. **Pre-arm markers stay out of receipt parsing.** The per-unit marker
   `launch_admission/units/<logical_id>.hold.json` matches the `units/*.json` receipt
   glob in `AdmissionEngine._states()`. `_states()` must skip `*.hold.json`, and the
   marker never carries an `identity` key.
6. **`condition_error` releases the hold too.** It is terminal without dispatch, so it
   releases the unit's hold alongside `skipped`, `launch_error`, and `cancelled`.
7. **The waiting-marker wire gets a third state.** The Rust scan wire collapses a
   missing `wait_priority_explicit` into `false`. The TUI then treats the implied
   priority 5 as authored. The wire field becomes `Option<bool>` (§4.5).

## 3. Shared vocabulary

- **Key:** `launch:<request_id>/<logical_id>`. Rust owns the format (§4.3), and Python's
  `unit_hold_key` calls the binding. A typed request with an empty `request_id` cannot
  pre-arm; its submission fails with a `%hold:` error.
- **Env var:** `SASE_LAUNCH_HOLD_KEY`, defined once as a constant in
  `src/sase/agent/launch_hold.py`.
- **Implied priority:** `HOLD_ARMER_WAIT_PRIORITY = 5`, defined beside
  `DEFAULT_WAIT_PRIORITY` in `src/sase/core/runner_slots/_admission_types.py` and
  exported from `sase.core.runner_slots`. Lower values start first. A value of 5 never
  triggers a deference window, because deference applies only when the priority is
  greater than the default.
- **Failure prefix:** every user-facing arm, rebind, or pre-arm failure message starts
  with `%hold: `.

## 4. Phase hold-core (Rust, in sase-core)

### 4.1 `launch` armer kind (`crates/sase_core/src/agent_hold.rs`)

- Add `AgentHoldArmerKindWire::Launch` (serde `launch`), and add `"launch"` to `as_str`.
- **Validation:** a `launch` armer requires both `pid` and `done_marker_path`.
  `agent_name`, `family`, and `clan` stay optional; when present, they are validated
  exactly as for agent armers.
- **Liveness:** add
  `AgentHoldArmerLivenessFactWire::Launch { pid_alive, done_marker_present }`, and add a
  `(Launch, Launch)` arm in `armer_is_alive` (`pid_alive && !done_marker_present`).
- Keep `AGENT_HOLD_WIRE_SCHEMA_VERSION = 1`. Older binaries drop an unknown-kind record,
  which fails open; the commit message must say so.

### 4.2 Arm-time kin rejection and rebind

- **Shared validator.** Add `validate_selectors_exclude_armer_kin(armer, selectors)`. It
  fails with `AgentHoldError::Validation` when any value in `names`, `families`,
  `clans`, or `workflows`:
  - equals the armer's `agent_name`, its family (`armer_family(armer)`), or its `clan`;
    or
  - is a dotted descendant of the armer's family (reuse `same_or_dotted_descendant`).

  The message names the offending selector and value, for example:
  `hold selector "planner" names the armer's own identity, family, or clan`. Hoods,
  tribes, and artifact dirs are never rejected.

- **Where it runs:** call the validator in `arm_agent_hold_until`, after
  `validate_and_normalize_record` and before taking the lock. This covers the CLI and
  the directive alike.
- **Rebind.** Add
  `pub fn rebind_agent_hold_armer(sase_home, old_key: &str, new_armer: AgentHoldArmerWire, liveness, now) -> Result<Option<AgentHoldRecordWire>, AgentHoldError>`.
  It validates `now` and `old_key`, then, under
  `with_hold_lock(…, "rebind_agent_hold")`:
  1. reads the records with `read_records_locked` (pruning expired and dead ones);
  2. returns `Ok(None)` without writing anything if `old_key` is absent;
  3. builds the record with the old `created_at`, `expires_at`, `scope`, and `selectors`
     and `new_armer`, then runs `validate_and_normalize_record` and the kin validator.
     On error it returns `Err` and leaves the old record untouched;
  4. removes `old_key`, inserts the record under `new_armer.key` (replacing any record
     already there), writes the state, and returns the record.

  Rebinding to the same key is allowed; the §6.3 re-anchor relies on it.

- Re-export `rebind_agent_hold_armer` from `crates/sase_core/src/lib.rs`.

### 4.3 Launch-unit armer builder (new `crates/sase_core/src/agent_launch/launch_hold.rs`)

- **`pub fn launch_unit_hold_key(request_id, logical_id) -> Result<String, AgentHoldError>`**
  - Both arguments must be non-empty and free of whitespace and control characters.
  - Returns `launch:<request_id>/<logical_id>`.
- **`pub fn launch_unit_hold_armer(unit: &LaunchUnitWire, request_id, project, pid: u32, done_marker_path) -> Result<AgentHoldArmerWire, AgentHoldError>`**
  builds a validated `launch` armer:
  - **`key`:** the value from `launch_unit_hold_key`.
  - **`display`:** `"<label> (launch <first 8 chars of request_id>)"`. The label is the
    agent's effective identity or the proc's `shell_name`, falling back to `logical_id`.
  - **Agent units:** set `agent_name = effective_identity()` only when
    `identity_explicit` is true or a non-`@` `family_attach_suffix` is present. Set
    `family = family_attach_parent` and `clan = clan`.
  - **Proc units:** set no identity fields. A pending proc excludes its own hold through
    the candidate's `armer_key` (§6.6).
- Export the module from `agent_launch`, and re-export both functions from `lib.rs`.

### 4.4 `future` edges in the typed hold-cycle check (`agent_launch/mod.rs`)

- In `hold_matches_unit` (used by `add_hold_cycle_edges`), return `true` when
  `hold.future` is set. Kin exclusion and self exclusion still apply first.
- Update the `hold-cycle` diagnostic message to mention `future`, for example "Typed
  launch holds and waits contain a cycle (a `future` hold fences every other unit in the
  plan)."

### 4.5 Waiting-marker wire keeps absent distinct from false

- In `agent_scan/wire.rs`, change `WaitingMarkerWire.wait_priority_explicit` to
  `Option<bool>` (`#[serde(default)]`).
- In `scanner.rs::waiting_marker_from_object`:
  - a key that is absent or null becomes `None`;
  - any other value becomes `Some(coerce_bool_truthy(..))`.
- Update `tests/agent_scan_parity.rs` (`== Some(true)`) and add a case each for an
  absent key and for `false`.
- Keep `AGENT_SCAN_WIRE_SCHEMA_VERSION` unless a golden or parity test forces a bump. If
  it does, mirror the constant in Python in the bootstrap-arm phase.

### 4.6 Bindings (`crates/sase_core_py/src/lib.rs`)

Add these beside the agent-hold bindings, each with a doc-index row:

- `agent_hold_rebind(sase_home, old_key, new_armer, liveness=None, now=None) -> dict | None`,
  using the same error mapping as `agent_hold_arm_relative`;
- `launch_unit_hold_key(request_id, logical_id) -> str`, which raises `ValueError`;
- `launch_unit_hold_armer(unit: dict, request_id, project, pid, done_marker_path) -> dict`,
  where `unit` is the JSON form of `LaunchUnitWire`.

### 4.7 Tests (`cargo test -p sase_core agent_hold agent_launch agent_scan`; `cargo test -p sase_core_py`)

- **Rebind:**
  - it keeps `created_at` and `expires_at`, so a `future` rule still matches a candidate
    created between arm and rebind;
  - with an absent old key it returns `None` and writes nothing;
  - a same-key rebind works;
  - a kin-invalid new armer returns `Err` and leaves the old record in place.
- **Kin rejection:**
  - rejected: the armer's own name, its family, its clan, and a descendant of its
    family, in each of `names`, `families`, `clans`, and `workflows`;
  - accepted: a hood containing the armer, and an ancestor family.
- **`launch` kind:**
  - validation requires both `pid` and `done_marker_path`;
  - liveness pruning works (dead pid, or done marker present);
  - an agent-kind fact on a launch record is ignored.
- **Builder:** the key format; the display fallbacks; identity fields for an explicit
  agent, a family-attach child, a clan joiner, an auto-named agent, and a proc; and
  errors for an empty request id.
- **Hold cycles:**
  - `future` plus a wait on a sibling gives `hold-cycle`;
  - two `future` siblings give `hold-cycle`;
  - a lone `future` unit gives no diagnostic;
  - a `future` unit whose waited sibling is kin gives no diagnostic.
- **Wire:** the scan parity cases for the tri-state.
- **Existing tests:** check that the CLI-style tests in `agent_hold.rs` still pass with
  kin rejection. `armer("a")` never names itself, so they should.

## 5. Phase hold-facade (Python)

### 5.1 `src/sase/core/agent_hold_facade.py` (edits stay additive)

- **Rename.** Rename `_agent_armer_wire` to the public
  `agent_armer_wire_for_artifacts(artifacts_dir, *, pid_fallback: int | None = None)`.
  When `agent_meta.json` has no integer pid, it uses `pid_fallback`. Update
  `current_armer_wire` and add the new name to `__all__`.
- **`arm_agent_hold`** gains two keyword arguments, both defaulting to today's CLI
  behavior:
  - `armer: Mapping | None = None` (default: `current_armer_wire(...)`);
  - `selectors: Mapping | None = None`. When given, it is the base selectors payload and
    `names`, `tribes`, `hoods`, and `future` are ignored.

  When `pending` is true, the captured artifact dirs are merged into
  `selectors["artifact_dirs"]` (sorted, deduplicated). The armer's own artifacts dir
  (the parent of its agent `done_marker_path`) is dropped from the capture.

- **`rebind_agent_hold(old_key, new_armer, *, now=None) -> dict | None`** calls the
  binding. Errors propagate, and it sends no notification.
- **`resolve_hold_ttl_seconds(requested: float | None) -> float`:**
  - returns the config default when `requested` is `None`;
  - raises `ValueError` naming the configured maximum when the value exceeds
    `get_agent_hold_max_ttl_seconds()`;
  - imports the config getters lazily.

  `cli_hold._resolve_ttl_seconds` keeps its CLI duration parsing and its error printing,
  and delegates the default and the cap to this helper.

- **`_validate_hold_record`** accepts the kind `launch`.
- **Liveness:** `_liveness_facts_for_holds` gains a `launch` branch using
  `_launch_liveness_fact(armer)`:
  - **`pid_alive`:** `_pid_alive(armer)`, or the integer pid in the `started.json`
    beside `done_marker_path` (when that file exists) is running. Reuse
    `sase.ace.hooks.processes.is_process_running`.
  - **`done_marker_present`:** when the marker's basename is `receipt.json`, true only
    if the file parses as an object with `complete is True`; otherwise, true when the
    file exists.

  Unit-test both marker shapes.

- **CLI rendering (`cli_hold.py`):** `_print_hold_detail` shows `Done marker:` when the
  armer has `done_marker_path`. List output needs no change.

### 5.2 New `src/sase/agent/launch_hold.py` (primitives only; no call sites in this phase)

The module imports the facade and bindings lazily. It holds:

- `LAUNCH_HOLD_KEY_ENV = "SASE_LAUNCH_HOLD_KEY"`, and
  `LAUNCH_HOLD_RELEASE_REASON = "launch unit ended without dispatch"`.
- `unit_hold_key(request_id, logical_id) -> str`, a binding wrapper.
- `hold_fields_for(payload) -> HoldFields | None`, which reads `payload.hold` (a
  `HoldFieldsWire`) or a directive mapping.
- `arm_hold_for_fields(fields, *, armer, now=None) -> AgentHoldArmResult`:
  - selectors come from `hold_fields_to_selectors(fields)`;
  - the scope comes from `fields.scope or "project"`;
  - the TTL comes from `resolve_hold_ttl_seconds(fields.ttl_seconds)`;
  - `pending=fields.pending`;
  - any exception is re-raised as `LaunchHoldError(f"%hold: {exc}")`, a new
    `RuntimeError` subclass.
- `rebind_hold(old_key, new_armer) -> dict | None`, which wraps errors the same way.
- `release_hold_best_effort(key, *, reason, display=None) -> bool`, which wraps
  `release_agent_hold` and logs instead of raising.
- `launch_unit_armer(unit, *, request_id, project, pid, done_marker_path) -> dict`, a
  binding wrapper that passes `agent_launch_wire_to_json_dict(unit)`.
- `runner_anchor_armer(armer, *, pid, artifacts_dir) -> dict`: a copy of a `launch`
  armer with `pid` and `done_marker_path=<artifacts_dir>/done.json` replaced.

### 5.3 Priority constant

Add `HOLD_ARMER_WAIT_PRIORITY = 5` (§3) and export it from `sase.core.runner_slots`. Add
a docstring comment saying it is implied and must never be written as authored metadata.

### 5.4 Tests (`tests/test_agent_hold_service.py`, new `tests/test_launch_hold.py`)

- **Arming:**
  - explicit armer and selectors override the defaults;
  - `pending` merges into explicit selectors and drops the armer's own dir;
  - the CLI defaults are unchanged.
- **Kin:** kin rejection surfaces as `ValueError` from `arm_agent_hold`, and the CLI
  `create -n <own name>` prints it.
- **TTL:** `resolve_hold_ttl_seconds` handles the default, the cap, and an over-cap
  value; the CLI reuses it with unchanged messages.
- **`launch` kind:**
  - record validation;
  - liveness: pid alive, pid dead but `started.json` pid alive, both dead, receipt with
    `complete` false or true, and `done.json` present.
- **Rebind:** `rebind_agent_hold` round-trips and keeps `created_at`.
- **`launch_hold` primitives:** the key format; `launch_unit_armer` for agent and proc
  units; `runner_anchor_armer`; the `%hold:` prefix on errors; and best-effort release
  swallowing a store error.

## 6. Phase typed-arm (typed plans)

### 6.1 Pre-arm at submission (`src/sase/agent/launch_admission.py`)

In `dispatch_typed_launch_request`, under the admission `flock`, after `write_sidecar`
and before `engine.run`, call `pre_arm_typed_plan_holds(root, plan, request_id)`. Add
that function to `launch_hold.py`. It does nothing when `agent_holds_enabled()` is false
or when no unit has a hold. Otherwise:

- It fails with `LaunchHoldError` if `request_id` is empty.
- For each hold-carrying unit in `source_order`:
  1. If `units/<logical_id>.hold.json` already exists, skip the unit. Re-entry never
     re-arms or re-freezes.
  2. Resolve the unit's project:
     - for a proc, `payload.selected_project`;
     - for an agent, move `_resolve_unit_project` from `launch_admission_runtime.py` to
       a shared helper and reuse it;
     - otherwise, `plan.selected_project`, then the cwd inference already used by
       `_proc_hold_project`.
  3. Build the armer with
     `launch_unit_armer(unit, request_id=…, project=…, pid=os.getpid(), done_marker_path=str(root / RECEIPT_FILENAME))`.
  4. Call `arm_hold_for_fields`.
  5. Write the marker atomically: `{logical_id, key, armed_at_unix, expires_at}`, with
     no `identity` key.
- **On failure:** if any unit fails, release every key armed in this call, delete their
  markers, and raise `LaunchRequestError("invalid_request", "hold", str(exc))` before
  any unit dispatches.
- **Receipt parsing:** `AdmissionEngine._states()` skips `units/*.hold.json` (§2 item
  5).

### 6.2 Coordinator re-anchor

In `run_coordinator_in_bundle`, under the lock and **before** writing `started.json`,
call `reanchor_pending_unit_holds(root, plan, request_id, pid=os.getpid())`. For each
unit that has a hold marker:

1. Read the raw store with the facade's no-liveness list.
2. Rebind only when the record under the marker's key still has kind `launch` and its
   `done_marker_path` equals this bundle's receipt path. A unit already re-anchored to a
   runner (§6.3) is left alone.
3. The new armer is the same armer with `pid=os.getpid()`.
4. Failures are logged. They never block the ack; liveness remains the backstop.

### 6.3 Agent dispatch (`launch_admission_runtime.py`, `chop_typed_admission.py`, the engine)

- **Env helper.** Add `launch_hold_dispatch_env(unit, request_id) -> dict[str, str]` to
  `launch_hold.py`. It returns `{LAUNCH_HOLD_KEY_ENV: key}` only when the flag is on,
  the unit has a hold, and `request_id` is non-empty.
- **Passing the key:**
  - `dispatch_agent_unit` gains `request_id: str | None = None` and merges the env into
    `extra_env`;
  - `make_approved_agent_dispatcher(data)` passes `data["request_id"]`;
  - the engine's default path passes `self.request_id`;
  - `make_axe_chop_agent_dispatcher` merges the same env into its `extra_env`, using
    `data.get("request_id")`.
- **Remote units.** Remote dispatch (`dispatch_target`) never carries a hold (the
  planner rejects `%hold` with `%dispatch`). As a guard, release the key best-effort
  when a remote unit has one.
- **Re-anchor.** In `AdmissionEngine._apply_action`, after a successful agent dispatch
  (`ok`) of a hold-carrying unit whose first spawned result has a pid and an
  `artifacts_dir`:
  - read the current record;
  - if it is still the bundle-anchored `launch` record, rebind it to the same key with
    `runner_anchor_armer(...)`;
  - log errors instead of raising.

  When no spawned result is available, the record stays anchored to the bundle.

### 6.4 Proc dispatch (`src/sase/agent/launch_proc_runtime.py`)

- `AdmissionEngine._proc_context` adds `request_id`.
- In `dispatch_proc_unit`, once `get_proc(proc.proc_id)` has confirmed the proc is
  reserved and visible:
  1. If the unit carries a hold, rebind `unit_hold_key(request_id, logical_id)` to
     `{kind: "proc", key: f"proc:{proc_id}", display: shell_name or label or proc_id, project: <unit project>, proc_id}`.
  2. If the rebind raises, release the launch key best-effort.
  3. If the proc is already terminal, also release `proc:<proc_id>` best-effort. The
     existing `settle_proc_shell` → `release_proc_agent_holds` path handles later
     settlement.

### 6.5 Units that never dispatch (`launch_admission_engine.py`)

- In `AdmissionEngine._journal`, when the phase is one of `skipped`, `condition_error`,
  `launch_error`, or `cancelled` and the unit carries a hold, call
  `release_hold_best_effort(unit_hold_key(...), reason=LAUNCH_HOLD_RELEASE_REASON)`.
  - This also covers `fail_dispatch` and `_cancel_open_units`.
  - A key already rebound to a runner or proc is simply absent, so the release does
    nothing.
- A complete receipt prunes any stragglers through liveness (§5.1).

### 6.6 Proc candidates and proc priority

- **Candidates.** `_proc_hold_candidate` sets
  `armer_key = unit_hold_key(self.request_id, unit.logical_id)` when the unit carries a
  hold and `request_id` is non-empty. A pending proc is then never held by its own
  `future` rule.
- **Priority.** In `proc_capacity_admission.evaluate_proc_capacity_admission`, the
  priority is:
  - the authored `wait_priority` when present;
  - otherwise `HOLD_ARMER_WAIT_PRIORITY` when `payload.hold` is set;
  - otherwise `DEFAULT_WAIT_PRIORITY`.

  Proc capacity admission still runs only for procs with authored queue fields.

### 6.7 Tests (extend `tests/test_launch_admission_*.py` and `tests/_launch_admission_helpers.py`)

- **Pre-arm:**
  - agent and proc units pre-arm with the right armer, scope, TTL, and pending capture;
  - re-entry is idempotent: no second arm and no re-freeze;
  - an over-cap TTL or a kin failure fails the submission, rolls back already-armed
    units, and dispatches nothing;
  - with the flag off, the store stays empty.
- **Parsing:** `_states()` ignores `.hold.json` markers.
- **Coordinator:**
  - it re-anchors bundle-anchored records to its own pid before `started.json` exists;
  - it leaves runner-anchored records alone;
  - a dead coordinator with an incomplete receipt gets its hold pruned;
  - a complete receipt prunes the hold.
- **Agent dispatch:**
  - the env carries the key (default dispatcher, approved dispatcher, chop dispatcher);
  - a single-unit inline plan that completes keeps the hold alive, anchored to the
    spawned pid, after the receipt completes;
  - simulating the runner rebind keeps `created_at`, so an agent created between
    submission and dispatch is still `future`-held;
  - a remote unit releases its key.
- **Proc dispatch:**
  - the proc rebinds to `proc:<id>` at dispatch and is released at settlement;
  - a failed rebind releases the launch key;
  - the candidate `armer_key` stops self-holding.
- **Non-dispatch:** `skipped`, `condition_error`, `launch_error`, and `cancelled` units
  release their hold with the reason text.
- **Proc priority:** the implied priority applies only without an authored priority.
- **Quiesce composition** `%proc(...) %q:1 %hold(pending, future)`:
  - the proc drains;
  - a later-launched agent (a runner-slot candidate built with the existing fixtures) is
    blocked with `hold-barrier`;
  - the agent is admitted once the proc settles.

## 7. Phase bootstrap-arm (agent runner)

### 7.1 Carry the hold

- In `src/sase/axe/run_agent_directives.py`, add `hold: HoldFields | None = None` as the
  last `AgentInfo` field, filled with `HoldFields.from_mapping(directives.hold)`.
- Leave `agent_meta` and `wait_priority` alone.

### 7.2 Arm or rebind (`src/sase/axe/run_agent_runner_bootstrap.py`)

- **Pop the key.** At the top of `bootstrap_agent_run`, before any other work, run
  `launch_hold_key = os.environ.pop(LAUNCH_HOLD_KEY_ENV, None)`.
- **Call site.** Immediately after `extract_directives_and_write_meta` returns (and
  after `retry_handoff` is known), call
  `arm_bootstrap_hold(state, info, retry_handoff, launch_hold_key)`. This runs before
  `_wait_flags`, the bead claim, and any dependency wait. Put the implementation in
  `launch_hold.py`, or in a thin `run_agent_runner_hold.py` if import weight matters.
- **What `arm_bootstrap_hold` does:**
  1. **Skip:** when `RUNNER_CODE_REFRESHED_ENV` is set, when
     `retry_handoff is not None`, or when `agent_holds_enabled()` is false. The original
     record lasts until the family settles.
  2. **Key but no hold:** release the key best-effort and return. This covers a flag or
     version skew.
  3. **Build the armer:**
     `agent_armer_wire_for_artifacts(state.artifacts_dir, pid_fallback=os.getpid())`. If
     `agent_meta.json` lacks a name at this point, that is a bug: raise a `%hold:`
     error.
  4. **With a key:** call `rebind_hold(key, armer)`.
     - If the record is gone (`None`), print a warning that the hold already ended, and
       continue without a hold.
     - If `rebind_hold` raises, release the key best-effort, then raise the
       `LaunchHoldError`.

     Never arm a fresh hold in this branch.

  5. **Without a key:** call `arm_hold_for_fields(info.hold, armer=armer)`.
  6. **Errors:** any `LaunchHoldError` propagates, so the runner records it as a failed
     launch whose message starts with `%hold:`.

### 7.3 Implied priority

- **Threading.**
  - `run_agent_runner._admit_and_launch` passes
    `wait_priority_implied=HOLD_ARMER_WAIT_PRIORITY if info.hold is not None and info.wait_priority is None else None`
    to `wait_for_runner_slot`.
  - The value continues through `_try_claim_runner_slot` (as
    `directive_priority_implied`) into
    `marker_priority_state(waiting_data, directive_priority, implied_priority=None)`.
  - `marker_priority_state` returns `(implied_priority, False)` in place of
    `(DEFAULT_WAIT_PRIORITY, False)` when no explicit priority exists and
    `implied_priority` is a non-negative int.
  - Keep `run_agent_phases` re-exports in sync.
- **Marker.** `waiting.json` then records
  `wait_priority: 5, wait_priority_explicit: false`.
- **Never authored.** Do not write the implied value into `agent_meta` or
  `AgentInfo.wait_priority`.
- **TUI enrichment.** In `src/sase/core/agent_scan_wire_markers.py`, change
  `WaitingMarkerWire.wait_priority_explicit` to `bool | None = None`. In
  `src/sase/ace/tui/models/_loaders/_meta_enrichment_wire.py`:
  - an explicit `True` or `False` wins;
  - only `None` falls back to the non-default inference, mirroring
    `_meta_enrichment_filesystem.py` and `_legacy_marker_priority_explicit`.

  Update any tests that build the wire with the old bool default.

### 7.4 Tests

- **Bootstrap arming:**
  - with `%wait:x %hold(pending)`, the agent hold is in the store (agent kind, runner
    pid) while the runner is parked on the dependency wait;
  - a refreshed pass and a retry handoff neither re-arm nor re-freeze;
  - a kin name fails the launch with a `%hold:` error;
  - with the flag off, nothing is written;
  - the env var is popped, so a child process does not see it.
- **Rebind:**
  - with `SASE_LAUNCH_HOLD_KEY` set, the pre-armed record is rebound (same `created_at`,
    agent armer), and no second record appears;
  - a missing record means a warning and no hold;
  - a rebind error means the key is released and the launch fails;
  - a key with no hold is released.
- **Implied priority:**
  - it applies only when no `%q(p=…)` is authored;
  - `waiting.json` records it as non-explicit;
  - an authored priority, or a TUI edit that sets explicit, still wins;
  - the TUI wire enrichment does not mark 5 as authored, so the render cache and the
    status text omit `p5`;
  - legacy markers without the key still infer.

## 8. Landing notes (for the land agent of this epic)

- **Verification.**
  - Run `just check-full` on the combined tree through `/sase_monitor`.
  - Run `cargo test -p sase_core agent_hold agent_launch agent_scan` and
    `cargo test -p sase_core_py` in sase-core.
  - Confirm that the pin (`sase-core-revision.txt`) points at the published sase-core
    commit, or document why it cannot yet.
- **Closing.**
  - Run `sase bead epic-symbols sase-11l.5.1.2` and resolve every leftover.
  - Close **only** bead sase-11l.5.1.2, with a note summarizing what was verified.
  - Never close sase-11l.5.1, sase-11l.5, or sase-11l.
- **Follow-ups.** Triage the `PROPOSED FOLLOW-UP` notes on sase-11l.5.1.2. Carry the §2
  refinements into that close note, because they refine the parent spec that
  sase-11l.5.1.3 and sase-11l.10 build on.

## 9. Non-goals

- Previews, the capture threshold, confirmation, and docs (sase-11l.5.1.3).
- Completion and LSP work (sase-11l.6), `held_by` rendering (sase-11l.8), and flag
  removal (sase-11l.10).
- Changing the CLI `-n` to directive name semantics.
- Cross-request or global hold-cycle detection beyond the in-plan check.
- Re-arming holds for a bundle whose coordinator died and was restarted later. The hold
  fails open by design.
