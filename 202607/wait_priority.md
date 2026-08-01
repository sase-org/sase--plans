---
tier: epic
title: Make %wait(priority=N) effective, observable, and editable
goal: 'A deprioritized agent (%wait with a priority above the default) reliably yields
  runner-slot capacity to normal-priority work that becomes runnable moments later,
  and the priority that decided a queue outcome is visible and editable from ACE.

  '
phases:
- id: deference
  title: Bounded deference window for deprioritized waiters
  depends_on: []
  size: medium
  description: '''Bounded deference window for deprioritized waiters'' section: add
    a priority-scaled, bounded hold-back gate so a below-default-priority waiter does
    not claim a freshly freed runner slot while a better-priority agent is still finishing
    its dependency wait, with an early exit when no such agent exists.

    '
- id: explicit-flag
  title: wait_priority_explicit marker symmetry
  depends_on:
  - deference
  size: small
  description: '"''wait_priority_explicit marker symmetry'' section: record whether
    a parked agent''s priority was user-specified or defaulted, mirroring wait_runners_explicit,
    so a marker written with the implicit default no longer shadows a later directive
    change."

    '
- id: ace-display
  title: Surface wait priority in ACE
  depends_on:
  - explicit-flag
  size: small
  description: '''Surface wait priority in ACE'' section: render the explicit wait
    priority (and deference state) on queued agent list rows and in the agent detail
    pane''s Wait line, including the render-cache key fix.

    '
- id: wait-modal
  title: Edit wait priority from the ACE wait modal
  depends_on:
  - explicit-flag
  size: medium
  description: '''Edit wait priority from the ACE wait modal'' section: add a priority
    field to the ACE wait modal and its directive/marker persistence path, and make
    sure editing other wait fields never clobbers an existing wait_priority.

    '
create_time: 2026-07-25 10:38:22
status: wip
bead_id: sase-9k
---

- **PROMPT:** [prompts/202607/wait_priority.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/wait_priority.md)
- **BEAD:** [sase-9k](https://github.com/sase-org/sase--beads/blob/main/pages/sase-9k/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-9k.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9k.land/README.md)
  - [bbugyi200.athena.sase-9k.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-9k.land.md#member-code)
- **COMMITS:**
  - [4b9281d](https://github.com/sase-org/sase/commit/4b9281d3d7d92f0de8a03c8bdea802d28eea6901) — docs: document bounded runner-slot deference (sase-9k)

# Plan: Make `%wait(priority=N)` effective, observable, and editable

## Background: what is already correct

This work started from a report that the `research.k.final` agent, which declares `%wait(priority=20)`, started running
while the default-priority `sase-95.4` agent stayed queued. That specific outcome was **not** a bug in the priority
implementation. Do not re-litigate the following; it was verified against live artifacts under
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/`:

- The full path works today: `%wait(priority=N)` → `DirectiveInfo.wait_priority` → `agent_meta.json["wait_priority"]` →
  `waiting.json["wait_priority"]` → `RunnerSlotWaiter.priority`.
- `_waiter_sort_key` in `src/sase/core/runner_slots/_admission.py` sorts ascending on
  `(priority, invalid, slot_requested_at, timestamp, artifact_dir)`, with `DEFAULT_WAIT_PRIORITY = 10`. Lower numbers
  start first, as documented in `src/sase/xprompt/_directive_types.py` ("Lower values start first").
- Priority survives the pending-question slot yield and reacquire (`src/sase/axe/run_agent_exec_questions.py:169`).

The incident timeline (UTC) explains itself:

| time         | event                                                                                            |
| ------------ | ------------------------------------------------------------------------------------------------ |
| 14:15:30.442 | `sase-95.3` stops (`stopped_at`), freeing a runner slot                                          |
| 14:15:32.867 | `research.k.final` claims the slot (`run_started_at`); parked since `wait_completed_at` 14:04:44 |
| 14:15:35.229 | `sase-95.4` finishes its dependency wait (`wait_completed_at`) — only now can it park for a slot |
| 14:18:12.322 | `sase-95.4` starts                                                                               |

`sase-95.4` entered the slot queue 2.4 seconds _after_ the priority-20 agent had already claimed. The two were never in
the queue together, so the sort never compared them.

## The actual problem

Priority only arbitrates among agents that are **already parked** with `slot_requested_at` at the instant a slot frees.
Dependency-chained work — a clan chain such as `sase-95.1 → .2 → .3 → .4`, where each member waits on its predecessor —
joins the runner-slot queue only after its dependency wait resolves, which lags the predecessor's exit by seconds (5s in
the case above: predecessor stopped at 14:15:30.4, successor's wait completed at 14:15:35.2).

The runner polls every `_RUNNER_SLOT_POLL_INTERVAL = 2` seconds (`src/sase/axe/run_agent_wait.py`). A long-parked waiter
therefore takes a freed slot within ~2 seconds, well before a chained successor can park. A deprioritized background
agent reliably wins that race against exactly the normal-priority work it was supposed to yield to, which makes
`priority` ineffective in the case users reach for it.

Three supporting gaps found in the same audit:

- **Observability.** `wait_priority` is never rendered in ACE. The detail pane shows
  `runners: N/M in use · eligible #P of Q` with no priority; queued list rows show the `▶10/10` marker with no priority.
  Only `sase agent list` JSON (`src/sase/agents/cli_list.py:86`) exposes it. Nothing on screen explains why the queue
  ordered the way it did.
- **Editability.** The ACE wait modal (`w`) supports agents, time, beads, and runners, but not priority, so a parked
  agent cannot be promoted or demoted.
- **Explicit-flag asymmetry.** `wait_runners` has a `wait_runners_explicit` marker flag; `wait_priority` has none. Once
  an agent parks, `_marker_priority` prefers the marker value, so a marker holding the implicit default `10` is
  indistinguishable from a deliberate `priority=10` and a later directive change can never take effect.

## Design decision and its tradeoff

**Deprioritized waiters defer; nobody else does.** A waiter whose priority is numerically _worse_ than
`DEFAULT_WAIT_PRIORITY` does not claim a slot the moment it becomes eligible. It must be _continuously_ eligible for a
bounded window first. Any better-priority waiter that parks during the window wins through the existing sort.

Default-priority and better-priority waiters never hold back. This asymmetry is deliberate: you cannot defer to work
that may never arrive, so only an agent that explicitly volunteered to be deprioritized pays a delay.

The honest tradeoff: a deprioritized agent can start up to the window later than it otherwise would, in exchange for
`priority` meaning something across time rather than only within a single instant. The early exit described below
removes that cost whenever no better-priority agent could plausibly arrive.

Two designs were rejected:

- _Preemption_ (stopping a running low-priority agent) — far too invasive, and agents are not restartable mid-work.
- _Treating `ready.json` as queue membership_ — the readiness marker is written by the wait-checks chop and lagged
  behind the predecessor's exit by ~3–5 seconds in the observed case, so it would not have changed the outcome on its
  own.

## Rust core backend boundary assessment

Per the `rust_core_backend_boundary` memory, runner-slot admission passes the litmus test for core backend logic: any
other frontend would need identical behavior. However, the existing decision logic already lives in Python at
`src/sase/core/runner_slots/_admission.py`, operating on wire records that the Rust scan facade supplies. Splitting a
single admission decision across two languages would be worse than the current arrangement.

**Decision: implement in Python beside the existing admission code.** Moving the whole `sase.core.runner_slots` module
to `../sase-core/crates/sase_core` is a worthwhile follow-up, but it is explicitly out of scope here and should not be
attempted as part of these phases.

One consequence shapes the phase split: the deference timestamp deliberately stays **out** of the scan wire in the
`deference` phase. The waiting agent reads its own `waiting.json` directly via `_read_json_dict` inside
`_try_claim_runner_slot`, and no other process needs to see it, so no Rust wire change is required to make priority
work. Only the display and editing phases need new fields to survive the Rust scan projection — see the note in
`explicit-flag`.

## Bounded deference window for deprioritized waiters

Phase id: `deference`.

### Admission helpers (`src/sase/core/runner_slots/_admission.py`)

Add pure helpers next to the existing ones and export them from `src/sase/core/runner_slots/__init__.py`:

- `deference_window_seconds(priority: int, *, seconds_per_step: int, max_seconds: int) -> float` Returns `0.0` when
  `priority <= DEFAULT_WAIT_PRIORITY`; otherwise
  `min((priority - DEFAULT_WAIT_PRIORITY) * seconds_per_step, max_seconds)`. With the proposed defaults, `priority=20`
  yields 30s, comfortably clearing the ~5s dependency-resolution lag observed in the incident.
- `deference_satisfied(eligible_since: str | None, now: datetime, window_seconds: float) -> bool` Returns `True` when
  `window_seconds <= 0`. Returns `False` when `eligible_since` is missing, unparseable, or in the future (the caller
  then rewrites it to now, so the window stays bounded rather than stranding the runner). Otherwise returns whether
  `now - eligible_since >= window_seconds`.
- `better_priority_agent_pending(records, is_live, *, priority, me) -> bool` The early exit. Returns `True` when some
  record satisfies all of: `is_runner_slot_user_agent_record(record)`, `record.artifact_dir != me`, `is_live(record)`,
  the agent has not started (`not record.agent_meta.run_started_at`), it is **not** already parked for a slot
  (`record.waiting is None or not record.waiting.slot_requested_at` — agents already in the queue are handled by
  `may_start`), and its prospective priority sorts better than `priority`. The prospective priority comes from
  `normalize_wait_priority(record.agent_meta.wait_priority)`, which is already present on `AgentMetaWire`
  (`src/sase/core/agent_scan_wire_markers.py:161`), so this needs no new wire field and no extra IO — it reuses the
  record list the claim path already scanned under the lock.

**Do not modify `may_start` or `_waiter_sort_key`.** Deference is a second, independent gate applied _after_ `may_start`
says "you are first in the queue". Keeping the queue-ordering functions untouched is what satisfies the constraint that
the TUI mirror in `src/sase/ace/tui/models/agent_runner_slots.py::_waiter_sort_key` stays consistent with the core:
queue positions and the `eligible #P of Q` display remain exactly as accurate as they are today, and the mirror needs no
change in this phase.

### Claim loop (`src/sase/axe/run_agent_wait.py::_try_claim_runner_slot`)

Inside the existing global `fcntl` lock, after `may_start(...)` returns `True`:

1. Compute `window = deference_window_seconds(priority, ...)`. If `window <= 0`, claim exactly as today — default and
   better-priority waiters must show no behavior change at all.
2. Otherwise, if `not better_priority_agent_pending(records, is_live, priority=priority, me=artifacts_dir)`, claim
   immediately. Nothing better can arrive, so there is nothing to defer to.
3. Otherwise, read `eligible_since` from the marker. If `deference_satisfied(...)`, claim. If not, write the marker with
   `eligible_since` set (to its existing value, or to now when missing, unparseable, or in the future) and return
   `(None, parked)` so the runner keeps polling.
4. Whenever `may_start(...)` returns `False`, drop `eligible_since` from the marker as part of the existing marker
   refresh. The window must measure _continuous_ eligibility, not total time parked; otherwise a waiter that was briefly
   eligible minutes ago would skip the window entirely.

The first time a waiter enters the window, print a one-line notice in the same spirit as the existing
`"Waiting for a runner slot"` message (for example `Deferring for up to Ns (priority N)`), so the behavior is
explainable from the agent log. Print it only on the transition into the window, not every poll.

Keep the whole addition cheap and fail-closed: it runs under the global lock on every poll of every waiter. It must add
no filesystem scan beyond the `records` list already in hand.

### Configuration

Add to `src/sase/default_config.yml`, with a comment explaining the semantics (lower priority number starts first; the
default is 10; only priorities above the default defer):

```yaml
runner_slots:
  deference_seconds_per_step: 3
  deference_max_seconds: 60
```

Add validated accessors in `src/sase/config/core.py` alongside `get_configured_max_running_agents`, following its
`type(value) is int` + range-check style, with module-level defaults. Unlike `get_max_running_agents`, these must
**not** propagate errors: fall back to the built-in defaults, because deference is a politeness optimization and a
config problem must never strand a runner. Document that difference in the accessor docstrings.

Note for the implementing agent: `sase/memory/xprompts.md` documents `%wait` but says nothing about `priority=`. Memory
files must not be edited without the user's explicit permission in the conversation, and a plan file does not grant it.
Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims. If you believe the memory should document
`priority=`, say so in your completion summary and let the user decide.

### Tests (`tests/test_runner_slots.py`, plus the claim-path tests)

- `deference_window_seconds`: `0.0` at and below the default; scales per step; clamps at the cap.
- `deference_satisfied`: true when the window is zero; false for missing, unparseable, and future timestamps; true once
  the elapsed time reaches the window.
- `better_priority_agent_pending`: true for a live, unstarted, unparked better-priority record; false when that record
  is dead, already started, already parked, terminal, or not a runner-slot participant; false when its priority is equal
  or worse.
- Claim path: a deprioritized waiter defers within the window, claims once the window elapses, and has `eligible_since`
  cleared when it becomes ineligible.
- Regression guard reproducing the incident: waiter A parked with `priority=20`, a slot free, and agent B live,
  unstarted, dependency-waiting, and default priority, but not yet parked → A must not claim. Then, with B absent, A
  claims immediately via the early exit.
- Regression guard that default and better-priority waiters claim on the first eligible poll, with no marker churn.
- The existing `may_start` and queue-order tests must pass unchanged; that is the evidence that the TUI mirror stays
  correct.

## wait_priority_explicit marker symmetry

Phase id: `explicit-flag`. Depends on `deference` because it edits the same marker-writing block in
`_try_claim_runner_slot`.

Mirror the `wait_runners` / `wait_runners_explicit` pair:

- Thread an "explicit" signal from the directive to `_try_claim_runner_slot` (`DirectiveInfo.wait_priority is not None`
  is the source of truth) and write `wait_priority_explicit` next to `wait_priority` in `waiting.json`.
- Update `_marker_priority` (`src/sase/axe/run_agent_wait.py:262`) to prefer the marker value only when the marker
  records it as explicit; otherwise re-read the directive value so a prompt edit takes effect on the next poll.
- Add `wait_priority_explicit` to `WaitingMarkerWire` (`src/sase/core/agent_scan_wire_markers.py:196`).
- Backward compatibility: markers written before this change carry no flag. Treat a missing flag as explicit when
  `wait_priority != DEFAULT_WAIT_PRIORITY` and as non-explicit when it equals the default. Cover this with a test and a
  comment; it is a heuristic, and it is the same one the display phase relies on to avoid labelling every queued agent
  "priority 10".

**Verify the Rust scan projection before relying on the new field.** `agent_scan_wire_conversion.py:148` builds
`WaitingMarkerWire` with `known_field_kwargs(...)`, which filters a dict supplied by the Rust-backed scan facade
(`src/sase/core/agent_scan_facade.py` uses `require_rust_binding`). If the Rust scanner emits a fixed projection of
`waiting.json` rather than passing unknown keys through, adding the Python field is not enough — the field must also be
added in `../sase-core/crates/sase_core`, opened through the `/sase_repo` skill, never by cloning or web-fetching it.
Determine which case applies first and state the finding in your completion summary; if a `../sase-core` change is
required, that is in scope for this phase.

Tests: explicit priority round-trips through the marker; a non-explicit marker lets a changed directive win; a legacy
marker without the flag is interpreted by the documented heuristic.

## Surface wait priority in ACE

Phase id: `ace-display`. Depends on `explicit-flag` so it can show priority only when it was actually requested.

- Detail pane (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py`, the slot-wait segment around
  lines 269–305): append a `· priority N` fragment to the existing `runners: N/M in use · eligible #P of Q` line, only
  when the priority is explicit. Use the muted `dim #AF87FF` style already used by the neighbouring fragments. This is
  the pane that was open in the report and showed nothing about `priority=20`.
- Agent list rows (`src/sase/ace/tui/widgets/_agent_list_render_agent.py`, around lines 285–296): add a compact suffix
  to the `▶10/10` slot marker, for example `▶10/10 p20`, again only when explicit. Keep it short; the row is dense.
- **Render cache:** add the priority (and the explicit flag) to the cache key tuple in
  `src/sase/ace/tui/widgets/_agent_list_render_cache.py:201-202`, which currently keys on `runner_slots_in_use` and
  `runner_slot_queue_position`. Without this, a row whose priority changes will keep rendering the stale marker.
- Optionally show the deference state (for example `deferring 12s`) if, and only if, the `eligible_since` field is
  actually reachable through the scan wire. It is intentionally not wired through in the `deference` phase; adding it
  may require the same `../sase-core` change discussed above. Skip it rather than forcing a wire change, and say so in
  your completion summary.

Tests: unit coverage for the new render fragments in both widgets, including the non-explicit case rendering nothing.
The row change touches ACE rendering, so run `just test-visual` and refresh the PNG goldens under
`tests/ace/tui/visual/snapshots/png/` with `--sase-update-visual-snapshots` only if the diff is the intended one.

## Edit wait priority from the ACE wait modal

Phase id: `wait-modal`. Depends on `explicit-flag`; independent of `ace-display`, so the two can run in parallel.

- `src/sase/ace/tui/modals/wait_modal.py`: add a priority input and `WaitModalResult.priority: int | None`, with a
  `_PriorityValidation` dataclass and validation modelled directly on the existing `_RunnersValidation` flow
  (non-negative integer, empty means unset). Prefill from the agent's current explicit priority.
- Directive round-trip: `prompt_wait_spec` / `set_prompt_wait` must emit and re-parse `priority=` inside `%wait(...)` so
  the edited prompt stays the source of truth, consistent with how `runners=` is handled.
- Persistence: add priority (and its explicit flag) to `_WaitingMarkerPatch`, `waiting_marker_patch_for_token`, and
  `wait_meta_patch_for_token` in `src/sase/ace/tui/actions/agents/_directive_persistence.py:157`, and apply it in
  `_apply_live_runner_wait` in `src/sase/ace/tui/actions/agents/_wait_actions.py`.
- **Verify first, then fix if needed:** `_WaitingMarkerPatch` currently has no priority field. Determine whether
  applying a wait edit that only changes beads/runners preserves an existing `wait_priority` in `waiting.json` or drops
  it. If it drops it, that is a live bug in its own right — fix it and cover it with a regression test.
- `_apply_run_now` (`src/sase/ace/tui/actions/agents/_wait_actions.py:271`) clears `wait_priority`, which is correct
  because run-now removes the wait entirely. Confirm and keep that behavior; add a test pinning it so a later change
  does not silently resurrect a stale priority.

Tests: modal validation accepts and rejects the documented inputs; the directive round-trips `priority=` through
`set_prompt_wait`; an edit that does not mention priority preserves the existing value; run-now still clears it.

## Verification

Every phase must run `just install` first (workspace virtualenvs go stale), then `just check` before reporting
completion. The `ace-display` phase must also run `just test-visual`.

End-to-end check for the `deference` phase, which is the phase that actually changes runtime behavior: with the runner
cap saturated, park one agent under `%wait(priority=20)` and a second default-priority agent behind a dependency that is
about to resolve, then confirm from `agent_meta.json` timestamps that the default-priority agent claims the freed slot
and the deprioritized one starts after it. The incident timeline in the Background section is the shape to reproduce.
