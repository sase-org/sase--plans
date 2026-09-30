---
tier: tale
title: sase-core starter scope, monitor join, and sync wait budget
goal: "Land the ToolRun starter record, atomic join and release-join APIs, envelope
  continuation_mode, and sync wait budget in sase-core, with the migration, Python
  bindings, fixtures, and tests, while wire schema version stays 1.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1cx.1
bead: sase-1cx.1
create_time: 2026-09-29 20:47:24
status: wip
---

- **PARENT:**
  [202609/tool_run_escalation.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_run_escalation.md)
- **BEAD:**
  [sase-1cx.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cx/sase-1cx.1.md)

# Plan

Implement phase `core-detach-join` of epic `sase-1cx` entirely in the linked `sase-core`
checkout. Open it with
`sase repo open sase-core -r "Implement starter scope, monitor join, and sync wait budget"`
and work only in the printed path. Read that checkout's `AGENTS.md` before editing.
Follow its recipe "Add a core function and expose it to Python". Never run bare `cargo`.
Never edit a `version` field, a path-dependency pin, or a `CHANGELOG.md`.

This phase is additive. Keep `TOOL_RUN_WIRE_SCHEMA_VERSION` at 1. Result wires stay
lenient. The commit subject is not `feat!`: released sase never sends the new fields.
Python CLI, the feature flag, the watchdog, and docs belong to later epic phases. Leave
`tool_run/projection/` queries unchanged; the TUI is out of scope. `show`, `claim`, and
`list` already return `ToolRunWire` through `load_run`, so `starter` and `join` appear
there once `load_run` reads the new columns.

## Starter record

In `crates/sase_core/src/tool_run/handoff_wire.rs`, add:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct ToolRunStarterWire {
    pub agent: String,
    pub pid: i64,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub boot_id: Option<String>,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub process_start_identity: Option<String>,
}
```

Add `starter: Option<ToolRunStarterWire>` to `ToolRunBeginRequestWire` and `ToolRunWire`
in `wire.rs`, each with `#[serde(default, skip_serializing_if = "Option::is_none")]`.
Place both fields at the end of their structs. Missing `starter` must keep every
existing begin request, stored run, and golden fixture deserializing, and serializing a
run without a starter must omit the key.

`validate_handoff_begin` in `store/lifecycle.rs` gains these rules, checked before the
existing handoff rules return:

- `starter: None` is valid for foreground and handoff begins.
- `Some` starter with `launch_mode` other than `handoff` is `ToolRunError::Invalid` with
  a message that says starter requires `launch_mode` handoff.
- Trim `agent`. After trimming, an empty agent is invalid.
- `pid <= 0` is invalid.
- Store the trimmed agent. Leave `boot_id` and `process_start_identity` as supplied,
  including when they are absent.

`insert_run` writes `starter_json`. Serialize with `serde_json::to_string` when
`starter` is `Some`; otherwise bind SQL `NULL`. Do not store the JSON literal `null`.
Add the column to the `INSERT` column list and parameter list.

## Join record and APIs

Add these types to `handoff_wire.rs`. Request wires use `deny_unknown_fields`. Result
wires and the stored record stay lenient, matching `ToolRunStopRequestWire` /
`ToolRunStopResultWire` / `ToolRunStopRecordWire`. Schema version defaults through the
existing `handoff_schema_version()` helper.

Stored record, field name `join` on `ToolRunWire`:

```rust
pub struct ToolRunJoinRecordWire {
    pub kind: String,
    pub id: String,
    pub joined_ts: i64,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub requested_by: Option<String>,
}
```

`ToolRunWire.join` uses the same `default` + `skip_serializing_if` attribute as
`stop_request`.

Request `ToolRunJoinRequestWire`:

- `schema_version` (default 1)
- `run_id`
- `joiner_kind`
- `joiner_id`
- `agent: Option<String>`
- `requested_by: Option<String>`
- `now_ts: Option<i64>`

Result `ToolRunJoinResultWire`:

- `schema_version`
- `outcome`: `joined` | `refused` (`ToolRunJoinOutcomeWire`, snake_case)
- `refusal: Option<ToolRunJoinRefusalWire>`: `not_detached`, `settled`,
  `stop_requested`, `joined_elsewhere`, `agent_mismatch`
- `replayed: bool`
- `run: ToolRunWire`
- `diagnostics: Vec<String>` (default empty; leave it empty on every path)

Request `ToolRunReleaseJoinRequestWire`:

- `schema_version`
- `run_id`
- `joiner_kind`
- `joiner_id`
- `now_ts: Option<i64>`

The epic names the release behavior and not the request fields. These are the fields
required to match `kind` and `id`, plus the same `now_ts` `request_stop` uses for
`touch_write_meta`.

Result `ToolRunReleaseJoinResultWire`:

- `schema_version`
- `outcome`: `released` | `not_joined` | `joined_elsewhere`
- `run: ToolRunWire`
- `diagnostics: Vec<String>` (default empty)

No `replayed` field on release. The epic does not define one.

Implement `join` and `release_join` in `store/handoff.rs` beside `request_stop`. Each
call validates schema, rejects a blank `run_id`, trims `joiner_kind` and `joiner_id`,
and rejects either of those when empty after trimming, all before `with_write_store`. A
supplied `agent` that is empty after trimming is the same kind of `Invalid` error, not
`agent_mismatch`. Compare and store the trimmed joiner strings and the trimmed agent.
Leave `requested_by` as supplied.

Inside one `TransactionBehavior::Immediate` transaction, follow `request_stop`: unknown
`run_id` returns `ToolRunError::NotFound` with that run id. Do not encode not-found as a
refusal outcome. Neither function updates `state`.

`join` checks, in this order, and commits on every path:

1. `starter` is `None` → `refused` / `not_detached`. A handoff run that has no starter,
   and a foreground run, both take this path. A terminal foreground run is
   `not_detached`, not `settled`. An unreadable `starter_json` already becomes `None`
   plus a diagnostic in `load_run`, so join fails closed as `not_detached`.
2. `run.state.is_terminal()` → `refused` / `settled`. This wins over a stored stop
   request.
3. `stop_request` is `Some` → `refused` / `stop_requested`.
4. `agent` is `Some` and not equal to `starter.agent` → `refused` / `agent_mismatch`.
   Check this before the replay shortcut so a wrong agent never observes
   `replayed: true`.
5. `join` is `Some` with the same `kind` and `id` → `joined`, `replayed: true`. Do not
   rewrite `joined_ts` and do not call `touch_write_meta`. An omitted `agent` still
   replays.
6. `join` is `Some` with a different kind or id → `refused` / `joined_elsewhere`. Leave
   the stored join in place.
7. Otherwise write `join_json` to the serialized record (`kind`, `id`, `joined_ts` =
   `now_ts` or `unix_now()`, `requested_by`), call `touch_write_meta`, and return
   `joined` with `replayed: false`.

`release_join`, any run state, same transaction shape:

1. No join → `not_joined`. Do not touch meta.
2. Stored kind and id both match → `UPDATE` `join_json` to SQL `NULL`,
   `touch_write_meta`, outcome `released`. The returned run has `join: None`.
3. Otherwise → `joined_elsewhere`. Do not clear the join.

Refusal and release results carry `refusal: None` except join refusals, which set
`refusal` and `replayed: false`. An unreadable `join_json` is already `None` plus a
diagnostic from `load_run`; the next successful join overwrites that column, matching
the unreadable-stop behavior.

Export `join` and `release_join` from `store/mod.rs` and add them to the
`pub use store::{...}` list in `tool_run/mod.rs`. Do not add root re-exports in `lib.rs`
or `core_*` aliases in `sase_core_py`'s prelude.

## Envelope continuation

Add `continuation_mode: Option<String>` at the end of `ToolRunLaunchEnvelopeWire` with
`#[serde(default, skip_serializing_if = "Option::is_none")]`. Absent mode keeps existing
stored envelopes and fixtures byte-identical: serde omits the key on write and accepts
its absence on read.

In `validate_handoff_begin`, when the envelope is present, accept only `None` or exactly
`always`, `never`, or `known`. Any other string, including empty, is `Invalid`. Do not
interpret the three values. `claim` already returns the stored envelope from
`load_launch_envelope`; no claim branch is required for the field to round-trip inside
`launch`.

Every `ToolRunLaunchEnvelopeWire { ... }` literal needs `continuation_mode: None`. The
compiler will list them. Known sites today:

- `store/tests/handoff.rs` (`handoff_envelope` and one inline envelope)
- `store/tests/compat.rs`
- `store/tests/reconcile_owner.rs`

Every `ToolRunBeginRequestWire { ... }` literal needs `starter: None`. Known sites
today:

- `store/tests.rs`
- `store/tests/handoff.rs`
- `store/tests/compat.rs`
- `store/tests/receipt.rs`
- `store/tests/reconcile_owner.rs`
- `store/tests/triage.rs`
- `store/tests/triage_stage.rs`
- `store/receipts_report/tests.rs`
- `projection/tests.rs`

The only `ToolRunWire { ... }` literal is `load_run` in `store/query.rs`.

## Budget policy

In `tool_run/duration.rs`, add named constants and use them in the function. Do not
repeat the numbers as literals at the call site:

- numerator `15` and denominator `100` (15 percent, truncating integer division)
- floor `90` seconds
- cap `300` seconds

```rust
pub fn sync_wait_budget(
    request: SyncWaitBudgetRequestWire,
) -> Result<SyncWaitBudgetResponseWire, ToolRunError>
```

Request (`deny_unknown_fields`), modelled on `DurationFitRequestWire`:

- `schema_version` defaulting to `TOOL_RUN_WIRE_SCHEMA_VERSION`
- `ceiling_seconds: Option<u64>`
- `soft_ceiling_seconds: Option<u64>`

Response, modelled on `DurationFitResponseWire`:

- `schema_version`
- `budget_seconds: Option<u64>`
- `source: Option<SyncWaitBudgetSourceWire>` (`hard` | `soft`)
- `margin_seconds: Option<u64>`
- `ceiling_seconds: Option<u64>`
- `soft_ceiling_seconds: Option<u64>`

Optional response fields use `skip_serializing_if = "Option::is_none"`.

Rules:

- Wrong `schema_version` → `ToolRunError::SchemaVersion`.
- `Some(0)` for either ceiling → `Invalid`. Name the field. Check the hard ceiling
  first. A positive companion does not forgive a zero.
- `checked_mul` overflow on `ceiling * 15` → `Invalid`.
- Hard margin = `clamp(ceiling * 15 / 100, 90, 300)`.
- Hard budget = `max(ceiling.saturating_sub(margin), ceiling / 2)` with `u64` division.
- Soft budget = the soft ceiling. No margin.
- Budget = the smaller of the budgets that are present. Equal budgets report source
  `hard`.
- With neither ceiling, return success and leave every optional response field `None`.
  That is "no budget", not an error.
- Echo each supplied positive ceiling. `margin_seconds` is present only when a hard
  ceiling was supplied, including when the soft budget wins.

Pinned results:

| Input                  | budget | source | margin |
| ---------------------- | ------ | ------ | ------ |
| hard 600               | 510    | hard   | 90     |
| hard 14400             | 14100  | hard   | 300    |
| hard 60                | 30     | hard   | 90     |
| hard 1                 | 0      | hard   | 90     |
| soft 1200              | 1200   | soft   | absent |
| hard 600 and soft 200  | 200    | soft   | 90     |
| hard 600 and soft 510  | 510    | hard   | 90     |
| hard 600 and soft 1000 | 510    | hard   | 90     |
| neither                | absent | absent | absent |
| hard 0, or soft 0      | error  |        |        |

`hard 1` is the integer formula, not a special case: raw margin truncates to 0, the
floor raises it to 90, and both `saturating_sub` and `ceiling / 2` yield 0. Input zero
is the only zero that errors.

Export `sync_wait_budget` and its request, response, and source types from the
`pub use duration::{...}` list in `tool_run/mod.rs`.

## Store migration

Add nullable `starter_json TEXT` and `join_json TEXT` to `runs` in `SCHEMA_SQL` in
`store/connection.rs`, after `stop_request_json`. Add the same pair to the
`ensure_child_observation_columns` list so an older store gains them on first write.
Reads must not migrate.

In `load_run`, extend the existing missing-column `NULL` fallback. Append `starter_json`
and `join_json` as the next positional columns (indexes 46 and 47). Do not insert them
earlier; indexes 0 through 45 stay as they are. Parse each value the way
`stop_request_json` is parsed: SQL `NULL` and JSON `null` become `None`; a valid object
becomes the record; unreadable JSON pushes one diagnostic
(`stored starter was unreadable` or `stored join was unreadable`) and becomes `None`.

Leave `OLD_SCHEMA_SQL` and `OLD_RUN_COLUMNS` in `store/tests/compat.rs` without the new
columns. That file is the old-reader fixture. `load_run`'s `NULL` fallback is what makes
`old-1` readable.

## Tests and fixtures

Begin, in `store/tests/handoff.rs`:

- Handoff begin with a starter persists `agent`, `pid`, and the optional identity
  fields, and `show_run` returns them. `state` stays `created`.
- Starter without `launch_mode: handoff` errors, and no row is written.
- Empty or whitespace `agent` errors. `pid` of `0` and `-1` error.

Show and claim:

- `show_run` round-trips `starter` and `join` once a join exists.
- Begin with `continuation_mode: "known"`, then `claim`. `launch` on the claim result
  carries `Some("known")`. `"always"` and `"never"` are accepted. `"sometimes"` and `""`
  fail begin. A begin that omits the field still claims, and the returned envelope has
  `continuation_mode: None`.

Join and release matrix, same file, one IMMEDIATE-transaction behavior each:

- First join of an unsettled starter run: `joined`, `replayed` false, `join` records
  kind, id, `joined_ts`, and `requested_by`. `state` is unchanged.
- Same kind and id again: `joined`, `replayed` true, `joined_ts` unchanged.
- Different id, or same id with a different kind: `joined_elsewhere`, original join
  intact.
- Omitted `agent` succeeds. A different `agent` is `agent_mismatch` and writes nothing,
  including when the joiner kind and id match an existing join.
- No starter: `not_detached`, including a settled foreground run.
- `request_stop` then join: `stop_requested`, `join` stays absent.
- Finish, then join: `settled`, even if a stop record is also present.
- Unknown run id: `NotFound`, same error `request_stop` returns.
- Blank `joiner_kind` or `joiner_id`: `Invalid`, before a store write.
- Release of the matching joiner: `released`, `join` is `None`, `state` unchanged.
  Repeat release: `not_joined`.
- Release with a different id: `joined_elsewhere`, join remains.
- Release after finish still returns `released`.
- Release of an unknown run: `NotFound`.

Budget tests live in the existing `duration.rs` test module and cover the pinned table,
including input zero, schema-version mismatch, and neither ceiling.

Compat, in `old_shape_store_is_readable_and_migrates_on_write`:

- The loaded `old-1` run has `starter: None` and `join: None`.
- After the first `begin`, `PRAGMA table_info(runs)` contains `starter_json` and
  `join_json` alongside the columns that test already lists.

Golden fixtures beside `crates/sase_core/src/tool_run/fixtures/claim_request.json`:

- `join_request.json`
- `join_result.json`
- `release_join_request.json`
- `release_join_result.json`
- `begin_starter_request.json`

Pin them from `golden_handoff_fixtures_pin_the_wire_shape` the same way claim and stop
are pinned: `include_str!`, deserialize, assert one identifying field (`outcome` or
`starter.agent`). Build the result JSON from `serde_json::to_string_pretty` of a real
success value so `ToolRunWire` stays a faithful fixture, then commit that text.
`begin_starter_request.json` must parse as `ToolRunBeginRequestWire` and include a
handoff launch plus a starter. Request fixtures must pass `deny_unknown_fields`.

## Bindings

In `crates/sase_core_py/src/telemetry/mod.rs`, add three functions modelled on the
neighbours named below. Parse with `telemetry_request_from_pydict`, map `ToolRunError`
to `PyRuntimeError`, and return `telemetry_result_to_py`. Import core items as
`sase_core::tool_run::...`.

- `tool_run_join(store_path, request, busy_timeout_ms=250)` modelled on
  `py_tool_run_claim`.
- `tool_run_release_join(store_path, request, busy_timeout_ms=250)` modelled on
  `py_tool_run_request_stop`.
- `tool_run_sync_wait_budget(request)` modelled on `py_tool_run_duration_fit`. It takes
  no store path.

Register each with `m.add_function(wrap_pyfunction!(...))` inside `register_telemetry`.
A missed registration only fails later as `AttributeError`.

Extend `tool_run_bindings_round_trip_python_dicts` in
`crates/sase_core_py/src/telemetry/tests.rs`:

- `tool_run_sync_wait_budget` with `ceiling_seconds: 600` returns `budget_seconds` 510
  and `source` `hard`.
- A second handoff begin in that test's store carries a starter and
  `continuation_mode: "always"`. `tool_run_join` returns `joined` and the run's
  `starter.agent` and `join.kind`. `tool_run_release_join` returns `released` and the
  run omits `join`.
- A request with `ceiling_seconds: 0` raises.

## Verification

From the sase-core checkout:

- While iterating, `just test -p sase_core` with a filter covering the handoff, compat,
  and duration tests, then `just test -p sase_core_py` filtered to
  `tool_run_bindings_round_trip_python_dicts`. A targeted run does not replace the gate.
  `-p sase_core` alone skips the binding tests.
- `just fmt` before the gate if the tree needs it.
- Gate: `sase tool run check` in that checkout. It takes about 5 minutes. Give the
  command a timeout of at least 10 minutes. Do not run `check-full`. Do not run bare
  `just check` or bare `cargo`.
- A test that fails under the gate and passes alone is a load flake. Look for its
  `sase-core flake:` bead before treating it as this change. Do not weaken an assertion
  to go green.

The host commit subject, written by the finalizer for the sase-core repository, is:

`feat(tool-run): add detached starter scope, monitor join, and sync wait budget`

Before closing, run `sase bead epic-symbols sase-1cx.1`. There were no `--epic-symbol`
entries when this plan was written. If any appear, re-key each Justfile line to a
still-open bead (parent `sase-1cx` or a later phase). `sase bead close` refuses while
leftovers remain.

Close only `sase-1cx.1`:

`sase bead close sase-1cx.1 --note "<what sase tool run check and the new tests verified>"`

Do not close parent epic `sase-1cx`, bead `sase-17g`, or any ancestor plan bead. Do not
create beads. Record any discovered follow-up with
`sase bead note sase-1cx.1 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`. A check
failure that reproduces on the clean base tree is that kind of note, citing any task
bead that already tracks it, and does not keep this phase open.
