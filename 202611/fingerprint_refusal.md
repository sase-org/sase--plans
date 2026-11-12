---
tier: tale
title: Preserve prepared-completion ownership and explain fingerprint refusals
goal:
  Preserve monitor predecessor baselines so unchanged verified work completes, while
  real fingerprint mismatches remain refused with component-level evidence.
size: medium
proposed_by: bbugyi200.athena.sase-1sd.5
proposed_with: codex/gpt-6-astra
bead: sase-1sd.5
create_time: 2026-11-12 12:26:28
status: wip
---

- **PARENT:**
  [202611/next_seven_salvage_and_unstrand.md](https://github.com/sase-org/sase--plans/blob/main/202611/next_seven_salvage_and_unstrand.md)
- **BEAD:**
  [sase-1sd.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1sd/sase-1sd.5.md)

# Preserve prepared-completion ownership and explain fingerprint refusals

Implement phase `sase-1sd.5` of `plan:202611/next_seven_salvage_and_unstrand.md`. The
seed task is `sase-1qm`. Keep the parent epic's accepted decisions unchanged. This is
one bounded coding task: repair the monitor's lost predecessor context, add Rust-owned
comparison diagnostics, and verify the Python host carries those diagnostics into
recovery. No memory edits or new CLI options are planned.

## Findings and existing work

The phase is in progress, assigned to this session, and had no prior implementation
notes when read. The inspected sase base was `1087ccd7f1`; the linked sase-core base and
sase CI pin were `fae59b46a9db7e7c37656bea7d1e7fa98ebaabc6`.

The earlier Rust fix `9e1ac1da` (2026-09-25) already scopes the worktree fingerprint to
repositories with declaration decisions and sorts repositories by ID. Its tests cover
ignored sidecars and reordered repositories. Do not redo that fix or remove metadata and
protection fields from the fingerprint.

A newer, concrete reproduction is available from this same epic:

- Monitor `3sefkg2pq2cg`, member `sase-1sd.2--mon`, started 2026-11-12 and finished
  `just check` with exit 0 after approximately 698 seconds. It recovered solely for
  `stale_worktree_fingerprint`.
- Its prepared intent is `file:explicit:3f2c9d9307bdb16af7458018`. There is only one
  repository decision, for main (`repo-c4d712b070d4`), so a cross-repository decision
  scope mismatch is not required for this recurrence.
- The starter run `20261112052955` has an empty main baseline. The monitor run
  `20261112063549` instead has a fresh main baseline containing all three edited files:
  `src/sase/axe/mentor_runner.py`, `src/sase/axe/run_agent_runner.py`, and
  `tests/axe/test_runner_import_sigterm.py`. All three XY statuses and content hashes
  match the prepared intent exactly. Both baseline HEADs are
  `e07272959937e0a39e1b7c31254b252e657978be`.
- The sealed paths have `protected=false` and `foreign=false`. Replaying the actual
  sealed intent through `bind_conditional_completion` and
  `evaluate_conditional_completion` with passing stages succeeds. Changing only those
  flags to the values derived from the monitor's fresh baseline makes the same evaluator
  return `stale_worktree_fingerprint`.
- The code explains the lost provenance: `monitor/member.py` calls
  `turns/member.py:create_agent_session_turn_member`, which calls
  `axe/run_agent_helpers.py:create_followup_artifacts` without `prev_artifacts_dir`. The
  implementation in `axe/run_agent_helpers_artifacts.py` passes that missing argument
  into `_seed_followup_commit_finalizer_baseline`. The inheritance helper cannot copy
  the starter baseline and falls back to `capture_dirty_baseline`.
  `monitor_starter_artifacts_dir` is layered into metadata only after this seeding. Host
  settlement then observes repositories using the monitor artifacts directory.

The replay confirms the protection-metadata mechanism; it does not recover a lost
host-time observation. `host_completion_receipt.json` for this run contains only the
generic reason and no comparison input. Keep that distinction in the phase note.

Other evidence to check without re-running expensive historical checks:

- Original 2026-11-01 monitor `enf0k7gv43fc`, intent
  `file:explicit:37585a8fa70d95de4d42d2e8`, covers main plus sase-core.
- 2026-11-07 member `19b--mon-0` and 2026-11-10 monitor `b2ynhwznjahm` (`1au--mon-0`,
  intent `file:explicit:21d526ae39cc3c734f4171c4`).
- Additional green refusals: `p0zxjxcyb4b6` on November 11 and `q277j9348xd0` on
  November 12. These are corroboration candidates, not proof of one shared cause.
- Research context:
  `research:202611/next_seven_task_beads_salvage_and_unstrand/next_seven_task_beads_salvage_and_unstrand__cld.md`.
  Its bob-cli examples combine fingerprint refusals with missing stage evidence; do not
  classify those as the same defect without checking their evidence.

Use `sase artifact read` for those artifacts and `sase monitor show` to resolve runtime
records. Do not inspect or mutate another agent's source checkout. Compare retained
inputs and explicitly label historical snapshots that were not preserved.

## Implementation

### 1. Preserve the authoritative predecessor at monitor creation

Add an integration regression before changing production code. Use temporary Git
repositories and real baseline capture/inheritance, with expensive process launch and
global services stubbed at their boundaries. Capture an empty starter baseline, edit
files, and create the monitor through the normal member-creation path. Prove that the
current path captures those edits as protected dirt and prevents the unchanged prepared
intent from completing.

Thread the already-resolved starter artifact directory through `create_monitor_member`
and `create_agent_session_turn_member` into the existing
`create_followup_artifacts(prev_artifacts_dir=...)` argument before baseline seeding.
Inspect every caller of the shared member helper; preserve its existing call
compatibility and avoid making unrelated gate semantics part of this phase. Where a
supported monitor path omits a starter directory, resolve its predecessor through the
existing authoritative artifact resolver/caller context. Do not build legacy timestamp
paths or guess a different predecessor.

Reuse `inherit_commit_finalizer_responsibility`, including canonical baseline and
opened-repository markers, workspace ownership validation, child-record precedence, and
audit behavior. This is Python lifecycle glue to existing behavior. Any new shared
ownership-selection policy belongs in sase-core rather than a parallel Python domain
implementation.

Verify the fix through all consumers: observations, current repository obligations,
prepared declaration installation, and commit ownership. Merely making the initial
comparison pass by reading a different baseline is insufficient if finalization then
still excludes the inherited work. Preserve genuinely pre-existing protected dirt. Do
not clear protection flags, exclude them from hashing, overwrite existing child
baselines opportunistically, or bypass workspace identity checks.

New monitors must inherit correctly; already-refused historical monitors are evidence
only. Missing or unavailable predecessor provenance must produce actionable recovery,
not permission to commit unknown dirt. Avoid a broad historical-state migration.

### 2. Generate precise mismatch evidence in Rust

In the linked sase-core checkout, keep the existing fingerprint equality check and
generic `stale_worktree_fingerprint` reason. Add a structured, additive
`fingerprint_differences` diagnostic to the completion decision wire, defaulting to
empty for older serialized decisions. Follow the current wire compatibility rules; do
not change the sealed fingerprint algorithm or invalidate existing valid seals.

Compute differences from the exact sealed and current observations already used by
`continuation/completion_eval.rs`, in a small dedicated continuation module. Use the
same decided-repository scope as `worktree_fingerprint`. Diagnostics must identify:

- Missing or extra decided repository observations, including duplicate observations if
  they can reach evaluation; repository ID plus available name/kind.
- HEAD, HEAD tree, index tree, completeness, and repository metadata differences.
- Added/removed paths, changed content hash, Git XY status, mode, path kind, protection
  flag, or foreign flag.
- Path ordering if it alone changes the current serialized fingerprint; never report an
  unexplained digest mismatch merely because a set-based comparison hid ordering.

Include expected and observed scalar values or hashes, never file contents. Sort
diagnostics deterministically. Bound serialized detail with explicit total/omitted
counts, retaining at least one explanatory component for every mismatch. Keep an
explicit digest-level fallback for an unexpected encoding discrepancy instead of
silently claiming the observations match. None of this diagnostic calculation may turn a
refusal into acceptance.

Update the existing PyO3 round-trip tests for
`continuation_evaluate_conditional_completion` and the relevant Python wire mirrors
where present. Shared comparison logic must remain in Rust.

### 3. Preserve and render the evidence through host recovery

Carry Rust diagnostics through `monitor/host_completion_run.py` and
`monitor/host_completion_complete.py` into the durable host-completion receipt and the
metadata or artifact reference used to build recovery prompts. Audit both initial
eligibility and later re-evaluation branches so a refusal cannot silently drop detail.
Capture the result from that evaluation; do not re-observe after refusal and mistake the
later tree for the rejected input.

Update `monitor/followup_prompt.py` to explain the actual differing repository,
component, and path, with bounded output and a pointer to retained full detail when
truncated. Replace the unconditional claim that the worktree changed with wording that
accurately covers ownership metadata or repository discovery changes. Older receipts
without diagnostics must remain readable and explicitly say component evidence was not
recorded; do not invent an explanation or retry the same check automatically. A
genuinely changed tree still requires fresh verification.

## Verification and acceptance

The regression suite must cover these behaviors, without mocking away observation or
inheritance in the decisive lifecycle test:

1. A monitor created after its starter edits an initially clean main repository keeps
   the starter's empty baseline; unchanged Git and ownership observations remain
   eligible and the inherited main obligation reaches host finalization.
2. Main plus a newly opened, edited linked repository keeps baseline and opened-repo
   identity through monitor creation, including the linked repository obligation.
3. Existing protected dirt stays protected; an existing child record retains its
   authority; degraded/different-workspace or unavailable provenance cannot become an
   eligible completion through the repair.
4. Changed bytes, HEAD, index tree, mode, XY, path set, protection/foreign flags,
   missing decided repository, and relevant ordering each still refuse with precise
   diagnostics. Identical observations, repository reordering, and changed undecided
   sidecars preserve their existing behavior.
5. PyO3 preserves diagnostics; Python settlement writes them before launching recovery;
   recovery prompts show bounded useful details; old decisions and receipts without the
   new field still work; unrelated stage refusals retain all their reasons.

Relevant suites include `tests/monitor/test_monitor_start_stage_prepared.py`, the
`test_monitor_host_completion_*` tests and their helper modules,
`tests/test_finalizers_prepared_bridge*`, existing follow-up inheritance tests, and Rust
continuation evaluation/binding tests. Extend the closest suites instead of copying
large fixtures or bringing back deprecated check-full contracts.

Read `lint_and_test` before finishing. Run focused tests, then `sase tool run check` in
each changed repository. Never run bare cargo in sase-core; use its `just test -p ...`
recipes for focused coverage. Follow the repo's supported local binding build workflow
when testing Python against modified Rust. Let host completion commit core and move
`sase-core-revision.txt` past that actual commit; never invent the SHA or perform a
global install. Use `/sase_monitor` for long verification, waiting for its staging
command to exit before `/sase_final`.

Re-check `sase agent list -j` before implementation. Planning found no live sase-1ra
member; the sase-1n6.11 steward was waiting on GitHub settings, not editing this
surface. If that changes, coordinate on the phase bead and narrow/wait rather than race
shared core work. Keep the outer epic's phases sequential.

Record on `sase-1sd.5`: base and host, before/after reproduction, which historical runs
corroborate the inheritance defect, unknowns due to absent snapshots, and test results.
Claim `TASK RESOLVED: sase-1qm` only if the reproduction and fix establish the
mechanism; if only diagnostics can land, explicitly leave the seed for the outer land
agent to judge, as the parent plan allows. Unrelated discoveries are
`PROPOSED FOLLOW-UP:` notes on this phase, never new beads. Identical clean-base check
failures are documented follow-ups and do not strand a completed phase.

Before closeout run `sase bead epic-symbols sase-1sd.5`; resolve every entry or re-key
it to a still-open owning bead. Close only `sase-1sd.5` with a verification note when
the accepted phase scope is complete. Never close `sase-1sd`, the seed task, or any
ancestor plan. Finish through `/sase_final` with all changed repositories declared.
