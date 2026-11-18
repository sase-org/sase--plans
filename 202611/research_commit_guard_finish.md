---
tier: tale
size: medium
title: Finish the research commit guard and land sase-1tj
goal:
  Warn reliably about uncommitted reports outside finalizer coverage, direct repairs to
  the correct repository, and close sase-1tj.
proposed_by: bbugyi200.athena.sase-1tj.land
proposed_with: codex/gpt-6-astra
bead: sase-1tj
create_time: 2026-11-18 13:08:55
status: wip
---

- **PARENT:**
  [202611/research_commit_gap.md](https://github.com/sase-org/sase--plans/blob/main/202611/research_commit_gap.md)
- **BEAD:**
  [sase-1tj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tj/README.md)

# Finish the research commit guard and land sase-1tj

This is the remaining work from the landing review of epic **sase-1tj**. One coding
agent implements this bounded repair and completes the landing in the same turn as its
final code. There is no later land agent for this tale.

Read `sase bead read sase-1tj -r "Need remaining landing scope and evidence"`, all three
child beads and their notes, and the approved plan with
`sase artifact read plan:202611/research_commit_gap.md "Need the approved scope"`. The
original decisions remain final: **artifact_create_guard = warn** and **recovery =
commit_and_park**. Do not reopen those choices.

## Verified baseline

The landing review used sase `fbfaf07123`, equal to freshly fetched `origin/master`. It
reviewed the two epic commits in sase, `03a48ab7fc` and `43ef590bb6`, and plugin commit
`73b3350` in `sase-research-artifacts`.

- The five research macros now resolve the checkout with `sase repo open research`
  during the agent run. Target/suffix/no-overwrite behavior and the month substitution
  remain intact. The new plugin raw-source regression passes.
- The 13 tests in `tests/test_unscanned_writes_guard.py` pass, but miss the two failures
  below. The five tests proposed by phase `.1` also pass in the same serial run: 18
  passed in 6.55 seconds.
- Recovery is committed: sase research `df90f1ef` has one report, bob-cli research
  `b83df6f6` has ten, and tailnet research `3db5ee60` has two. All 13 committed Git blob
  IDs match the parked Apollo originals under
  `~/.local/state/sase/recovered-research/<project>/202611/`. Three bob-cli reports
  appeared after the plan's inventory, and were recovered too. Audited read-only opens
  on Apollo confirm all three primary research checkouts now have empty porcelain
  status; the recovered paths have arrived as committed files. Do not recover or
  re-register those files again.
- Apollo still has the old research macro installed. Skill redeployment and plugin
  reinstall are the original plan's post-landing activation steps. Report them in the
  final response; do not make this tale's closeout wait for its own future commit, push,
  deployment, SHA, or CI result. Do not globally reinstall from an unlanded checkout.
- `sase bead epic-symbols sase-1tj` is currently empty. The epic currently has no
  `parent_bead`.

## Remaining defects

### Coverage disagrees with the finalizer

`src/sase/llm_provider/commit_finalizer_state/_scan_roots.py` duplicates the enumeration
in `collect_dirty_state` and `_dirty_repos.py`; the finalizer never calls the new
helper. This misses the original plan's shared-enumeration requirement.

A temporary-git reproduction at the reviewed revision creates main, A and B checkouts,
with untracked files in both A and B. Set the configured sibling target to
`SiblingTarget(name="research", workspace_dir=B)` and the opened linked map to
`{"research": A}`. With unrelated SDD/external sources empty:

- `dirty_configured_sibling_repos` returns only A because the dirty A consumes the name
  in `seen_names`.
- `commit_scan_roots` returns main, A and B.
- `find_unscanned_writes(main, artifacts, extra_paths=(B/report.md,), include_run_paths=False)`
  returns an empty tuple, falsely saying the dirty report needs no warning.

Test both A dirty and A clean: current selection makes the configured fallback eligible
when the opened candidate is clean. Do not silently change that behavior.

### Repair instructions can name the wrong project's repository

`_match_inventory_record` calls `collect_repo_inventory()` across every project.
`_suggest_fix` then emits the matched record's bare name and `_own_clone_path` compares
only workspace numbers, without the owning project.

While running in sase, asking `_suggest_fix` about bob-cli's primary research checkout
returns `repo_name="research"`, action `open_and_move`, and the command
`sase repo open research`. That command opens **sase's** research repository, so
following the advice puts the report in the wrong repository. Use temporary inventory
fixtures for the regression; do not mutate real primary checkouts.

### Shared policy belongs in Rust

The new classification, coverage, and repair-target decisions are implemented only in
Python. They are shared domain behavior used by artifact creation and finalization, so
the project's Rust boundary applies. Keep Python configuration, environment,
filesystem/git collection and CLI rendering as thin glue. Move the new deterministic
policy to `sase_core`, with a registered `sase_core_rs` binding and a thin Python
adapter. Do not migrate unrelated finalizer subsystems.

## Implementation

1. Open `sase-core` through `/sase_repo` and read its `AGENTS.md`. Read the
   `lint_and_test`, `generated_skills`, and `symvision` memory notes; read
   `docs_writing` before changing docs. Preserve the existing advisory-only context
   contract and the warn-and-register behavior, including the artifact command's exit
   code. No new feature flag, obligation type, or Python fallback is needed.
2. Extract one ordered repository-candidate enumeration used by both the actual
   dirty-state collector and the guard. Preserve main/sibling/external/SDD ordering,
   name-only opens, canonical path identity, existing dedupe and baseline rules, SDD
   priority, and prompt-archive ownership filtering. The enumeration itself makes no git
   status calls. A superset of possible candidates is not proof of coverage: derive the
   guard's coverage from the same effective eligibility and selection rules as the
   finalizer, including duplicate names and paths. Gather status facts once per
   normalized checkout within a scan and pass those facts to the shared policy. Keep
   current finalizer output for existing cases. Remove the parallel
   `_blocking_candidates` / `_blocking_sibling_candidates` algorithms.
3. Implement the bounded pure coverage/classification/repair policy and its typed
   request/response in `sase_core` (the existing finalizer domain is the starting
   point). Register and test its Python binding in `sase_core_py`. Carry normalized
   candidate and checkout identities, selection facts, the current project identity, and
   inventory ownership/clone facts rather than reading Python environment from the
   policy. Keep canonicalization consistent on both sides, including macOS symlinked
   ancestors. Python collects tool-call and explicit-artifact sources, preserves the
   candidate cap and portable-label exclusions, performs bounded git IO, and renders the
   selected action. Candidate errors remain fail-open.
4. Fix repo targeting using project-qualified identity, not a global bare-name match.
   For a repository in the current project's inventory, the local configured name
   remains usable. For another project's repository, use its unambiguous supported
   external reference (for example `gh:bobs-org/bob-cli--research`), or the existing
   safe unknown-repository guidance if no valid target can be established. Never tell
   the caller to open a same-named repository owned by another project. An `open` action
   must mean the command will register the exact checkout; another project's equal
   workspace number is not evidence of ownership. Preserve repo-relative destinations
   and the existing advisory fields; add an explicit target field only if needed without
   changing the finalizer wire or submission digest.
5. Replace the policy in `_unscanned_writes.py` with the thin adapter and wire both
   `sase final context` and `sase artifact create -p` to it. Update docs/skill wording
   only if the repair command contract changes. Do not hand-edit deployed skills.
   Declare both changed repositories through `/sase_final`: the existing host
   `repos.linked[].revision_pin` mechanism commits/pushes core first and writes its SHA
   to `sase-core-revision.txt` before committing sase. See
   `docs/contribute/rust-backend.md`, "The CI source revision pin". Do not wait for or
   manually manufacture this tale's commit SHA before closeout.

## Verification and integration

Add behavioral regressions for duplicate-name dirty/clean fallback cases, name-only
opens, normalized duplicate paths and SDD priority, nested unopened checkouts,
explicit-artifact sources without tool calls, and cross-project `research` names with
colliding workspace numbers. Exercise actual temporary git status results rather than
mocking away the coverage decision. Test real git error/timeout handling: the existing
`test_git_error_drops_candidate_without_raising` only passes a nonexistent file and
never causes a git failure. Include a Rust policy matrix and a real binding round trip.

Keep the existing finalizer output/dedupe/baseline/ownership tests and the public
artifact-create/final-context contract tests passing. Check that requirements,
obligations, submission decisions and digests are unchanged by advisory entries. Run the
appropriate focused tests, then `sase tool run check` in both changed repositories (the
wrapped `just check`). **Do not run `just check-full`.** Follow `/sase_monitor` if a
verification command needs a handoff; retain all concrete closeout instructions for its
recovery successor instead of ending with a promise to resume. A check-full-only failure
is test infrastructure, not this epic's work.

Before closing, review default-branch drift after `fbfaf07123` and integrate any new
guard or finalizer consumers. The lander already reviewed the earlier non-epic changes
`5f4807e61b` (SDD facade split), `4e5c1b9730` (cancellation contract), `1ccd5c0579`
(update wait), and `fbfaf07123` (compact-list cleanup); none required a research-guard
integration edit. Preserve the cancellation companion assertions.

## Final step: complete the epic landing

The lander has already completed `/sase_new_task` triage and recorded every outcome on
`sase-1tj`. Carry those dispositions into the close note: phase `.1` note #1 belongs to
causal active epic `sase-1tc`; its general cwd hazard was recorded there. Phase `.1`
note #2's four cancellation failures are existing tasks `sase-1tl` and `sase-1tm`, now
passing after companion `4e5c1b9730`; passing evidence was recorded on those tasks and
causal epic `sase-1tc.3.1.6`. Its fifth failure corroborates existing flake `sase-1tn`.
New tasks were declined as duplicate or obsolete paired revision failures. No other
proposed follow-ups existed. The two detector bugs in this tale are epic work and must
be fixed before closing.

Re-read `sase-1tj` and every descendant/note and confirm linked-plan readiness. Run
`sase bead epic-symbols sase-1tj`. Resolve every returned exemption by wiring,
privatizing, adding a justified non-test pragma, or deleting the symbol; re-key only
when a still-open later bead really needs it. Then close normally:

```bash
sase bead close sase-1tj --note "Verified all phases and decisions, runtime research paths, 13 committed and parked recoveries, effective guard coverage and project-correct repairs through Rust; integrated post-start drift and passed the required checks. Follow-up dispositions: general cwd hazard on sase-1tc; cancellation reports on sase-1tl/sase-1tm pass after 4e5c1b9730; gate-capacity flake corroborated on sase-1tn; no new duplicate tasks."
just symvision
```

Expand that close note with the actual verification evidence and any new drift. If the
close rejects leftover exemptions, finish the cleanup and retry. If it rejects
unfinished descendants, finish or reopen them deliberately; never use `--force` merely
to succeed or to advance a successful nested landing. A genuine
cancellation/supersession requires
`--force --reason ... --resolution canceled|superseded`, not a false successful close.

Open the plans repo using `/sase_repo` and set **`status: done`** in the frontmatter of
the original epic's linked file **`202611/research_commit_gap.md`**. Re-read
`sase bead read sase-1tj -r "Need the parent link"`. No parent existed at landing
review, so no ancestor currently needs closing. If a parent link has been added, read
it: close only a satisfied phase parent normally and leave its containing epic to its
lander; for a plan parent recheck its prior landing note, all descendants and notes,
linked-plan readiness and post-child drift, retire its exemptions, close normally, run
`just symvision`, mark its plan done, and repeat only through fully complete directly
parented plan ancestors. At the first incomplete or ambiguous parent, record the blocker
on it and stop the ancestor walk.

Finish through `/sase_final` with commit decisions for every changed repository.
Closeout occurs before this turn's host commits; do not defer it to obtain this turn's
commit SHA, a push, or CI result. Report the remaining installed-plugin and
generated-skill activation requirement, without claiming the installed end-to-end smoke
has already run.
