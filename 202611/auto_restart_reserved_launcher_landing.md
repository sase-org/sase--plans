---
tier: tale
size: medium
title: Finish auto-restart reservation and ledger recovery, then land sase-1se
goal:
  Unnamed update-skew failures recover owned planned names without split ledger
  identities, and sase-1se closes with verified integration and a completed plan.
proposed_by: bbugyi200.athena.sase-1se.land
proposed_with: codex/gpt-6-astra
bead: sase-1se
create_time: 2026-11-12 09:36:56
status: wip
---

- **PARENT:**
  [202611/auto_restart_update_skew_fixes.md](https://github.com/sase-org/sase--plans/blob/main/202611/auto_restart_update_skew_fixes.md)
- **BEAD:**
  [sase-1se](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1se/README.md)

# Finish auto-restart reservation and ledger recovery, then land sase-1se

Implement this bounded remaining work directly, then finish the landing in this same
turn. This tale has no separate land agent. Do not make closeout depend on this work's
eventual commit SHA, push, or CI result.

## Context and fixed decisions

The parent epic is `sase-1se`; its approved plan is
`plan:202611/auto_restart_update_skew_fixes.md`. Read that artifact and the epic's
latest landing notes through their audited commands before editing. Read the landing
evidence with
`sase artifact read file:explicit:9023ae26f2429704cab3f420 "Need the reproduced reservation blocker and completed follow-up triage"`.
All four original phases are closed. The parent epic has no parent_bead, so there is no
containing phase or ancestor to close.

Keep the accepted decisions: launcher_relaunch_mode=resubmit,
marker_scope=exception_first, stale_ledger_keys=leave. No new feature flag,
historical-ledger migration, or forced-name wipe is authorized by this repair.

The landed implementation consists of sase f613149ecc (canonical ledger keys),
7665654aaa (launcher resubmission), cb0a3c9a80 (binding tests, docs and core pin), and
sase-core 47f1af1cb8fd196220098298dc1970044d4251ee (classifier fixes). The core pin
already covers the classifier commit. A focused audit run passed 37 tests, including the
real-binding classifier and healer tests.

Post-start integration reviewed 063c98293b (entry-owned SIGTERM handlers), cc236079f4
(config facade split and preprocessing-test environment fix), and 8bb7bdc231 (nameless
boot-failure bead-work ownership and registry cleanup). The last change makes it
especially clear that an unnamed artifact can still own a name reservation. Refresh the
drift audit before closeout.

## Remaining defect

In `src/sase/agent/auto_restart/healer_relaunch.py`, `_is_launcher_row` is true when the
failed row has no name/workflow_name in either metadata file and its target name is the
timestamp. `_launch_launcher_prompt` then sends raw_prompt.md unchanged to
`launch_agents_from_cwd`.

But launch-time planning can already have reserved the explicit name before the runner
writes metadata. The normal `launch_cwd_single.launch_single_agent` calls
`validate_launch_name_requests`, which rejects that still-owned name.
`names._registry_scan_collectors.collect_planned_reservation_entries` retains planned
reservations even through rebuild. The healer turns the exception into terminal
execute_failed after begin_launch, consuming the only restart attempt.

The lander's isolated reproduction seeds an unnamed failed row, calls
`names.reserve_registered_name("bootstrap-agent", row)`, confirms launcher detection is
true, then calls the real guard on `%id:bootstrap-agent\nResume the requested work.`. It
raises `AgentNameLaunchCollisionError`. Calling the existing owner-checked
`release_planned_registered_name("bootstrap-agent", row)` makes the guard pass without
deleting any artifacts. Treat this as a proven failure and a reusable primitive, not as
proof that an unlocked release/relaunch sequence is sufficient. The current success test
mocks `launch_agents_from_cwd`, so it cannot catch this.

A second, small contract gap is in `healer_targets._project_for_artifacts_dir` and
`history._project_for_done`: their broad `except Exception` blocks silently return the
positional project fallback when the canonical parser fails. The approved plan permits
this fallback only when parsing returns None. In a deterministic probe, patch
`parse_agent_artifact_path` to raise `RuntimeError("parser unavailable")` and pass
`/isolated/projects/gh_demo__repo/artifacts/ace-run/202611/12/20261112052956`: both
helpers return `202611`. That recreates the split ledger identity this epic is supposed
to eliminate whenever a parser/binding error occurs.

## Implementation

1. Reproduce the failed launch using the actual registry and name-validation path in
   isolated test state, stubbing only the process-spawn boundary and other external
   effects. Cover the ordinary named-row route unchanged.

2. Integrate reservation ownership into the unnamed-launcher resubmission. Discover the
   planned reservation actually owned by this failed artifact; use existing locked,
   owner-checked name APIs to release/transfer only that reservation into the
   replacement launch. Preserve the intended concrete identity and applicable
   bead/clan/session metadata. Do not steal another row's reservation, delete registry
   entries by name alone, invoke a forced name wipe, or destroy/dismiss the failed
   artifacts. Recheck ownership and liveness at the mutation boundary, including races
   with manual cleanup or a concurrent launch. If ownership changed, fail closed with an
   actionable outcome rather than launching a duplicate or consuming unrelated state.

   Keep ledger claim-before-mutation ordering and the one-attempt guarantee. Successful
   recovery must persist the replacement directory, provenance, evidence and terminal
   recovery marker and send one notification. A failed or interrupted launch must retain
   honest terminal state and cannot leak an unowned reservation or re-enable a second
   automatic launch. Keep missing, empty and multi-segment prompt refusals before
   reservation mutation.

   Reuse the normal workspace allocation path: a nondeferred project launch uses
   `claim_next_axe_workspace` through `launch_executor_workspace` and can leave the
   failed hold intact. Add a regression proving a replacement can obtain a different
   slot while the failed hold remains, rather than relying solely on a mock of the
   top-level launch function. Handle home/deferred launches using their existing
   allocation contracts.

   Use existing owner-checked reservation primitives and the normal launch guards; keep
   local Python code as orchestration and adapters. Shared reservation/domain policy
   belongs in sase-core and must not be duplicated here. No new binding or core pin
   change is expected for this repair. Read applicable reference memories before editing
   ACE, documentation, or tools.

3. Stop canonical parser errors from becoming guessed project identities. Preserve the
   positional compatibility fallback when the parser returns None, as approved.
   Propagate/report a parser failure so candidate enumeration skips or defers the row
   without claiming a month-keyed ledger entry. Reuse the required Rust parser through
   its thin adapter; do not add another parser or a Python fallback for unavailable Rust
   behavior. Cover both target and history paths, valid legacy/sharded paths, explicit
   None, and an exception.

4. Add meaningful integration regressions: planned reservation owned by the failed
   unnamed row relaunches exactly once under the intended identity; a foreign or live
   owner's name is preserved; an ownership change during recovery cannot be stolen;
   invalid prompts do not mutate reservations; failed spawn does not create false
   success or permit a second attempt; the original artifacts and workspace hold
   survive; replacement lineage, directory mapping and notification are correct.
   Exercise the real launch name guard. Retain the classifier overlap/log-marker,
   legacy/sharded ledger, and ordinary named-restart coverage. Update the auto-restart
   documentation to describe safe reservation handling. Correct the stale module
   docstring in `src/sase/agent/auto_restart/managed_roots.py`, which still says any
   ImportError under the workspace never matches; workspace-only origin is the actual
   classifier rule now.

5. Run focused tests and the guarded `sase tool run check` (`just check`). Do not run
   `just check-full`. Run the linked repo's guarded check only if changed. If a long
   check requires a monitor, stage it through `/sase_monitor` with a successor
   explicitly owning the remaining closeout below; do not end an ordinary turn with an
   unmonitored live command. Do not wait for this turn's own commit to close the epic.

## Final step: complete this epic's landing

The lander already triaged both original child PROPOSED FOLLOW-UP notes and recorded
their final outcomes on sase-1se. The preprocessing proposal from .2 was declined
because cc236079f4 fixes the environment leak and the exact test passes. The unrelated
lifecycle-row flake from .3 is tracked by large flake task `sase-1sg`, with retained
fail/pass evidence and a related link to the history-loader epic `sase-zu`; causality to
that epic is unproven. Do not reopen those investigations as product acceptance work.

Read sase-1se and every descendant/note again, confirm the approved acceptance criteria
and this repair, and review any further commits since cb0a3c9a80 in sase and since
47f1af1c in sase-core for integration. Use `/sase_new_task` for any genuinely distinct
new out-of-scope follow-up; unresolved defects caused by this epic remain epic work.

Run `sase bead epic-symbols sase-1se`. It was empty at the lander's audit but must be
rechecked. Resolve each entry by wiring, privatizing, a justified non-test pragma, or
deletion under Symvision policy; re-key only if a still-open later bead actually needs
that exemption. Never leave an entry keyed to a closing epic or phase.

Then close normally:

```bash
sase bead close sase-1se --note "Verified all four original phases and notes, accepted classifier/key/launcher decisions, containing core pin, and post-start integration; repaired owned planned-name recovery and parser-error ledger identity handling with real launch-guard, ownership-race, workspace-hold and at-most-once regressions; focused tests and just check passed. Original follow-up triage is recorded in the landing notes. No stale epic-symbol entries remain."
```

Tailor the note to actual verified results. If closure rejects leftover symbols, finish
cleanup and retry. If named phases are incomplete, finish or reopen them; never use
--force merely to make close succeed or to advance a successful nested landing.
Cancellation/supersession is only for a deliberate scope outcome with the required
reason and resolution, not for this expected successful landing.

After closing, run `just symvision` to confirm the whitelist is clean. Open the plans
repo with `/sase_repo`, resolve the audited PLAN artifact to its path, and set
`status: done` in the frontmatter of `plan:202611/auto_restart_update_skew_fixes.md`.
Re-read `sase bead read sase-1se -r "Need the parent link"`: the audited parent is
absent, so finish normally when still absent. If the relationship unexpectedly changed,
record the ambiguity and do not force an ancestor closed. End through `/sase_final`,
declaring every changed repo for host completion.
