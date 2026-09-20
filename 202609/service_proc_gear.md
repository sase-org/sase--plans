---
tier: tale
title: Service-host procs never count as session background work
goal:
  Service-host daemon procs are excluded from every proc gear count and never delay an
  ACE restart, and a core binding that cannot record the service marker is reported
  instead of silently degrading.
size: medium
proposed_by: bbugyi200.athena.0o9
create_time: 2026-09-20 16:05:36
status: wip
---

# Plan: Service-host procs never count as session background work

## Problem

On athena the three `sase-11y` service procs (`gateway`, `scheduler`,
`telegram_receiver`) showed up in the blue proc gear on the ACE top bar, and a `,E`
update toasted `restart queued until 3 procs finish` — for procs that by design never
finish. apollo did not show either symptom. These are two separate defects plus one
latent third.

### Root cause 1 — the gear count (athena only, already self-cleared)

`_is_gear_eligible_row` (`src/sase/ace/tui/_proc_observer_models.py:106`) decides that a
row belongs to the service host by exactly one signal: the wire `service` block on the
durable proc row.

athena's service-host process had loaded a `sase_core_rs` build that predated sase-core
`4cee31a` ("feat(procs): add service proc wire metadata", 2026-09-16), which is the
commit that added `service` to `ProcReserveWire` / `ProcWire`. Serde drops unknown
fields silently, so every row that host reserved reached `~/.sase/procs/procs.jsonl`
with **no** `service` key at all — while every other field of the same `ProcReserve`
(`origin: service-host`, `shell_kind: service`, `tags: ["service", "service:<name>"]`)
round-tripped fine. With `row.service` `None`, all three daemons were gear-eligible.

Evidence gathered while diagnosing (do not re-derive, but do re-confirm the code paths
before editing):

- athena rows created `2026-09-20T16:02Z` through `19:03Z`: `service` key absent.
- athena rows created `2026-09-20T19:58Z`, after the local sase-core `.so` was rebuilt
  at 15:56 EDT and the host restarted: `service` block present and correct.
- apollo rows: `service` block present. apollo's host was started with a current
  binding, which is why apollo never showed the gear symptom.

So the athena gear symptom has already stopped on its own. The defect that remains is
that a single missing additive wire field silently misclassifies a row on every UI
surface, with three other authoritative host-written markers sitting unused on the same
row, and with no diagnostic anywhere.

### Root cause 2 — the restart gate (reproduces everywhere, including apollo)

`running_background_procs` (`src/sase/ace/tui/update_restart.py:90`) filters out only
monitor shells:

```python
return [
    row
    for row in proc_projection_for(app).active_rows()
    if not is_monitor_shell_row(row)
]
```

Service-host daemon rows are active, sessionless procs, so they are always blockers.
`restart_after_update_when_ready` therefore toasts
`… - restart queued until 3 procs finish.`, re-arms its 1s timer for the whole
`_RESTART_WAIT_SECONDS = 60` window, and then warns
`restart wait expired; restarting with 3 procs: service:gateway, … still active.` A
daemon service proc never exits, so this wait is always wrong and always burns 60
seconds.

This is a live bug on master and is **not** fixed by a host restart — it reproduces with
a perfectly correct `service` block. apollo almost certainly avoided it only because
`,E` there found nothing to update: `,E` reaches `_restart_after_update` only when
`result.code_changed` (`src/sase/ace/tui/actions/update_run.py:215`). Confirm this
during implementation rather than assuming it.

### Root cause 3 — Procs pane title drifts from the top-bar gear

`_title_text` (`src/sase/ace/tui/modals/procs_pane_selection.py:467`) computes its own
blue chip as `proc_running = running - monitor_running`, with no service exclusion,
while the top-bar `ProcIndicator` is fed by `gear_eligible_count`. With a correct
`service` block the two chips disagree by the number of running service procs — on
apollo today the pane would read `⚙2` blue against the top bar's `⚙0`. This violates the
same "a gear and the count it feeds can never disagree" invariant that
`_agent_list_render_agent_prefix.py:79-82` and `models/agent_family_members.py:50-51`
state for the agent lane.

## Implementation

### 1. One shared, degradation-tolerant service-row classification

Add named origin constants beside the existing service-proc constants in
`src/sase/procs/service_meta.py`:

- `SERVICE_HOST_ORIGIN = "service-host"` — the origin `ServiceHost._launch` writes for
  every daemon (`src/sase/service/host.py:313`).
- `SERVICE_ONESHOT_ORIGIN = "service-proc"` — the origin transient oneshots write
  (`src/sase/procs/oneshot.py:34`).

Replace the hard-coded string in `host.py` with the constant, and make
`oneshot.ONESHOT_ORIGIN` an alias of `SERVICE_ONESHOT_ORIGIN` so the public name and its
exports keep working. `service_meta.py` is the right home: it already owns the service
mode/source vocabulary, it has no heavy imports, and both the TUI read models and the
host already import from it.

Add two predicates next to `is_monitor_shell_row` in
`src/sase/ace/tui/_proc_observer_models.py`, and export both:

```python
def is_service_row(row: ObservedProc) -> bool:
    """Return whether a row is owned by the service host or is a oneshot."""

def is_service_daemon_row(row: ObservedProc) -> bool:
    """Return whether a row is a service daemon that never terminates."""
```

- `is_service_row`: true when `row.service is not None`, **or** when `row.origin` is
  `SERVICE_HOST_ORIGIN` or `SERVICE_ONESHOT_ORIGIN`. The origin fallback is what makes a
  row written by a lagging host — or one already sitting in the store — classify
  correctly, so no store migration or backfill is needed.
- `is_service_daemon_row`: true when the row carries a `service` block whose `mode` is
  `SERVICE_PROC_MODE_DAEMON`, **or** when `row.service is None` and
  `row.origin == SERVICE_HOST_ORIGIN`. The second arm is sound because the host reserves
  daemons and nothing else under that origin; a transient oneshot is never a daemon.

Deliberately do **not** add `shell_kind == "service"` as a third arm. It is redundant
with the origin on every row the host writes, and one narrow, testable fallback is
easier to reason about than three overlapping ones. Say so in a short comment so the
omission reads as a decision, not an oversight.

### 2. Gear eligibility uses the shared predicate

In `_is_gear_eligible_row`, replace `row.service is None` with
`not is_service_row(row)`. Behavior is unchanged for correctly marked rows and correct
for degraded ones. Update the docstring, which currently explains the exclusion purely
in terms of the wire marker.

### 3. The restart gate ignores procs that never terminate

In `running_background_procs`, drop rows for which `is_service_daemon_row` is true,
alongside the existing monitor-shell exclusion, and update the docstring to say why: a
daemon service proc has no terminal state, so waiting on it can only expire.

Leave transient oneshots (`!` background commands) blocking the restart exactly as they
do today. They are ordinary user background work with a real terminal state; narrowing
this change to "never terminates" keeps the behavior change to the case that is
unambiguously wrong.

### 4. Procs pane title reuses the same counts

Rewrite `_title_text`'s chip computation so the blue chip counts only rows that
`gear_eligible_count` would count — i.e. active, not a monitor shell, not a service row
— instead of `running - monitor_running`. Keep the `[N running · M done]` inventory
totals as they are: the pane deliberately lists service rows (`_proc_query.py:75`
exposes a `service:` filter field), and only the _gear chip_ is claiming to be the
session proc count.

### 5. The host reports a binding that cannot record the marker

`ServiceHost._launch` currently discards the `reserve_proc` outcome
(`src/sase/service/host.py:302-322`). Capture it, and when
`outcome.proc.service is None` even though the request carried a block, emit one warning
per host process to stderr in the existing house style (`print(..., file=sys.stderr)`,
as at `host.py:118`), naming the likely cause — a `sase_core_rs` build that predates the
service proc wire metadata — and the remedy (rebuild/update sase-core). Guard it with a
flag so a restart loop cannot spam the journal.

Do not refuse the launch: running the gateway without a cosmetic metadata field is
strictly better than not running it, and step 1's fallback already keeps the UI correct.
The point is that this can never again present to a user as an unexplained gear count.

## Tests

Extend the existing suites rather than adding new files:

- `tests/ace/tui/test_services_phase_closure.py` — the gear block already has
  `test_gear_excludes_monitor_and_sessionless_service_rows`. Add a case for a
  service-host row with **no** `service` block (`origin="service-host"`, `service=None`)
  and assert it is still excluded from `gear_eligible_count`.
- `tests/ace/tui/test_update_restart.py` — add: a daemon service row (both with a
  `service` block and, separately, with only `origin="service-host"`) is not returned by
  `running_background_procs`; a transient oneshot row (`origin="service-proc"`,
  `mode=oneshot`, `source=transient`) still **is**; and the end-to-end assertion that
  with only daemon rows active, `restart_after_update` restarts immediately and emits no
  `restart queued until` notice.
- Procs pane title: add a case to the existing procs-pane tests asserting the pane's
  blue chip equals `gear_eligible_count` for a mixed projection (ordinary + monitor +
  service rows). `tests/ace/tui/_procs_pane_helpers.py` already builds projections for
  this.
- `tests/test_procs_facade_models.py` or the service-host tests — cover the new origin
  constants and the `_launch` warning path (reserve returning a row whose `service` is
  `None` warns once and still claims the supervisor).

## Verification

- `just install` first if the workspace venv is stale, then `sase tool run check`
  (preferred over raw `just check`). Run `just fix` inline beforehand.
- These changes alter rendered TUI text (the Procs pane title chip), so run
  `just fix-tui-screenshots` and inspect the report and golden diffs before accepting
  any update; generation is not approval. Hand a full run to `/sase_monitor` with the
  `TESTING` / `TESTED` status pair.
- Do not run `just check-full`; nothing here names it.

Manual confirmation on athena, after the change is installed and the TUI restarted:

1. `sase service status` shows three running service procs.
2. The top-bar blue gear and the Procs pane blue chip both read the same number and
   neither includes those three.
3. `,E` with a real code change restarts promptly and never toasts
   `restart queued until 3 procs finish`.

## Out of scope

- Backfilling the `service` block onto proc rows already in the store. Step 1's origin
  fallback makes those rows classify correctly as they are.
- A `sase doctor` check for a stale `sase_core_rs` build. The host warning in step 5
  covers the surface that actually produced this report; a general binding-freshness
  check is a separate concern worth its own task bead.
- Any change to how the Services tab or the `#1`-`#9` oneshot surface renders these
  rows. Both are correct today.
