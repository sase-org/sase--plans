---
tier: tale
title: Complete the Services tab in the TUI
goal:
  The beta service host is fully observable and controllable from a responsive Services
  tab, with correct service health, proc filtering, compatibility aliases, and phase
  closure evidence.
size: medium
proposed_by: bbugyi200.athena.sase-11y.7
bead: sase-11y.7
create_time: 2026-09-18 05:48:22
status: wip
---

- **PARENT:**
  [202609/service_host_1.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_1.md)
- **BEAD:**
  [sase-11y.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.7.md)

# Complete `sase-11y.7`: Services tab in the TUI

## Outcome

Turn the user-facing AXE tab into the Services tab while retaining `axe` as its internal
tab id until the sunset phase. When the existing `service_host` beta flag is enabled,
the tab must present the service host and every configured service proc, nest the
existing routine/job tree beneath the Scheduler proc, and route host/proc controls
through the service control plane. When the flag is disabled, preserve the legacy AXE
runtime and controls so the later sunset phase has one explicit Off branch to delete.

The phase also makes service health visible in the footer, prevents service rows from
inflating the session gear, and seeds the Procs query from the existing
`ace.procs.default_query` setting without overwriting an explicitly cleared session
query.

## Boundaries and decisions

- Keep `axe` as the canonical internal tab id, widget ids, and broad module namespace.
  Add `services` only as an input alias and change user-facing copy. Canonical id and
  wholesale internal naming changes belong to `sase-11y.10`.
- Do not migrate legacy `!` background-command slots in this phase. Keep their current
  rows/actions available below the daemon service tree; `sase-11y.8` owns durable
  oneshot service procs and the final oneshot section.
- Use the already-landed `service_host` beta flag. Do not add or remove flags. Cover
  both enabled and disabled branches in tests.
- Treat `ServiceStatusSnapshot` as the sole TUI read model for host/proc runtime health.
  Do not reconstruct desired state in Textual code. All status reads, config reads, log
  tails, state mutations, host lifecycle calls, and nudges run outside the Textual event
  loop.
- Keep the Rust-backed service config/state/status facades authoritative. Python may add
  a thin shared action adapter, but must not duplicate merge, enablement, stop-lifetime,
  or health classification rules.
- Render `snapshot.procs` (configured procs) in the main tree. Do not mix
  `snapshot.orphans` into the selectable configured-proc inventory; surface orphan
  diagnostics in details if needed.
- A configured but disabled or unavailable proc remains visible. Disable start/restart
  for an unavailable proc with a useful explanation; machine enable/disable policy is
  available for rendered configured procs and its provenance remains visible.

## Implementation

### 1. Establish one proc-control adapter shared by CLI and TUI

Add a small service-domain action module (or equivalently extend
`src/sase/service/control.py` without importing Textual) that exposes typed operations
for a named configured proc:

- start: clear the boot-scoped stop and nudge the host;
- stop: record the boot-scoped stop with actor/reason and nudge the host;
- restart: request stop, allow the existing short settle delay, clear the stop, and
  nudge after each mutation;
- enable/disable: write the machine-local enablement override and nudge the host.

Return enough typed mutation information for callers to distinguish an accepted/no-op
request and produce their own UI/CLI messages. Reuse `ServiceStateMutationOutcome` in
the public adapter contract rather than wrapping away the Rust-owned mutation result.
Validate that the selected name is configured and report unavailable/startability errors
before mutation. Keep host start/stop/restart on the existing `sase.service.control`
functions.

Refactor `src/sase/main/service_handler.py` and the service-host branch of
`src/sase/main/scheduler_handler.py` to call this adapter. This prevents the TUI from
introducing a second interpretation of stop, restart, or enablement and removes the
current duplicate CLI mutation sequences. Add focused service action/handler tests for
the state mutation ordering, actor/reason, nudges, validation failures, and unchanged
human/JSON command behavior.

### 2. Rename the surface without renaming its internal identity

Update the tab bar and related user-facing help/copy so the visible label is `Services`
and uses a service-health color rather than the AXE failure-red identity. Teach
`normalize_tab_name()` that `services` aliases canonical `axe`; include `services` in
`sase tui --tab` choices and completion while retaining `axe` as a sunset alias. Update
the TUI command help to name Services.

Give the existing startup flags canonical service spellings while retaining their
destinations and backward compatibility:

- `-x/--no-service/--no-axe`
- `-R/--restart-service/--restart-axe`

The public help and restart re-exec argv should emit the service spellings. Restart
argument filtering must recognize both names so it never duplicates either alias. Keep
options sorted and add parser, normalization, help, and restart-forwarding tests.

### 3. Add service snapshots to the existing off-thread AXE/Services refresh lane

Extend `actions/axe_display/_data.py`, `_loader_state.py`, `_loader_refresh.py`, and the
event-refresh surface token plumbing with a feature-gated service payload containing:

- the atomic `ServiceStatusSnapshot` (using persisted status when current and the
  existing current-status fallback when needed);
- the snapshot `change_token`/generation used to reject a stale collector result;
- only bounded detail data that the selected service row needs, including a log tail
  loaded off-thread when its selection changes rather than reading every proc log on
  every refresh.

Include the service status/state paths in the cheap token probe so host reconciliation
wakes an idle TUI, but keep file stat/JSON/config work off the event loop. Preserve the
existing refresh coordinator's single-flight/trailing-refresh behavior. Capture a
selection identity before awaiting collection and re-resolve it against the returned
generation before applying details.

When `service_host` is disabled, skip service snapshot collection and retain the
existing AXE collector/startup path. When enabled, map normal auto-start/restart startup
behavior to the service host functions and run it through the current pump-free worker
path; `--no-service` must suppress auto-start. Tests must cover missing/corrupt/stale
snapshots, a newer generation winning over an older completion, selection changes during
an await, and both feature-flag states.

### 4. Build the Services tree and detail/status chrome

Add a `ServiceProcItem` (and stable key such as `("service", name)`) to
`widgets/bgcmd_list.py` and the AXE display loader/render code. Under the enabled flag,
build the flat visual tree in this order:

1. one row for each configured daemon proc from `snapshot.procs`;
2. for `scheduler`, the current lumberjack/routine and chop/job rows indented beneath
   that proc, with their current fold state, navigation, manual-job action, output, and
   scope-rail editing intact;
3. the existing legacy background-command section, unchanged except for surrounding
   Services terminology, pending the oneshot phase.

Render clear glyph/style/inline summaries for running, stopped, disabled (dimmed with
enablement provenance), failed, and unavailable (including the reason) states. Keep the
Scheduler presentation label capitalized, but use its canonical `scheduler` name for
lookup/actions. Preserve stable selection across refreshes and sensible fallback when a
configured proc is added or removed.

Evolve `AxeInfoPanel` into Services chrome that always renders the host state, uptime or
platform unit when available, and the `!x` start hint when down. Extend the existing
dashboard/detail renderer for a selected service proc using `ServiceStatusHost`,
`ServiceStatusProc`, and `ServiceEnablement`: summary, desired/current state, enablement
provenance, source/declaring layer, launcher, pid/restart/last-exit data,
unavailability, and the bounded cached log tail. Continue to render the current
routine/job/bgcmd detail views for their row types. Keep disk access out of render,
navigation, and footer code.

Add pure item formatting, tree construction, identity restoration, host chrome, and
detail renderer tests, plus targeted Textual tests that prove routine/job behavior is
unchanged beneath Scheduler.

### 5. Wire service-aware actions, keys, quit behavior, and startup

Update the keymap model, `src/sase/default_config.yml`, binding metadata/help, dynamic
footer bindings, and action dispatch together:

- `x` starts a stopped selected service proc or stops a running/desired selected service
  proc;
- `r` restarts a selected daemon, retains run-job behavior on a nested job, and retains
  the legacy rerun behavior on a legacy background-command row;
- `!e` toggles the selected configured proc's machine-local enablement policy, with help
  text that explicitly calls it machine policy;
- `!x` starts/stops the service host;
- `!!` remains the current background-command creation path until `sase-11y.8`.

Run each service operation through a pump-free task and the shared adapter from step 1.
Recapture the selected stable service key after the await, notify success/failure, and
request a coalesced Services refresh. Suppress inapplicable bindings and explain
unavailable actions rather than mutating an unavailable proc.

Under the enabled flag, change the quit modal's user-facing option from stopping AXE to
stopping Scheduler and route it through the proc adapter; quitting must never stop the
service host. Under the disabled flag, preserve the legacy AXE action/startup/quit
branch exactly. Add action tests for all row types, action failures, selection changes
during workers, host controls, quit semantics, and both flag states.

### 6. Replace the AXE footer pill with service health

Teach the keybinding footer to consume the cached snapshot. For the enabled branch,
derive health from the host plus configured enabled/desired procs:

- healthy: `SVC <running>/<desired>`;
- any unhealthy host or desired proc: persistent loud `SVC !` styling and a concise
  failure summary/notification;
- disabled procs do not inflate the denominator, while enabled-but-unavailable or
  failed/stopped desired procs do degrade health.

Deduplicate notifications by health/change-token transition so a failed proc stays
visually loud without producing a toast every countdown tick. Retain the current startup
stopwatch and background-command badges. Keep the legacy AXE footer state in the feature
flag's disabled branch. Update footer unit tests for all host/proc health combinations
and transition deduplication.

### 7. Correct the gear count and initialize the Procs default query

In `_proc_observer_models.py` expose an explicit gear-eligible active-row/count helper:
an active row counts only when it is neither a monitor row nor marked as a service proc.
Use it from `_proc_action_observer.py` instead of inferring ownership from
`session_id is None` or subtracting only monitor rows. Keep the full projection and
Procs pane inventory unchanged. Add the regression case for a live sessionless
Telegram/service receiver alongside ordinary and monitor rows.

Add an initialization marker to `ProcsSessionState` so its first construction/open can
seed `query` from merged `ace.procs.default_query` (already defaulted to `-service`).
Any committed query, including the empty string, marks the state initialized; reopening
the pane must therefore preserve a user-cleared query. Validate a malformed non-string
config value without crashing and keep an explicit empty configured default empty.

The `service` boolean, `svc:<name>` field, and service name/mode cache-key components
already exist in `_proc_query.py`; retain them and add/assert regression coverage rather
than reimplementing them.

### 8. Resolve this phase's Symvision ownership before closure

Run `just install` before Symvision, then `just _lint-symvision`. Remove each
`sase-11y.7(...)` entry from the `Justfile` as this phase creates a real non-test
consumer (expected for the service status/enablement types and the state mutation
result). Do not add artificial imports solely to satisfy Symvision.

For any remaining public service facade that is still intentionally reserved for a
future epic phase—currently candidates include detailed config provenance,
enablement-reset, direct composition, or standalone enablement resolution—either
delete/private it if genuinely dead or re-key its existing `--epic-symbol` entry to the
still-open parent `sase-11y` or the concrete later phase that will consume it. Document
the semantic reason in the diff. Immediately before closing, run:

```text
sase bead epic-symbols sase-11y.7
```

It must report no entries owned by this phase.

## Verification and closure

1. Run focused service-domain, parser, tab normalization, Services tree/render/action,
   footer, proc gear, Procs filter/default-query, refresh race, and feature-flag tests.
2. Run the dedicated TUI visual snapshot suite. Inspect actual/expected/diff artifacts
   for Services-label/tree/footer changes before accepting only intentional goldens; do
   not change the canonical internal tab id in snapshots.
3. Exercise both `service_host` flag states, including `sase tui --tab services`, the
   old `--tab axe` alias, both startup-flag spellings, host-down chrome, configured
   disabled/unavailable procs, Scheduler child folding/actions, service action errors,
   quit-with-Scheduler, footer degradation, and a user-cleared Procs query.
4. Run the existing TUI navigation benchmark with status snapshots changing in a
   restart-storm loop (`SASE_TUI_PERF=1`) and record p95 below 16 ms. Navigation and
   render callbacks must show no sync disk/config/subprocess work.
5. Run `just fix`, then `just check` (using the monitor skill if it becomes long). If
   dependency drift prevents the checks, run `just install` and retry. Because this is
   one phase rather than the epic landing, leave `just check-full` to the host landing
   gate unless scoped verification escalates or the changed-file set requires it.
6. Run `sase bead epic-symbols sase-11y.7` again and confirm it is empty. Record any
   genuinely out-of-scope discovery only as `PROPOSED FOLLOW-UP:` on `sase-11y.7`; do
   not create a bead.
7. Close only the assigned phase with a note naming the tests, visual inspection,
   performance result, `just check`, and empty epic-symbol audit:

   ```text
   sase bead close sase-11y.7 --note "<what was verified>"
   ```

   Do not close `sase-11y` or any ancestor.
