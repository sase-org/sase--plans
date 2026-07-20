---
tier: tale
title: Migrate epic bead work to just-in-time claims
goal: Epic work launches associate each phase and land agent with its bead, leave
  waiting work unclaimed, and preserve recoverable state across launch failures while
  runner-side claims remain the sole transition to in_progress.
create_time: 2026-07-20 17:09:28
status: done
prompt: 202607/prompts/jit_epic_bead_work.md
---

# Plan: Migrate epic bead work to just-in-time claims

Complete bead `sase-8f.3` by moving the final epic-work launch path onto the atomic, runner-side claim lifecycle
delivered by the preceding phases. The generic `%id(bead=...)` parser, metadata propagation, runner ordering, and
`BeadProject.claim_for_agent_launch` operation already exist; this work makes `sase bead work` produce those
associations and stops it from claiming all phase beads before the agents are ready to execute.

## Render phase and land bead associations

Update the epic multi-prompt renderer so every worker identity carries exactly one authoritative bead association. The
first phase's full-name `%id` must include `bead=<phase_id>` while retaining the separate clan declaration; subsequent
phase joins must combine their suffix, `clan=<epic_id>`, and `bead=<phase_id>` in the same `%id`; and the land join must
use `bead=<epic_id>`. Preserve force-reuse syntax and rewriting, deterministic names, clan/tribe metadata, model
selection, `%auto`, agent and bead waits, VCS/ChangeSpec prefixes, and work/land xprompt selection.

Keep the existing per-segment environment metadata. `SASE_BEAD_ID` remains commit attribution and must match the
rendered directive, while `SASE_EPIC_BEAD_ID`, `SASE_PHASE_BEAD_ID`, the plan reference, the epic tribe, and the
internal-name bypass continue to describe the worker's richer role. Refresh renderer documentation and exact-prompt
assertions so dry-run and live paths exercise the same associations before force-reuse rewriting.

## Transfer mutation ownership to each runner

Remove the `preclaim_epic_work` call, its timing stage, rollback payload, and phase-status assumptions from
`launch_epic_bead_work`. Launch approval should still mark the epic `is_ready_to_work`, because readiness records that
the epic bundle was approved and launched; it must not set phase or epic status. A successful launch should commit only
readiness and other launch-owned bead metadata. The phase and epic transitions to `in_progress` must occur later through
each runner's existing post-wait, post-workspace-preparation `claim_for_agent_launch` call immediately before model
execution.

Retain retry behavior: the work plan still includes all non-closed, non-delegated phases, including phases already in
progress, so an explicit rerun can reassign a surviving phase when its replacement runner reaches execution. Closed
phases remain bead-only dependencies and are never rendered as workers. Do not remove the legacy core preclaim API
solely because this launch path no longer calls it.

## Make launch-failure cleanup honor the new ownership boundary

Refactor the cleanup helper and orchestration around actual spawn results rather than eager-claim restoration. For a
failure before any runner is spawned, undo `is_ready_to_work` only when this launch attempt set it, then persist that
rollback using the existing no-push/deferred-push policy. For a partial multi-prompt launch, terminate the spawned
runners using the identity-aware cleanup path (with the PID fallback retained), keep `is_ready_to_work`, never roll back
phase or epic claims that a runner may already have durably made, and persist the recoverable launch state so a later
`sase bead work` rerun can select the remaining non-closed work. An epic that was already ready must remain ready in
either failure mode.

Keep post-launch commit failures distinguishable from spawn failures: once all requested runners were created, report
the commit failure without terminating them or reverting any runner-owned claim. Update diagnostics to describe
readiness recovery rather than pre-claim rollback.

## Keep worker prompts truthful

Review the built-in phase-worker and land-agent prompt contracts in the normal configuration source. The model runs only
after its runner-side claim, so the phase prompt may continue to say that the bead is already claimed; adjust wording
only if needed to remove any implication that the parent launcher eagerly claimed it. The land prompt must continue to
close the epic only after verification and integration, now observing the epic as `in_progress` because the land runner
claimed it after all phase waits.

## Regression and integration coverage

Update pure rendering snapshots and VCS/ChangeSpec/dry-run assertions to require the correct `bead=` keyword on every
phase and land `%id`, including first-clan declaration, later clan joins, retries of an existing clan, nested bead IDs,
force-reuse rewrites, and custom model/size variants. Confirm each rendered association matches its segment's
`SASE_BEAD_ID` and that no segment gains duplicate or lost identity metadata.

Replace eager-preclaim lifecycle expectations with ownership-boundary assertions: a successful mocked launch marks
readiness but leaves phases and the epic status unchanged until a simulated runner claim; dry runs mutate nothing; retry
launches preserve closed phases and prior in-progress assignees until their replacement runners execute; zero-spawn
failure restores newly-set readiness; and partial-spawn failure terminates launched agents while retaining readiness and
any simulated claims. Keep JSON, plan-file launch/resume, collision, commit/push, and cleanup tests aligned with this
behavior.

Add an integration-style dependency-chain/parallel-wave exercise that connects rendered epic segments to the existing
runner claim point. Assert downstream phases stay open through agent/bead waits, parallel workers claim independently
immediately before their model callbacks, completed phases unblock dependents, and the epic stays open until the land
runner crosses all phase waits and claims it. Cover closed phase and closed epic claim failures, ensuring their model
callbacks are never invoked and the runner reports the bead-specific failure. Exercise both the generic CWD launcher and
planned bead-work fast path where practical, since both consume the same rendered lifecycle through different launch
adapters.

## Validation

Run `just install` before repository checks. Start with focused tests for epic plan/rendering, launch lifecycle and
cleanup, plan-file work/resume, generic and planned launcher adapters, directive bead preservation, and runner claim
ordering/failures. Then run the mandatory `just check`. If targeted failures expose stale eager-preclaim assumptions
elsewhere, update those tests and callers without broadening the runtime or core APIs beyond the ownership change
described above.
