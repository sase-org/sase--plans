---
tier: tale
title: Finish integration and land wait-priority epic sase-9k
goal: A published Rust scan binding carries explicit wait priority into ACE, the concurrent
  documentation is integrated, and sase-9k closes only after release-backed verification
  and post-close cleanup.
bead: sase-9k
create_time: 2026-07-25 12:33:24
status: done
---

- **PROMPT:** [202607/prompts/finish_wait_priority_epic.md](prompts/finish_wait_priority_epic.md)
- **PARENT:** [202607/wait_priority.md](https://github.com/sase-org/sase--plans/blob/main/202607/wait_priority.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-9k.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9k.land/README.md)
  - [bbugyi200.athena.sase-9k.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-9k.land.md#member-code)
- **COMMITS:**
  - [4b9281d](https://github.com/sase-org/sase/commit/4b9281d3d7d92f0de8a03c8bdea802d28eea6901) — docs: document bounded runner-slot deference (sase-9k)

# Plan: Finish integration and land wait-priority epic sase-9k

## Context

The landing audit verified the original epic bead `sase-9k`, all four closed child beads, their plans/notes, their
commits, and the current implementations:

- `43ba5daf7` adds bounded, priority-scaled runner-slot deference, continuous eligibility tracking, the early exit when
  no better-priority agent is pending, fail-open configuration accessors/schema/defaults, logging, and focused admission
  tests.
- `64ac40d38` adds `wait_priority_explicit` marker symmetry, the legacy-marker heuristic, and the Python scan wire
  field. The corresponding `sase-core` commit is `e63f1ab`.
- `68723bedb` carries the explicit flag into ACE state/enrichment/dedup, renders explicit priorities in rows and detail
  metadata, and includes both priority fields in the row cache key. The optional deference countdown was correctly
  skipped under the plan's explicit instruction because `eligible_since` is not in the scan wire.
- `3a8540f32` adds priority validation/prefill/clear semantics to the ACE wait modal, round-trips `priority=N`,
  preserves an omitted priority during unrelated wait edits, updates live marker/meta state, and clears priority on
  run-now.

After the required `just install`, 254 focused Python tests covering those runtime, wire, config, loader, rendering,
modal, directive, and persistence paths passed.

The audit of every non-epic main-repository commit since `43ba5daf7` found no code conflict with the feature. One
concurrent documentation landing does need integration: `0e7e36185` refreshed the central configuration reference after
the deference settings were added, but it could not document the still-incomplete `runner_slots` feature. The current
configuration reference has no `runner_slots` section, and the xprompt/runner-slot documentation still describes
priority without the new bounded cross-time deference behavior.

There is also a release-critical cross-repository gap. `sase-core` master contains `e63f1ab` and its scanner parity
test, but tag `v0.9.1` predates that commit. The main repo declares `sase-core-rs>=0.9.1,<0.10.0` and locks the
published 0.9.1 artifact. An exact clean install of the registry's 0.9.1 wheel contains no `wait_priority_explicit`
projection. The existing remote 0.9.2 release branch commit `834fd71` is a sibling of `e63f1ab`, not a descendant, and
the registry currently offers no 0.9.2. Local tests pass only because `just install` builds the linked `sase-core`
master checkout. Do not close the epic until a published binding that contains `e63f1ab` is consumed and tested by the
main repo.

## Phase 1: Release the Rust projection

Open the linked `sase-core` repository through the required `sase_repo` workflow. Reconfirm that its worktree and
`origin/master` are clean/aligned, and inspect the current release state rather than assuming the stale remote release
branch is safe.

Use the repository's established release process to produce and publish the next compatible `sase-core-rs` version from
a history that contains `e63f1ab`. Refresh or replace the stale release proposal as appropriate; do not publish
`834fd71` as-is because its ancestry omits the epic's scanner change. Preserve any newer release work that lands
meanwhile.

Before consuming the release, prove all of the following:

1. The release tag/commit is a descendant of `e63f1ab`.
2. Rust formatting, Clippy, and the scanner parity tests pass, including the `WaitingMarkerWire.wait_priority_explicit`
   fixture.
3. The exact registry artifact for the new version is available, and a clean install—not the editable linked
   checkout—projects `waiting.json["wait_priority_explicit"]` through the Python binding.

If publication requires an external approval or has not propagated to the registry, leave `sase-9k` open and report that
blocker. Do not make the main repo depend on an unpublished version and do not proceed to the landing phase.

## Phase 2: Consume the release and integrate concurrent documentation

In the main SASE repository, raise the lower bound of `sase-core-rs` to the first released version that contains
`e63f1ab`, retain the existing `<0.10.0` compatibility ceiling unless the actual release requires a justified change,
and refresh `uv.lock` from the registry artifact.

Add the missing documentation integration:

- Add `runner_slots` to the configuration-reference table of contents and document `deference_seconds_per_step` and
  `deference_max_seconds`, their defaults, valid ranges, priority-10 boundary, bounded continuous-eligibility semantics,
  and fail-open configuration fallback.
- Update the `%wait(priority=N)` and runner-slot troubleshooting documentation to explain that default/better priorities
  remain immediate, priorities above 10 may defer while a live unstarted better-priority agent can plausibly arrive, the
  deference window resets when eligibility is lost, and it exits early when no such agent remains. Mention the explicit
  priority display/edit surfaces in ACE where useful for diagnosis.
- Keep the documented lack of priority aging accurate: bounded deference does not turn into aging or preemption.

Run `just install` first as required, then the focused epic test set, `just test-visual`, `just rust-check`, and
`just check`. Also run a clean registry-artifact probe that does not inherit the linked editable Rust build, so release
integration cannot pass solely because of the local checkout. Review the complete main and linked-repository diffs and
confirm every later non-epic commit remains integrated without duplicated or conflicting behavior.

## Phase 3: Land epic sase-9k

This is the final phase, and its ordering is mandatory:

1. Re-run `sase bead show sase-9k` and each child bead; confirm all four children remain closed and the release-backed
   verification above is complete.
2. Close the original epic with `sase bead close sase-9k`.
3. Only after the close succeeds, run `just symvision`. Epic-symbol whitelist entries for `sase-9k` expire at close. If
   Symvision reports stale whitelist entries or unused code, use the required `sase_memory_read` procedure for
   `symvision.md` before editing, remove exactly the stale entries/unused code it identifies, and rerun proportionate
   tests plus `just check` for source changes.
4. Open the plans sidecar through `sase_repo` and set `status: done` in the frontmatter of the original linked epic
   plan, `202607/wait_priority.md`. Do not mark this handoff plan as a substitute for finalizing the original epic plan.
5. Verify `sase bead show sase-9k` reports the epic closed, the original plan reports `status: done`, post-close
   Symvision is clean, all required checks pass, and every affected worktree contains only intentional landing changes.
