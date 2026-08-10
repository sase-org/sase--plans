---
tier: tale
title: Clear the flake-baseline gate and retire the sase-ct umbrella
goal:
  The flake-baseline exit criterion passes non-vacuously on post-fix records, the live
  residue and the gate defect are filed as node-specific task beads, sase-ct is closed
  as a retired umbrella that will not reopen, and the sase-iy epic plan is closed out.
size: medium
proposed_by: bbugyi200.athena.sase-j0.w1
create_time: 2026-08-10 15:10:36
status: wip
---

# Finish `sase-iy.5`: clear the flake-baseline gate and retire `sase-ct`

## Goal

Complete the `retire` phase of epic `sase-iy` (phase bead `sase-iy.5`): clear the one
remaining red exit criterion, file the true residue as node-specific beads, close
`sase-ct` as a retired umbrella, and close out the epic plan.

This plan does **not** re-open any question that is already settled. Read "Already done"
before doing anything.

## Already done — do not re-plan or redo these

1. **The skill change landed.**
   `8501a19ac fix(skills): route retired umbrella duplicates to new tasks` changed
   `src/sase/xprompts/skills/sase_new_task.md` so step 4's duplicate branch routes a
   retired umbrella to a new node-specific task bead with a `RELATED:` note instead of a
   `+1`. Generated skills were deployed with `sase skill init --force` (chezmoi
   `c4759318`). This satisfies the epic plan's ordering constraint that the skill change
   must land _before_ `sase-ct` is closed. Verify it is still present, but do not
   re-implement it.

2. **The test-cost budget blocker is fixed.** Phase `sase-iy.5` recorded a
   `PROPOSED FOLLOW-UP` that `tools/check_test_cost_budgets` failed six budgets. That
   became task bead `sase-j0`, which is **closed `done`** via
   `c8e4016c7 fix(test-cost): recalibrate suite-cost budgets against real recorded history`.
   Re-verified on master `c8e4016c7`: `.venv/bin/python tools/check_test_cost_budgets`
   exits **0** against recording `20260810T185645Z-773456.json`. Do not re-plan the cost
   budgets.

3. **Three of the four exit criteria already passed** on the combined tree per
   `sase-iy.5`'s own notes: `just test-visual` 648 passed / 1 skipped in 322.17s; the
   `residue` node set under `just test-contention` reported 0 node failures across 3
   repeats in 183.9s; `.venv/bin/python tools/check_test_wait_helpers` exits 0. They
   must be re-run once on the final tree (step 4), but they are not expected to be work.

## The one remaining blocker

`just selection-health --fail-on-new-flake` (Justfile recipe `selection-health`) exits
1:

```
flake baseline gate: 8 reproducible flake(s) exceed tests/reproducible_flake_baseline.txt
(records after 2026-08-08T19:56:29Z, at most 5 failures per run)
```

The gate reads durable full-run records from the selection store, keeps those recorded
after the baseline file's `effective-after:` timestamp with a resolvable change set and
at most `MAX_GATED_FAILURES_PER_FULL_RUN = 5` failures, and calls a node a reproducible
flake when it fails across two disjoint change sets with an independent interleaved pass
(`reproducible_flake_nodeids`, `tests/_test_selection_health.py:183`). Records are never
pruned when a fix lands, so a node stays flagged until the cutoff moves past the
failures — which is exactly what commit
`607b72bb0 test: bump flake-baseline cutoff past fixed historical xprompt records` did
before, and is the precedent this plan follows.

The current cutoff, `2026-08-08T19:56:29Z`, predates every `sase-iy` mechanism fix. It
is therefore judging the epic's own pre-fix history and reporting the very flakes the
epic just fixed.

### Per-node triage (already performed — reproduce it, do not redo it from scratch)

All timestamps UTC. "Head" is the commit the failing run was recorded against.

| #   | Node                                                                                                                                | Last failure                                                                       | Verdict                                                                                                                                                                                      |
| --- | ----------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `tests/test_run_pytest_main.py::test_main_cost_mode_arms_only_the_cost_recorder`                                                    | 14:33Z                                                                             | **Test no longer exists** — renamed to `test_main_cost_mode_arms_cost_and_health_recorders` by `1417de7db`. This node ID can never pass again.                                               |
| 2   | `tests/test_vcs_provider_vcs_log.py::test_remote_log_ops_fetch_partition_and_union_log`                                             | 2026-08-09 18:37Z                                                                  | Historical; clean for over 24h.                                                                                                                                                              |
| 3   | `tests/ace/tui/widgets/test_prompt_glossary_navigation.py::test_k_on_glossary_term_pushes_glossary_preview_card`                    | 15:21:57Z @ `344a0b8ff`                                                            | All 11 failures predate its own fix, `waitgate` commit `c49452c47` (15:48:33Z).                                                                                                              |
| 4   | `tests/test_contract_manifest.py::test_contract_set_manifest_entry_budget_has_no_hidden_headroom`                                   | 16:44:25Z @ `47b2a74aa`                                                            | Head predates the manifest updates in `dcb243b75` / `8501a19ac`.                                                                                                                             |
| 5   | `tests/test_agent_group_revival_e2e.py::test_mark_save_preview_and_revive_saved_agent_group`                                        | 16:44:25Z @ `47b2a74aa`                                                            | Head predates `residue` fix `ebd3a91bc` (16:41:54Z).                                                                                                                                         |
| 6   | `tests/test_agent_group_revival_e2e.py::test_saved_group_revive_restores_deleted_artifacts_and_tribe_real_loader`                   | 16:52:44Z @ `47b2a74aa` (also 16:50:41Z @ `187085a80`, an ancestor of `ebd3a91bc`) | Both heads predate the `residue` fix.                                                                                                                                                        |
| 7   | `tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection`                                                  | 17:24:55Z @ `43337c3f7`                                                            | A **deterministic** stale-manifest break (task `sase-iu`), not an ACE flake; the manifest was refreshed by `8501a19ac` (17:38:13Z) and the node passes on master.                            |
| 8   | `tests/ace/tui/widgets/test_agent_display_xprompt.py::TestAgentXPromptRendering::test_agent_xprompt_highlights_warm_catalog_skills` | 17:22:46Z @ `dcb243b75`                                                            | **Genuinely live.** `dcb243b75` already contains `c49452c47`, `ebd3a91bc`, and `128b326ea`, so this failure is post-fix. It is the recurrence agent `xd` routed to epic `sase-iy` at 17:26Z. |

All seven still-existing nodes pass in isolation on master `c8e4016c7`
(`7 passed in 29.46s`). **Isolation passes are not evidence of a fix** — passing alone
is the `sase-ct` signature. The evidence that matters is record-based: for nodes 2–7
every failure is charged to a head that predates the relevant fix, and eligible records
recorded after those fixes are clean.

## Step 1 — Bump the flake-baseline cutoff, with the evidence recorded

Edit `tests/reproducible_flake_baseline.txt` and change only the header line:

```
# effective-after: 2026-08-08T19:56:29Z
```

to

```
# effective-after: 2026-08-10T16:50:24Z
```

`2026-08-10T16:50:24Z` is the commit time of `128b326ea`, the last of the three
`sase-iy` mechanism fixes (`catalog` `128b326ea`, `waitgate` `c49452c47`, `residue`
`ebd3a91bc`). It is the **minimal** bump that excludes only trees predating the epic's
fixes.

**Do not add any node IDs to the baseline list.** The file's own header says entries are
"debt to remove, not suppressions to grow." This step removes stale _records_ from the
judging window; it does not suppress any node.

### Verify the bump is correct and non-vacuous

The epic plan warns explicitly that `sase-h8` was misled by a vacuously-passing gate, so
this is a required check, not a nicety. Reproduce this locally before committing:

```bash
.venv/bin/python - <<'PY'
import sys
from pathlib import Path
sys.path.insert(0, ".")
from tests._test_selection_health_records import load_records
from tests._test_selection_health_store import store_directory
from tests._test_selection_health import reproducible_flake_nodeids, git_commit_order_oracle
full = list(load_records(store_directory(Path("."))).full_runs)
oracle = git_commit_order_oracle(Path("."))
cut = "2026-08-10T16:50:24"
elig = [r for r in full if r.recorded_at and r.recorded_at > cut
        and r.changed_files is not None and len(r.failures) <= 5]
flagged = sorted(reproducible_flake_nodeids(elig, max_failures_per_run=5, commit_order=oracle))
print("eligible:", len(elig), "flagged:", len(flagged))
for n in flagged: print("   ", n)
PY
```

At authoring time this reported **27 eligible records** and **2 flagged nodes**:

- `tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_directory_key_spelling_also_resolves`
- `tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_offers_project_local_xprompts_by_canonical_name`

Both are **already in the baseline**, so the gate exits 0 — while still detecting two
real reproducible flakes from post-fix history. That is the non-vacuity proof: the gate
is judging live records and still catching things, it is simply no longer catching the
flakes this epic fixed. `MIN_GATED_FULL_RUNS` is 2, and 27 >> 2.

Two facts to confirm rather than assume, because the store grows as other agents run:

- The eligible count is still comfortably above 2. If it has dropped near 2, say so in
  the close note instead of counting the criterion as met.
- The flagged set is still non-empty and still a subset of the baseline. **If a bumped
  cutoff makes the gate flag nothing at all, that is the vacuous outcome the epic plan
  warns about** — a cutoff of `2026-08-10T18:00:00Z` was measured to flag 0 nodes, which
  is why this plan picks the earlier, evidence-backed timestamp instead of "now".

Record the measured numbers in the commit message and in the `sase-iy.5` bead note.

## Step 2 — File the live residue as its own node-specific task bead

Node 8, `test_agent_xprompt_highlights_warm_catalog_skills`, is real post-fix residue.
The cutoff bump drops it from the gate only because a single post-cutoff failure cannot
meet the gate's own two-disjoint-change-set definition of "reproducible" — not because
it is fixed. Leaving it unfiled would be the dishonest reading this epic exists to stop.

Use `/sase_new_task` to file it. This is step 3 of the epic plan's `retire` phase and
the whole point of the retirement: demonstrate the pattern on the first real case.

- Name the bead for the specific failing node ID.
- Include the evidence: failed 2026-08-10T17:22:46Z under the full parallel lane at head
  `dcb243b75` (a tree already containing `c49452c47`, `ebd3a91bc`, `128b326ea`); passes
  in isolation; independently observed by agent `xd` at 17:26Z and routed to epic
  `sase-iy`.
- Attach `sase bead note <new-id> "RELATED: sase-ct — <how it bears on this task>"`.
- Choose `--size` deliberately. Default to `large` unless you can state the precise root
  cause in the bead.

Note that `/sase_new_task` should now route you here itself once `sase-ct` is closed —
but at this point `sase-ct` is still open, so file it directly and do **not** `+1`
`sase-ct`.

## Step 3 — File the gate defect the triage exposed

Node 1 is a genuine `tools/selection_health` defect, independent of this epic: a test
that is renamed or deleted leaves a node ID that can never pass again, so it stays
flagged until someone bumps the cutoff. The cutoff bump in step 1 clears it
incidentally, which means it will silently recur the next time a flaky test is renamed.

File this through `/sase_new_task` as its own task bead: the flake gate should skip (or
report separately as stale) node IDs that no longer exist in the collected suite.
Include the concrete instance — `test_main_cost_mode_arms_only_the_cost_recorder`
renamed to `test_main_cost_mode_arms_cost_and_health_recorders` by `1417de7db`.

Do **not** implement this fix in this plan; it is out of the `retire` phase's scope and
would widen a `medium` tale into gate-design work.

One lesser observation to include in that bead as context, not as separate scope: of 201
distinct record heads in the store, `git_commit_order_oracle` failed to resolve exactly
1 (a commit from another workspace not present in this checkout). Unresolved heads fall
back to sequence order and sort last (`_ordered_flake_candidate_runs`,
`tests/_test_selection_health.py:244`). At 1 of 201 this is not currently distorting the
verdict, but it is the mechanism by which cross-workspace records could be mis-ordered.

## Step 4 — Run the four exit criteria on the combined tree

All four, on the integrated tree, **before** touching `sase-ct`:

1. `just check-full` green, end to end.
2. `just test-visual` green.
3. `just test-contention` on the `residue` node set, zero failures in the tally:
   ```bash
   just test-contention -- tests/test_agent_group_revival_e2e.py \
     tests/ace/tui/test_commits_pane_filters.py \
     tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py \
     tests/ace/tui/test_plugins_browser_pane_sase_update_dev.py
   ```
4. `.venv/bin/python tools/check_test_wait_helpers` exits 0, **and**
   `just selection-health --fail-on-new-flake` passes non-vacuously per step 1.

Run `just install` first — this workspace may be stale.

If a criterion cannot be run, say which and why. **If a criterion fails, fix it or file
it and report; do not close `sase-ct` on a criterion you did not meet.** `sase-h8.5`
closed `done` with zero commits and looked identical to a real closure from the outside.

`just test-contention` deliberately starves the host and other agents share this machine
— keep it scoped to the files above.

## Step 5 — Close `sase-ct`

`sase-ct` currently sits at `READY [+60] [↺8]`. Close it with resolution `done` and this
reason, which is the epic plan's verbatim text with the real references substituted:

> RETIRED UMBRELLA — DO NOT `+1` THIS BEAD.
>
> `sase-ct` tracked a class ("an ACE TUI test failed under the full parallel run and
> passed in isolation") that matches every future ACE timing failure, so every reporter
> corroborated it and every `+1` reopened it: 60 `+1`s, 8 closures, 8 reopens, two epics
> (`sase-h8`, `sase-h8.10`). The bead could not stay closed because the tracking
> pattern, not the tests, was the defect.
>
> The live instances are fixed by mechanism: `sase-iy.2` (`128b326ea`) fixed the
> deterministic `prompt-catalog:0` convergence hang that held `wait_for_visual_idle`
> open for its full 30s deadline and made the PNG lane red in isolation; `sase-iy.3`
> (`c49452c47`) widened `tools/check_test_wait_helpers` past its `pilot`-receiver and
> `_wait_until`-name blind spots and migrated the attempt-bounded pause loops it then
> reported, including `test_k_on_glossary_term_pushes_glossary_preview_card`;
> `sase-iy.4` (`ebd3a91bc`) fixed the remaining contention-sensitive nodes against a
> measured `just test-contention` before/after tally of commits-pane 2/3 and agent-group
> 2/3 and 1/3 failing → 0 node failures across 3 repeats.
>
> If you hit an ACE TUI test that fails under the full parallel run and passes in
> isolation: **do not `+1` this bead, and do not reopen it.** File a new task bead
> through `/sase_new_task` named for the specific failing node ID, and record
> `RELATED: sase-ct` on it with a note explaining how it bears on the new task. A
> node-specific bead can be fixed and can stay closed; this umbrella could not.
>
> Re-run the measurement with `just test-contention`, `just test-visual-contention`, and
> `tests/reproducible_flake_baseline.txt`.

Confirm the `+1` count and reopen count against `sase bead show sase-ct` at close time
and correct the numbers if they have moved. Confirm `8501a19ac` is still an ancestor of
the tree you are closing from — if the skill change were ever reverted, the window
between close and re-landing is exactly when the next `+1` reopens `sase-ct` a ninth
time.

## Step 6 — Report to the owner without acting

Report, do not act:

- `sase-iu` is `READY` and `sase-iv` is an `OPEN` byte-identical duplicate of it. Both
  name `tests/test_contract_manifest.py` nodes that **now pass on master** (verified in
  this session), so one should likely be closed as a duplicate of the other and both as
  already-fixed. Out of this epic's scope; the owner's call.
- The two task beads filed in steps 2 and 3, by ID.

## Step 7 — Close out the epic plan

Use `/sase_repo` to open the `plans` sidecar before editing it; do not reach for the
path directly.

- Set `status: done` in the frontmatter of `202608/retire_sase_ct_umbrella.md`.
- Do **not** close `sase-h8` or `sase-h8.10`. Note on each that `sase-ct` was retired
  here, by what criteria, and that it will not reopen, so their land agents are not left
  waiting on a bead that will never come back.
- Record on `sase-iy.5` the measured numbers from step 1, the four exit-criteria results
  from step 4, and the bead IDs from steps 2, 3, and 5.

## Watch out for

- **Do not grow `tests/reproducible_flake_baseline.txt`.** The only edit to that file is
  the one-line cutoff change.
- **Do not treat isolation passes as proof.** Passing in isolation while failing under
  the full lane is the definition of the class being retired. The record-based argument
  in the triage table is the evidence; reproduce it rather than substituting a green
  isolated run.
- **Do not pick a "safe" late cutoff.** Bumping past every failure produces a gate that
  flags nothing, which is the vacuous pass the epic plan calls out by name. The measured
  `16:50:24Z` value keeps the gate live.
- **Other agents are writing to the record store concurrently.** The eligible-record
  count and flagged set will drift between your measurement and your commit. Re-measure
  immediately before committing and report the numbers you actually saw.
