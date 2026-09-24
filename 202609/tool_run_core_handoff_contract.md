---
tier: tale
size: medium
title:
  "sase-17p.1: extend the sase-core ToolRun contract for reservation, adoption, and
  owner-aware settlement"
goal:
  sase-core's ToolRun store and bindings support hand-off reservation with a private
  launch envelope, atomic claim, durable stop requests, typed terminal causes, persisted
  finish diagnostics, the owner log locator, and owner-aware reconcile — additive at
  wire schema 1, readable by older cores — with thin sase adapters, tests, and
  smoke/validator coverage ready for phase standalone-handoff.
proposed_by: bbugyi200.athena.sase-17p.1
bead: sase-17p.1
create_time: 2026-09-24 08:51:28
status: wip
---

- **PARENT:**
  [202609/tool_e2_durable_handoff.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_e2_durable_handoff.md)
- **BEAD:**
  [sase-17p.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17p/sase-17p.1.md)

# Plan: core-contract phase of E2 (bead sase-17p.1)

## Goal

Give the sase-core ToolRun store and its Python bindings everything the later E2 phases
need, while keeping wire schema 1 and staying additive. That means:

- a hand-off reservation that carries a private launch envelope;
- an atomic claim;
- a durable stop request;
- typed terminal causes;
- persisted finish diagnostics (absorbing `sase-145`);
- the owner log locator;
- owner-aware reconcile.

On the sase side, add thin adapters, tests, and smoke/validator coverage. Nothing in
`src/sase/` calls the new adapters yet. The one exception is a small executor change
that makes the `sase-145` log-write facts durable.

Authority: the epic plan `plan:202609/tool_e2_durable_handoff.md`, sections "Decisions
this plan settles", "Durable record additions", "Reconciliation", and "1.
core-contract". This plan follows them exactly, except for three refinements listed
under "Deliberate refinements" below. Each refinement exists only to honor the epic's
own decision 6 (older cores share `~/.sase/tools/runs.sqlite`).

## Ground rules

- Open sase-core with `/sase_repo` (`sase repo open sase-core -r "<why>"`), read its
  `AGENTS.md`, and use only the printed path. Never run bare `cargo`. Iterate with
  `just fast` and `just test -p sase_core tool_run`. Gate with `sase tool run check`
  from inside that checkout, with an explicit tool timeout of at least 15 minutes.
- sase-core conventions:
  - `mod.rs` files are facades that hold only `mod` and `pub use` lines;
  - new files stay at or under 1,500 lines;
  - no `macro_rules!`;
  - add nothing to the root `pub use` list in `crates/sase_core/src/lib.rs` and no new
    `core_*` aliases to `crates/sase_core_py/src/prelude.rs`. Import new items by module
    path (`sase_core::tool_run::claim`, and so on);
  - never edit versions or changelogs.
- `wire.rs` currently has 1,061 lines and `store/tests.rs` has 1,341 lines, so new code
  goes into new files (see "Layout").
- Keep the existing `ToolRunError` `Display` text byte-for-byte. Add only the new
  variant.
- Agents do not commit. Host finalizers commit both repos.

## Deliberate refinements of the epic design

1. **Terminal cause, settled_by, and diagnostics never enter the event payload.**
   `ToolRunEventWire` is `deny_unknown_fields`, and an older core's `show_run` parses
   every stored event payload. A new field in `events.payload_json` would therefore make
   old cores fail `sase tool show` on new runs. These facts are written only to the new
   `runs` columns in the same transaction as the lifecycle event, so `canonical_event`
   stays byte-for-byte replay-stable. No new field is added to `ToolRunEventWire`.
2. **Hand-off launcher identity is stored apart from the wrapper columns.** A hand-off
   begin stores the request's `wrapper_pid`/`boot_id`/`process_start_identity` (the
   launcher's) in a new `launcher_json` column and leaves `wrapper_pid`, `boot_id`, and
   `process_start_identity` NULL until the claim writes the worker's identity there. The
   reason: the launcher exits right after acknowledging, while the run is still
   `created` for the roughly one second the worker takes to import and claim. An older
   sase or core in another workspace venv reconciling in that window would otherwise see
   a dead wrapper and mark the run `lost` (old cores allow `created → lost`), and the
   worker's claim would then be refused. With the wrapper pid NULL, old Python reports
   `unknown` ("wrapper pid was not recorded"), and old cores never settle on `unknown`.
   The begin _request_ shape is unchanged, and the run exposes the launcher as
   `launcher` on `ToolRunWire`. For a `created` hand-off run, the reconcile
   wrapper-liveness fact refers to that launcher; phase `standalone-handoff` teaches the
   Python liveness pass to observe it.
3. **Diagnostics persist into the existing `runs.diagnostics_json` column.** It exists
   and is always `'[]'` today. No new column is needed.

Everything else in the epic's "Durable record additions" table is implemented as
written.

## Layout (sase-core, `crates/sase_core/src/tool_run/`)

| File                              | Change                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `wire.rs`                         | new optional fields on the existing begin, finish, and liveness-fact requests, on `ToolRunWire`, on `ToolRunLogMetadataWire`, and on `ToolRunReconcileResultWire`                                                                                                                                                                                                                                                                                                                            |
| `handoff_wire.rs` (new)           | enums (`ToolRunLaunchModeWire`, `ToolRunTerminalCauseWire`, `ToolRunSettledByWire`, `ToolRunOwnerStateWire`, `ToolRunClaimOutcomeWire`, `ToolRunClaimRefusalWire`, `ToolRunStopOutcomeWire`), `ToolRunLaunchEnvelopeWire`, `ToolRunProcessIdentityWire`, `ToolRunStopRecordWire`, `ToolRunOwnerFactWire`, claim and stop request/result wires, `ToolRunReconcileSettlementWire`. Each enum has `as_str`, plus `from_db` where the value is stored. `mod.rs` gains `pub use handoff_wire::*;` |
| `mod.rs`                          | `ToolRunError::DuplicateRun { run_id }` (Display `tool run {run_id} already exists`); export `claim` and `request_stop`                                                                                                                                                                                                                                                                                                                                                                      |
| `store/connection.rs`             | a generalized column-ensure helper and a column-set probe (below)                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `store/lifecycle.rs`              | begin/finish changes, a cause-aware transition check, and shared helpers made `pub(super)`. `reconcile` moves out                                                                                                                                                                                                                                                                                                                                                                            |
| `store/handoff.rs` (new)          | `claim` and `request_stop`                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `store/reconcile.rs` (new)        | `reconcile`, moved from `lifecycle.rs`, plus a pure decision function for the owner-aware table                                                                                                                                                                                                                                                                                                                                                                                              |
| `store/query.rs`                  | column-aware `load_run`; a private `load_launch_envelope` used only by claim                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `store/mod.rs`                    | `mod handoff; mod reconcile;` plus the re-exports                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `store/tests.rs` + `store/tests/` | keep `tests.rs` and declare `mod handoff; mod reconcile_owner; mod compat;` in it, with bodies in `store/tests/{handoff,reconcile_owner,compat}.rs`. They reuse the parent's private helpers through `super::`                                                                                                                                                                                                                                                                               |
| `fixtures/`                       | `claim_request.json`, `claim_result.json`, `stop_request.json`, `stop_result.json`, `owner_fact.json`                                                                                                                                                                                                                                                                                                                                                                                        |

## Wire details (all additive, schema_version stays 1)

- Every new optional field uses `#[serde(default, skip_serializing_if = ...)]`. New
  request wires are `deny_unknown_fields`, like their siblings. Result wires and
  `ToolRunWire` are not.
- `ToolRunBeginRequestWire` gains:
  - `launch_mode: Option<ToolRunLaunchModeWire>` (`foreground` | `handoff`);
  - `launch: Option<ToolRunLaunchEnvelopeWire>`;
  - `owner_log_path: Option<String>`.
- `ToolRunLaunchEnvelopeWire` (`deny_unknown_fields`) holds:
  - `argv: Vec<String>`, `cwd: Option<String>`, `tool_name: Option<String>`,
    `extra_args: Vec<String>`, `display_argv: Vec<String>`,
    `private_argv: Option<Vec<String>>`;
  - `definition: ToolDefinitionWire`, `digest: Option<String>`, `adhoc: bool`. These are
    exactly the fields of Python's `ResolvedToolArgv`.
- `ToolRunFinishRequestWire` gains `terminal_cause: Option<ToolRunTerminalCauseWire>`.
- `ToolRunLivenessFactWire` gains `owner: Option<ToolRunOwnerFactWire>`. The owner fact
  holds:
  - `kind`, `id`;
  - `state: active | terminal | missing | unknown`;
  - optional `exit_code: i32`, `termination_reason: String`, `stop_requested: bool`.
- `ToolRunWire` gains:
  - `launch_mode: Option<String>`, `terminal_cause: Option<String>`,
    `settled_by: Option<String>`. These are read back verbatim, so unknown stored values
    survive (lenient reads);
  - `stop_request: Option<ToolRunStopRecordWire>` with
    `{requested_ts, requested_by?, reason?}`. An unparsable stored record reads as
    `None`, and a diagnostic is appended to the run's `diagnostics`;
  - `launcher: Option<ToolRunProcessIdentityWire>` with
    `{pid?, boot_id?, process_start_identity?}`.
- `ToolRunLogMetadataWire` gains `owner_log_path: Option<String>`.
- `ToolRunReconcileResultWire` gains
  `#[serde(default)] settled: Vec<ToolRunReconcileSettlementWire>` with entries
  `{run_id, state, terminal_cause, settled_by}`. It lists every run this pass settled.
  `marked_lost` keeps its meaning (every run settled `lost`) so existing callers are
  unaffected.
- Claim:
  - `ToolRunClaimRequestWire` holds
    `{run_id, owner_kind, owner_id, wrapper_pid?, boot_id?, process_start_identity?, owner_log_path?, running_event_id?, now_ts?}`.
  - `ToolRunClaimResultWire` holds
    `{outcome: claimed | refused | stopped, refusal?: not_created | owner_mismatch | already_claimed | not_handoff, replayed: bool, run: ToolRunWire, launch?: ToolRunLaunchEnvelopeWire, diagnostics}`.
    `launch` is present only when `outcome == claimed`.
- Stop:
  - `ToolRunStopRequestWire` holds `{run_id, requested_by?, reason?, now_ts?}`.
  - `ToolRunStopResultWire` holds
    `{outcome: recorded | already_requested | already_settled, run, stop_request?, diagnostics}`.

## Store schema

- Add these nullable TEXT columns to `SCHEMA_SQL`: `launch_mode`,
  `launch_envelope_json`, `launcher_json`, `terminal_cause`, `settled_by`,
  `stop_request_json`, `owner_log_path`. Fresh stores get them there.
- Generalize `ensure_child_observation_columns` into one helper, called on write-open.
  It reads `PRAGMA table_info(runs)` once and runs `ALTER TABLE runs ADD COLUMN` for
  each missing column in a fixed list: `child_process_start_identity` plus the seven new
  columns.
- Replace `runs_has_child_observation_column` with a column-set probe. `load_run` builds
  its projection from that set and selects a `NULL` placeholder for every absent
  optional column, so a read-only open of an old-shape store keeps working. Reads never
  migrate. `meta.schema_version` stays 1.
- `load_run` never selects `launch_envelope_json`, the same way it never exposes
  `private_argv`. Only claim reads the envelope, through `load_launch_envelope`.
- Retention and stats SQL is unchanged, because no state is added.

## Semantics

### begin

- A duplicate `run_id` (checked inside the IMMEDIATE transaction before the insert)
  returns `ToolRunError::DuplicateRun`, not a raw SQLite constraint error.
- Setting `launch` without `launch_mode: handoff` is `Invalid`.
- `launch_mode: handoff` requires all of the following, and anything else is `Invalid`:
  - `commit_running == false`;
  - `launch` is present;
  - non-empty `owner_kind` and `owner_id`;
  - `parent_run_id` is absent.
- Envelope consistency, where any mismatch is `Invalid`:
  - `envelope.argv` equals the protected argv: `private_argv` when present, else
    `display_argv`;
  - `envelope.display_argv`, `private_argv`, `tool_name`, and `extra_args` equal the
    request's;
  - `envelope.adhoc == tool_name.is_none()`;
  - for a named tool, `normalize_tool_definition(envelope.definition)` digests to the
    `definition_digest` computed by `identity_for_begin`, and `envelope.digest`, when
    present, equals it.
- Store the following:
  - `launch_mode` as given (NULL when absent means foreground);
  - the envelope JSON and `owner_log_path`;
  - for hand-off only: the launcher identity in `launcher_json`, with the wrapper
    columns left NULL (refinement 2).

### Transitions

- Replace `can_transition(from, to)` with a check that also takes an optional terminal
  cause:
  - `created → failed` is allowed only with `launch_failed`;
  - `created → signaled` is allowed only with `stop_requested`;
  - both of those also require that no exit code and no signal are present.
- `append_event` passes no cause, so it can never take the two new edges. Every existing
  edge is unchanged.
- Errors stay `InvalidTransition`, with the same text.

### finish

- Accepts `terminal_cause`. When present, the cause must be compatible with the state,
  or the call is `Invalid`:

  | Cause                        | Allowed states            |
  | ---------------------------- | ------------------------- |
  | `exited`                     | `succeeded`, `failed`     |
  | `launch_failed`              | `failed`                  |
  | `signal`, `timeout`          | `signaled`                |
  | `stop_requested`             | `signaled`, `interrupted` |
  | `interrupt`                  | `interrupted`             |
  | `owner_lost`, `wrapper_lost` | `lost`                    |

- Only when the event ingest is not a replay, write the following. A replayed finish
  changes nothing new:
  - `terminal_cause` (COALESCE, so the first write wins);
  - `settled_by = 'wrapper'`;
  - the request's `diagnostics` appended to `runs.diagnostics_json`, de-duplicated with
    order preserved.
- Everything else is unchanged. A finish without a cause stays valid (existing Python
  callers).

### claim (`store/handoff.rs`)

Claim runs in one IMMEDIATE transaction, in this order:

1. A missing run returns `NotFound`.
2. If the run is `created`:
   - `launch_mode` is not `handoff` → refused `not_handoff`;
   - `(owner_kind, owner_id)` differs from the request → refused `owner_mismatch`;
   - a stored `stop_request` is present → settle `signaled` with
     `terminal_cause = stop_requested`, `settled_by = wrapper`, and the diagnostic
     "command was not run: stop requested before the claim". Outcome `stopped`, no
     envelope;
   - otherwise:
     - ingest a `running` event;
     - set `wrapper_pid`/`boot_id`/`process_start_identity` to the claimant;
     - set `owner_log_path` with COALESCE of the request over the begin value;
     - load the envelope. An unparsable or missing envelope is `Invalid`;
     - outcome `claimed` with `launch`, `replayed: false`.
3. If the run is `running`:
   - the owner matches and the claimant's pid, boot_id, and process_start_identity all
     equal the stored wrapper columns → `claimed`, `replayed: true`, with the envelope;
   - otherwise → refused `already_claimed`.
4. If the run is settled:
   - `signaled` with `terminal_cause = stop_requested` → `stopped` (a replay-safe
     no-op);
   - otherwise → refused `not_created`.

Refusals are result values, never errors.

### request_stop (`store/handoff.rs`)

- A missing run returns `NotFound`.
- A settled run returns `already_settled` and writes nothing.
- An existing record returns `already_requested` with the stored record; the first
  request wins.
- Otherwise, write `stop_request_json` with `{requested_ts: now, requested_by, reason}`
  and return `recorded`.
- A stop request never transitions the run itself.

### reconcile (`store/reconcile.rs`)

For each fact, load the run. Missing or already-settled runs are a no-op, which gives
idempotence. Then branch on `launch_mode`.

- **Foreground (NULL or `foreground`).** Keep today's behavior, with two changes:
  - a `Dead` wrapper settles `lost` with `terminal_cause = wrapper_lost`,
    `settled_by = reconcile`, and the fact's reason (default reason unchanged);
  - reap candidates are emitted only when the run has no `owner_kind`. A nested
    owner-mode child shares the owner's process group, so reaping it could kill the
    owner.
- **`handoff`.** Apply the epic's reconcile table through a pure function
  `decide_handoff(run, wrapper_observation, owner_fact) -> Decision`, with first match
  winning. The helpers it uses:
  - An owner fact that is absent, or whose `kind`/`id` does not match the run's owner,
    counts as `unknown` and gets a diagnostic.
  - "Stop requested" means any of: the run's `stop_request`,
    `owner.stop_requested == true`, or `termination_reason == "stop"`.
  - The proc `termination_reason` vocabulary is `success`, `error`, `stop`,
    `total-timeout`, `idle-timeout`, `launch-failure`, `supervisor-loss`, and `reboot`.

  `created` hand-off runs (the wrapper fact describes the launcher):
  1. The owner is `active`, or the launcher is `alive` → no transition.
  2. The owner is `terminal` and a stop was requested → `signaled` / `stop_requested`,
     diagnostic "command was not run".
  3. The owner is `terminal` otherwise → `failed` / `launch_failed`, diagnostic "command
     was not run".
  4. The owner is `missing` and the launcher is `dead` → `failed` / `launch_failed`,
     diagnostic "command was not run".
  5. Anything else → no transition, with a diagnostic. A dead launcher alone is never
     proof.

  `running` hand-off runs (the wrapper fact describes the worker):
  1. The worker is `alive` or `unknown`, or the worker is `dead` and the owner is
     `active` → no transition.
  2. The worker is `dead`, the owner is `terminal` with
     `termination_reason ∈ {success, error}` and `exit_code >= 0` → `succeeded` if the
     code is 0, else `failed` with that code. `exited`, `settled_by = owner`, diagnostic
     "recovered from owner result; fingerprints and stages may be missing".
  3. The worker is `dead`, the owner is `terminal`, and the reason is `total-timeout` or
     `idle-timeout` → `signaled` / `timeout`, with no invented exit code or signal.
  4. The worker is `dead`, the owner is `terminal`, and a stop was requested →
     `signaled` / `stop_requested`.
  5. The worker is `dead`, and either the owner is `terminal` with reason
     `supervisor-loss`, `reboot`, `launch-failure`, or `error` without a usable exit
     code, or the owner is `missing` → `lost` / `owner_lost`.
  6. Anything else, including an owner that is `unknown` or an unrecognized reason → no
     transition, with a diagnostic.

  Non-recovered settlements use `settled_by = reconcile`. Hand-off runs never produce
  reap candidates.

A settlement from reconcile sets `terminal_cause`, `settled_by`, and diagnostics through
the same helper that finish uses. Reconcile never re-executes anything.

## Bindings (`crates/sase_core_py/src/telemetry/mod.rs`)

- Add `py_tool_run_claim` (`tool_run_claim`) and `py_tool_run_request_stop`
  (`tool_run_request_stop`). Model them on `py_tool_run_observe`: same
  `(store_path, request, busy_timeout_ms=250)` signature, same
  `PyRuntimeError(error.to_string())` mapping. Call
  `sase_core::tool_run::{claim, request_stop}` by module path.
- Register both in `register_telemetry`.
- Extend `tool_run_bindings_round_trip_python_dicts` in `telemetry/tests.rs` with this
  sequence: hand-off begin, claim, replayed claim, stop on a settled run, and a
  reconcile with an owner fact.

## Core tests (in `store/tests/`)

- **`handoff.rs`:**
  - a hand-off begin commits `created`, stores the launcher in `launcher` with NULL
    wrapper columns, and records `launch_mode` and `owner_log_path`;
  - each invalid begin combination is refused, and so is each envelope mismatch;
  - a duplicate `run_id` returns `DuplicateRun`;
  - claim succeeds and returns the envelope, sets the worker identity and the locator;
  - claim refusals: `not_handoff`, `owner_mismatch`, `already_claimed` (a different
    claimant), `not_created` (a settled run);
  - a claim replay by the same claimant returns `replayed: true`;
  - a claim with a pending stop returns `stopped` and settles `signaled` /
    `stop_requested` with "command was not run". A second claim returns `stopped` again;
  - stop requests before claim, during running, and on a settled run; first-wins;
  - both new `created` transitions through finish, each refused with a wrong cause and
    refused with an exit code;
  - `append_event` cannot take either new edge;
  - the finish cause/state compatibility table;
  - finish diagnostics persist and show up on `show_run`, and a replayed finish does not
    duplicate them;
  - `list_runs`, `show_run`, and the serialized JSON never contain the envelope or any
    of its argv fields beyond what `display_argv` already shows. Use a secret-bearing
    `private_argv`.
- **`reconcile_owner.rs`:**
  - one test per table row for `created` and for `running` hand-off runs, including the
    `unknown`/mismatched-owner no-op rows;
  - re-applying the same facts to a settled run is a no-op, with no new event;
  - the foreground `lost` path now records `wrapper_lost` and `settled_by = reconcile`;
  - an owned foreground run that dies yields no reap candidate, while an unowned one
    still does (the existing tests stay green);
  - hand-off runs never yield reap candidates;
  - the `settled` result list is correct.
- **`compat.rs`:**
  - an old-shape store (created with the pre-change `runs` DDL as a literal in the test)
    is readable through `show_run`/`list_runs`, with new fields absent. A write then
    migrates it by adding the columns;
  - a new-shape store containing a hand-off run and a stop request stays loadable by the
    pre-change query shape. Keep the old `load_run` SELECT list as a test constant, run
    it, and assert that `state`, `source`, and `executor` parse through the unchanged
    `from_db` sets and that event payloads deserialize into `ToolRunEventWire`.
- **Golden fixtures:** add `golden_handoff_fixtures_pin_the_wire_shape`, which
  round-trips the five new fixture files through their wires, beside the existing golden
  observe test.

## sase side

- `src/sase/core/tool_run.py`: add
  `tool_run_claim(request, *, store_path=None, busy_timeout_ms=250)` and
  `tool_run_request_stop(...)`, mirroring `tool_run_observe`. Give each a one-line
  docstring naming its contract.
- `Justfile` `_lint-symvision`: add `--epic-symbol 'sase-17p(tool_run_claim)'` and
  `--epic-symbol 'sase-17p(tool_run_request_stop)'`. Add a comment line above the recipe
  in the style of the sase-16h precedent: phase `sase-17p.2` (standalone-handoff)
  consumes `tool_run_claim`, and phase `sase-17p.4` (lifecycle-controls) consumes
  `tool_run_request_stop`; each removes its entry. Key the entries on the epic, never on
  `sase-17p.1`, so `sase bead epic-symbols sase-17p.1` stays empty.
- `tools/smoke_sase_core_rs_tool_runs`:
  - add both names to `TOOL_RUN_BINDINGS`;
  - add a hand-off leg to `validate_round_trip`: begin a hand-off, confirm that `show`
    has no `launch` key, claim and check that the envelope argv matches, run a replayed
    claim, stop the now-running run (`recorded`), and finish with `terminal_cause`
    `stop_requested` and a diagnostic that persists;
  - extend `tests/test_sase_core_rs_tool_runs_smoke_tool.py` to assert the new result
    keys, skipping when `tool_run_claim` is absent, like the existing skip.
- `tools/validate_sase_core_rs`:
  - add `tool_run_claim` and `tool_run_request_stop` to `REQUIRED_BINDINGS`;
  - add a small `_validate_tool_run_handoff_contract(module)` behavior probe, wired into
    `main`, in a temporary store: a hand-off begin succeeds; a claim with a mismatched
    owner is refused with `owner_mismatch`, not raised; the envelope is absent from
    `tool_run_show`. This catches a local core built before this change;
  - add a `tests/test_validate_sase_core_rs_tool.py` test asserting both names are
    required, plus a probe test following the neighboring contract-probe test style.
- `tools/check_sase_core_rs_bindings` needs no list edit, because its static scan picks
  up the adapters' literals. Verify that with
  `tools/check_sase_core_rs_bindings --list`.
- `tests/core/test_tool_run_store.py`: add a real-binding round trip of reserve
  (hand-off begin), claim, stop, finish (with a cause and diagnostics), and reconcile
  (an owner fact that recovers a dead-worker run). Mark these tests
  `skipif(not hasattr(sase_core_rs, "tool_run_claim"))`, the same way the module already
  skips on `tool_run_begin`.
- `sase-145` log-write facts, the one `src/sase` behavior change:
  - in `src/sase/tool/logs.py`, add a helper that returns one explicit fact per retained
    log sink whose write failed (which log and path);
  - in `src/sase/tool/executor.py`, pass
    `[*ingest_diagnostics, *truncation, *log_write_facts]` (or `None`) to
    `finish_tool_run`;
  - the spawn-failure diagnostic (127/126) is already sent;
  - add a `tests/tool/test_executor.py` case, skipped without `tool_run_claim`. A spawn
    failure and a truncated or write-failed log must each leave its diagnostic in
    `run.diagnostics` of the `sase tool show RUN -j` envelope. Keep executor edits
    minimal, because phase `standalone-handoff` splits this file.
- Run the sase tool tests (`tests/tool`, `tests/core/test_tool_run_store.py`, the
  smoke/validator tests). If an exact-shape JSON assertion now sees `settled_by` or
  `terminal_cause`, update it deliberately, never by pasting output. If a liveness test
  relied on reaping an owned run, update it to the new rule and cite the epic's
  decision 8.

## Verification

1. sase-core: `just fmt`, `just fast`, `just test -p sase_core tool_run`,
   `just test -p sase_core_py`, then `sase tool run check` from inside the sase-core
   checkout (explicit timeout of at least 15 minutes). Treat known `sase_gateway` fleet
   flakes per sase-core `AGENTS.md` and never weaken an assertion.
2. sase: rebuild the local core from the linked checkout (`just rust-install`, with a
   generous explicit timeout) so the new bindings are importable. Run the focused pytest
   files above, then `just fix`, then `sase tool run check`, per
   `sase/memory/lint_and_test.md`.
3. `sase bead epic-symbols sase-17p.1` must print no leftovers.

## Core revision pin

Agents do not commit, so the sase-core commit carrying these bindings does not exist on
the remote during this turn. At the end, check whether sase-core's remote HEAD contains
the new bindings (`just ratchet-core-revision --report-only`). If it does, move
`sase-core-revision.txt` with `just ratchet-core-revision`. If it does not (expected),
leave the pin and record
`PROPOSED FOLLOW-UP: ratchet sase-core-revision.txt past the sase-core tool_run_claim/tool_run_request_stop commit before sase-17p.2 lands — CI's pinned-bindings check fails on the new adapters until then`.
Also state in the closing note that phase `standalone-handoff` must ratchet first. Never
touch the `sase-core-rs` window in `pyproject.toml`.

## Closing the bead

- Do not close `sase-145`. The launch prompt limits closure to this bead. If its three
  cases are covered and visible in `show -j`, record
  `PROPOSED FOLLOW-UP: close sase-145 — acceptance verified in sase-17p.1 (<evidence>)`;
  otherwise leave a note for phase `settlement`.
- Record any other discovered work as `PROPOSED FOLLOW-UP:` notes on `sase-17p.1`.
  Create no beads.
- Close with `sase bead close sase-17p.1 --note "<what was verified>"` only after the
  epic-symbols check is clean.
