---
tier: tale
title: Quiet completed remote-machine onboarding
goal:
  Keep bare sase init offline and silent about discovered remote machines after the
  controller completes its first explicit machine-init review.
size: medium
proposed_by: bbugyi200.athena.0mo
create_time: 2026-09-18 05:34:14
status: wip
---

# Plan: Quiet completed remote-machine onboarding

## Context and behavioral contract

Today, a successful explicit `sase machine init` writes `machine_init_review.json`, but
later interactive `sase init` runs still invoke every configured discovery provider to
look for candidates absent from the record. Provider health diagnostics are then
rendered as init warnings even when the machine plan says the review is current. In a
multi-project `sase init --all` run, the same machine-local diagnostics can consequently
appear beneath every project.

Change the acknowledgment from a per-candidate reminder mechanism into a one-time
onboarding boundary:

- Before a valid completed review exists, interactive bare `sase init` continues to
  offer `sase machine init` once, without performing discovery during planning.
- Any successful explicit `sase machine init` review continues to mark the initial
  review complete, including an empty discovery or a review in which every candidate was
  skipped.
- Once that completion is recorded, bare `sase init`, `sase init --all`, and
  `sase init --project` do not invoke remote-machine discovery, do not prompt for newly
  discovered candidates, and do not surface provider health failures. They report the
  machine initializer as current using local state only.
- `sase machine init` remains the user-invoked rescan and enrollment workflow.
  `sase machine discover` and `sase machine status` also retain their explicit network
  behavior and diagnostics.
- Missing or valid-but-incomplete review state still requests the initial review.
  Unreadable or invalid state still emits its local-state warning and requests a
  catch-up review; unrelated dispatch configuration warnings remain visible.
- Keep the existing review-state and Rust wire schemas readable and writable. No state
  migration or schema-version bump is needed; existing `reviewed` entries may remain as
  compatible historical data even though automatic onboarding no longer compares
  candidates after completion.

## Implementation

1. Update the shared machine-setup policy in the linked `sase-core` repository. Open
   that repository through `/sase_repo`, then change
   `crates/sase_core/src/machine_setup/review.rs` so `assess_machine_init_review` treats
   `initial_review_completed` as the terminal onboarding condition: absent or incomplete
   state offers the initial review, while completed state never offers enrollment
   regardless of candidate or enrolled-machine inputs. Remove reconciliation-only
   implementation dependencies that become unused, but keep the request/result wire
   shape and merge behavior stable. Rewrite the Rust policy tests and the PyO3
   round-trip assertion to prove that a new candidate after a completed review does not
   reopen onboarding and that `unreviewed_candidates` is empty.

2. Make the Python bare-init planner honor that policy before any network work. In
   `MachineInitService.assess_onboarding`, read and validate the local review state,
   assess initial completion without discovering candidates, and return either the
   original initial-review offer or a local-only "review is current" plan. Delete the
   completed-review `discover_detailed` path, its provider-diagnostic propagation, its
   discovery-error fallback, and the assessment cache/key that existed only to avoid
   repeating those probes. Preserve the explicit `MachineInitService.apply` discovery
   and completed-review recording paths unchanged.

3. Remove the obsolete cache plumbing from the init coordinator while retaining the
   batch-wide `machine_offer_handled` guard used by the initial prompt. This includes
   the cache field on `InitOnboardingBatchContext`, the cache argument passed by
   `plan_init_machine`, and cache invalidation after config initialization. Continue to
   re-plan the machine initializer after config changes so a controller with no
   completed review can still receive its first offer when discovery providers become
   enabled.

4. Replace candidate-reminder tests with regressions for the one-time boundary. Cover
   missing, incomplete, completed, and unreadable review states; make the completed case
   use a discovery function that fails the test if called; include provider health
   diagnostics in that forbidden result so the assertion proves they cannot leak into
   init warnings. Add or adjust a multi-project onboarding test to show a completed
   machine review produces neither machine prompts nor repeated health warnings across
   project headings. Retain explicit-apply tests that prove direct `sase machine init`
   still discovers, reports diagnostics, and records successful reviews.

5. Update `docs/init.md` and `docs/remote_dispatch.md` to describe the one-time local
   acknowledgment, the absence of automatic post-setup discovery, and the explicit
   commands users can run later to discover, enroll, repair, or check remote machines.
   Remove claims that new unreviewed candidates or discovery failures can trigger a
   later bare-init offer or warning.

## Acceptance criteria

- After one successful `sase machine init`, an interactive `sase init -a` can plan any
  number of projects without calling a discovery provider and without printing offline
  tailnet-machine health warnings.
- A newly appearing tailnet machine does not create a future bare-init prompt; the user
  must invoke `sase machine init` to review it.
- A fresh controller, an incomplete review record, or an invalid/unreadable record still
  gets one actionable initial/catch-up review opportunity with the existing local-state
  warning where applicable.
- Explicit machine discovery, initialization, repair, and status behavior is unchanged.
- Existing review files remain valid and no credential, enrollment pin, or project
  configuration is rewritten by the planner.

## Verification

- In `sase-core`, run `just check` so the core policy and PyO3 binding tests both pass.
- In the main SASE repository, run focused tests for
  `tests/dispatch/test_machine_init.py`, the review-store/apply coverage, and
  `tests/main/test_init_onboarding_all.py`.
- Run `just fix` and then the required main-repository `just check`; use the project's
  monitor workflow if a verification command becomes long-running.
