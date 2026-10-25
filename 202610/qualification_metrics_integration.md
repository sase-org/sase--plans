---
tier: tale
title: Integrate release metrics with the qualification inventory and finish landing
goal:
  Release metrics rejects invalid inventories through the shared qualification
  validator, and the verified qualification epic and its completed parent plan and phase
  are closed normally.
size: small
proposed_by: bbugyi200.athena.sase-1n6.5.1.5.land
proposed_with: codex/gpt-6-astra
bead: sase-1n6.5.1.5
create_time: 2026-10-25 15:04:53
status: wip
---

- **PARENT:**
  [202610/qualification_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202610/qualification_landing_repairs.md)
- **BEAD:**
  [sase-1n6.5.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1n6/sase-1n6.5.1.5.md)

# Finish the qualification landing

This is the only remaining work from land review of `sase-1n6.5.1.5`. One coder can
finish it directly. This tale has no land agent; its coder must perform the concrete
closeout below after implementing and verifying the repair. Close in the same turn as
the final code. Do not wait for your own commit, push, SHA, or CI result.

Read the current epic's **LAND AUDIT AND FOLLOW-UP TRIAGE (2026-10-25)** note through
`sase bead read sase-1n6.5.1.5 -r "Need the completed audit and remaining integration"`.
Read the linked plans with audited artifact reads:

- `plan:202610/qualification_landing_repairs.md` (this epic);
- `plan:202610/published_core_qualification.md` (parent plan `sase-1n6.5.1`);
- `plan:202610/e2_release_train_steward.md` (parent phase `sase-1n6.5`, qualify
  section).

The inherited decisions stay fixed: workflow steward, daily cadence, one eligible
failed-test rerun, memory work reserved for the outer epic. No memory, Rust behavior,
workflow, dependency-pin, publishing, or admission-policy change is needed here.

## Confirmed defect

Reviewed source and fetched master are `ba59799ce95e7a9697dc7877276adec4274898f1`.
Post-start commit `b15cc3fb5b` added `tools/_release_metrics.py:load_inventory`. It
duplicates `tools/_release_qualification_inventory.py:load_inventory`, uses permissive
`json.loads`, and silently skips repeated check IDs. This conflicts with the shared
inventory's fail-closed contract.

Reproduce using the real `.github/release-qualification.json`: append a copy of
`checks[0]` with `job={"id":"other","name":"other required lint","matrix":{}}`. The
shared loader rejects `duplicate inventory check id 'lint'`. The metrics loader accepts
it and omits the second required job. The 287 focused tests passed during the audit,
including metrics, so existing tests do not cover this disagreement.

## Implement and verify

1. Inspect post-audit drift, then change `tools/_release_metrics.py:load_inventory` to
   read bytes and call the authoritative shared qualification inventory loader. Import
   it using the existing release-tool import convention. Translate its
   `QualificationInputError` into the existing `MetricsError` boundary, retaining useful
   path/error context and existing file-read failure behavior. Keep only the projection
   metrics needs: blocking check ID, workflow basename, job name, exercise required
   legs, and returned path. Preserve valid-input output and advisory exclusion. Remove
   duplicate schema/ID/mapping validation; do not weaken the shared validator.
2. Add focused regressions in `tests/test_release_metrics.py` or a small adjacent
   inventory test module. Start with actual valid inventory bytes, prove their current
   projected shape, and exercise the reproduced conflicting duplicate, duplicate JSON
   keys, conflicting workflow/job mapping, and unmapped blocking evidence. Assert that
   the metrics boundary refuses these inputs with useful errors. Preserve existing
   calculations and CLI behavior. This is release tooling, so no Rust binding or CLI
   option is needed. Read `tools.md` and `lint_and_test.md` through memory before edits.
3. Run focused metrics and inventory/evaluator tests, plus the existing transport test
   that revalidates a typed fetch-error red through the steward. Run `just fix`, inspect
   its diff, then `sase tool run check` (the governed `just check`) for changed files.
   Do not run `just check-full`. Use `/sase_monitor` for long verification with an
   explicit continuation that retains this tale's closeout obligations; do not end a
   turn with an unmonitored command still running. Preserve real check failures and
   their triage; fix regressions from this integration.

## Already verified evidence and follow-up disposition

The lander read both direct phases, all nested descendants/notes, parent plan's four
original phases/notes, epic commits and source, and post-start drift. Scoped job IDs,
pagination, immutable canonical observation filenames, missing-binding diagnostics,
source defaults, advisory independence, and the steward evaluator adapter are intact.
All descendants are closed, all four linked qualification plans validate in archived
mode, and plan links validate. This epic, parent plan and original phase had no
`--epic-symbol` entries at audit time. The detailed audit note is the authoritative
follow-up ledger; do not manufacture duplicate reports for the same historical runs.

Every proposal was triaged through `/sase_new_task` before this tale: the direct-phase
toobig item is resolved by `b7f145ef96`; cache ordering by `b2a4c380cf`; ref-prefix
allowlisting by `a9f4b3d1b7`; import budget is retained on `sase-1ic`; modal/visual
ownership is retained on now-closed `sase-1nn`; Ctrl-J on `sase-1a7`; descriptor
ordering on `sase-1nr`; diagnostic-environment failures on resolved `sase-1nq`;
typeahead and bootstrap corroborations on `sase-1mu` and `sase-12f`. Outer `sase-1n6`
owns wheel exercise, SC2012, and source-containment enforcement. Each proposing
bead/note and each decline rationale is in the current epic's audit note. No new
untriaged proposal remains.

Retained proof:

- `file:explicit:ee80ee363315fb88efc6d612`: complete landing verification bundle with
  manifests, actual uploaded observations, history lists, detailed source evidence and
  source logs. Read with `sase artifact read`; binary output supplies the audited path.
  For CLI replay use `green-history.json` and `probe-history.json` (lists), not the
  similarly named `*-observations.json` individual objects.
- `file:explicit:d214a552d318b5ebbe68e847`: acceptance mapping;
  `file:explicit:e38ba06b3bdb3c52a52a2657`: source summary;
  `file:explicit:dfa30156a15eeca38cbfce64`: abandoned-run flake evidence.
- Green rehearsal `40189341606` passes qualification against exact published 0.42.6.
  Probe `40189365714` is typed `QUALIFY_RED` with sealed 0.42.6 versus tested 0.42.2;
  recomputed lint diagnostics name `autonomy_effective_record`,
  `classify_stale_publication_request`, `project_question_round`, and
  `stranded_work_reduce`; it is not rerun-eligible.
- Source Full CI `40189680772` has first-attempt lint and Python 3.12/3.13/3.14 success,
  57521 collected each, zero failures/errors. Its second attempt was advisory-only.
  Every compared run is at exact SHA `7423fa8380eceee38cc3e4fbb37dceb2feb8b641`. Source
  pin `3156d11f33d5a5277ff806a78b92a9490f78d6be` is contained by `v0.42.6`; isolated
  published 0.42.6 exposes all 880 current bindings. Required inventory digest is
  `a4ff4aa99b83777d444dcc526bc392f0f8249be21086443de821d76ac43d1165`.
- Rehearsal aggregate failures and evaluator exit 2 reflect the separate exercise
  obligation; they do not overturn the passing qualification clause. The failed host
  source run `40199727112` remains historical failed evidence, not acceptance.

This metrics-only change does not invalidate the deployed producer/evaluator proof. Do
not introduce another live-run dependency or wait for your own future commit.

## Final step: finish this landing and its authorized parent chain

After the code and required verification are complete:

1. Re-read `sase-1n6.5.1.5`, confirm every descendant is closed and the repair/proof
   acceptance above remains valid after drift. Resolve or re-key every entry from
   `sase bead epic-symbols sase-1n6.5.1.5`; re-key only to a still-open later bead that
   actually needs the exemption, following Symvision memory. Close normally with
   `sase bead close sase-1n6.5.1.5 --note "<audit, metrics integration, tests, retained proof and every follow-up disposition>"`.
   The close note must include the audit's proposal outcomes and decline reasons (it may
   reference the detailed prior audit while summarizing each group). Then run
   `just symvision` and set `status: done` in
   `plan:202610/qualification_landing_repairs.md`. Open `plans` with `/sase_repo` before
   edits and use the printed checkout; do not infer another workspace path.
2. Read `sase-1n6.5.1.5` again to verify its parent link is `sase-1n6.5.1`. This parent
   is a plan bead. Review its previous LANDING REVIEW, every descendant/note, linked
   `plan:202610/published_core_qualification.md`, and post-child drift. Its earlier
   transport/proof blockers are now satisfied by this epic. Rerun descendant readiness,
   archived-mode plan validation and `sase plan links validate -q` before closing.
   Retire all entries from `sase bead epic-symbols sase-1n6.5.1`, then close normally
   with
   `sase bead close sase-1n6.5.1 --note "<rechecked contracts, descendants, integration, exact-SHA proof, follow-ups and checks>"`.
   Run `just symvision`, then mark `plan:202610/published_core_qualification.md`
   `status: done`. Previously closed nested plans
   `plan:202610/deployed_qualification_proof.md` and
   `plan:202610/reprove_published_qualification.md` are already done; confirm readiness.
3. The next direct parent is phase `sase-1n6.5`. Verify that the completed child plan
   satisfies its qualify scope and all notes, preserve its publication integration seam,
   clear any `sase bead epic-symbols sase-1n6.5` entries, and close only this phase
   normally with
   `sase bead close sase-1n6.5 --note "<verified published qualification, wrong-core refusal, same-SHA source proof and implementation checks>"`.
   Leave containing epic `sase-1n6` and its plan to its waiting land agent, with its
   existing containment/exercise/build issues intact. If a parent is incomplete or
   ambiguous, leave it open and append a concrete blocker note instead.
4. Never use `--force` to advance a successful nested landing. If close rejects stale
   symbols or unfinished descendants, finish the cleanup/work and retry normally.
   Inspect final diffs and finish through `/sase_final`; the changed primary source and
   opened plans repository both require host commit decisions. No manual commit, branch,
   tag, push, or release is part of this tale.
