---
tier: tale
title: Service host runtime and CLI
goal: 'Ship the flag-gated per-machine SASE service host, service-proc and scheduler
  command surfaces, scheduler ownership handover, durable proc recording, and detached
  fallback required by phase sase-11y.4 without exposing unfinished platform-unit
  integration or allowing two supervisors to own the scheduler.

  '
size: medium
proposed_by: bbugyi200.athena.sase-11y.4
bead: sase-11y.4
status: done
---

- **PARENT:**
  [202609/service_host_1.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_1.md)
- **BEAD:**
  [sase-11y.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.4.md)

# Plan: Service host runtime and CLI

## Context and invariants

The Rust-backed service config, state, status, restart-decision, and proc-wire facades
already exist, as does the shared `sase.supervision` library. This work is the Python
runtime and command layer that consumes those prerequisites. It must keep the
architecture's strongest invariant: the legacy AXE launcher and the new service host
must never supervise the scheduler simultaneously.

All new user-facing behavior stays behind a default-off `service_host` beta flag. With
the flag disabled, existing `sase axe` behavior remains intact and the new service
commands fail with a concise opt-in diagnostic rather than mutating service state.
Create the flag through `sase flag new`, with authored enabled, disabled, and
removal-gate text, so its registry entry and required flag lifecycle record stay
consistent. This required flag record is part of the assigned epic scaffolding; do not
create any unrelated task beads. Discovered follow-up work is recorded only as a
`PROPOSED FOLLOW-UP:` note on `sase-11y.4`.

The host resolves only the machine-level config composition supplied by
`load_service_config`; it must not read project-local service definitions. A named
daemon is always a direct host child, while a transient oneshot always uses the existing
detached proc-shell path. Host restart, config reload, or proc-store recovery must never
replay a oneshot.

## Implementation

1. Add the service runtime primitives under `src/sase/service/`.
   - Add a single-instance lifetime flock and host probe/control helpers under
     `~/.sase/service`. A foreground `run` process owns the lock for its entire
     lifetime, publishes a PID-guarded heartbeat through the Rust state facade, and
     clears only its own host record on clean shutdown. Treat a held lock as
     authoritative even when an old heartbeat or PID is stale.
   - Implement signal handling for graceful `SIGTERM`/`SIGINT` shutdown and a `SIGUSR1`
     reconcile nudge. The main loop reconciles at roughly one-second intervals and
     refreshes the heartbeat and atomic status snapshot without restarting itself.
   - Recompose service config on the existing config stat token. Compare a stable
     launch-definition signature so added enabled procs start, removed or
     disabled/stopped procs stop, and changed definitions restart. Honor machine-local
     enablement overrides and boot-scoped stops from the locked service state. A
     concurrent stop/restart race is resolved by reading desired state again before
     every launch or scheduled retry.
   - Resolve configured argv, shell commands, cwd, `${VAR}` environment references, and
     the v1 service-proc environment contract. Add `SASE_SERVICE_PROC` plus a per-proc
     state directory, and read an optional bounded/validated `status.json` report for
     the host snapshot. Builtin launchers remain explicit; this phase implements
     `scheduler`, while a configured `gateway` remains visible but unavailable until its
     later phase supplies that launcher.
   - Spawn daemon procs as direct children in their own process sessions. Pump combined
     output into the stable bounded `~/.sase/service/procs/<name>/output.log` using the
     configured byte limit. Stop with the configured signal and timeout, then escalate
     to `SIGKILL`.
   - Give every launch a new durable proc row carrying `{name, mode: daemon, source}`
     service metadata. Claim and settle those rows from the host as children start and
     exit, and expose their IDs and exit details in the service status snapshot. Route
     generic `sase proc kill` for a live named service row into the host's boot-scoped
     stop intent instead of signaling the host/supervisor PID.
   - Feed each exit or spawn failure through the Rust restart-decision facade with the
     entry's policy and `success_exit_codes`; preserve returned history, backoff,
     reason, and crash-loop state in observations. Give up after a clean `on-failure`
     exit, notify only when the decision requests it, and never reschedule a proc whose
     current desired state is no longer running.

2. Implement scheduler ownership and host lifecycle control.
   - Make the scheduler builtin invoke the canonical foreground scheduler entrypoint
     (`sase scheduler run`) with the existing `axe.*` configuration and CLI override
     semantics, without adding a second config model.
   - Before the first host-owned scheduler launch, probe the existing AXE lifecycle
     lock. If a legacy orchestrator owns it, stop that orchestrator without publishing a
     persistent legacy stop, wait for the lock to clear, and only then start the
     scheduler child. Refuse the launch with a visible status error if handover cannot
     establish single ownership.
   - Implement the lock-guarded detached fallback for `sase service start`. It launches
     `sase service run` outside the caller's terminal/cgroup using the existing
     detach-scope mechanism, waits for a fresh heartbeat, and reports
     `running detached (no platform unit installed)`. Concurrent starts converge on one
     host. `stop` signals and waits for that host; `restart` performs a verified
     stop/start. Do not daemonize inside `service run` itself.
   - Keep platform-unit installation out of this phase. `service init` and
     `service uninstall` exist behind the beta flag but return explicit phase-specific
     errors that point to the detached fallback until `sase-11y.5` implements them.

3. Add the public CLI and rendering surfaces following the CLI rules.
   - Register lazy top-level `service` and canonical `scheduler` command trees, retain
     `axe` as an accepted alias, route both from the main entrypoint, and update
     exhaustive/compact command inventories where appropriate. Keep subcommands and
     options alphabetical, give every public long option a short alias, and provide
     contrasting help for host `service run` versus transient `service proc run`.
   - Add `sase service init|logs|restart|run|start|status|stop|uninstall`. Bare
     `sase service` must be harmless (help or status), never foreground `run`. Status
     and logs support stable human output and JSON where the surrounding command family
     exposes a machine contract.
   - Add `sase service proc disable|enable|list|logs|restart|run|show|start|stop`. Bare
     `service proc` centrally delegates to `list`. Enable/disable write machine-local
     policy overrides; start clears the boot stop; stop records it; restart is ordered
     stop/clear/nudge; show includes config and effective enablement provenance.
     List/show/status are useful even while the host is down by deriving a fresh
     snapshot from config and state when the persisted host snapshot is absent or stale.
   - Extend `ProcSubmitRequest` and reservation plumbing to carry a validated
     `ProcServiceBlock`. Implement `service proc run` with `--cwd/-c`, `--label/-l`,
     `--project/-p`, and `--workspace/-w`, submitting exactly once through
     `submit_proc_request` with `{mode: oneshot, source: transient}` and no stable
     service name. It is never added to daemon desired state and is never replayed by
     the host.
   - Under the beta flag, make `sase scheduler run` the foreground legacy orchestrator
     entrypoint and make scheduler `start|stop|restart|status` delegate to the
     corresponding `service proc` control for `scheduler`. With the flag off, retain the
     current AXE lifecycle paths so both flag states are testable and the unfinished
     epic does not change production ownership.

4. Add focused tests before enabling any path.
   - Unit-test flag-off diagnostics and flag-on parser/handler routing, alphabetical
     help, harmless bare groups, scheduler/AXE alias behavior, JSON/human status,
     enablement provenance, and transient request metadata.
   - Exercise host single-instance behavior with concurrent starts, stale heartbeat/lock
     recovery, signal shutdown, periodic and nudged reconcile, config add/remove/change
     reload, stable bounded logs, direct-child proc recording/settlement, and
     environment/status-report handling.
   - Exercise restart behavior for clean success codes, spawn failures,
     backoff/crash-loop give-up, a stop arriving while a restart is pending, and config
     disable/removal while a retry is pending.
   - Exercise scheduler handover with a simulated legacy lock holder and prove the host
     does not spawn until the old owner exits. Add a negative case where failed handover
     leaves the scheduler unstarted.
   - Exercise detached fallback startup/status/stop without touching real units, and
     prove two simultaneous starts result in exactly one host.
   - Seed unsettled transient oneshot rows at host startup and prove the host never
     launches or replays them; use existing proc reconciliation only to settle genuinely
     lost supervisors as unknown/error.

## Verification and completion

Run the focused service, proc, parser, feature-flag, AXE lifecycle, and shared
supervision tests while iterating. After all tracked edits, read the `lint_and_test.md`
reference memory and run its required verification, including `just check`; use the
repository's targeted test commands rather than invoking an unnecessarily exhaustive
matrix unless the memory requires it.

Before closing, run `sase bead epic-symbols sase-11y.4`. Remove phase-owned Justfile
exemptions for symbols now consumed by the runtime/CLI. Re-key any symbol whose first
real consumer belongs to a still-open later phase (for example platform integration or
the Services tab) to that precise bead rather than leaving it attached to this closing
phase. Re-run the symbol check until no `sase-11y.4` entries remain, then rerun the
required checks.

Close only the assigned phase with:

```bash
sase bead close sase-11y.4 --note "<concise tests, checks, flag-state coverage, handover, race, and no-replay verification>"
```

Do not close `sase-11y` or any ancestor. If an out-of-scope issue remains, append one
`PROPOSED FOLLOW-UP:` note to `sase-11y.4` and leave triage to the epic land agent.
