---
tier: tale
title: Land the unlanded sase-core detach join
goal: "Land phase sase-1cx.1 by applying the already verified sase-core patch for the
  ToolRun starter record, atomic join and release-join, envelope continuation_mode, and
  sync wait budget, then re-running the sase-core gate and closing only that phase bead.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1cx.1
bead: sase-1cx.1
create_time: 2026-09-30 07:25:38
status: wip
---

- **PARENT:**
  [202609/tool_run_escalation.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_run_escalation.md)
- **BEAD:**
  [sase-1cx.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cx/sase-1cx.1.md)

# Plan: Land the unlanded sase-core detach join

Phase `core-detach-join` was implemented and verified once, then the host commit failed
before any commit landed. Bead note #1 records that verification. Bead note #2 attaches
the patch and note #3 says to apply it and re-verify. This plan is that relaunch. Do not
redesign the feature and do not retype it when the patch applies.

## Where the bytes are

On 2026-09-30 the patch applied cleanly with `git apply --check` onto sase-core
`a354a8a96c06443fb2ed47b5699e9ae2142862d7`. That commit is one commit after the patch
base `1e51ff3ce9c53ee1a4bc9f52c3642ac4eea8f423`. The only newer commit is
`feat(agent-tab): name the default tab from the local machine alias`, and it does not
touch any patched path. `sase bead epic-symbols sase-1cx.1` printed no `--epic-symbol`
entries.

The patch is bead note #2's attachment `sase-1cx.1-core.patch`, sha256 prefix
`d591cf58fde6`, at
`/home/bryan/.sase/attachments/views/d591cf58fde68d05/sase-1cx.1-core.patch`. It is 25
files: 20 tracked `tool_run` edits plus five new fixture JSON files. Keep working from
that attachment if the view path is gone. Read the bead with
`sase bead read sase-1cx.1 -r "Confirm the unlanded patch and close rules"`.

## Apply

Open the linked checkout and work only in the printed path:

```bash
sase repo open sase-core -r "Land starter scope, monitor join, and sync wait budget"
```

Read that checkout's `AGENTS.md` before editing. Follow its recipe "Add a core function
and expose it to Python" only for a hunk that does not apply. Never run bare `cargo`.
Never edit a `version` field, a path-dependency pin, or a `CHANGELOG.md`. Leave the sase
checkout unchanged. This phase does not move `sase-core-revision.txt`.

Confirm the sase-core tree is clean. If `HEAD` is still
`a354a8a96c06443fb2ed47b5699e9ae2142862d7`, apply the patch:

```bash
git apply --check <patch>
git apply <patch>
```

If `HEAD` has moved or `--check` fails, apply with `git apply --3way` and resolve
conflicts so the result still meets the contract below. `--3way` implies `--index`; do
not commit. The host finalizer commits. Do not restyle or rewrite hunks that applied.
Run `just fmt` only when the tree needs it.

If the patch cannot be applied at all, implement the contract below in place. The
approved write-up is `plan:202609/core_detach_join.md` (`sase artifact read`). Match it.
Do not invent a second design.

## Contract the patch already implements

Keep `TOOL_RUN_WIRE_SCHEMA_VERSION` at 1. Result wires stay lenient. The commit subject
is not `feat!`: released sase never sends the new fields. Python CLI, the feature flag,
the watchdog, and docs belong to later epic phases. Leave `tool_run/projection/` queries
unchanged. `show`, `claim`, and `list` already return `ToolRunWire` through `load_run`,
so `starter` and `join` appear once `load_run` reads the new columns.

### Starter

`ToolRunStarterWire` in `crates/sase_core/src/tool_run/handoff_wire.rs` denies unknown
fields and has `agent: String`, `pid: i64`, and optional `boot_id` and
`process_start_identity` (skip the key when absent). Add
`starter: Option<ToolRunStarterWire>` at the end of `ToolRunBeginRequestWire` and
`ToolRunWire` in `wire.rs`, with `default` and `skip_serializing_if` so a missing
starter still deserializes and a run without one omits the key.

`validate_handoff_begin` in `store/lifecycle.rs`, before the existing handoff rules
return:

- `starter: None` is valid for foreground and handoff begins.
- `Some` starter with `launch_mode` other than `handoff` is `ToolRunError::Invalid` and
  the message says starter requires `launch_mode` handoff.
- Trim `agent`. Empty after trim is invalid. `pid <= 0` is invalid.
- Store the trimmed agent. Leave `boot_id` and `process_start_identity` as supplied,
  including when absent.

`insert_run` writes `starter_json`: `serde_json::to_string` when `Some`, SQL `NULL`
otherwise. Do not store the JSON literal `null`.

### Join and release

Stored `ToolRunJoinRecordWire` (`kind`, `id`, `joined_ts`, optional `requested_by`) is
field `join` on `ToolRunWire`, same skip attribute as `stop_request`. Request wires deny
unknown fields. Result wires stay lenient. Schema version defaults through
`handoff_schema_version()`.

`ToolRunJoinRequestWire`: `schema_version`, `run_id`, `joiner_kind`, `joiner_id`,
optional `agent`, `requested_by`, and `now_ts`. `ToolRunJoinResultWire`:
`schema_version`, `outcome` (`joined` | `refused`), `refusal` (`not_detached`,
`settled`, `stop_requested`, `joined_elsewhere`, `agent_mismatch`), `replayed`, `run`,
and `diagnostics` (default empty; leave empty on every path).

`ToolRunReleaseJoinRequestWire`: `schema_version`, `run_id`, `joiner_kind`, `joiner_id`,
optional `now_ts`. `ToolRunReleaseJoinResultWire`: `schema_version`, `outcome`
(`released` | `not_joined` | `joined_elsewhere`), `run`, and `diagnostics`. No
`replayed` field on release.

Implement `join` and `release_join` in `store/handoff.rs` beside `request_stop`. Each
validates schema, rejects a blank `run_id`, trims `joiner_kind` and `joiner_id`, and
rejects either when empty after trim, all before `with_write_store`. A supplied `agent`
that is empty after trim is `Invalid`, not `agent_mismatch`. Compare and store the
trimmed joiner strings and the trimmed agent. Leave `requested_by` as supplied.

Inside one `TransactionBehavior::Immediate` transaction, unknown `run_id` is
`ToolRunError::NotFound`. Do not encode not-found as a refusal. Neither function updates
`state`.

`join` checks, in this order, and commits on every path:

1. `starter` is `None` → `refused` / `not_detached`. A handoff run with no starter and a
   foreground run both take this path. A terminal foreground run is `not_detached`, not
   `settled`. Unreadable `starter_json` is already `None` plus a diagnostic, so join
   fails closed as `not_detached`.
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
7. Otherwise write `join_json` (`kind`, `id`, `joined_ts` = `now_ts` or `unix_now()`,
   `requested_by`), call `touch_write_meta`, and return `joined` with `replayed: false`.

`release_join`, any run state:

1. No join → `not_joined`. Do not touch meta.
2. Stored kind and id both match → set `join_json` to SQL `NULL`, `touch_write_meta`,
   outcome `released`, returned `join: None`.
3. Otherwise → `joined_elsewhere`. Do not clear the join.

Join refusals set `refusal` and `replayed: false`. Other results carry `refusal: None`.
Export `join` and `release_join` from `store/mod.rs` and from the `pub use store::{...}`
list in `tool_run/mod.rs`. Do not add root re-exports in `lib.rs` or `core_*` aliases in
`sase_core_py`'s prelude.

### Continuation mode

Add `continuation_mode: Option<String>` at the end of `ToolRunLaunchEnvelopeWire` with
the same skip attribute. Absent mode keeps existing envelopes and fixtures
byte-identical. When the envelope is present, `validate_handoff_begin` accepts only
`None` or exactly `always`, `never`, or `known`. Any other string, including empty, is
`Invalid`. Do not interpret the three values. `claim` already returns the stored
envelope. Every `ToolRunLaunchEnvelopeWire { ... }` literal needs
`continuation_mode: None`, and every `ToolRunBeginRequestWire { ... }` literal needs
`starter: None`. The only `ToolRunWire { ... }` literal is `load_run` in
`store/query.rs`.

### Budget

In `tool_run/duration.rs`, add `sync_wait_budget` and use named constants for numerator
`15`, denominator `100`, floor `90` seconds, and cap `300` seconds. Do not repeat those
numbers as literals at the call site.

Request (`deny_unknown_fields`), modelled on `DurationFitRequestWire`: `schema_version`
defaulting to `TOOL_RUN_WIRE_SCHEMA_VERSION`, `ceiling_seconds: Option<u64>`,
`soft_ceiling_seconds: Option<u64>`.

Response, modelled on `DurationFitResponseWire`: `schema_version`, `budget_seconds`,
`source` (`hard` | `soft`), `margin_seconds`, `ceiling_seconds`, and
`soft_ceiling_seconds`. Optional response fields skip when `None`.

Rules:

- Wrong `schema_version` → `ToolRunError::SchemaVersion`.
- `Some(0)` for either ceiling → `Invalid`. Name the field. Check the hard ceiling
  first.
- `checked_mul` overflow on `ceiling * 15` → `Invalid`.
- Hard margin = `clamp(ceiling * 15 / 100, 90, 300)`.
- Hard budget = `max(ceiling.saturating_sub(margin), ceiling / 2)`.
- Soft budget = the soft ceiling. No margin.
- Budget = the smaller of the budgets that are present. Equal budgets report source
  `hard`.
- With neither ceiling, succeed and leave every optional response field `None`.
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

`hard 1` is the integer formula: raw margin truncates to 0, the floor raises it to 90,
and both `saturating_sub` and `ceiling / 2` yield 0. Export `sync_wait_budget` and its
request, response, and source types from the `pub use duration::{...}` list in
`tool_run/mod.rs`.

### Store

Add nullable `starter_json TEXT` and `join_json TEXT` to `runs` in `SCHEMA_SQL` in
`store/connection.rs`, after `stop_request_json`. Add the same pair to
`ensure_child_observation_columns` so an older store gains them on first write. Reads
must not migrate.

In `load_run`, extend the missing-column `NULL` fallback. Append `starter_json` and
`join_json` as indexes 46 and 47. Do not insert them earlier. Parse each the way
`stop_request_json` is parsed: SQL `NULL` and JSON `null` become `None`; a valid object
becomes the record; unreadable JSON pushes `stored starter was unreadable` or
`stored join was unreadable` and becomes `None`.

Leave `OLD_SCHEMA_SQL` and `OLD_RUN_COLUMNS` in `store/tests/compat.rs` without the new
columns.

### Tests and fixtures the patch adds

In `store/tests/handoff.rs`: starter persistence and validation; `show_run` round-trip
of `starter` and `join`; `continuation_mode` `known` / `always` / `never` through
`claim`, rejection of `sometimes` and `""`, and `None` when omitted; the join and
release matrix (first join, replay, `joined_elsewhere`, omitted agent, `agent_mismatch`
that writes nothing, `not_detached` including a settled foreground run,
`stop_requested`, `settled` winning over a stop record, `NotFound`, blank joiner ids,
release and repeat release, release after finish).

Budget tests in the existing `duration.rs` test module cover the pinned table, input
zero, schema-version mismatch, and neither ceiling.

Compat test `old_shape_store_is_readable_and_migrates_on_write`: loaded `old-1` has
`starter: None` and `join: None`; after the first `begin`, `PRAGMA table_info(runs)`
contains `starter_json` and `join_json`.

Golden fixtures beside `claim_request.json`: `join_request.json`, `join_result.json`,
`release_join_request.json`, `release_join_result.json`, and
`begin_starter_request.json`. Pin them from `golden_handoff_fixtures_pin_the_wire_shape`
with `include_str!`, deserialize, and assert one identifying field. Request fixtures
must pass `deny_unknown_fields`.

### Bindings

In `crates/sase_core_py/src/telemetry/mod.rs`, add three functions. Parse with
`telemetry_request_from_pydict`, map `ToolRunError` to `PyRuntimeError`, and return
`telemetry_result_to_py`. Import core items as `sase_core::tool_run::...`.

- `tool_run_join(store_path, request, busy_timeout_ms=250)` modelled on
  `py_tool_run_claim`.
- `tool_run_release_join(store_path, request, busy_timeout_ms=250)` modelled on
  `py_tool_run_request_stop`.
- `tool_run_sync_wait_budget(request)` modelled on `py_tool_run_duration_fit`. It takes
  no store path.

Register each with `m.add_function(wrap_pyfunction!(...))` inside `register_telemetry`.

Extend `tool_run_bindings_round_trip_python_dicts`:

- `ceiling_seconds: 600` returns `budget_seconds` 510 and `source` `hard`.
- A second handoff begin carries a starter and `continuation_mode: "always"`.
  `tool_run_join` returns `joined` and the run's `starter.agent` and `join.kind`.
  `tool_run_release_join` returns `released` and the run omits `join`.
- `ceiling_seconds: 0` raises.

## Verify

From the sase-core checkout:

- While iterating, `just test -p sase_core` with a filter covering the handoff, compat,
  and duration tests, then `just test -p sase_core_py` filtered to
  `tool_run_bindings_round_trip_python_dicts`. A targeted run does not replace the gate.
  `-p sase_core` alone skips the binding tests.
- Gate: `sase tool run check` in that checkout. Give it at least 10 minutes. Do not run
  `check-full`, bare `just check`, or bare `cargo`.
- A test that fails under the gate and passes alone is a load flake. Look for its
  `sase-core flake:` bead before treating it as this change. Do not weaken an assertion
  to go green.
- Do not trust bead note #1. Re-run the gate on the tree you applied.

The host commit subject for the sase-core repository is:

`feat(tool-run): add detached starter scope, monitor join, and sync wait budget`

## Close only this phase

Before closing, run `sase bead epic-symbols sase-1cx.1`. There were no `--epic-symbol`
entries when this plan was written. If any appear, re-key each Justfile line to a
still-open bead (parent `sase-1cx` or a later phase). `sase bead close` refuses while
leftovers remain.

Close only `sase-1cx.1`:

`sase bead close sase-1cx.1 --note "<what sase tool run check and the new tests verified>"`

Do not close parent epic `sase-1cx`, bead `sase-17g`, or any ancestor plan bead. Do not
create beads. Record a discovered follow-up with
`sase bead note sase-1cx.1 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`. A check
failure that reproduces on the clean base tree is that kind of note, citing any task
bead that already tracks it, and does not keep this phase open.
