---
tier: epic
title: Finish gate-decision integrity landing gaps
goal:
  Close the residual contract, atomicity, notification-lifecycle, and requester
  acceptance gaps found while landing sase-zr.7.1.1.5, without repeating work that its
  three completed phases already delivered.
phases:
  - id: atomic-lifecycle-completion
    title: Complete released-core adoption and atomic failure transitions
    depends_on: []
    size: medium
    description:
      "atomic-lifecycle-completion: ratchet the published sase-core floor to the
      released gate-decision contract, delete the older-binding capability fallback,
      make receipt and attempt supersession atomic and correctly acceptance-scoped,
      preserve current post-response failures across plain replay, retain enough
      selection identity for every recovery command, and add the missing deterministic
      transition races."
  - id: requester-recovery-acceptance
    title: Complete requester and plan-gate recovery acceptance
    depends_on:
      - atomic-lifecycle-completion
    size: medium
    description:
      "requester-recovery-acceptance: prove actionable failure recovery through launch
      approval, workflow HITL, plan approval, CLI and ACE; restore the specifically
      required plan archive and terminal-preparation recovery cases; and verify the
      integrated tree with focused, governed, and full checks."
proposed_by: bbugyi200.apollo.sase-zr.7.1.1.5.land
parent_bead: sase-zr.7.1.1.5
create_time: 2026-09-17 23:21:14
status: wip
---

- **PROMPT:**
  [prompts/202609/finish_gate_decision_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_gate_decision_landing.md)
- **PARENT:**
  [202609/gate_decision_integrity_completion.md](https://github.com/sase-org/sase--plans/blob/main/202609/gate_decision_integrity_completion.md)

# Finish Gate-Decision Integrity Landing Gaps

## Why this child epic exists

The land audit for `sase-zr.7.1.1.5` reviewed its linked plan, all three phase beads and
notes, the Rust and Python implementation commits, current source, and interleaved
mainline drift. The phases delivered the central Rust policy, serialized gate
transitions, deterministic failure notifications, ACE routing, CLI wait output, and
requester adapters, but the current tree still does not satisfy several explicit
completion requirements. This plan contains only those residual requirements.

Do not include the parent epic's close, Symvision cleanup pass, or plan-file status
update as a phase. The child epic's `parent_bead` relationship returns control to the
waiting land workflow after this work lands.

## Verified starting point — preserve, do not redo

- Linked `sase-core` commit `41a98303` implements the richer gate-decision policy and is
  published by release commit `bd5946c1` as `sase-core-rs` 0.34.50. The main repository
  already pins `bd5946c1` in `sase-core-revision.txt`.
- Main commits `df0090f0` and `e91fa138` provide terminal-transition serialization and
  the failure-recovery surfaces. Keep their working behavior and extend it narrowly.
- Concurrent drift commit `76df5477` and later integration commit `6e06a3e2` establish
  stable nonempty wire attempt IDs for failures that have no execution attempt. The
  three regressions recorded on the parent epic now pass together; preserve that
  projection instead of restoring empty IDs or inventing a second identity scheme.
- `response.json` remains the terminal answer. A post-response side-effect or follow-up
  failure remains a recovery obligation, and ordinary replay must not execute the failed
  stage again.
- The one phase follow-up was independently triaged: the two pager rendered-link tests
  were routed as `DISCOVERED ISSUE` evidence to active flake epic `sase-j7` because
  exact-task searches found no duplicate and the failure is unrelated to gate-decision
  code. It is not part of this child epic.

## Residual invariants

1. The declared runtime dependency must guarantee the Rust wire/policy API Python uses;
   Python must not probe for and silently omit fields from an older supported binding.
2. `acceptance_id` is the receipt identity for every failure and transition. Replacing a
   failed or owner-lost receipt atomically terminalizes its old attempt under the old
   acceptance, and a stale recovery claimant cannot act on the replacement receipt.
3. A current failure notification stays current through passive polling and plain
   idempotent replay. It is dismissed only when the matching recovery is durably claimed
   or its failed stage succeeds, or when that receipt is deliberately superseded or
   cancelled.
4. Every current failure exposes the exact actions permitted for its stage. Failures
   before `attempt_started` and owner-loss failures still retain the accepted selection
   needed to render resume/restart commands; they do not degrade to cancel-only output.
5. Shared lifecycle permission and owner-loss judgments come from `sase-core`. Python
   collects facts, persists transitions under the bounded lock, and renders outcomes; it
   does not reconstruct Rust policy from host booleans.

## Phase 1 — `atomic-lifecycle-completion`

Finish the backend adoption and the missing durability behavior before expanding the
surface tests.

- Ratchet `pyproject.toml` and `uv.lock` to the complete published release containing
  the policy contract (0.34.50 at the audited starting point), using the repository's
  core-window tooling and its floor-repair procedure if the stale 0.34.48 floor is no
  longer a complete non-yanked release. Keep the revision pin at the newest compatible
  descendant; never rewind the linked core checkout.
- Remove `gate_lifecycle_supports_post_response_failure` and its cached runtime probe.
  Always send `post_response_failure` through the required Rust binding. Add or update
  binding/floor validation coverage so a normal install cannot select a binding that
  rejects the field.
- Remove Python owner-loss policy reconstruction such as `_facts_show_dead_owner`.
  Consume the Rust outcome's `owner_lost`, `owner_loss`, liveness, and permissions as
  authoritative structured results.
- When a failed or dead-owner receipt is superseded, append `decision_superseded`, the
  relevant `owner_lost`, and `attempt_superseded` exactly once while still holding the
  bounded acceptance lock. Scope the terminal attempt event to the old attempt and old
  acceptance, and name the replacement acceptance explicitly. A crash after receipt
  replacement must not leave the old attempt apparently resumable.
- Recheck current receipt/response state at every transition commit point. Reject a
  stale resume, restart, reclaim, cancel, or execution claim without mutating the newer
  receipt. Lock timeout and unknown-owner liveness remain explicit non-mutating
  conflicts.
- Fix the response-present fast path so plain replay does not dismiss a current
  `side_effects` or `follow_up` failure notification. Dismiss only after the matching
  recovery is claimed/completes, or after deliberate receipt supersession/cancellation.
- Persist or derive selected option IDs from the accepted receipt for pre-attempt and
  owner-loss failures, so their deterministic `GateExecutionFailed` payloads contain
  every permitted exact resume/restart/cancel command without exposing raw inputs.
- Add deterministic tests for supersede versus completion, cancel versus a live owner,
  stale-receipt recovery, bounded acceptance-lock timeout, simultaneous recovery
  claimants/pollers, and `SIGKILL` owner loss. Assert exact receipt identity and journal
  order, including one old-acceptance `attempt_superseded` event. Extend the existing
  post-response test to assert that plain replay preserves the notification and that a
  successful matching recovery dismisses it.
- Re-run the three parent-note regressions together and the gate decision, executor,
  lifecycle, notification, and CLI-show focused modules.

## Phase 2 — `requester-recovery-acceptance`

Complete the acceptance coverage and repair any surface behavior those tests expose.

- Add exact-contract tests for launch approval and workflow HITL failed outcomes. Each
  requester must retain the redacted failure stage/code/message, error-report path, and
  permitted exact commands; a cancellation refused by a still-live owner stays a
  conflict instead of being converted into polling or a generic rejection.
- Exercise plan approval through the same failed lifecycle, including its gate-shell
  branch/continuation payload, so failed archive or terminal preparation is neither
  reported as approval success nor allowed to launch or archive twice.
- Add the original plan's required generic and plan-specific recovery cases in
  `tests/test_plan_approval_actions_archive.py` and
  `tests/test_plan_archive_approval_recovery.py`. Cover a terminal-preparation/archive
  failure, durable inspectable error and receipt, actionable retry, successful recovery
  without duplicating already-completed work, and isolation from a newer acceptance.
- Extend notification tests for deterministic ID/tags/actions across pre-attempt,
  command, terminal-prepare, side-effect, follow-up, owner-loss, dedupe, receipt
  replacement, and dismissal timing. Keep the ACE missing/malformed-report fallback safe
  and actionable.
- Recheck CLI `show`/`wait` JSON and human output, launch approval, workflow HITL, and
  plan approval against the same failure payload. Update documentation only where the
  now-proven contract differs from the current text.
- Run focused tests while implementing, then `just fix`/`just check`. Finish with
  `just check-full` through the SASE monitor. Record exact commands and any independent
  flakes on the phase bead using the normal proposal policy.

## Completion evidence

The child epic is complete only when a normal installation requires the released Rust
contract; Python contains no compatibility or duplicated owner-loss policy branch; old
attempts terminalize atomically under their own receipt; stale claimants and lock
timeouts are non-mutating; post-response failures survive passive replay and clear only
on a matching lifecycle transition; every failure stage yields the permitted exact
recovery commands; the named plan-archive tests and all requester contract tests pass;
and the governed and full repository checks pass apart from separately triaged,
unrelated flakes.
