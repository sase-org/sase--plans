---
tier: tale
title: Surface retry backoff immediately for agent families
goal: 'ACE shows a live agent family as RETRYING during provider backoff as soon as
  retry_state.json is observed, even when its newest plan or coder child has a failed
  workflow snapshot, without reviving terminal failures or slowing the TUI.

  '
create_time: 2026-07-17 12:15:10
status: done
prompt: 202607/prompts/early_retrying_status.md
---

# Plan: Surface retry backoff immediately for agent families

## Context and diagnosis

The photographed agent was not waiting for SASE to decide whether to retry. Its second attempt failed at approximately
11:51:34, the runner immediately entered a five-minute retry backoff, and execution resumed at approximately 11:56:34.
The first screenshot was captured inside that interval, so the `FAILED` row concealed an already-scheduled retry.

The retry producer in `src/sase/axe/run_agent_exec_retry.py` is behaving correctly: it atomically writes
`retry_state.json` with `status: retrying`, the retry number, the maximum retries, and the next-attempt time before
sleeping. ACE's watcher also already treats that marker as agent-relevant and routes it through the bounded
artifact-delta refresh path. The gap is later in the Agents-tab model pipeline:

- family status aggregation mirrors the failed coder child onto the live plan-family root;
- `_apply_loaded_agent_disk_projections()` only reads retry state for a row whose current display status is exactly
  `RUNNING`, so the now-`FAILED` root is skipped;
- an exact artifact-delta refresh can project the root independently and then lose that projection when cached family
  children are reattached and relationships are normalized again.

The earlier direct-workflow dedup fix therefore covers anonymous workflows but not the plan-to-coder family handoff
shown here. This is an ACE presentation and aggregation issue; it does not require changing retry scheduling or the
shared Rust agent-scan contract.

## Live-runner-aware retry projection

Make live runner provenance independent of the mutable display status. Record a runtime-only, non-serialized indication
that an agent came from a verified live RUNNING claim (including the equivalent home-agent marker), and preserve it when
a RUNNING row is merged into its richer workflow row. Do not infer this solely from `status == "RUNNING"`, because
family normalization is intentionally allowed to replace that label with semantic plan, question, working, or failure
states.

Use that provenance when applying cached `retry_state.json` data so a live family root remains eligible after a child
failure has been mirrored onto it. Keep the normal fast path for ordinary running agents, and handle exact artifact
deltas without broad project or artifact scans: preserve matching cached live provenance across the bounded Tier-1
replacement, or perform an equivalently bounded PID liveness check before allowing an exact delta to promote a
non-running row. A dead runner plus a stale retry marker must remain terminal.

Project all retry metadata together (`retry_status`, counts, countdown target, fallback fields, and last-known mode) and
let `status: retrying` promote the visible row to `RETRYING`. `running_retry` and `running_fallback` should retain the
semantic working-family status while still exposing the retry count/fallback annotation. Keep these reads in the
existing worker-side snapshot cache; do not add filesystem or process probes to render paths, the UI thread, or the
Textual message pump.

## Family-status precedence and delta convergence

Teach serial agent-family aggregation that a verified live root with an active `retrying` projection outranks a terminal
snapshot from the child whose work will be retried. The collapsed family row should therefore stay in the Running bucket
as `RETRYING (<countdown>)`, while the expanded child can retain its failed-attempt evidence. Once the retry state
advances to `running_retry` and the child workflow is active again, the family should naturally return to `WORKING PLAN`
or `WORKING TALE`; when the runner truly exits without another retry, normal terminal `FAILED` behavior must win.

Apply the same precedence after incomplete Tier-1/artifact-delta rows are merged with cached children. A
`retry_state.json` watcher event must be sufficient to move an already-rendered failed family to `RETRYING` on the next
normal Agents refresh; it must not wait for the later child `workflow_state.json` update that announces the next
attempt. Reuse the existing coalesced, bounded delta refresh and snapshot cache so the fix respects navigation and
prompt-input activity gates and introduces no new refresh loop.

Review active/dismissed filtering touched by this projection so `RETRYING` is consistently treated as live during the
same load. Do not allow a previously dismissed terminal bundle or a stale retry file to delete, hide, or resurrect the
current live row.

## Regression coverage

Add a loader-level reproduction of the photographed state with:

- a live plan-family root whose root workflow has already completed planning;
- an approved planner child and a newest coder child with a failed workflow state;
- a root `retry_state.json` containing `status: retrying`, retry `2/3`, and a future retry timestamp;
- a matching live runner claim/PID.

Assert that a full/Tier-1 load yields one family root with `RETRYING`, the correct retry counts and countdown, and
Running-bucket membership. Cover the negative cases where the PID/claim is no longer live or no retry marker exists,
which must remain `FAILED`.

Add a focused family-aggregation test proving that a failed coder child cannot overwrite an active retry projection, and
that `running_retry` plus an active coder returns to `WORKING TALE`/`WORKING PLAN`. Add an incomplete-load merge test
that starts with a cached failed family, applies an exact root-artifact retry delta, and still finishes as `RETRYING`
after children are reattached and relationships are renormalized.

Extend the real-loader ACE retry visual coverage with a plan-family countdown case. Verify both the visible `RETRYING`
countdown and the detail-panel retry fraction so the test catches the original misleading `FAILED` plus `Retries: 0/0`
combination. Keep the existing ordinary-agent retry, fallback, and completed retry-chain cases passing to guard against
regressions outside plan families.

## Validation

Run the focused loader, family-status, delta-merge, retry snapshot-cache, and ACE retry visual tests while iterating.
Confirm the watcher still recognizes `retry_state.json` and that the change uses the established artifact-delta refresh
rather than broad reloads. Run `just install` first as required for an ephemeral workspace, then finish with
`just check`; run `just test-visual` as the dedicated snapshot verification when the intentional visual case is added.
Inspect any PNG failure artifacts before accepting a snapshot update.

Acceptance means a retrying family becomes visibly `RETRYING` within the existing watcher/auto-refresh cadence after the
backoff marker is written, instead of remaining `FAILED` until the next attempt begins, while terminal failures and TUI
responsiveness remain unchanged.
