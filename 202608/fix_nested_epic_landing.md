---
tier: tale
title: Fix nested epic landing handoffs
goal:
  Nested remediation epics return completion ownership to their parent bead so
  successful epic chains close normally without force.
size: medium
proposed_by: bbugyi200.athena.02j
create_time: 2026-08-15 13:23:54
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.02j](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.02j.md)
- **COMMITS:**
  - [87a5698](https://github.com/sase-org/sase/commit/87a569884f80ece1aca82ee011235eeb22ae69ec)
    — fix: resume nested epic landing handoffs

# Plan: Fix nested epic landing handoffs

## Diagnosis

The descendant-close guard is working as designed and must remain unchanged. A normal
`sase bead close <epic> --note ...` succeeds once every descendant is closed. `--force`
is only for deliberately canceling or superseding unfinished descendants, requires a
reason, and cannot record the normal `done` resolution.

The failure is a contradictory nested-landing prompt contract:

- `bob-cli-t.4.land` found legitimate remaining work and proposed the child epic
  `bob-cli-t.4.5`. Proposal ended the original land-agent turn and associated the new
  plan with `bob-cli-t.4` through `parent_bead`.
- `bd/land_epic` currently tells such a lander to put the original epic close into the
  nested plan's final phase.
- `bd/work_phase_bead` correctly tells every phase worker—including `bob-cli-t.4.5.3`—to
  close only its assigned phase and never its parent epic. At that point `bob-cli-t.4.5`
  was also still open, so closing `bob-cli-t.4` normally would have failed; forcing it
  would have incorrectly swept valid work.
- `bob-cli-t.4.5.land` then closed only `bob-cli-t.4.5`, its own target. No prompt or
  orchestration step resumed landing of the directly parented epic. The same gap had
  already left `bob-cli-t` open after its remediation child `bob-cli-t.4` was created.

Canonical bead events now show all phases and `bob-cli-t.4.5` closed with resolution
`done`, while `bob-cli-t.4` and its ancestor `bob-cli-t` remain `in_progress`. Both can
be closed normally after a final state recheck; neither needs `--force`.

## Implementation

1. Correct the built-in landing contract in `src/sase/default_config.yml`.
   - Keep phase ownership strict: a phase worker closes only its assigned phase. Make
     explicit that an ancestor-close instruction in a phase description is preparation
     and evidence for the land agent, not authorization to close the ancestor.
   - Change `bd/land_epic` so a lander that discovers remaining work plans only that
     work; it must not delegate closing the current epic to a child phase.
   - After a child epic lands, have its land agent inspect the linked `parent_bead` and
     continue the interrupted handoff according to the parent type:
     - for a parent phase, verify the child plan completed that phase, close the phase
       normally, and leave the containing epic to its already-waiting land agent;
     - for a parent plan bead, review the parent's landing note, descendants, linked
       plan, and post-child drift, then close the fully completed parent normally and
       finish its post-close cleanup/plan status; repeat through directly parented plan
       ancestors while each remains fully complete.
   - Stop at the first incomplete or ambiguous parent and record/report the blocker.
     Never use `--force` to advance a successful nested landing.

2. Add focused regression assertions in `tests/test_bead_xprompt_tags.py`.
   - Assert the phase prompt retains single-bead ownership and explicitly defers any
     ancestor closure.
   - Assert the land prompt no longer assigns its own close to a remediation phase.
   - Assert the land prompt distinguishes parent phases from parent plans, resumes
     completed plan ancestors, requires descendant/readiness rechecks, and preserves
     normal non-force close semantics.
   - Retain the existing tag-resolution, override, wait-directive, and follow-up
     contracts.

3. Repair the stranded real bead chain after the workflow regression is covered.
   - Open the `bob-cli` project and its bead/plan sidecars through `/sase_repo`; do not
     use hard-coded workspace paths.
   - Re-read `bob-cli-t.4`, `bob-cli-t.4.5`, every descendant, notes, and linked plan
     status. If the observed all-`done` state still holds, close `bob-cli-t.4` normally
     with a note tying the repair to the completed nested landing.
   - Recheck `bob-cli-t` after that close. If all of its descendants are closed and its
     linked plan is already complete as the recorded landing evidence says, close it
     normally as the next stranded plan ancestor. If either condition has changed, leave
     it open and report the exact blocker. Do not force either close.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral SASE
   workspace.
2. Run the focused xprompt contract tests while iterating.
3. Run `just check` and fix any failures caused by this change.
4. Re-read the built-in expanded prompt bodies to confirm the phase/land instructions
   are mutually consistent and contain no wait directives.
5. Re-read the repaired `bob-cli` bead hierarchy and histories to confirm each intended
   close used resolution `done`, no descendant was swept, and no `--force` close event
   was introduced.
