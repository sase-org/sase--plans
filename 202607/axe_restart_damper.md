---
tier: tale
title: Journal and damp axe fleet restarts
goal: 'Axe lifecycle activity is attributable through a bounded journal, stale maintenance
  ownership cannot survive PID reuse or block healing, and repeated orchestrator starts
  are damped with one durable notification instead of causing a silent fleet-wide
  restart storm.

  '
create_time: 2026-07-20 16:43:17
status: done
prompt: 202607/prompts/axe_restart_damper.md
---

# Plan: Journal and damp axe fleet restarts

## Context and current baseline

The axe lifecycle already has serialized start/stop helpers, a persistent desired-state marker, verified restart
retries, and rate-limited ensure-failure notifications. It does not retain successful lifecycle history, so a dead or
repeatedly healed orchestrator cannot be traced back to the CLI, ACE, update, plugin, or ensure source that initiated
it. Maintenance markers contain only a PID and wall-clock timestamp, which leaves a live-but-recycled PID
indistinguishable from the original owner. The ensure path also checks desired state and liveness without first clearing
a stale maintenance marker, allowing a marker to remain while the fleet is down.

Prometheus exposition and its HTTP bind were removed from the current telemetry implementation. Legacy
`telemetry.prometheus` configuration is ignored and already covered by a regression test, so there is no active
bind-conflict path to change in this phase. Keep that behavior intact rather than reintroducing server lifecycle code.

## Bounded lifecycle journal and source attribution

Add an axe-backend lifecycle journal under the axe state directory with a stable JSONL record shape. Each record will
identify the lifecycle event and outcome, source/reason, relevant orchestrator PID, current desired-state snapshot, and
active maintenance marker. Writes must be best-effort and must never turn a successful lifecycle action into a failure.
Serialize cross-process append/compaction with a host-local lock and retain complete recent JSONL records within a fixed
byte cap so the journal remains parseable and bounded.

Record actual orchestrator starts at the point where the process owns the lifecycle lock and publishes its PID, using
the existing start-source environment handoff so direct `sase axe start`, helper-launched starts, ensure healing,
verified restarts, and update-triggered starts share the same path. Record stop outcomes in the stop backend and overall
restart outcomes in the verified restart backend, including unsuccessful/no-op outcomes where they explain operator
intent. Thread explicit source strings through ACE lifecycle actions and update/plugin restart helpers while keeping
dependency-injected test callables supported, so dev-update and other callers no longer collapse into an unattributed
generic restart.

Expose a tolerant journal reader for recent valid records. It should skip malformed or historically truncated lines and
provide the recent successful-start window used by the damper without requiring callers to understand persistence
details.

## Maintenance owner identity and startup cleanup

Extend newly written maintenance markers with a process identity that remains stable for the life of the owner and
changes when a PID is recycled (Linux process start ticks, paired with boot identity when available). Preserve
compatibility with existing markers. Stale detection will continue to clear malformed, expired, and dead-owner markers,
and will additionally clear a marker when the live PID's current identity differs from the recorded owner identity.

Run stale-maintenance cleanup near the start of every serialized `ensure_axe` check, before desired-state/liveness
decisions, and during actual orchestrator startup before lumberjacks are spawned. Keep cleanup best-effort and preserve
a genuinely live maintenance marker. Journal snapshots should therefore show any maintenance state that is still
legitimately active rather than a known dead owner.

## Restart-storm damper and notification

Before `ensure_axe` heals a missing orchestrator, inspect recent successful start records. Use conservative module-level
defaults matching the design intent (five starts in thirty minutes) and permit test injection of time/threshold values.
When the threshold is reached, return a rate-limited ensure result without starting another fleet, write the normal
ensure cadence marker, and emit a durable notification that names the recent sources and points operators to the
lifecycle journal.

Persist a small storm-notification marker/signature so repeated ensure polls for the same episode do not produce
duplicate notifications. A later episode outside the window can notify again. Notification and marker failures remain
best-effort; damping itself must still hold. Existing explicit-stop behavior remains authoritative and running
orchestrators remain healthy no-ops.

## Tests and validation

Add focused tests for bounded, parseable journal retention and lifecycle record contents; source propagation across
direct start/stop/restart, ensure healing, and update-triggered restart paths; and tolerant reading of malformed
historical rows. Extend maintenance tests for recorded process identity, recycled-PID clearing, legacy-marker
compatibility, and startup/ensure cleanup while preserving live owners.

Extend ensure tests to seed recent start history, assert the threshold suppresses the start callback, verify one
notification containing the contributing sources, and prove repeated polls do not notify again. Keep existing
desired-state serialization, retry verification, and notification tests passing. Retain the legacy-Prometheus-config
regression as evidence that bind conflicts cannot affect current orchestrator startup.

Run targeted axe lifecycle, maintenance, notification, telemetry-config, and update/plugin restart tests during
development. After implementation, run `just install` and the repository-required `just check`; address all failures
before closing `sase-8g.6`. Do not alter or close the parent epic.
