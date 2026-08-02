---
tier: tale
title: Audit open task beads and close duplicate, stale, and non-reproducible ones
goal:
  The non-closed task-bead backlog contains only beads that still describe real unresolved work, with every closed bead
  carrying an accurate resolution and evidence trail.
proposed_by: bbugyi200.athena.ru
create_time: 2026-08-02 09:45:50
status: wip
---

- **PROMPT:**
  [prompts/202608/open_task_bead_triage_audit.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/open_task_bead_triage_audit.md)

# Plan: Audit open task beads and close duplicate, stale, and non-reproducible ones

## Problem

Twenty task beads are currently non-closed (`open`, `ready`, or `in_progress`). Most of them were filed between
2026-08-01 and 2026-08-02 by `toobig-*` file-splitting agents and epic land agents that hit pre-existing repository-wide
gate failures while running their own required `just check`. Because each agent filed independently, the backlog now
contains:

- several exact semantic duplicates (the same Symvision symbol set, the same failing test node, the same plan-link
  error, reported minutes or hours apart by different agents), and
- a large number of reports whose underlying failure has since been fixed on `master`, so they are stale and no longer
  reproduce.

Every `ready` task bead raises a `TaskTriage` gate for the project owner, so this backlog is actively noisy. The goal of
this plan is to shrink it to the beads that still describe real, unresolved work, without discarding evidence that a
future reporter would need.

This plan makes **no source-code changes**. It only reads verification gates and mutates bead state.

## Baseline evidence already collected

All of the following was verified on `master` at commit `d0f0b6161` ("test: stabilize ci restoration checks") in a
freshly `just install`-ed workspace on 2026-08-02:

| Gate              | Command                                                                                                               | Result                                                                                   |
| ----------------- | --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Symvision         | `just _lint-symvision`                                                                                                | exit 0 — `All public/private classes/functions are used properly!`                       |
| SDD validation    | `sase validate`                                                                                                       | exit 0 — all five checks `ok`, including `plan links validate`                           |
| Plan links detail | `sase plan links validate`                                                                                            | `SDD validation passed: 3401 files, 521 warnings`; **0** `uppercase_active_subtabs` hits |
| PNG visual suite  | `just test-visual`                                                                                                    | **405 passed, 1 skipped** in 281s                                                        |
| Full suite        | `just test`                                                                                                           | **2 failed, 25,389 passed, 7 skipped** in 258s                                           |
| Cited stale tests | `uv run pytest tests/ace/tui/test_admin_center_selection_resume.py tests/ace/tui/test_copy_as_palette_entrypoints.py` | 17 passed                                                                                |

The two `just test` failures were:

1. `tests/test_bead/test_cli_work_contention_regressions.py::test_concurrent_bead_mutations_wait_past_the_old_lock_timeout`
   — a third reproduction of the flake already tracked by `sase-dy`. This is why `sase-dy` stays open and only its
   duplicate `sase-e2` is closed.
2. `tests/ace/tui/widgets/test_prompt_at_prefix_completion.py::TestAtPrefixIntegration::test_at_prefix_directory_drilldown`
   — passes alone in 5.40s, so another load-sensitive flake. `sase bead search at_prefix` returns no match, so no
   existing bead covers it. See Step 9.

Critically, **none** of the failures reported by the Group A, B, or C beads reproduced in that run.

Corroborating repository facts:

- `tests/ace/tui/test_admin_center_selection_resume.py:39` now imports `patch_store_loader as _patch_store_loader`, so
  the `ImportError` cited by `sase-du` and `sase-dw` is fixed.
- `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py:363` defines `_has_bulk_read_undo_available`, so the
  missing-helper failure cited by `sase-dw` is fixed.
- The plans sidecar (`sase repo path plans`) is on `main` in sync with `origin/main` and no longer has a
  `202607/prompts/` directory, so the `prompt-in-plans-store` errors behind `sase-e0` are migrated away. Commit
  `404fac3b5` ("fix(validation): skip unavailable prompt archive context") is the matching code fix.
- The `BulkUnreadToggleResult`, `prune_prompt_artifact_pool`, `find_pending_task_triage`, and `resolve_task_launch_cwd`
  symbols named by `sase-dq`, `sase-ds`, and `sase-dv` no longer exist anywhere under `src/`.

## Guardrails

These rules exist because a previous audit closed a load-sensitive flake that then recurred, and the bead had to be
reopened by a `+1` (see `sase-cf` below). Follow them exactly.

1. **Never touch a bead whose status is `in_progress`.** A runner owns it. In this backlog that is `sase-d1` and
   `sase-di`.
2. **Never close a load-sensitive flake bead just because the node passes in isolation.** "Fails under the full parallel
   suite, passes alone" is the _symptom being reported_, not evidence of resolution. Only a duplicate of another flake
   bead may be closed, and only by consolidating onto the survivor.
3. **Transfer evidence before closing a duplicate.** Append the loser's distinguishing evidence to the survivor with
   `sase bead note` first, then close. Do not use `sase bead +1` for this — a `+1` is first-hand independent
   reproduction by its reporter, and re-attributing another agent's report as your own `+1` corrupts the corroboration
   count.
4. **Re-verify before every close.** `master` may have moved since this plan was written. Re-run the gate named in the
   disposition table and close only if it still passes. If a gate now fails, leave that bead open and record what you
   saw with `sase bead note`.
5. **Use the right resolution.** `-R superseded` for a bead closed because another bead already tracks the same defect;
   `-R canceled` for a bead whose reported failure no longer reproduces. Never `-R done` — this audit did not do the
   work. Every close needs a `--reason`.
6. **Closing never cascades and re-closing is a safe no-op**, but a conflicting resolution or reason is refused. If a
   close is rejected, read the error and record findings with `sase bead note` instead of forcing.
7. **Create no beads except the one named in Step 8**, and never run `sase bead rm`.

## Dispositions

### Group A — Symvision unused-public / private-misuse reports (6 beads, all close)

Gate to re-verify: `just _lint-symvision` (exit 0).

| Bead      | Status | Disposition     | Rationale                                                                                                                                                                                                                                         |
| --------- | ------ | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sase-dj` | open   | `-R superseded` | Exact duplicate of `sase-dk` — same three symbols (`bead_status_display_order`, `bead_type_chip`, `hierarchical_id_key`), same reporting agent, filed 8 seconds apart. `sase-dj`'s description is a raw paste of the failing `just check` output. |
| `sase-dk` | ready  | `-R canceled`   | Canonical survivor of the pair; Symvision is now clean, so the finding no longer reproduces.                                                                                                                                                      |
| `sase-dv` | ready  | `-R superseded` | Exact duplicate of `sase-ds` — identical seven-symbol finding, filed ~30 minutes later by a sibling `toobig-1d` agent.                                                                                                                            |
| `sase-ds` | ready  | `-R canceled`   | Canonical survivor of that pair; Symvision is now clean.                                                                                                                                                                                          |
| `sase-dq` | ready  | `-R canceled`   | Subset of the same seven-symbol finding; four of its named symbols no longer exist under `src/` and Symvision is clean.                                                                                                                           |
| `sase-dm` | ready  | `-R canceled`   | Reports a private-misuse finding (`_hierarchical_id_key` imported across modules). Symvision now passes, so the finding no longer reproduces.                                                                                                     |

Close `sase-dj` before `sase-dk`, and `sase-dv` before `sase-ds`, so each `superseded` close names a bead that is still
open at the time it is written.

### Group B — SDD / plan-link validation reports (2 beads, both close)

Gate to re-verify: `sase validate` (exit 0) and `sase plan links validate`.

| Bead      | Status | Disposition     | Rationale                                                                                                                                                                                                                                                                                                                                                                                                                           |
| --------- | ------ | --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sase-dt` | ready  | `-R superseded` | Duplicate of `sase-dn` ("Repair uppercase active subtabs plan links"), which was closed `done` on 2026-08-02T10:19:36Z. `sase plan links validate` reports zero `uppercase_active_subtabs` findings.                                                                                                                                                                                                                                |
| `sase-e0` | ready  | `-R canceled`   | Reported 5,764 `plan links validate` errors dominated by `prompt-in-plans-store` and `link-missing-target`. `sase validate` now passes with zero errors; the legacy `202607/prompts/` directory is gone from the plans sidecar and `404fac3b5` fixed the validator's prompt-archive handling. Note that `sase-e0` carries a `+1` from `rn` at 2026-08-02T11:13Z, so state the verified-clean re-run explicitly in the close reason. |

### Group C — ACE test-suite and PNG-golden reports (5 beads, all close)

Gates to re-verify: `just test-visual` (405 passed / 1 skipped) plus the full `just test` run from Step 2.

These five beads were filed by three different `toobig-*` agents plus one land agent within about two hours on
2026-08-01, and all describe the same branch-wide mismatch: Artifacts key `5` opening `files` instead of `prs`,
`plans`/`chats` moved under `Files`, ChangeSpec onboarding order missing `Beads`, the removed `_patch_store_loader`
helper, and the PNG drift cascading from that stale subtab setup.

| Bead      | Status | Disposition     | Rationale                                                                                                                                                                                                                                                                                        |
| --------- | ------ | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `sase-do` | ready  | `-R superseded` | 308-failure report of the branch-wide mismatch; `sase-dw` describes the same root cause most completely.                                                                                                                                                                                         |
| `sase-du` | ready  | `-R superseded` | 313-failure report of the same mismatch, filed ~90 minutes later by a sibling agent.                                                                                                                                                                                                             |
| `sase-dp` | ready  | `-R superseded` | The narrow symptom of the same root cause: the Tasks PNG fixture timing out because `page.press("5")` left `artifacts_subtab` on `files`. The whole `tests/ace/tui/visual/test_ace_png_snapshots_config_center_tasks.py` file now passes.                                                        |
| `sase-dl` | ready  | `-R canceled`   | Widespread PNG baseline drift (307 failing snapshots, 264/401 under `just test-visual`). The full visual suite now passes 405/405. This bead is _not_ folded into `sase-dw` because it tracks renderer/baseline drift as its own recurring theme (follow-up to `sase-bl`, `sase-c5`, `sase-c6`). |
| `sase-dw` | ready  | `-R canceled`   | Canonical survivor for the mismatch cluster; every named failure now passes. Close this **last**, after `sase-do`, `sase-du`, and `sase-dp`.                                                                                                                                                     |

### Group D — Load-sensitive flakes (6 beads, close exactly one)

| Bead      | Status      | Disposition      | Rationale                                                                                                                                                                                                                                                                                                                                   |
| --------- | ----------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sase-e2` | ready       | `-R superseded`  | Duplicate of `sase-dy`: both report `tests/test_bead/test_cli_work_contention_regressions.py::test_concurrent_bead_mutations_wait_past_the_old_lock_timeout` exhausting the exclusive-lock timeout under full-suite load. `sase-dy` was filed first (2026-08-01T18:40Z). Per the task-bead rules this should have been a `+1` on `sase-dy`. |
| `sase-dy` | ready       | **keep open**    | Canonical bead for that flake. Reproduced again on 2026-08-02 at 13 workers, at 16 workers, and a third time in this plan's own baseline `just test` run, so it is live.                                                                                                                                                                    |
| `sase-cf` | ready       | **keep open**    | Precedent case: it was closed by the 2026-08-01 audit as no longer reproducible, then re-promoted to `ready` by an independent `+1` on 2026-08-02T12:04Z after recurring in a 28-worker `just check`. Do not close it again.                                                                                                                |
| `sase-e1` | ready       | **keep open**    | Distinct node (`test_mirror_counts_global_detached_and_this_sessions_command_tasks`), single report, no duplicate.                                                                                                                                                                                                                          |
| `sase-dx` | ready       | **keep open**    | Distinct subject (Artifacts navigation p95 benchmark outliers), no duplicate.                                                                                                                                                                                                                                                               |
| `sase-d1` | in_progress | **do not touch** | Runner-owned.                                                                                                                                                                                                                                                                                                                               |

### Group E — Unrelated in-progress work

| Bead      | Status      | Disposition                                                                              |
| --------- | ----------- | ---------------------------------------------------------------------------------------- |
| `sase-di` | in_progress | **do not touch** — runner-owned; unrelated subject (file-hint markers inside HTTP URLs). |

**Totals:** 14 beads closed (6 `superseded`, 8 `canceled`); 6 beads left alone.

## Implementation steps

### Step 1 — Refresh the workspace and confirm the backlog

```bash
git -C "$(sase repo path plans)" pull --ff-only
just install
sase bead list --type task --format json --limit 0 > /tmp/task_beads_before.json
```

Compare the result against the disposition table. If a bead listed here is already closed, skip it. If a new non-closed
task bead appears that is not in the table, leave it alone and report it at the end — do not improvise a disposition for
it.

### Step 2 — Re-verify every gate

```bash
just _lint-symvision            # Groups A
sase validate                   # Group B
sase plan links validate 2>&1 | grep -c uppercase_active_subtabs   # Group B, expect 0
just test-visual                # Group C
just test                       # Group C
```

Record the actual pass/fail counts; you will quote them in the close reasons. If `just test` reports failures, classify
each one:

- A failure that matches a bead in Groups A–C means that bead is **not** stale. Remove it from the close list, append a
  `sase bead note` with the fresh reproduction, and say so in the final report.
- A failure that matches a Group D flake is expected and changes nothing; the flake beads stay open either way.
- A failure matching nothing in the table is new discovered work. Do not close anything on account of it. If it is the
  `at_prefix` flake, Step 8 handles it. If it is something else, report it at the end and leave it to the owner rather
  than expanding this plan's scope.

### Step 3 — Consolidate duplicate evidence onto the survivors

Run these before any close, so no evidence is lost.

```bash
sase bead note sase-dk "Consolidating duplicate sase-dj, filed 8s earlier by the same toobig-1c config_pane_widget agent against the identical three-symbol Symvision finding."

sase bead note sase-ds "Consolidating duplicate sase-dv, which reported the identical seven-symbol Symvision unused-public finding from the toobig-1d artifacts_beads split."

sase bead note sase-dw "Consolidating duplicate reports sase-do (308 failures), sase-du (313 failures), and sase-dp (Tasks PNG fixture timing out on artifacts_subtab=files after page.press(\"5\")). All three describe the same branch-wide Artifacts subtab/test/golden mismatch."

sase bead note sase-dy "Consolidating duplicate sase-e2, which reported the same node test_concurrent_bead_mutations_wait_past_the_old_lock_timeout. Additional evidence carried over from sase-e2: 13-worker just test on 2026-08-02 failed the node after 103.37s with two mutation workers exhausting the 12,000ms exclusive-lock timeout, passing alone in 3.67s; and reporter ro independently reproduced it in a 16-worker just check on 2026-08-02, failing after 31.71s while 25,371 tests passed and passing alone in 3.40s. sase-e2 was sized medium."
```

`sase-dy` has no recorded size while its duplicate `sase-e2` was sized `medium`. Carry that over:

```bash
sase bead update sase-dy -z medium
```

`sase bead update` accepts `-z/--size` with the values `xsmall`, `small`, `medium`, `large`, and `xlarge`.

### Step 4 — Close Group A (Symvision)

```bash
sase bead close sase-dj -R superseded --reason "Duplicate of sase-dk: identical Symvision unused-public finding for bead_status_display_order, bead_type_chip, and hierarchical_id_key, filed 8 seconds apart by the same agent. Consolidated onto sase-dk."
sase bead close sase-dk -R canceled --reason "No longer reproducible on master <SHA>: just _lint-symvision exits 0 with 'All public/private classes/functions are used properly!'. Absorbed duplicate sase-dj."
sase bead close sase-dv -R superseded --reason "Duplicate of sase-ds: identical seven-symbol Symvision unused-public finding. Consolidated onto sase-ds."
sase bead close sase-ds -R canceled --reason "No longer reproducible on master <SHA>: just _lint-symvision exits 0. Absorbed duplicate sase-dv."
sase bead close sase-dq -R canceled --reason "No longer reproducible on master <SHA>: just _lint-symvision exits 0, and BulkUnreadToggleResult, prune_prompt_artifact_pool, find_pending_task_triage, and resolve_task_launch_cwd no longer exist under src/."
sase bead close sase-dm -R canceled --reason "No longer reproducible on master <SHA>: just _lint-symvision exits 0, so the private _hierarchical_id_key cross-module import finding is gone."
```

Replace every `<SHA>` with the actual `git rev-parse --short HEAD` you verified against.

### Step 5 — Close Group B (SDD validation)

```bash
sase bead close sase-dt -R superseded --reason "Duplicate of sase-dn, which was closed done on 2026-08-02T10:19:36Z after repairing the 202607/uppercase_active_subtabs.md prompt links. Verified on master <SHA>: sase plan links validate reports zero uppercase_active_subtabs findings."
sase bead close sase-e0 -R canceled --reason "No longer reproducible on master <SHA>: sase validate passes all five checks and sase plan links validate reports 3401 files with 0 errors (521 warnings), against the reported 5,764 errors / 519 warnings. The plans sidecar no longer contains 202607/prompts/, and 404fac3b5 fixed the validator's unavailable-prompt-archive handling. This supersedes the 2026-08-02T11:13Z +1 from rn."
```

### Step 6 — Close Group C (ACE tests and PNG goldens)

```bash
sase bead close sase-do -R superseded --reason "Duplicate of sase-dw: same branch-wide Artifacts subtab / stale-test / PNG-golden mismatch. Consolidated onto sase-dw."
sase bead close sase-du -R superseded --reason "Duplicate of sase-dw: same branch-wide mismatch, including the _patch_store_loader import and the nested plans/chats subtabs. Consolidated onto sase-dw."
sase bead close sase-dp -R superseded --reason "Narrow symptom of the sase-dw mismatch (artifacts_subtab stayed on files after page.press(\"5\")). Verified on master <SHA>: tests/ace/tui/visual/test_ace_png_snapshots_config_center_tasks.py passes in the full just test-visual run."
sase bead close sase-dl -R canceled --reason "No longer reproducible on master <SHA>: just test-visual reports 405 passed, 1 skipped, against the reported 264/401 and 307 failing snapshots. The representative node test_axe_generated_instance_warning_png_snapshot passes."
sase bead close sase-dw -R canceled --reason "No longer reproducible on master <SHA>: <just test summary>. test_admin_center_selection_resume.py now imports patch_store_loader as _patch_store_loader, _has_bulk_read_undo_available exists in _leader_mode.py, and just test-visual passes 405/405. Absorbed duplicates sase-do, sase-du, and sase-dp."
```

Replace `<just test summary>` with the real counts from Step 2.

### Step 7 — Close the one Group D duplicate

```bash
sase bead close sase-e2 -R superseded --reason "Duplicate of sase-dy: both report tests/test_bead/test_cli_work_contention_regressions.py::test_concurrent_bead_mutations_wait_past_the_old_lock_timeout exhausting the exclusive-lock timeout under full-suite load. sase-dy was filed first on 2026-08-01T18:40Z; per the task-bead rules this belonged there as corroboration. Both reproductions and the medium size are recorded on sase-dy."
```

Do not close `sase-cf`, `sase-dy`, `sase-e1`, or `sase-dx`.

### Step 8 — File the one genuinely new flake

The baseline `just test` run surfaced a load-sensitive failure that no bead covers:
`tests/ace/tui/widgets/test_prompt_at_prefix_completion.py::TestAtPrefixIntegration::test_at_prefix_directory_drilldown`
failed under the full parallel suite and passed alone in 5.40s. `sase bead search at_prefix` found no match.

Invoke `/sase_new_task` for it (that skill owns duplicate detection and sizing) — do not run `sase bead create`
directly. If your own Step 2 `just test` run reproduced it, use your run as the evidence; if it did not, cite this
plan's baseline run. If `/sase_new_task` finds that a bead now exists, corroborate that bead instead of creating a new
one.

This is the only bead this plan may create.

### Step 9 — Verify the resulting backlog

```bash
sase bead list --type task --format json --limit 0
sase bead ready
sase bead doctor
```

Confirm the non-closed task beads are exactly `sase-cf`, `sase-d1`, `sase-di`, `sase-dx`, `sase-dy`, and `sase-e1`, plus
whatever bead Step 8 produced. Confirm `sase bead doctor` reports no new problems.

## Acceptance criteria

- `just _lint-symvision`, `sase validate`, and `just test-visual` were re-run and their real results are quoted in the
  close reasons; `just test` was run and its outcome classified per Step 2.
- Fourteen beads are closed: `sase-dj`, `sase-dk`, `sase-dq`, `sase-ds`, `sase-dv`, and `sase-dm` (Symvision); `sase-dt`
  and `sase-e0` (SDD validation); `sase-do`, `sase-dp`, `sase-du`, `sase-dl`, and `sase-dw` (ACE tests/goldens);
  `sase-e2` (flake duplicate).
- Each close carries `-R superseded` or `-R canceled` and a `--reason` that names either the surviving bead or the
  specific gate output proving non-reproducibility. No close uses `-R done`.
- Every surviving duplicate target (`sase-dk`, `sase-ds`, `sase-dw`, `sase-dy`) has a note recording the absorbed
  reports before its own disposition.
- `sase-cf`, `sase-d1`, `sase-di`, `sase-dx`, `sase-dy`, and `sase-e1` are untouched apart from the consolidation note
  and size update on `sase-dy`.
- `sase bead doctor` is clean, and the only bead created is the Step 8 `at_prefix` flake (or nothing, if
  `/sase_new_task` resolves it to an existing bead).
- No files under `src/`, `tests/`, or `sase/memory/` were modified.

## Out of scope

- Fixing any of the flakes that remain open (`sase-cf`, `sase-dx`, `sase-dy`, `sase-e1`). This plan only triages.
- Touching plan beads, epic beads, or phase beads. Task beads only.
- Adding Symvision pragmas, editing goldens, or changing production code to make a gate pass. If a gate fails, the
  corresponding bead stays open.

## Risks

- **`master` moves between plan approval and execution.** Mitigated by re-verifying every gate in Step 2 and by the Step
  1 instruction to skip already-closed beads.
- **A "stale" report is actually load-sensitive and will recur**, as happened with `sase-cf`. Mitigated by keeping every
  flake bead open and by the fact that `-R canceled` closes are cheap to reverse: a future reporter's `+1` promotes a
  closed task straight back to `ready`.
- **Evidence loss when closing duplicates.** Mitigated by Step 3 running before any close.
