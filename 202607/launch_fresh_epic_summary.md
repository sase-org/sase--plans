---
tier: tale
title: Launch-fresh epic summary lookup
goal: 'Epic clan-summary generation recovers from a warm but stale sidecar bead store,
  records actionable fallback diagnostics, and retains a bounded launch delay without
  allowing summary failures to abort an agent launch.

  '
create_time: 2026-07-20 11:03:15
status: done
prompt: 202607/prompts/launch_fresh_epic_summary.md
---

# Plan: Launch-fresh epic summary lookup

## Context

The built-in epic clan-summary script currently opens the workspace-visible bead store once and silently emits a minimal
fallback when the requested epic is absent. A newly created epic can therefore be missing from an otherwise valid, warm
sidecar clone at launch time. The launcher already routes a summary script's stderr to the agent log, but the script
discards the exception, and its ten-second execution budget is too short for a synchronous remote integration.

This phase keeps the existing `summary_script=` architecture and summary wire format. It does not change rendering
richness, the TUI, the Rust core, or the parent epic lifecycle.

## Implementation

1. Add a focused blocking-refresh entry point in the bead sync layer for the currently resolved remote-backed sidecar
   store. It will reuse the existing transactional SDD integration path and cooperative store-write lock to fetch and
   rebase the checkout, report unsuccessful integration as an exception, and deliberately omit the worker's push step so
   a read-recovery attempt cannot publish unrelated local state. In-tree and non-remote stores remain no-ops.
2. Restructure the epic summary loader around a warm-read fast path. Only a missing epic lookup (`KeyError`) may trigger
   the blocking refresh, and only when the configured bead refresh mode is not `off`; after a successful refresh
   attempt, reopen the bead project and retry once. Invalid bead types and other read errors fall through directly to
   the existing best-effort fallback.
3. Make every final fallback diagnosable by writing a concise explanation and the causal traceback to stderr before
   printing the existing Rich-markup fallback to stdout. Preserve exit status zero so malformed, unavailable, or slow
   summary data never prevents launch.
4. Raise the clan summary script's default timeout from 10 to 20 seconds to cover the one blocking integration attempt
   while preserving the launcher's existing process-group timeout and failure behavior.

## Verification

Extend the epic summary script tests to cover:

- an initially missing epic that becomes available through the monkeypatched integration hook and renders the full
  summary after exactly one retry;
- refresh mode `off`, which never invokes the integration hook and emits the fallback plus stderr diagnostics;
- failed refresh or failed retry, which still exits zero, keeps stdout limited to the fallback, and exposes traceback
  text and the explanation on stderr;
- the existing happy path, including no refresh call on a warm-store hit.

Add or adjust a focused timeout assertion so the 20-second launcher default is explicitly pinned. Run the targeted bead
and clan-summary runner suites, then run `just install` followed by the repository-mandated `just check`.

## Risks and constraints

- Blocking integration can delay the declaring member, but it occurs only after a missing lookup and remains bounded by
  the launcher timeout.
- Concurrent background refreshes must remain safe by using the established transactional integrator and store lock
  rather than ad hoc Git commands.
- The retry is intentionally single-shot; persistent remote, repository, or data failures degrade to the logged fallback
  instead of looping or failing launch.
- Do not edit SASE memory files, generated provider instructions, rendering fixtures, or the parent `sase-85` epic, and
  do not create additional beads.
