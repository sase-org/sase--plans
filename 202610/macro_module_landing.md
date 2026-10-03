---
tier: tale
title: Finish the macro module rename landing
goal:
  The macro terminology guard participates in the curated contract set, and epic
  sase-1eq.3.1 and parent phase sase-1eq.3 close with verified evidence.
size: medium
proposed_by: bbugyi200.athena.sase-1eq.3.1.land
bead: sase-1eq.3.1
create_time: 2026-10-03 05:11:35
status: wip
---

- **PARENT:**
  [202610/sase_modules_rename.md](https://github.com/sase-org/sase--plans/blob/main/202610/sase_modules_rename.md)
- **BEAD:**
  [sase-1eq.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eq/sase-1eq.3.1.md)

# Plan: Finish the macro module rename landing

## Goal and remaining work

Finish epic `sase-1eq.3.1` and its parent phase `sase-1eq.3` in this coding turn. The
module rename is implemented; its new terminology contract was not integrated into the
committed contract manifest. Repair that integration and finish the nested landing. Do
not wait for this turn's commit, SHA, push, or CI.

The approved epic plan is `plan:202610/sase_modules_rename.md`; its file is
`202610/sase_modules_rename.md` within the plans repository printed by
`sase repo open plans`. The containing epic is `sase-1eq`, with an already assigned land
agent. Close only the child epic and parent phase, not `sase-1eq`.

## Verified evidence and scope

The land agent audited the epic and all four closed children, every child note, the
approved plan and parent requirements, and commits `d9d0cae9f0`, `117f5779d3`,
`5541d4c697`, and `8de1add72c`. It reviewed the 25 other commits after the first epic
commit through clean master `0676975ef3`; a remote fetch confirmed `origin/master`
matched. Later MRU, prompt-lifecycle, pager, keymap, and memory-history work follows the
moved modules. No further rename gap was found in that integration review.

Verification on that tree:

- Shorthand wire/digest round-trip, guard, legacy readers, and prompt-key MRU filename
  probe: 39 passed.
- Loader, catalog, skill-resource, and wheel-packaging checks: 100 passed.
- Fresh interpreters: four required legacy import proofs, all 22 shim exports,
  synthetic-module isolation, and `mock.patch` isolation passed.
- Proposed-failure batch: 155 passed, with only the two unrelated failures below.
- All four phase descendants are closed. `sase plan links validate --json` reports
  `ok: true`. `sase bead epic-symbols sase-1eq.3.1` and
  `sase bead epic-symbols sase-1eq.3` report no entries.

Epic-caused failure: `tests/test_macro_terminology.py` declares
`pytestmark = pytest.mark.contract`, but `tests/contract_manifest.txt` omits it. A fresh
serial
`tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection` fails
with exactly that missing file. The other two manifest tests pass. The committed
manifest currently has 73 entries, and `tests/test_contract_manifest.py` pins its budget
to exactly 73 with a measured 57.25-second serial-cost comment. Blind regeneration
creates 74 entries and breaks that count invariant.

Source changes should stay in the contract guard, its manifest/curation tests, and any
justified contract-marker adjustment. The only other intended edits are the approved
epic plan's status and bead closeout notes. No core, plugin, TUI rename, syntax/config
migration, documentation, memory, or PNG work belongs in this tale. Preserve the
existing legacy/shim/deferred-syntax guard exceptions.

## Follow-up triage already finished

The land agent completed `sase_new_task` searches, the last-week task sweep, and
active-epic review, and recorded every proposal's outcome on `sase-1eq.3.1`. Do not file
those proposals again:

- `sase-1eq.3.1.1` note 1: legacy MRU filename assertion fixed by `d8efa2a6e5`, passing
  now. Note 2: busy-cluster load flake corroborated existing task `sase-1ak` with +1;
  unchanged-tree isolated rerun passes.
- `sase-1eq.3.1.2` note 1: old three-pane flag removed by `c62e4f1491`/`d8efa2a6e5`;
  current feature-flag lint passes.
- `sase-1eq.3.1.3` note 1: three-pane pager, editor harness, unmount-hook, and split-key
  failures fixed by `d8efa2a6e5`, all passing now. MRU pruning still reproduces,
  returning only `#gh:sase` instead of also `#git:otherproj`; exact existing task
  `sase-172` received +1. Its report predates this epic.
- The publication binding failure from `.3` note 1 and `.4` note 1, and the publication
  Symvision residue from `.3` note 4, belong to active epic `sase-1ex`. A detailed
  `DISCOVERED ISSUE:` was appended there. Commit `5c7e7514ae` (`sase-1ex.7`) introduced
  `src/sase/core/publication_payload_facade.py`; it has only a test consumer, and the
  current core source/wheel lacks `plan_publication_payload_batches`.
  `tests/test_check_sase_core_rs_bindings_tool.py::test_dev_extension_exposes_every_collected_name`
  therefore still fails after `just install`.
- `.3` note 2: config token/teardown/isolation, link-follow, and previously failing
  collections now pass after later fixes. Generic load-flake proposals lacked exact
  failing nodes/durable evidence and were declined on current green reruns. Contract
  collection completes in 55.62 seconds; the slowness proposal was declined. Its newly
  exposed manifest omission is this tale.
- `.3` note 3 was a guard-phase heads-up: surviving wire/env/plugin/discovery and TUI
  spellings are deferred parent scopes, not new tasks. The pr/commit and split-file path
  leftovers were fixed by `5541d4c697`.
- `.3` note 4: old `sase-1eu` whitelist residue removed by `5541d4c697`; PaneGrid public
  symbols fixed by `d8efa2a6e5`; current lint has no PaneGrid failures. The shim's
  `__getattr__` visibility was repaired.

`sase tool run check` ToolRun `6541adcd5aba30801281d54d4c0423ad` passed every lint stage
before Symvision, then stopped on 26 unrelated public symbols: two in the publication
facade (`sase-1ex`), and 24 from memory-history epic `sase-1ev` in
`memory/history/feed_model.py`, `ace/tui/modals/memory_pane_rail_glance.py`,
`memory_pane_instructions.py`, and `config_hub_pane.py`. The latter epic received a
detailed `DISCOVERED ISSUE:`. Triage labeled `glance_glyph_only` NEW, but git attributes
it to `8b27e3f011` (`sase-1ev.8`), present before this land turn; the tree had no source
changes. Keep this evidence separate from tale-caused failures.

## Implementation

1. Read the latest `sase-1eq.3.1` and `sase-1eq.3` notes, and the approved epic plan
   through `sase artifact read`. Check source/base drift since `0676975ef3`. Read
   `lint_and_test.md` and `symvision.md` via `sase memory read` before finishing
   changes. Consult `plan:202608/test_suite_tier1.md` through an audited artifact read
   for the contract curation procedure.

2. Integrate `tests/test_macro_terminology.py` into the generated contract manifest
   without paying for unnecessary whole-tree parsing on every check. Its NAME-token and
   import guards currently tokenize/AST-parse every source file even when no
   case-insensitive `xprompt` substring exists. Add a conservative text prefilter before
   those expensive operations, retaining the existing path scan and all case-insensitive
   NAME/import detection, legacy exceptions, and useful diagnostics. Keep the required
   contract marker. Do not widen the allowlist or change user-facing/deferred spellings.

   Re-curate membership by value per second. Prefer displacing an existing
   module-behavior test that is already reached through imports rather than raising the
   entry cap. A concrete candidate to assess is `tests/test_core_eligibility_facade.py`:
   its six tests import and exercise one facade rather than audit the repository; retain
   them as ordinary tests if removing only their contract marker. Prove its relevant
   source changes remain selected, and consider core-identity broadening. Do not remove
   assertions/tests or demote unrelated global audits merely to satisfy a count. If
   measured curation supports a different justified membership, document it.

   Run `just refresh-contract-manifest`, review the sorted manifest diff, and measure
   the resulting serial contract set with the project's
   `.venv/bin/python -m pytest -m contract` over the manifest paths,
   `-p no:randomly --durations=0`. Aggregate per-file durations and record actual count,
   cost, date, and selection rationale in `tests/test_contract_manifest.py`, keeping the
   exact-count/no-hidden-headroom assertion. Preserve the current budget policy; never
   blindly relax it. Separately classify the already documented publication-binding
   failure if it remains in that measured lane.

3. Run the terminology guard, all `tests/test_contract_manifest.py` tests, and any
   changed/demoted contract module tests. Confirm generated manifest matches marker
   selection and the exact budget. Recheck macro legacy/shim behavior only if changes or
   drift affect it. Run `just fmt`, then `sase tool run check` (the recorded form of
   `just check`). Resolve every tale/epic-caused failure. Preserve evidence for
   unrelated current-base failures on `sase-1ex`/`sase-1ev` instead of broadening this
   rename. Never run `just check-full` for this tale. If a long verification needs a
   monitor, its successor must execute the closeout below; a passing prepared completion
   must not skip the explicit target-epic and parent-phase closure.

4. Finish the landing in this same turn after verification, with no dependency on this
   turn's future commit, push, SHA, or CI:
   - Re-read readiness for `sase-1eq.3.1`, all descendants (including any tale bead the
     host created), and its linked plan. If this tale is itself an open descendant,
     close that verified tale bead normally first so the epic's descendant check is
     satisfied. Never force a successful landing.
   - Run `sase bead epic-symbols sase-1eq.3.1`. Resolve every listed symbol by wiring,
     privatizing, a justified non-test pragma, or deletion per Symvision policy. Re-key
     only when a concrete still-open later bead needs the exemption. Rerun until no
     entries remain.
   - Close normally with
     `sase bead close sase-1eq.3.1 --note "<source/commit/integration verification, manifest and measured curation repair, test results, and all follow-up triage outcomes>"`.
     Include the follow-up dispositions above, referring to the land audit notes for
     detailed evidence. If rejected, finish the named blockers; do not force.
   - Run `just symvision`, verify no whitelist entries remain for closed `sase-1eq.3.1`
     or its closed phases, and record any independently owned `sase-1ex`/`sase-1ev` lint
     errors accurately.
   - Open the plans repository with
     `sase repo open plans -r "Mark the verified macro module epic plan done"`,
     audit-read `plan:202610/sase_modules_rename.md`, and set `status: done` in the
     frontmatter of its `202610/sase_modules_rename.md` file in the printed repository.
     Include that repository in the final declaration.
   - Read `sase bead read sase-1eq.3.1 -r "Need the parent link"` and verify parent
     `sase-1eq.3` remains the phase whose required work this plan completes. Run
     `sase bead epic-symbols sase-1eq.3` and retire/re-key entries by the same policy
     before closing only that phase normally with
     `sase bead close sase-1eq.3 --note "Verified child epic sase-1eq.3.1 completed query shorthands, module/resource moves, token-aware identifiers, isolated plugin shim, and the committed contract guard; reviewed concurrent changes and verification evidence."`.
     Leave containing epic `sase-1eq` to its waiting land agent.
   - Declare completion with `sase_final` for every repository changed, including the
     plans repository. Let host finalizers commit after the turn.
