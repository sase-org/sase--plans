---
tier: tale
title: Self-healing agent dependency waits
goal: "Waiting agents start automatically when their dependencies complete even if the
  AXE waits lumberjack is stopped or temporarily unavailable, without duplicating
  expensive scans or weakening successful-completion semantics.

  "
create_time: 2026-09-09 19:53:22
status: wip
---

# Plan: Self-healing agent dependency waits

## Context and root cause

The `sase-6k.3` phase was correctly configured to wait only for `sase-6k.1`, and the
dependency had an authoritative successful `done.json` by 12:16:10. However, the AXE
waits lumberjack had received a shutdown signal at 12:12:33 and did not restart until
12:31:03. The phase runner had performed its one startup dependency check before `.1`
completed and then polled only for a `ready.json` marker normally written by the
external `wait_checks` chop. The Agents tab independently recomputed the dependency as
complete, which explains why it showed a checkmark while the runner remained parked. The
manual **Run now** action finally wrote the missing marker at 12:20:01, matching the
phase's persisted `wait_completed_at` timestamp.

The defect is therefore an availability dependency in the wait protocol: a live waiting
runner cannot make progress after its initial check unless a separate AXE process (or a
user action) publishes readiness. AXE should remain the normal centralized resolver, but
a temporary daemon outage must not strand otherwise runnable agents.

## Shared wait-resolution pass

Extract the marker scan and dependency-index evaluation currently owned by
`sase_chop_wait_checks` into a reusable, non-CLI wait-resolution service. Keep the
existing dependency semantics in `sase.core.wait_dependency_resolution` as the single
source of truth: only successful current dependencies resolve; failed, killed, stale,
malformed, or incomplete dependencies continue to fail closed; family/workflow
generations, submitted planners, identity waits, memoized dependencies, retries, and
self-exclusion retain their current behavior.

Have one resolution pass build one dependency index, evaluate all current waiting
markers against that snapshot, and publish compatible `ready.json` markers. Make
publication race-safe and idempotent when AXE, a runner fallback, and the TUI's manual
run-now action overlap. Return structured counts/reasons so the AXE chop can preserve
its summaries and the runner can log useful fallback diagnostics without duplicating the
resolver.

## Runner-owned outage fallback

Extend the named-dependency loop in `src/sase/axe/run_agent_wait.py` so `ready.json`
remains the normal handoff, while the waiting process can recover when the AXE waits
lumberjack is absent or its cycle heartbeat is stale. Derive health from durable
lumberjack state with a conservative grace period relative to the configured wait-check
interval. When fallback is needed, coordinate waiting processes through a global
non-blocking lock/cadence so only one of them performs the shared all-waiter resolution
pass; the rest continue cheap marker polling. This preserves the existing single-index
efficiency instead of making every parked process repeatedly scan artifact history.

Keep fallback best-effort and fail-closed. A scan/read/lock error must leave the agent
waiting for a later fallback or AXE cycle, while kill signals still exit promptly.
Preserve time-floor behavior after dependency completion, durable `wait_completed_at`,
marker cleanup, post-wait code refresh, edited wait directives, and manual run-now
compatibility. Emit a concise diagnostic when a runner takes over for stale AXE state
and when that pass resolves waiters, so a future outage is visible in runner and AXE
artifacts.

## Verification and regression coverage

Add deterministic tests around the shared service, runner loop, and existing chop
adapter:

- Reproduce the incident in an isolated SASE home: park a runner while its dependency is
  unfinished, publish a successful `done.json` with no waits lumberjack running, and
  verify the runner proceeds automatically without a manual ready marker.
- Verify a healthy waits heartbeat keeps AXE as the primary resolver and does not
  trigger redundant fallback scans.
- Exercise multiple concurrent parked agents during an AXE outage and assert that one
  coordinated pass can release all satisfied waiters without duplicate scans or
  marker-write races.
- Confirm failed, killed, malformed, missing, and superseded dependencies stay parked,
  while later successful retries, workflow/family completion, submitted-planner waits,
  identity waits, and self-exclusion still follow the shared resolver's established
  rules.
- Cover races with AXE-written and TUI-written `ready.json`, kill handling, optional
  duration/absolute-time floors, durable completion timestamps, and cleanup of both
  waiting and ready markers.
- Keep the existing `wait_checks` summaries and tests passing through the new service,
  and document that AXE is the primary fast resolver with a runner-owned outage
  fallback.

Run the focused wait/chop/runner test suites first, then install the workspace as
required and run `just check` for the repository-wide validation gate.
