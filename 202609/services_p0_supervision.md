---
tier: epic
title:
  "Service supervision P0: honest restarts, loud failures, safe config and environment"
goal: "The service host obeys its own restart decisions, an explicit restart is a
  confirmed transition with a new pid, every failure is visible in a notification, the
  CLI, and the Services tab, a broken config layer never silently stops a proc, and
  `sase service init` cannot freeze an agent shell's feature flags into the 24/7 host.

  "
phases:
  - id: core
    title: Core restart, request, and config-layer semantics
    depends_on: []
    size: medium
    description: "core: make crash-loop stickiness survive capped backoff, add a
      per-proc restart request with a generation to the service state store, surface the
      pending request in the status snapshot, and make an errored or unknown-kind config
      layer fatal instead of invisible.

      "
  - id: restartgen
    title: Restart is a confirmed transition
    depends_on:
      - core
    size: medium
    description: "restartgen: move the sase-core revision pin, rebuild start and restart
      on the new request generation, have the host consume requests and record the pid
      it produced, and make the CLI wait for that generation, print old pid to new pid,
      and fail honestly.

      "
  - id: config
    title: Last-known-good config keeps the host supervising
    depends_on:
      - restartgen
    size: medium
    description: "config: keep the last composition that loaded, observe exits and stop
      children without reloading config, and publish the config error through the
      heartbeat and the status snapshot instead of going silently stale.

      "
  - id: giveup
    title: The host honors give_up and says so
    depends_on:
      - config
    size: medium
    description: "giveup: stop relaunching procs the restart policy gave up on, keep a
      signature-keyed given-up record that an explicit request clears, emit a durable
      notification on crash-loop and on give-up while desired running, and make the
      Telegram receiver exit retryable instead of reporting missing credentials as
      success.

      "
  - id: surfaces
    title: The CLI and the Services tab show the real failure states
    depends_on:
      - giveup
    size: medium
    description: "surfaces: replace the invented failed/error vocabulary with the wire
      states the core emits, render restart evidence and the host config error in the
      CLI tables and the Services tab, and make an unreadable snapshot read as unknown
      instead of healthy.

      "
  - id: env
    title: The captured service environment is context-safe
    depends_on: []
    size: medium
    description:
      "env: stop capturing SASE_FEATURE_FLAGS into the host environment, settle the
      currency check by ignoring volatile keys and normalizing PATH, and refuse an
      agent-context `sase service init --yes` unless it is explicitly allowed."
proposed_by: bbugyi200.athena.0pe
create_time: 2026-09-22 12:59:03
status: wip
---

- **PROMPT:**
  [prompts/202609/services_p0_supervision.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/services_p0_supervision.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                                          | Why                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| derives-from | [research:202609/sase_services_post_landing_hardening/sase_services_post_landing_hardening.md][1] | Consolidated post-landing report whose P0 section this epic implements |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202609/sase_services_post_landing_hardening/sase_services_post_landing_hardening.md

<!-- sase:links:end -->

# Plan: Service supervision P0 — honest restarts, loud failures, safe config and environment

## Why

The service host, its service procs, and the Services tab shipped as designed, but four
supervision defects make the host quietly lie about what it is doing. Each one below is
confirmed in code, and the first is confirmed live on athena.

1. **The host overrides its own "give up" decisions.** `_settle_exit`
   (`src/sase/service/host.py:202-213`) only queues a pending restart when
   `decision.action == "restart"`. On `give_up` the name lands in neither `_children`
   nor `_pending`, so loop 3 of `_reconcile_desired` (`host.py:252-256`) relaunches it
   about 1 s later; `_desired_running` (`host.py:258-265`) never consults the restart
   decision. Live evidence: 17 consecutive `service:telegram_receiver` rows, all
   `success/exit 0`, about 2 s apart. Each relaunch resets `restarts` to 0, so the
   Services pill stays green and the retained 20-row proc history is flushed in about 40
   s.
2. **Restarts are edge-triggered commands built on level-triggered state.**
   `restart_service_proc` (`src/sase/service/actions.py:82-104`) writes a stop marker,
   nudges, sleeps `delay`, clears the marker, and nudges again. `sase scheduler restart`
   passes `delay=0.0` (`src/sase/main/scheduler_handler.py:72-77`), so the marker is
   cleared before the nudged host reads state: it is a no-op. Even at 0.5 s the request
   is lost whenever the host is blocked in `_stop_child` or `handover_scheduler`. No
   caller waits for or reports a new pid, and `sase update` prints `status="restarted"`
   when it only requested one.
3. **Failures are silent.** `decision.notify` is set by the core once per crash-loop
   episode and read by nothing. Crash-loop switches itself off, because backoff caps at
   60 s while the crash-loop window is 60 s, so a proc that fails forever reads
   `crash_loop` only for failures 3-7. The TUI treats `{"failed", "error"}` as failure
   (`src/sase/ace/tui/_service_health.py:14` plus four widgets) although the core emits
   `running`, `unavailable`, `disabled`, `stopped`, `crash_loop`, `backoff`, `exited`,
   and test fixtures use a made-up `state="failed"` so the tests pass while real states
   render dim. `derive_service_health(None)` returns a healthy `SVC 0/0`. The CLI tables
   never show `restarts`, `last_exit`, the restart decision, or stop provenance,
   although the snapshot already carries all of it.
4. **A bad config blinds the host and misleads the TUI.** `_reconcile_once`
   (`host.py:108-124`) loads the config first, so a raising `load_service_config` skips
   exit observation, restarts, and stops; the fallback status write reloads the config,
   fails again, and swallows the error, so `status.json` goes stale and the TUI falls
   back to `SVC 0/0`. In the core, `compose_service_config` never reads a layer's
   `error` field (`crates/sase_core/src/config/wire.rs:154`), so an overlay saved
   mid-edit with a YAML error composes as if it did not exist, `gateway` and
   `telegram_receiver` fall back to `enabled: false`, and the host stops them within a
   second.

Plus one environment hazard: `capture_service_environment` deliberately captures
`SASE_FEATURE_FLAGS` (`src/sase/service/env.py:54`) and the host applies captured values
with `override_existing=True` (`src/sase/service/host_lifecycle.py:25`), so a single
`sase service init --yes` run from an agent or ACE shell freezes that shell's resolved
flag snapshot into the longest-lived process on the machine and into every agent it
launches. `environment_files_match` (`env.py:195-200`) is exact dict equality, so
`init --check` reports `needs_attention` from every shell and advises exactly that
dangerous command.

## What lands

- A proc the restart policy gave up on stays down until something explicit revives it,
  and the operator is told, once, why it is down.
- `sase service proc restart NAME`, `sase service proc start NAME`, and
  `sase scheduler restart` are confirmed transitions: they wait for the host to consume
  a generation and print `pid <old> -> pid <new>`, or fail with a non-zero exit and an
  honest reason.
- A crash-looping or given-up desired-running proc raises a durable notification, stays
  `crash_loop` until a healthy run, and reads as a failure in the CLI, the Services tab,
  and the `SVC` pill.
- A broken config layer is fatal at compose time, and a fatal composition leaves the
  host supervising its last-known-good set while publishing the error instead of
  freezing.
- The captured service environment no longer carries feature flags, its currency check
  settles from any shell, and `init --yes` refuses to run from an agent shell by
  default.

## Non-goals

Everything in the report's P1 and P2 lists stays out: graceful bounded parallel stops
and SIGTERM forwarding to chops (§3.7), `sase update` restarting the host and the code
fingerprint (§3.8), honest CLI outcomes for `start`/`enable` on a disabled proc and
moving the init audit out of `sase service status` (§4.1), `service logs` journal
fallback, orphan rendering, `--follow`, timestamped run separators (§4.2), TUI
`!x`/`!e`/`x` safety (§4.3), control-plane hardening such as the boot-id prune guard and
lock-held signal check (§4.4), performance (§4.5), proc-wire `extra` preservation
(§3.9), and the §5 polish batch. Do not add a `healthcheck` DSL, a Unix-socket control
plane, or `sase service proc kill`. §3.6's boot-before-login secret audit for `doctor`
is also out of scope; the `SASE_*` path-override capture already landed in `529d7d325`
for `sase-15q`.

Two owner actions outside this code, for the land agent to surface when the epic
finishes: create `~/.sase/telegram_bot_token` (mode 600) on athena so the receiver never
depends on gpg-agent at boot, and restart the service host once at a quiet moment so it
picks up the already-landed `SSH_AUTH_SOCK` fix.

## Rules every phase follows

- **Repos.** `core` works only in `sase-core`; `giveup` also touches `sase-telegram`;
  every other phase is sase-only. Open a linked repo with
  `sase repo open <name> -r "<why>"` and use the printed path; read that repo's
  `AGENTS.md` first. Never edit a linked checkout found any other way.
- **The core pin.** sase CI builds `sase_core_rs` from the SHA in
  `sase-core-revision.txt`, not from sase-core HEAD, so sase code that calls new core
  behavior is red until the pin moves. `restartgen` is the only phase that moves it
  (`just ratchet-core-revision` after `core` has landed and pushed); later phases
  inherit it and must not bump it again. Do not touch the `sase-core-rs` window in
  `pyproject.toml`: the release job owns it.
- **Wire compatibility.** Every new field on an existing wire struct is additive and
  optional, with `#[serde(default)]` and `skip_serializing_if`, and no
  `*_WIRE_SCHEMA_VERSION` constant moves in this epic. A newer schema number would make
  the running host and TUI refuse the file they already read.
- **Verification.** In sase run `sase tool run check` (never `just check-full`); run
  `just fix` or at least `just fmt` inline first. `just install` may be needed first in
  a fresh workspace. In sase-core and sase-telegram run that repo's `just check`
  (sase-core's takes about five minutes, so give it a tool timeout of ten minutes or
  more). Hand a long run to `/sase_monitor` rather than letting it outrun the turn.
- **Tests belong to the phase that changes behavior.** Every phase below names the tests
  it must add or update; a phase is not done while the old test still asserts the old
  lie.

---

## Core restart, request, and config-layer semantics

All in `sase-core`, in one phase so the pin moves once. No sase changes here.

### 1. Crash-loop is sticky until a healthy run

`crates/sase_core/src/service/restart.rs`. `crash_loop` today is
`recent_failures.len() >= threshold` over a 60 s window, while backoff caps at 60 s
(`SERVICE_RESTART_MAX_BACKOFF_SECONDS`), so a permanently failing proc drops out of
`crash_loop` once its failures are spaced further apart than the window. In `restart()`,
compute it as:

```
crash_loop = recent_failures.len() >= threshold
    || (history.alert_sent && history.consecutive_failures >= threshold)
```

`alert_sent` and `consecutive_failures` are already cleared by the existing healthy-run
branch, so recovery still ends the episode, and `notify = crash_loop && !alert_sent`
still fires once per episode. Keep the existing `reason` wording for the sticky case (it
should still read `crash-looping (...)`, using `recent_failures.len()` for the count).

Tests in the file's `mod tests`: a proc failing every 120 s past the cap stays
`crash_loop` with `notify` false after the first alert; a healthy run longer than
`healthy_run_seconds` clears `alert_sent` so the next crash loop notifies again; a proc
below the threshold is unaffected.

### 2. A per-proc restart request with a generation

`crates/sase_core/src/service/state.rs`. Today every restart path is a stop marker
written and cleared around a sleep, which is an edge command built on level state. Add a
durable, monotonically numbered request the host consumes.

New wire struct `ServiceProcRequestWire`:

- `generation: u64` — bumped on every request.
- `action: String` — `"start"` or `"restart"` (validate; reject anything else).
- `requested_at: f64`, `requested_by: String`, `reason: Option<String>`.
- `completed_generation: Option<u64>`, `completed_at: Option<f64>`,
  `completed_by: Option<String>`, `pid: Option<u32>`, `outcome: Option<String>`,
  `error: Option<String>`.

Add `requests: BTreeMap<String, ServiceProcRequestWire>` to `ServiceStateWire` with
`#[serde(default, skip_serializing_if = "BTreeMap::is_empty")]`, so a machine that never
requests anything writes a byte-identical `state.json`.

Two new `ServiceStateMutationWire` variants:

- `RequestProc { name, action, actor, reason }` — validates the proc name, the actor,
  and the action; sets `generation` to the existing entry's `generation + 1` (else 1);
  records `requested_at`/`requested_by`/`reason`; clears every completion field; **and
  removes `stops[name]`**, because both `start` and `restart` mean "this proc should be
  running now" and that is what today's stop-marker dance already achieved.
- `CompleteProcRequest { name, generation, pid, outcome, error }` — a no-op (returns
  `changed = false`) when the entry is missing, when `generation` is older than the
  stored `generation`, or when `completed_generation` is already at or past
  `generation`; otherwise records the completion fields. Validate `outcome` against a
  small allow-list: `started`, `restarted`, `already_running`, `not_desired`,
  `unknown_proc`, `failed`.

The existing `prune_expired_stops` is untouched; requests are not boot-scoped, because a
request made while the host is down must still be honored when it comes up.

### 3. The pending request is visible in the status snapshot

`crates/sase_core/src/service/status.rs`. Add `request: Option<ServiceProcRequestWire>`
to `ServiceStatusProcWire` (additive, optional), populated in `derive_configured_proc`
from `request.state.requests.get(&entry.name)` exactly the way `stop` is populated. Do
not derive a new proc state from it; it is evidence, and it also makes `change_token`
move when a request is written, which wakes the TUI.

### 4. An errored or unknown-kind config layer is fatal

`crates/sase_core/src/service/config.rs`. `ConfigLayerInputWire.error` is already filled
by sase whenever a layer's YAML failed to parse or was not a mapping
(`src/sase/config/layers.py:129-149`), and compose ignores it, so a broken overlay
composes as if it did not exist and its `enabled: true` entries fall back to disabled.
In the per-layer loop, before the `service` lookup:

- `layer.error.is_some()` and `layer.kind == "local"` → a warning diagnostic
  (`service_config_layer_error`) naming the layer and its path, and skip the layer: a
  local project layer's `service` section is ignored anyway.
- `layer.error.is_some()` for any other kind → `fatal = true` plus an error diagnostic
  naming the layer, its path, and the parse error text, then skip the layer.
- `layer.kind` not in the allow-list `{builtin, plugin, user, overlay, local}` →
  `fatal = true` plus `service_config_unknown_layer_kind`. `classify_source` currently
  maps every unknown kind to `user`, which silently grants an unknown layer user
  precedence.

Tests in this file's `mod tests` (the shared `layer()` helper hardcodes `error: None`,
so extend it or add a sibling): an errored overlay is fatal and names the file; an
errored local layer warns and is not fatal; an unknown kind is fatal; the existing
composition tests still pass unchanged.

### 5. Bindings

No new Python binding names: requests ride the existing `service_state_mutate` and
`service_state_read`, stickiness rides `service_restart_decide`, the layer rule rides
`service_config_compose`, and the new status field rides `service_status_build`. Add
round-trip coverage for the new mutations and the new status field in
`crates/sase_core_py/src/config/tests.rs` next to
`service_config_compose_binding_round_trips_python_dicts`.

**Done when** `just check` passes in sase-core and the commits are pushed, because
`restartgen` pins the SHA they land on.

---

## Restart is a confirmed transition

sase only. Depends on `core` being pushed.

### 1. Move the pin

Run `sase repo open sase-core -r "..."` so the linked checkout is current, then
`just ratchet-core-revision` to move `sase-core-revision.txt` to the pushed core HEAD,
and commit it with this phase. This is the epic's only pin bump.

### 2. Python facade for requests

`src/sase/service/state.py`: add a frozen `ServiceProcRequest` dataclass mirroring the
wire (with `from_wire`), add `requests: dict[str, ServiceProcRequest]` to
`ServiceState`, and add two functions beside the existing mutators:

- `request_service_proc(name, action, actor, *, reason=None) -> ServiceStateMutationOutcome`
- `complete_service_proc_request(name, generation, *, pid=None, outcome=None, error=None)`

`src/sase/service/status.py`: read and write the additive `request` field on
`ServiceStatusProc` so the snapshot round-trips it.

### 3. Actions become requests

`src/sase/service/actions.py`:

- `start_service_proc` and `restart_service_proc` issue one `request_service_proc`
  mutation instead of clearing/writing stop markers, then nudge. Delete the `delay`
  parameter, the `time.sleep`, and the double nudge.
- `ServiceProcActionOutcome` gains `generation: int | None` (read back from the mutation
  outcome's snapshot) so callers can wait on it. `stop`, `enable`, and `disable` keep
  their current shape.
- Add `wait_for_service_proc_request(name, generation, *, timeout, poll=0.2)` returning
  the completed `ServiceProcRequest` or `None` on timeout. It polls
  `read_service_state()`; it must never raise on a transient read.

### 4. The host consumes requests

`src/sase/service/host.py`: in `_reconcile_desired`, before the "launch anything
desired" loop, consume every request whose `completed_generation` is behind its
`generation`. Per name, wrapped in its own try/except so one bad name cannot stop the
loop:

- unknown to the composition → complete with `unknown_proc` and an error string.
- not `_desired_running` → complete with `not_desired` and a reason naming the disable
  or the stop, and do not launch. (`RequestProc` cleared the stop marker, so this means
  disabled or unavailable.)
- otherwise: drop any `_pending` entry and reset `_restart_history[name]` to a fresh
  `ServiceRestartHistory()` — an explicit request is a new episode, so backoff and
  `alert_sent` start clean — then:
  - `restart` with a live child → `_stop_child`, `_launch`, complete `restarted` with
    the new pid.
  - `start` with a live child → complete `already_running` with the existing pid.
  - no live child → `_launch`, complete `started` with the new pid.
  - `_launch` that produced no child (spawn failure) → complete `failed` with the
    recorded spawn error.

`_launch` does not return a pid; read it from `self._children[name].process.pid` after
the call. Completion is one `complete_service_proc_request` mutation carrying
`completed_by=f"service-host:{os.getpid()}"`.

### 5. The CLI waits and tells the truth

- `src/sase/main/parser_service.py`: delete `-d/--delay` from `service proc restart`.
  Add to both `service proc restart` and `service proc start` (keep options
  alphabetical, every long option gets a short alias): `-n/--no-wait` ("Return as soon
  as the request is recorded") and `-t/--timeout SECONDS` ("How long to wait for the
  host to confirm (default: derived from the proc's stop timeout)"). Mirror both onto
  `sase scheduler restart` and `sase scheduler start`.
- `src/sase/main/service_handler.py`: `_handle_proc_start` and `_handle_proc_restart`
  capture the current pid from `current_service_status()` before requesting, then wait.
  Default timeout is `max(15.0, entry.stop_timeout_seconds + 10.0)`. Print
  `service proc scheduler restarted: pid 1523548 -> pid 2018178`, or
  `service proc gateway started: pid 91234`, or `already running: pid 91234`. Exit
  non-zero with an honest message when the wait times out
  (`requested; the service host did not confirm within 25s`), when the outcome is
  `not_desired`/`failed`/`unknown_proc`, and when the host is not running at all (detect
  it from `nudge_service_host()` returning False and say so immediately instead of
  waiting).
- `src/sase/main/scheduler_handler.py`: `restart` and `start` route through the same
  helper. The `delay=0.0` call disappears with the parameter.
- `src/sase/main/update_restart.py`: `restart_after_update` reports `restarted` only
  when a completion confirms it, with the message naming both pids; an unconfirmed
  request reports a distinct status whose rendering is a yellow warning, not a green
  tick. Keep `RestartInfo`'s shape compatible with `src/sase/main/update_types.py` and
  its renderer.
- `src/sase/ace/tui/actions/axe.py` (~line 447) and
  `src/sase/integrations/chat_install.py` (~line 410): drop the removed keyword
  argument. The TUI never waits on the UI thread; its existing refresh shows the new
  pid.

### 6. Tests and docs

- `tests/service/test_actions.py`: rewrite the start/restart cases against
  `request_service_proc` (one mutation, one nudge, a generation on the outcome, no
  sleep).
- `tests/service/test_service_host_scenarios.py`: a restart request replaces the child
  with a new pid and completes the generation; a start request on a stopped proc
  launches it and completes; a request for a disabled proc completes `not_desired` and
  launches nothing; a second request while one is pending is consumed exactly once.
- CLI tests for the wait, the `--no-wait` path, the timeout exit code, and the host-down
  path.
- Docs: `docs/cli.md` for the changed options, and the service-control sections of
  `docs/init.md` / `docs/configuration.md` where the stop-marker restart is described.

---

## Last-known-good config keeps the host supervising

sase only. Lands right after the pin so the core's new fatal-layer rule can never freeze
a host for long.

`src/sase/service/host.py`:

- Keep `self._last_good_config: ServiceConfigComposition | None` and
  `self._config_error: str | None`. `_reconcile_once` loads the config in its own try;
  on failure it records the error string and continues the tick with `_last_good_config`
  when there is one. A successful load replaces the last-known-good and clears the
  error.
- Exit observation must not need the config. `_settle_exit` already has everything it
  needs on `running.entry`; change its `config` parameter to
  `ServiceConfigComposition | None` and fall back to `running.entry` when the
  composition is unavailable (the current `config.get(name)` lookup is only there to
  pick up a newer entry).
- `_stop_child` (`host.py:501-503`) must not reload config or state just to settle an
  exit; pass the composition and state the caller already holds, or `None`. A fatal
  config during shutdown must not abort `_stop_all_children` after the first child.
- With no last-known-good at all (a bad config at startup), still write a status
  snapshot and a heartbeat carrying the error, so the TUI and CLI see a degraded host
  instead of a file that silently ages out.

`src/sase/service/host_reporting.py`: `write_current_host_status` takes the composition
it is given (never reloading when one was passed) and an optional `config_error`, and
puts that error on the `ServiceHostRecord` it builds. The core already copies
`record.error` into `host.error`, which is how the CLI and the Services tab will show it
in `surfaces`.

`src/sase/service/host_lifecycle.py`: no behavior change, but confirm the shutdown path
still writes a final status.

Tests in `tests/service/test_service_host_scenarios.py` (its `_service_host` helper
already swaps the composition through a mutable cell, so a raising `get_config` is
easy): a composition that starts raising leaves existing children running, keeps
settling their exits, keeps heartbeating with `error` set, and keeps `status.json`
fresh; recovery re-adopts the new composition; a child that exits while the config is
raising is still settled and restarted per its own entry.

---

## The host honors give_up and says so

sase plus sase-telegram. This is the report's `§3.1 + §3.2 together`, and the receiver
change must land with it so Telegram is not left down after the next reboot.

### 1. The host stops relaunching what it gave up on

`src/sase/service/host.py`:

- Add `self._given_up: dict[str, _GivenUp]` (a new record in
  `src/sase/service/host_models.py` holding the entry signature, the give-up
  `ServiceRestartDecision`, the `ServiceProcLastExit`, the restart count, and the time).
- In `_settle_exit`, when `decision.action == "give_up"` **and the exit was not
  `stop_requested`**, record it. The stop-requested exclusion is load-bearing: a
  signature change and a stop both settle through the same path, and recording those
  would block the relaunch that must follow.
- `_record_spawn_failure` records a give-up the same way, so a `never` proc that cannot
  spawn stops retrying.
- In `_reconcile_desired`, skip a name in loop 3 while `_given_up[name].signature`
  equals the current entry's signature. Drop the record when the signature differs, when
  the entry disappears, or when the proc stops being desired-running, so
  `disable`/`enable` and `stop`/`start` revive it naturally; the `restartgen` request
  path already clears it explicitly.
- `src/sase/service/host_reporting.py`: emit an observation for a given-up proc too
  (alive false, its `last_exit`, its give-up decision, its restart count), so the
  snapshot derives `exited` with the real reason instead of a bare `stopped` with no
  evidence.

### 2. Failures are loud

New `src/sase/service/notifications.py`, called from the host and guarded so a
notification failure can never break the reconcile loop (try/except plus a stderr line,
and the `best_effort_test_state_write_allowed` check the AXE orchestrator's
`_surface_crash_loop` uses, `src/sase/axe/orchestrator.py:197-259`, is the pattern to
copy):

- On `decision.notify` (the core sets it once per crash-loop episode).
- On `give_up` while the proc is desired running.
- `upsert_notification` with `sender="service"`, `icon="!"`, a red `color`, tags
  `["service", "proc", <name>, "error"]`, notes carrying the decision `reason`, the
  restart count, and the proc's log path, and an episode-scoped `dedup_key` such as
  `service:<name>:crash-loop:<host_started_at>:<episode>` so a retry within one episode
  appends a `+1` while a genuinely new episode creates a new row (a `+1` never
  re-delivers, per `docs/notifications.md`). The give-up note must name the command that
  revives it: `sase service proc start <name>`.

### 3. The Telegram receiver stops reporting "not ready" as success

In `sase-telegram` (`src/sase_telegram/scripts/sase_tg_inbound.py`, `_run_receiver`,
around line 5008): keep `return 0` when Telegram is disabled, and return a retryable
`EX_TEMPFAIL` (75, as a named module constant beside
`_RECEIVER_CHAT_ID_MISSING_EXIT_CODE`) when `credentials.get_bot_token()` raises, so the
`restart: on-failure` policy retries with capped backoff until gpg is usable after login
instead of ending the episode as a clean exit. The chat-id-missing exit (78) keeps its
current meaning. Add tests for both exit codes, and document in the receiver's
README/docs that a disabled Telegram now leaves the proc down until
`sase service proc start telegram_receiver` (or a host restart) after
`~/.sase/telegram_is_enabled` comes back.

### 4. Tests

`tests/service/test_service_host_scenarios.py` (no host test covers `give_up` today,
which is why this shipped):

- `on-failure` plus exit 0 stays down across several ticks, and exactly one proc row is
  recorded.
- `restart: never` plus exit 1 stays down; the status snapshot shows `exited` with the
  last exit, not `stopped`.
- A spawn failure under `never` stays down.
- An explicit start request revives both cases and clears the given-up record.
- A signature change revives it without a request.
- A child that fails repeatedly then succeeds (the credential-outage shape) backs off,
  keeps `crash_loop` sticky past the 60 s cap, notifies once, and recovers.
- The notification path is exercised by monkeypatching the module's upsert seam and
  asserting one row per episode.

---

## The CLI and the Services tab show the real failure states

sase only.

### 1. One shared state vocabulary

The core emits `running`, `unavailable`, `disabled`, `stopped`, `crash_loop`, `backoff`,
`exited` for procs and `running`, `stale`, `starting`, `stopped` for the host. Add one
helper (next to `src/sase/ace/tui/_service_health.py`) that maps a state plus its
`desired` to a severity (`ok`, `warn`, `fail`, `muted`) and a style/glyph, and route
every current copy through it:
`src/sase/ace/tui/widgets/_axe_dashboard_output.py:56-69`,
`widgets/bgcmd_list.py:545-604`, `widgets/axe_info_panel.py:236-243`, and
`widgets/_axe_dashboard_status.py:296-303,335`. `{"failed", "error"}` disappears from
all of them.

Severity rules: `crash_loop` and `backoff` are failures; `exited` is a failure while
`desired == "running"` (a clean give-up reads as a warning, using the decision's
`clean_exit` when the snapshot carries one); `unavailable` is a warning; `disabled` and
a `stopped` that the operator asked for are muted, not failures; host `stale` is a
failure distinct from `stopped`, and `starting` is not a failure.

### 2. The pill stops lying about an unreadable snapshot

`src/sase/ace/tui/_service_health.py`: replace `_FAILED_STATES` with the shared severity
call, and give `ServiceHealth` a `known: bool`. `derive_service_health(None)` returns
`known=False` with a summary such as `service status unavailable`, and
`src/sase/ace/tui/widgets/_keybinding_status.py:207-215` renders `SVC ?` in a warning
style for that case (keeping `n/m` teal when healthy and `!` red when unhealthy). Host
chrome (`widgets/axe_info_panel.py:77-90`) grows a `stale` and a `starting` rendering
instead of collapsing everything that is not `running` into `stopped`.

Keep the reason: `src/sase/ace/tui/actions/axe_display/_data.py:340-351` currently
discards the exception message when the snapshot cannot be read. Carry a short message
through `AxeCollectedData` the way `AxeStatusDegradation` already carries an AXE config
error, and show it in the Services header.

### 3. Evidence is rendered

- Services tab: render the per-proc `summary` (the core's human string, today rendered
  nowhere in the TUI) as a chip on the row, and show `restarts` whatever the state — not
  only while running. In the detail pane add the restart decision's `reason`
  (`retrying in 12s`), the stop provenance (`stop.stopped_by`, `stop.reason`), the
  description, and any pending `request` from `restartgen`.
- Host: render `host.error` (the config error `config` now fills) in the Services header
  and in `sase service status`.
- CLI: `sase service status` and `sase service proc list` gain `Restarts` and
  `Last exit` columns; `sase service proc show` gains uptime, restart count, last exit
  and when, the current restart decision, stop provenance, the description, and any
  pending request. Keep the tables scannable and colored by the same severity
  vocabulary.

### 4. Fixtures, goldens, docs

- `tests/ace/tui/test_services_phase_closure.py` uses `state="failed"` at lines 85 and
  143 and locks `derive_service_health(None) == (0, 0, True)` at line 100; move every
  fixture onto real wire states and assert the new unknown behavior.
  `tests/ace/tui/_axe_collector_helpers.py` fakes get the same treatment where they
  assert health.
- PNG goldens change, because every golden captured with `axe_data=` renders the pill
  through `derive_service_health(None)` and will move from teal `0/0` to the unknown
  rendering (the `axe_*`, `launch_context_bar_services_*`, `help_guide_axe_*`, and
  `link_rail_axe_*` families under `tests/ace/tui/visual/snapshots/png/`). Run
  `just fix-tui-screenshots`, inspect the report and every update group before applying,
  and do not let generation stand in for approval.
- Docs: `docs/ace.md` (the Service Health Pill table at ~4821-4840 and the Services tab
  section at ~2818-2842), `docs/cli.md` for the new columns, and a `service` sender row
  in `docs/notifications.md`'s sender table for the notifications `giveup` added.

---

## The captured service environment is context-safe

sase only, and independent of the rest of the epic, so it can run in parallel from the
start.

### 1. Never capture feature flags

`src/sase/service/env.py:54`: drop `SASE_FEATURE_FLAGS` from the captured names. The
host must resolve flags from saved state, because it applies the captured file with
`override_existing=True` and outlives every flag change otherwise. Delete the
now-meaningless drift warning at `src/sase/service/platform.py:496-501` and its coverage
in
`tests/service/test_service_platform_readiness.py::test_readiness_warnings_omit_secrets_and_compare_paths`,
and simplify `tests/service/service_platform_helpers.py::_stub_non_ssh_readiness`, which
only scrubs `SASE_FEATURE_FLAGS` because every agent shell has one. Add a test that pins
the captured allow-list and asserts the flag snapshot is never in it. Make the
allow-list an explicit, documented module constant while you are there.

### 2. `init --check` settles

`environment_files_match` (`env.py:195-200`) is exact dict equality over every captured
key, so the per-shell SSH handles and PATH differences make `sase service init --check`
report `needs_attention` from every shell — and then advise the one command §3 makes
dangerous. Compare only what the host actually depends on: ignore `SSH_AUTH_SOCK` and
`SSH_AGENT_PID` (the host already drops a dead socket at start, `env.py:181-186`), and
compare `PATH` after normalizing it — strip empty and duplicate entries, strip trailing
slashes, and drop ephemeral workspace venv entries (`src/sase/axe/_process_start.py`
already recognizes a `sase_<N>` workspace path). A real PATH change still rewrites the
file. Add direct tests for `environment_files_match`; there are none today.

### 3. `init --yes` refuses an agent shell

`apply_service_init` (`src/sase/service/platform.py:203-272`) is the single choke point
for both `sase service init --yes` and `sase init service`. Refuse, with exit 2 and a
message that names a login shell as the fix, when the calling process is an agent
context (`os.environ.get("SASE_AGENT")` or `SASE_AGENT_NAME`, the same signal
`src/sase/artifact_read_links.py:29-30` uses) or when PATH contains an ephemeral
workspace venv. Add `-a/--allow-agent-env` to `sase service init` (and the
`sase init service` alias) to override it deliberately; do not overload `-f/--force`,
which already means "non-default `SASE_HOME`". Planning (`service_init_plan`, `--check`,
`--diff`) stays available everywhere — only the capture-and-write path is guarded.

### 4. Docs

`docs/configuration.md:4054-4060` still lists `SASE_FEATURE_FLAGS` in the captured set
and has not been updated for the `SASE_TMPDIR`/`SASE_HOME` capture that landed in
`529d7d325`; fix both. `docs/init.md:192-221` should state the currency rule (which keys
are compared, and that SSH handles are not), and the new refusal with its override flag.

---

## Acceptance

The epic is done when, on a live host:

- Stopping `~/.sase/telegram_is_enabled` or breaking the receiver's credentials produces
  bounded retries and one notification, never a 2-second relaunch loop, and
  `~/.sase/procs/procs.jsonl` no longer flushes its retained history in under a minute.
- `sase scheduler restart` changes the pid and prints both, and exits non-zero when it
  cannot confirm.
- `sase service status`, `sase service proc show scheduler`, and the Services tab all
  show the same state word, the same restart evidence, and the host's config error when
  there is one.
- Saving a deliberately broken machine overlay leaves every running proc running, shows
  the error in the header, and does not disable `gateway` or `telegram_receiver`.
- `sase service init --check` reports `current` from a login shell with no pending
  action, and `sase service init --yes` refuses from an agent shell.
