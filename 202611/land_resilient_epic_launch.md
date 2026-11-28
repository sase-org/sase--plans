---
tier: tale
size: medium
title: Finish landing sase-1vt (resilient epic launches)
goal: "The resilient-epic-launch work is correct end to end. Integration errors name
  their real cause and are not all labeled as bead-ID collisions. The conflict replay
  honors the bead-store write sandbox. The canonical bead-work docs describe the new
  launch behavior. Epic sase-1vt is closed and its plan is marked done.

  "
proposed_by: bbugyi200.athena.sase-1vt.land
proposed_with: claude/opus
bead: sase-1vt
status: done
---

- **PARENT:**
  [202611/resilient_epic_launch.md](https://github.com/sase-org/sase--plans/blob/main/202611/resilient_epic_launch.md)
- **BEAD:**
  [sase-1vt](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vt/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1vt.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1vt.land.md)
- **COMMITS:**
  - [a8c75d4](https://github.com/sase-org/sase--plans/commit/a8c75d45dc5f95630095bbc4106170ca221eb2a8)
    — chore(plans): mark resilient_epic_launch plan done

# Plan: Finish landing epic sase-1vt

Epic `sase-1vt` ("Resilient epic launches under concurrent bead-ID minting", plan
`plan:202611/resilient_epic_launch.md`) has all five phases closed. The land review
checked the Rust merge rule (sase-core `7e857edd`, pinned in `sase-core-revision.txt`),
the counter floor, the pre-mint refresh, the linked-epic identity check, the conflict
replay, and the three-clone regression. All of them match the plan, and the epic's
focused test modules pass (59 tests). The review also found four gaps caused by the
epic. This tale fixes them and then performs the epic closeout. No other agent will
resume this landing, so the closeout below is mandatory.

Governing decisions: `decisions:rust-core-required` (the integration-error summary and
the replay orchestration are Python SDD/launch glue and stay in Python; nothing here
changes stream-merge semantics in `sase_core`), `decisions:check-full-is-explicit`
(verify with `just check`, never `just check-full`), and
`decisions:host-owned-completion` (no manual commits; finish through the declaration).

The land agent already finished the follow-up triage and recorded it on `sase-1vt`, so
do not re-triage:

- No phase recorded a `PROPOSED FOLLOW-UP:` note.
- The plan's out-of-scope plans-clone conflict recurred and is filed as `sase-1vu`.
- Task-bead minting stays out of scope by design. The `cli_work_from_plan_launch.py`
  size is owned by the toobig_split routine. Slow artifact-link adds are already tracked
  by `sase-1vp`.

## 1. Cause-first integration errors name the real cause

**Bug.** `_cause_first_integration_error` in
`src/sase/sdd/_repository_integration_repair.py` prefixes every failed semantic repair
with `bead-ID collision with concurrently published history could not be merged:`.
`repair_or_abort_rebase` runs for every SDD repository (plans, agents, beads). Its
failure details include `non-bead conflicts remain: <paths>`, artifact-link,
plan-header, and revision-pin resolver messages, `not inside a git repository`, and any
exception text. So a plans-store archive conflict now reads like this:

```text
bead-ID collision with concurrently published history could not be merged: non-bead conflicts remain: 202611/ctrl_jk_section_map.md; git rebase failed: ...
```

That is a live case: the bob-cli primary plans clone hits it every AXE checks run (see
`sase-1vu`).

**Fix.**

- Keep the cause-first ordering and the `_strip_git_noise` cleanup.
- Choose the leading label from the semantic detail:
  - When the detail is a bead-ID collision (it matches `duplicate issue_created event`
    or `duplicate bead`, the texts the Rust merge and the bead resolver emit), keep the
    existing
    `bead-ID collision with concurrently published history could not be merged:` label.
  - Otherwise use a neutral label such as
    `conflict with concurrently published history could not be merged:`. Keep the detail
    itself, for example `non-bead conflicts remain: <paths>`, right after the label.
- Do not change any `SddIntegrationStatus` value.
- Keep the replay trigger in `publish_epic_graph_before_launch_result` keyed only on
  `ABORTED_UNSUPPORTED_CONFLICTS`.

**Tests.** Add them to a new focused module, for example
`tests/test_bead/test_integration_error_cause_first.py`.
`tests/test_bead/test_cli_work_from_plan_publication_flow.py` is already 640 lines.

- The existing incident-shaped case keeps the bead-ID label. Keep the current test in
  `test_cli_work_from_plan_publication_flow.py` passing.
- A `non-bead conflicts remain: 202611/x.md` detail leads with the neutral label, does
  not contain `bead-ID collision`, and still omits `hint:` and `Rebasing (n/m)`.
- A raised-exception detail (`semantic conflict resolution failed: boom`) without a
  duplicate-creation message also gets the neutral label.

## 2. Replay honors the bead-store write sandbox

**Gap.** The epic plan's replay step says:

> Honor the bead-store write sandbox (`assert_bead_store_write_sandboxed`) and the
> workspace ownership guards. If they refuse, fail without resetting.

`_replay_publication_conflict` in `src/sase/bead/cli_work_from_plan_launch.py` calls
`authorize_store_mutation`, but it never calls
`sase.core.state_write_guard.assert_bead_store_write_sandboxed` before its
`git reset --hard`.

**Fix.**

- Next to the ownership guard, before taking the store write lock, call
  `assert_bead_store_write_sandboxed(beads_dir, operation="publication conflict replay")`.
  Use a lazy import, matching the module's style.
- Route any exception it raises through the existing `_refuse(...)`, with a reason such
  as `the bead-store write sandbox refused the reset: <exc>`. The refusal then keeps the
  cause-first message, the manual recovery steps, and `sase doctor -v`, and nothing is
  reset.

**Test.** Use a new small module, or the new module from step 1 if it fits. Start from
the setup in `test_publication_conflict_replay_refuses_foreign_unpublished_commit` in
`test_cli_work_from_plan_publication_flow.py`, or call `_replay_publication_conflict`
directly against a real two-clone store.

- Monkeypatch `sase.core.state_write_guard.assert_bead_store_write_sandboxed` to raise.
- Assert that `EpicFromPlanError` carries the sandbox reason, that no
  `refs/sase/recovery/` ref was created, and that the beads clone `HEAD` is unchanged.

## 3. Canonical bead-work docs describe the new launch behavior

Before editing, read `docs_writing.md` with `/sase_memory_read`.

**Gap.** The epic added a short section to `docs/reference/work-units/cli-workflows.md`
that links to [Starting bead work](bead-work.md#sase-bead-work-target) "for the full
launch and recovery behavior". The plan-file part of
`docs/reference/work-units/bead-work.md` (the numbered "Plan-file mode … It:" list and
the paragraph after it) was never updated, and it now contradicts the code:

- It says "A publication failure after the checkpoint preserves the linked, preassigned
  epic as the safe retry point". That no longer holds for an unmergeable publication
  conflict with no runner spawned, which is discarded and re-minted.
- It says a linked `bead_id` that is missing "fails with instructions". In the code, a
  missing bead is replaced with a fresh epic. Only a missing bead store, or a non-epic
  bead, fails.
- Step 4 ("Resumes the linked epic when the archived plan already has a valid
  `bead_id`") does not say that the epic's `design` must name this plan.

**Fix.** Update that list and paragraph so they match the code:

1. **Pre-mint refresh.** A remote-backed bead store is refreshed before the linked epic
   is resolved and before each mint attempt. Transient failures (remote unavailable,
   timeout, busy lock) warn and continue. Any other integration failure stops before
   creating beads, with a resume command.
2. **Linked-epic identity.** A linked epic resumes only when it exists and its `design`
   names this plan, or when it has no `design`. An epic that names another plan, or a
   missing bead, is replaced with a new epic, and the launch renders a warning. A
   missing bead store or a non-epic bead still fails.
3. **Publication conflict replay.** When graph publication cannot merge with
   concurrently published history and no agents have spawned, plan-file mode:
   - restores the plan link;
   - preserves the unpublished graph under `refs/sase/recovery/…`;
   - resets the beads clone to upstream;
   - re-mints, within the three creation attempts.

   When a safety check refuses the replay, it prints manual recovery steps and
   `sase doctor -v`. Every other publication failure still preserves the checkpoint as
   the retry point.

Keep `cli-workflows.md` as the short summary that links to `bead-work.md`, and trim
anything there that would now be duplicated. Run `just fix` if a generated reference
changes.

## 4. One beads-repo HEAD helper

Phase 3 added `_beads_repo_head_sha` in `src/sase/bead/cli_work_from_plan_premint.py`.
Phase 4 then added `bead_store_head_sha` in `src/sase/bead/_sync_git.py`, re-exported
from `sase.bead.sync`. The two are behaviorally identical: both resolve the git root
with `find_git_root`, then run `rev-parse HEAD`.

- Delete `_beads_repo_head_sha` and use `sase.bead.sync.bead_store_head_sha` in
  `refresh_store_before_epic_mint`.
- Update `test_pre_mint_refresh_records_launch_base` in
  `tests/test_bead/test_cli_work_from_plan_premint.py` to patch the helper where it is
  looked up. If the import is lazy inside the function, that is
  `sase.bead.sync.bead_store_head_sha`.

## 5. Verify

- Run the focused modules: the new test modules,
  `tests/test_bead/test_cli_work_from_plan_publication_flow.py`,
  `test_cli_work_epic_launch.py`, `test_cli_work_from_plan_premint.py`,
  `test_epic_launch_collision_chain.py`, `test_conflict_resolver_reincarnation.py`, and
  `test_conflict_resolver_counter.py`. Also run any `tests/sdd` tests that assert on
  integration error text; find them with
  `grep -rn "semantic conflict resolution failed\|non-bead conflicts remain" tests`.
- Then read `lint_and_test.md` with `/sase_memory_read` and run `just check` as it
  directs. Do not run `just check-full`.

## 6. Close out epic sase-1vt (final step; do it in this same turn)

This is the epic's landing closeout. Do not wait for this tale's commit, SHA, push, or
CI before doing it.

1. Run `sase bead epic-symbols sase-1vt`. At land review it reported
   `No --epic-symbol entries for sase-1vt.` If an entry has appeared since, resolve it
   per the Symvision epic-whitelist policy: read `symvision.md` with
   `/sase_memory_read`, then wire the symbol up, privatize it, add a non-test pragma, or
   delete it. Do not re-key an entry to a closed bead.
2. Close the epic. The note must summarize what was verified:

   ```bash
   sase bead close sase-1vt --note "Land verified: sase-core 7e857edd published-ID collision rule (pinned), counter floor past every stream/relocation ID, pre-mint refresh with transient/conflict policy plus linked-epic identity check, conflict replay with recovery ref, bounded attempts and refusal recovery, three-clone regression and docs. Landing tale fixed: neutral cause-first label for non-bead semantic failures, sandbox guard before the replay reset, bead-work.md plan-file behavior, single bead_store_head_sha helper. Integration since start (8c77e6fa64, 1b461588eb, f6dd22d4dd, sase-core 34461b7f) needed no changes. Follow-ups: sase-1vu filed; other proposals declined (see land triage note)."
   ```

   Adjust the note if anything above changed during this tale.
   - Never use `--force` just to make the close succeed.
   - If the close is rejected for leftover `--epic-symbol` entries, clean them up per
     step 1 and close again.
   - If it is rejected for unfinished phases, stop and report. At land review, all five
     phases were closed.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set the epic plan file's frontmatter to `status: done` (it is currently
   `status: wip`). The file is the PLAN path printed by
   `sase bead read sase-1vt -r "Need the epic plan path for closeout"`
   (`202611/resilient_epic_launch.md` in the plans sidecar). Open the plans repo with
   `/sase_repo` if you need a writable path. Change only the `status` field.
5. `sase-1vt` has no `parent_bead`, so there is no parent landing to continue. Finish
   normally.
