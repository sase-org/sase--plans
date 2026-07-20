---
tier: epic
title: Priority-aware runner-slot queue via %wait(priority=...)
goal: 'The %wait directive gains an integer priority= keyword (default 10) that orders
  SASE''s max-runner slot queue: lower values are admitted first, and equal priorities
  keep today''s longest-waiting-first FIFO order. As the first consumer, the toobig_split
  chop in bugyi-chops launches all of its split agents at priority 20.

  '
phases:
- id: core-wire
  title: Rust core waiting-marker wire and %wait completion
  depends_on: []
  size: small
  description: '''Rust core: waiting-marker wire and editor completion'' section:
    add wait_priority to the agent-scan WaitingMarkerWire and scanner, extend the
    parity test, and offer priority= in %wait editor completions.'
- id: priority-directive
  title: Parse %wait(priority=...) and make admission priority-aware
  depends_on:
  - core-wire
  size: medium
  description: '''SASE: %wait(priority=...) directive and priority-aware admission''
    section: parse and validate the new keyword, thread it through agent meta and
    the waiting marker, sort the runner-slot queue by priority ahead of wait time,
    and cover parsing, admission, locked-gate, and fakey e2e tests.'
- id: toobig-priority
  title: toobig_split proposals launch at priority 20
  depends_on:
  - priority-directive
  size: small
  description: '''bugyi-chops: toobig_split proposals at priority 20'' section: add
    %wait(priority=20) to every toobig_split proposal prompt in the bugyi-chops repo
    and update its tests.'
create_time: 2026-07-20 14:00:01
status: done
bead_id: sase-8c
---

# Plan: Priority-aware runner-slot queue via `%wait(priority=...)`

## Context

SASE caps concurrently running root agents at `max_running_agents` (default 10, from `src/sase/default_config.yml`).
When the cap is hit, each launching root agent parks by writing a `waiting.json` marker with a `slot_requested_at`
timestamp and polls the global runner-slot gate:

- Every root agent passes through `wait_for_runner_slot` → `_try_claim_runner_slot` in `src/sase/axe/run_agent_wait.py`,
  which takes the `~/.sase/runner_slots.lock` file lock, scans live agents through the Rust-backed
  `scan_agent_artifacts`, and asks `may_start` whether this agent is the first eligible waiter.
- Queue ordering lives in `src/sase/core/runner_slots/_admission.py`: `live_runner_slot_waiters` builds
  `RunnerSlotWaiter` entries from the scanned `waiting.json` projections and `_waiter_sort_key` sorts them
  `(invalid, slot_requested_at, timestamp, artifact_dir)` — pure FIFO by wait start.
- The `waiting.json` fields reach Python through the agent-scan wire: the Rust scanner
  (`crates/sase_core/src/agent_scan/scanner.rs`, `waiting_marker_from_object`) parses a fixed set of keys into
  `WaitingMarkerWire` (`crates/sase_core/src/agent_scan/wire.rs`), mirrored by the tolerant Python dataclass in
  `src/sase/core/agent_scan_wire_markers.py`. Unknown JSON keys are ignored in both directions, so an additive field is
  backward and forward compatible.

The `%wait` directive already supports keyword args `bead=`, `runners=`, and `time=`
(`src/sase/xprompt/_directive_collect.py`, `_directive_values.py`), which flow through `PromptDirectives` → `AgentInfo`
(`src/sase/axe/run_agent_directives.py`) → `agent_meta.json` and the runner (`src/sase/axe/run_agent_runner.py`).

This plan adds a fourth keyword, `priority=`, that controls **which parked agent is admitted first** when a slot frees
up.

## Semantics and design decisions

- **Spelling:** `%wait(priority=<int>)` (alias `%w(priority=<int>)`), combinable with the existing kwargs and positional
  agent names, e.g. `%wait(builder, time=5m, priority=5)`. Validation mirrors `runners=`: a single non-negative integer;
  duplicates across `%wait` occurrences are rejected (`Multiple %wait(priority=...) values are not allowed`).
- **Ordering rule:** lower value = admitted sooner. Default is **10** when unset (define `DEFAULT_WAIT_PRIORITY = 10` in
  `src/sase/core/runner_slots/`). Priority dominates wait time: the queue sort key becomes
  `(priority, invalid, slot_requested_at, timestamp, artifact_dir)`, so equal priorities preserve today's
  longest-waiting-first order, and unparseable `slot_requested_at` values still sort last within their priority band.
  `may_start` itself is unchanged — it already picks the first _eligible_ entry of the sorted queue, so an ineligible
  high-priority waiter (per-agent `runners=` threshold not yet satisfiable) still does not block eligible lower-priority
  waiters.
- **Persistence and override:** the resolved priority is written into the parked `waiting.json` marker as
  `wait_priority` (naming consistent with the existing `wait_runners` key) and into `agent_meta.json` as `wait_priority`
  when the directive was given. Marker resolution mirrors `_marker_threshold`: a valid non-negative integer already
  present in a parked marker wins over the directive value, so a manual (or future TUI) edit of `waiting.json`
  re-prioritizes a parked agent live; otherwise the directive value applies, else the default 10. No
  `wait_priority_explicit` flag is needed because the default is a constant, unlike the live-config-derived runners
  threshold. The TUI wait-edit flow (`_directive_persistence.py`) patches specific keys rather than rewriting the
  marker, so an existing `wait_priority` survives TUI edits.
- **Old markers / mixed versions:** markers and agent metas written before this change simply lack the key; the wire
  defaults it to `None` and admission treats that as priority 10, which preserves current behavior exactly.
- **Deferred-start interplay:** `has_deferred_start_directive` already treats any `%wait` token as deferring the
  workspace claim, so a priority-only prompt (e.g. `%wait(priority=20)`) launches with
  `SASE_AGENT_DEFERRED_WORKSPACE=1`. The runner guard at `src/sase/axe/run_agent_runner.py` (`has_wait`, currently
  `has_dependency_wait or info.wait_runners is not None`) must also accept `wait_priority is not None`, otherwise a
  priority-only prompt would hit the "refusing to continue in the placeholder workspace" RuntimeError. This deferral is
  desirable: a priority-queued agent claims its numbered workspace only after winning a slot (`claim_deferred_workspace`
  runs after `wait_for_runner_slot`).
- **Accepted side effects:** like the other kwargs-only `%wait` forms, a priority-only `%wait` in a multi-segment fanout
  marks the slot `wait_for_previous` for launch ordering (Rust `has_wait_directive` in
  `crates/sase_core/src/agent_launch/mod.rs` only checks token presence). `has_bare_wait_directive`
  /`rewrite_bare_wait_directives` already skip paren forms with non-empty content, so `%wait(priority=20)` is never
  rewritten into a dependency on the previous agent. No changes needed for either.
- **Chop wiring:** chop proposal prompts flow through normal directive extraction at agent startup (the scaffolding in
  `src/sase/axe/chop_proposals.py` prepends `%id`/ `%model`/`%wait:<dep>` lines and appends the chop's own prompt text,
  and `%wait` is multi-valued), so toobig_split can adopt the feature by splicing `%wait(priority=20)` into its proposal
  prompt string. Adding a structured `priority` field to the chop SDK's `launch_proposal` was considered and rejected
  for now — the prompt already carries directives (`%auto`) and a one-repo text change is enough for the first consumer.

## Phase 1 — Rust core: waiting-marker wire and editor completion

All work in the `sase-core` repo (open it with `sase repo open sase-core`), crate `crates/sase_core`:

- `src/agent_scan/wire.rs`: add `pub wait_priority: Option<i64>` (serde default) to `WaitingMarkerWire`, next to
  `wait_runners`.
- `src/agent_scan/scanner.rs`: parse it in `waiting_marker_from_object` via `coerce_int(data.get("wait_priority"))`.
- `tests/agent_scan_parity.rs`: extend `waiting_marker_carries_runner_slot_fields` (and the fixture writer
  `build_waiting`) to cover the new key, including its absence defaulting to `None`.
- Editor surfaces: add `("priority=", ...)` to the `%wait` keyword completion tables in `src/editor/directive.rs` and
  `src/editor/completion.rs`, and update the completion tests that assert the expected kwarg lists (e.g. the
  `["time=", "runners=", ...]` expectations).
- Run the crate's test suite and any binding build the repo's own instructions require.

The scan-wire change is additive and tolerated by older Python readers, so this phase can land independently ahead of
Phase 2.

## Phase 2 — SASE: `%wait(priority=...)` directive and priority-aware admission

All work in the main sase repo.

**Directive parsing** (`src/sase/xprompt/`):

- `_directive_collect.py`: extend the `%wait` paren branch — `supported_keys` becomes
  `{"bead", "priority", "runners", "time"}`, route `named_args["priority"]` into a new
  `_CollectedDirectives.wait_priority_args` list, and update the unsupported-keyword error text to enumerate the four
  kwargs.
- `_directive_values.py`: add `resolve_wait_priority_args` modeled on `resolve_wait_runners_args` (single occurrence,
  non-negative integer, targeted DirectiveError messages).
- `_directive_types.py`: add `wait_priority: int | None = None` to `PromptDirectives` with a docstring entry.
- `_directive_extract.py`: resolve and populate the new field.

**Runner plumbing** (`src/sase/axe/`):

- `run_agent_directives.py`: add `wait_priority` to `AgentInfo`, write `agent_meta["wait_priority"]` when set, and
  return it.
- `run_agent_runner.py`: include `info.wait_priority is not None` in the `has_wait` deferred-workspace guard and pass
  `wait_priority=info.wait_priority` to `wait_for_runner_slot`.
- `run_agent_wait.py`: thread a `wait_priority` parameter through `wait_for_runner_slot` → `_try_claim_runner_slot`; add
  a `_marker_priority` resolver mirroring `_marker_threshold` (parked marker's valid value wins, then directive, then
  `DEFAULT_WAIT_PRIORITY`); always write the resolved `"wait_priority"` into the queue marker dict so the existing
  `waiting_data != marker` comparison republishes edits.

**Admission** (`src/sase/core/runner_slots/_admission.py`):

- Define `DEFAULT_WAIT_PRIORITY = 10` (export via the package as needed by `run_agent_wait.py`).
- Add `priority: int = DEFAULT_WAIT_PRIORITY` to `RunnerSlotWaiter`; populate it in `live_runner_slot_waiters` from
  `waiting.wait_priority` with the same type/non-negative guard used for `threshold`.
- Lead `_waiter_sort_key` with the priority value as described in the design section.

**Wire mirror and TUI hints:**

- `src/sase/core/agent_scan_wire_markers.py`: add `wait_priority: int | None = None` to the Python `WaitingMarkerWire`
  (rehydration via `known_field_kwargs` needs no other change); check `tests/agent_scan_golden/fixture_builder.py` and
  the wire conversion round-trip tests for fixtures that should now include the field.
- `src/sase/ace/tui/widgets/directive_completion.py`: extend the `%wait` usage hint and the kwarg suggestion list with
  `priority=` (lower runs first; default 10), updating any tests that assert those strings.

**Testing:**

- Parsing: extend `tests/test_directives_wait.py` — value sets `wait_priority`, non-integer/negative values raise,
  duplicates raise, combinable with `time=`/ `runners=`/positional names, and update `test_wait_unknown_keyword_raises`
  for the new error text.
- Admission units: extend `tests/test_runner_slots.py` — a later-parked priority-5 waiter beats an earlier priority-10
  waiter; equal priorities keep FIFO; missing priority defaults to 10; an ineligible high-priority waiter still doesn't
  block eligible lower-priority ones.
- Locked gate: extend `tests/test_run_agent_runner_slots.py` — the parked marker carries `wait_priority` from the
  directive; a marker edit re-prioritizes (mirroring `test_parked_marker_edit_overrides_original_directive`); a
  priority-only deferred-workspace launch passes the placeholder guard.
- End-to-end: add one case to `tests/fakey/test_runner_slots_e2e.py` asserting release order honors priority ahead of
  park order.
- Run `just check` before finishing, per repo policy.

## Phase 3 — bugyi-chops: toobig_split proposals at priority 20

In the external `bugyi-chops` repo (open it with `sase repo open gh:bbugyi200/bugyi-chops`):

- `src/bugyi_chops/toobig_split.py`: change the proposal prompt in `build_result` from `f"%auto #split_file:{path}"` to
  `f"%auto %wait(priority=20) #split_file:{path}"`, introducing a named constant (e.g. `LAUNCH_PRIORITY = 20`).
- `tests/test_toobig_split.py`: update the expected proposal prompts.
- Run that repo's own checks (see its `justfile`).

This phase must land only after Phase 2 is deployed on the host: an older sase parser rejects the unknown `priority=`
keyword at agent startup with a DirectiveError.

## Risks and edge cases

- **Starvation:** a steady stream of default-priority (10) launches can starve priority-20 toobig agents indefinitely.
  That is the requested semantic ("lower priority first, regardless of submission time"); no aging mechanism is added.
- **Mixed-version window:** between Phases 1 and 2 landing, markers never contain `wait_priority`, and after Phase 2,
  older binaries ignore the extra key — behavior degrades to today's FIFO, never to an error.
- **Marker hygiene:** the marker always records the resolved priority, so operators can inspect and hand-edit queue
  order under the runner-slot lock semantics that already govern `wait_runners` edits.

## Non-goals

- No TUI rendering of queue priority (Agents-tab waiting hints, wait modal `w` editor) and no new keybindings; the TUI
  keeps working unchanged because marker edits are key-targeted. A follow-up can add a priority field to the wait modal.
- No config knob for the default priority; it is the constant 10.
- No structured `priority` field on the chop-proposal SDK/schema.
- No change to `%wait(runners=...)`, dependency waits, time floors, or the `max_running_agents` cap itself.
