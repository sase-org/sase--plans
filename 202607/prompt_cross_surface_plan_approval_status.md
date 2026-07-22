---
tier: tale
title: Prompt cross-surface plan approval status reconciliation
goal: 'ACE promptly replaces pending tale and epic review labels with their canonical
  approved or handoff status after the gate is resolved from Telegram or any other
  external surface, without broad notification scans or TUI stalls.

  '
create_time: 2026-07-22 06:41:33
status: wip
---

- **PROMPT:** [202607/prompts/prompt_cross_surface_plan_approval_status.md](prompts/prompt_cross_surface_plan_approval_status.md)

# Plan: Prompt cross-surface plan approval status reconciliation

## Context and verified diagnosis

ACE deliberately keeps an in-memory status override for notification-driven states. When a tale or epic review arrives,
the matching family/root row gets a pending `TALE` or `EPIC` override immediately, before all runner artifacts have
necessarily converged. The TUI approval path replaces that same override with `TALE APPROVED` or `EPIC APPROVED` in its
modal callback, which is why approval from ACE looks instantaneous.

Telegram already uses the shared v2 notification-gate executor. A successful Telegram selection writes the terminal gate
response, persists `plan_approved`/`plan_action` into the agent metadata, refreshes the artifact index, marks the shared
action handled, and dismisses the source notification. The linked Telegram integration therefore does not need a
separate approval or status implementation.

The lag is caused by ACE's reconciliation boundary:

- Notification polling reads active rows only. Once the shared executor dismisses a Telegram-resolved notification, it
  is absent from the next snapshot, so `_auto_dismiss_external_plan_response()` never gets a chance to replace the
  pending override.
- Artifact refresh can load the durable approved/handoff state, but `should_clear_loaded_agent_status_override()` has no
  transition for a pending `PLAN`/`TALE`/`EPIC` override that has been overtaken by a loaded `PLAN APPROVED`,
  `TALE APPROVED`, `EPIC APPROVED`, committed, working, or terminal handoff state. The stale pending override is
  reapplied over the fresher loaded status.
- The remaining v2 response-inspection branch assumes the obsolete `response["result"]` shape. Current query-backed
  gates store `selected_option_ids` plus `option_results` and must be interpreted through the canonical plan-gate
  translator. Thus even a v2 response that remains visible would not reliably derive the tale/epic action.

The reported run corroborates this chain: its Telegram gate response was durably written at 06:30:05, while the 06:30:35
ACE screenshot still rendered the cached pending `TALE` label. This is a projection/reconciliation defect, not Telegram
callback latency or lost approval data.

## Canonical override progression

Teach the loaded-state override reconciliation to recognize when durable plan state has overtaken a
notification-installed pending review override.

- Define the transition in terms of shared status/action semantics rather than Telegram as a special case. `PLAN`,
  `TALE`, and `EPIC` overrides should yield when loaded metadata and family normalization prove the corresponding
  approval, commit, active coder handoff, epic creation, rejection/feedback, or completed handoff.
- Preserve a pending override across raw `STARTING`, `RUNNING`, `WAITING`, or an unchanged pending state when no durable
  post-review evidence exists. This prevents the review label from flickering away during the short interval between
  notification delivery and artifact persistence.
- Apply the rule to family roots and their concrete/synthetic planner members, using the existing canonical
  `plan_action` and normalized statuses so tale, epic, commit-only, and ordinary plan approvals remain distinct.
- Keep optimistic approved overrides until the loaded graph advances to a working/terminal handoff, preserving the
  current direct-TUI behavior.

## Bounded cross-surface refresh trigger

Make notification polling treat the disappearance of a previously cached active `PlanApproval` or `EpicApproval` row as
a reconciliation event, not as a new alert.

- Capture the prior cached active notifications before installing the new snapshot, and identify only plan/epic approval
  IDs that disappeared.
- Resolve their already-known agent/artifact identity from the cached row and route a refresh through the existing exact
  artifact-delta scheduler. Coalesce duplicates and retain the established fallback only when an exact artifact
  directory cannot be resolved.
- Do not load all dismissed notifications, add a new full agent-list rebuild, ring the bell, or toast for removals. A
  Telegram/TUI/CLI race and repeated polls must be idempotent.
- Keep snapshot and response-file I/O off the Textual event loop. The UI-thread portion should only apply a prepared
  transition, update the in-memory override, and patch/refilter the affected row through the existing fast path.
- Preserve watcher-independent convergence: a handled-notification transition should request the targeted delta even if
  inotify misses or coalesces the metadata write, rather than waiting for the long sanity refresh.

## V2 and legacy response compatibility

Clarify the remaining external-response compatibility path instead of leaving two response formats partially conflated.

- For v2 plan gates, derive any needed response action with the canonical gate translation used by the executor
  (`selected_option_ids`/`option_results`), or rely on the executor-persisted agent metadata when that is the
  authoritative signal. Do not read a nonexistent top-level `result` field.
- Retain the legacy `plan_response.json`/marker/request-disappearance behavior for genuinely in-flight legacy approvals,
  including its rejection behavior.
- Avoid duplicating archive/launch side effects during ACE reconciliation; the shared executor remains their owner. ACE
  only reconciles projection state and performs idempotent compatibility persistence where legacy data requires it.
- Keep the solution in the main SASE repository. The linked Telegram caller is already correctly transport-agnostic and
  should be covered as an external `source="telegram"` executor path rather than changed independently.
- Treat this as ACE presentation/cache invalidation work: the shared gate domain contract is already correct, so no
  Rust-core wire/API change is expected unless implementation uncovers contradictory backend evidence.

## Regression coverage

Add focused tests that reproduce the stale override before verifying the full cross-surface flow.

- Parameterize pending `PLAN`/`TALE`/`EPIC` overrides against their approved, committed, working, epic-created,
  rejected/feedback, and completed handoff successors; assert that stale pending overrides clear. Also assert that raw
  pre-persistence `STARTING`/`RUNNING`/`WAITING` states and unchanged pending states retain the override.
- Seed the notification poller's previous cache with tale and epic approvals, return a new snapshot where each is
  dismissed, and assert that exactly the matching artifact delta is requested without a bell, toast, unread increase,
  broad refresh, or `include_dismissed=True` scan. Cover muted reviews and an unrelated disappeared notification.
- Exercise current v2 tale and epic response envelopes, including the tale `approve + commit` selection, and verify
  canonical `tale`/`epic` action and status derivation. Retain explicit legacy response tests.
- Add an integration-style regression that resolves a real plan gate with `source="telegram"`, confirms the notification
  is dismissed and metadata is persisted, starts from ACE's pending override, runs the targeted
  reconciliation/finalization, and observes `TALE APPROVED` or `EPIC APPROVED` on the affected row.
- Preserve direct TUI approval, question override, alert-suppression, family aggregation, and subsequent
  `WORKING TALE`/terminal status behavior.

## Validation and acceptance

Before running repository checks, install the workspace dependencies as required by the project instructions. Run the
focused notification polling, status-override clearing, plan-gate, family-status, and artifact-delta tests, then run
`just check` for the full lint/type/test suite.

The change is accepted when a tale or epic resolved from Telegram (or another external executor surface) leaves the
pending review state and displays the canonical approved/handoff status during the next bounded notification/artifact
reconciliation, without requiring an ACE restart or the long sanity refresh; direct TUI approval remains immediate,
legacy in-flight gates still reconcile, and no new event-loop disk work or unbounded dismissed-notification scan is
introduced.
