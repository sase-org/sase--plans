---
tier: tale
title: Launch the epic lander after every phase is closed
goal:
  Make `sase bead work <epic-id>` launch or recover the deterministic epic land agent
  when all authored phases are already closed, without reopening phase work or
  duplicating an active lander.
size: medium
proposed_by: bbugyi200.athena.0b0
create_time: 2026-08-22 15:33:36
status: wip
---

# Plan

## Context and invariant

`sase bead work` already treats an epic invocation as a recoverable launch: it filters
closed phases, renders the land agent after the remaining phase waves, selects only
missing or safely replaceable deterministic agent names, and preclaims only the selected
beads. The shared Rust work-plan builder currently stops that flow early whenever its
filtered non-closed phase list is empty. That conflates two different states:

- an invalid epic with no authored phase children; and
- a valid, completed phase graph whose land agent still needs to verify and close the
  epic.

The target contract is that the first state remains invalid, while the second returns a
land-only work plan. For an all-closed epic, the plan must retain the total authored
phase count and every authored phase bead ID, contain zero phase waves and zero
agent-name land waits, and still name the deterministic `<epic-id>.land` agent. The
rendered land segment must keep bead-closure waits for every authored phase so the
existing closure invariant remains explicit even though those waits are immediately
satisfied.

This is recovery behavior, not unconditional duplication: a missing or stale lander is
launched through the existing cleanup/reuse path, while a matching live lander is
preserved as an idempotent no-op. Closed phase beads must never be reopened, reassigned,
or included in worker launch metadata.

## Implementation

1. In the linked `sase-core` repository, update `crates/sase_core/src/bead/work.rs` so
   epic validation rejects only an epic with no authored phase children. Allow an epic
   whose authored phases are all closed to continue through the existing empty-schedule
   path and return an `EpicWorkPlanWire` with the land-only shape described above. Keep
   the wire schema, phase ordering, authored phase count, model routing inputs,
   delegated-phase behavior, cycle checks, and cross-epic blocker checks unchanged.

2. Add direct Rust regression coverage beside the work planner. Prove that an epic with
   no phase children still returns a validation error, and that an epic with one or more
   closed phases returns all authored phase IDs and the original total count, empty
   waves and `land_waits_on`, and the expected land agent/model fields. This pins the
   shared backend contract independently of the Python host.

3. In the `sase` repository, replace the Python facade expectation that an all-closed
   epic raises `EpicPlanError` with contract assertions for the land-only plan. Exercise
   `render_multi_prompt` and `epic_work_segment_env` for this shape: the output contains
   exactly one land segment, declares or joins the epic clan correctly, uses the land
   model selected from the total authored phase count, contains `%w(bead=...)` for every
   closed phase, contains no phase-worker xprompt or phase-agent `%w:` dependency, and
   emits exactly one environment record attributed to the epic bead.

4. Add CLI-level relaunch coverage around `launch_epic_bead_work` using an epic whose
   phases are all closed and whose land agent is not currently active. Assert that the
   command selects and launches only `<epic-id>.land`, passes an empty phase-assignment
   batch plus the land assignment to preclaim, leaves every phase closed and unchanged,
   and reports the zero-wave plus one-lander summary. Add the complementary land-only
   retry case with a matching live lander to prove the command preserves that agent and
   performs no readiness mutation, preclaim, checkpoint, cleanup, or duplicate launch.
   Keep the existing stale-name replacement and mixed open/closed retry behavior intact.

5. Update the canonical user-facing bead-work documentation in `docs/beads.md` to state
   that a retry with no non-closed phases produces a land-only launch, just as an
   all-delegated schedule does, while an epic with no authored phase children remains
   invalid. Do not edit generated memory or provider instruction files, and do not
   manually change the published `sase-core-rs` dependency window; the release workflow
   owns that version ratchet.

## Verification

1. In `sase-core`, run the focused epic work-planner tests while iterating, then run the
   repository-required `just check` so formatting, clippy, all Rust crates, and the PyO3
   binding tests validate the changed backend contract.

2. In `sase`, run `just install` after the core change so the local `sase_core_rs`
   extension is rebuilt from the linked checkout. Run the focused work-plan, rendering,
   relaunch, cleanup, mutation, and model-routing tests that cover land-only and normal
   epic launches.

3. Run `just check` in `sase` and inspect its scoped-test selection. If it escalates,
   identifies an unusual selection, or the change is treated as part of the broadening
   set, hand `just check-full` to `/sase_monitor` with the required `TESTING`/`TESTED`
   statuses and a follow-up that reviews the result.

## Acceptance criteria

- Re-running the reported all-closed epic shape no longer emits
  `has no non-closed phase children`; it launches the missing `sase-s1.land` segment
  without launching any `sase-s1.<phase>` worker.
- Every authored closed phase remains closed and retains its existing assignee and
  metadata; the epic is assigned only to the selected land agent through the normal
  preclaim/checkpoint path.
- The land prompt keeps bead waits for all authored phases, uses the authored total for
  regular-versus-big lander model routing, and has no nonexistent phase-agent waits.
- A matching active lander is preserved rather than duplicated, while stale/missing
  lander ownership continues through the existing safe force-reuse workflow.
- An epic with zero authored phase children still fails validation with a specific,
  actionable error.
- Focused regressions and both repositories' required checks pass.
