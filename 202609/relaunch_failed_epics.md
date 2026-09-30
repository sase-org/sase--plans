---
tier: epic
title: Recover and relaunch the five epics broken by the pinned-sibling commit regression
goal: 'Epics sase-1d5, sase-1cx, sase-1cj.12, sase-1co, and sase-1ck are running again
  from a correct bead state. The host finalizer can commit sase-core changes for bead-assigned
  agents again. No verified-but-unlanded work from last night''s failed runs is lost.

  '
phases:
- id: pinned-sibling-bead-action
  title: Pass -B keep for revision-pinned sibling stitches
  depends_on: []
  size: small
  description: 'pinned-sibling-bead-action: stop commit_dispatch from dropping bead_action
    for revision-pinned siblings (downgrade to keep instead), fix the test that enshrined
    the bug, and prove keep passes the Rust bead-action policy.'
- id: salvage
  title: Preserve each failed run's unlanded diff on its bead
  depends_on: []
  size: medium
  description: 'salvage: without touching the five pinned workspaces, export each
    failed run''s uncommitted sase and sase-core changes (including untracked files)
    as verified patches, and attach them with base SHAs and intended commit messages
    to sase-1d5.1, sase-1cx.1, sase-1cj.12.1, sase-1co, and sase-1ck.'
- id: relaunch
  title: Make the fix live, reopen the early-closed beads, and relaunch
  depends_on:
  - pinned-sibling-bead-action
  - salvage
  size: small
  description: 'relaunch: make sure the host install runs the fixed finalizer, reopen
    the five beads that closed before their work landed, dry-run and then run sase
    bead work -Y for each epic (sase-1d5 waits on bead sase-1ck), and verify that
    the replacement agents exist.'
proposed_by: bbugyi200.athena.0ua
create_time: 2026-09-30 06:21:06
status: wip
bead_id: sase-1d6
---

- **PROMPT:** [prompts/202609/relaunch_failed_epics.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/relaunch_failed_epics.md)
- **BEAD:** [sase-1d6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1d6/README.md)

# Plan: Recover and relaunch the five epics broken by the pinned-sibling commit regression

## Review findings

Last night, all five failures ended the same way. The agent finished and verified its
work and submitted a valid final declaration. Then the host commit finalizer failed on
the **sase-core** stitch:

```
sase stitch create failed for sase-core: ❌ missing_bead_action: bead_action is required
when a bead is assigned; use -B keep or -B close
```

A second attempt was refused as "identical inputs". The run then failed. **No commit
landed in sase or sase-core for any of the five runs.** Neither repo's `origin/master`
has a commit from these agents. In every case the agent had already closed its bead
(`sase bead close`) during its turn, before the host tried to commit.

**Root cause (still on master):**
`c257a3f220 feat(finalizer): add revision_pin for linked repos…` (2026-09-29 17:48)
added this code to `src/sase/finalizers/commit_dispatch.py` (around lines 276-279):

```python
if repo.kind != "main" and repo.name in revision_pins:
    bead_action = None
```

sase-core is revision-pinned through `sase-core-revision.txt`, so every sase-core stitch
from a bead-assigned agent now runs `sase stitch create` without `-B`. The sase-core
bead-action policy (`crates/sase_core/src/bead_action/`) rejects that with
`missing_bead_action`. It also rejects `-B close` on a non-primary repository
(`close_requires_primary_repository`). So `keep` is the only valid sibling action. The
main stitch still carries the declared `close`. The unit test
`tests/test_commit_revision_pin_dispatch.py::test_dispatch_commits_pinned_sibling_first_and_follows_pin`
mocks the stitch runner and asserts `bead_actions["sase-core"] is None`, which is how
the bug shipped.

| Epic        | Failed agent (run / workspace claim)                                        | Bead closed too early | Unlanded work (base SHA at failure)                                                                                                                                                                                                                                                                                                             |
| ----------- | --------------------------------------------------------------------------- | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sase-1d5    | `sase-1d5.1` phase `core_audience`, large (`ace(run)-260930_015851`)        | `sase-1d5.1`          | sase-core only: about 20 paths on `a354a8a`, including untracked `note_attachment/audience.rs`, `public_objects.rs`, `scanner.rs`, and `Cargo.lock`/`Cargo.toml`. Approved phase plan `plan:202609/core_attachment_audience.md`                                                                                                                 |
| sase-1cx    | `sase-1cx.1` phase `core-detach-join`, large (`ace(run)-260929_203400`)     | `sase-1cx.1`          | sase-core only: about 25 `tool_run` paths on `1e51ff3`. Approved phase plan `plan:202609/core_detach_join.md`                                                                                                                                                                                                                                   |
| sase-1cj.12 | `sase-1cj.12.1` phase `core-correctness`, medium (`ace(run)-260929_183051`) | `sase-1cj.12.1`       | sase-core only: 7 `prompt_prediction` paths on `1e51ff3`. Message: `fix(prompt-prediction): block structural tails, count support once, real origin inventory`                                                                                                                                                                                  |
| sase-1co    | `sase-1co.land--1`, land (`ace(run)-260929_195614`)                         | epic `sase-1co`       | sase: 6 paths on `859140f025` (`feat(xprompt): alternation parity after literal { and adjacent paren openers`). sase-core: 4 paths on `1e51ff3` (`feat(alternation): scan openers after literal { and adjacent paren openers`). The plan-done mark was never committed either. Lander tale `plan:202609/midword_alternation_scanner_parity.md`  |
| sase-1ck    | `sase-1ck.land--3`, land (`ace(run)-260930_023625`)                         | epic `sase-1ck`       | sase: 37 paths on `b5d3021f5d` (`feat(bead-attachments): land note-attachment items 1-7 fixes, docs, and proving tests`). sase-core: 3 paths on `a354a8a` (`fix(core-attachments): same-text attachment reuse and manifest fidelity`). The plan-done mark was never committed either. Lander tale `plan:202609/finish_bead_note_attachments.md` |

**Verdict under the user's rule:** in all five epics the failing agent closed its bead
before its changes landed. So all five beads must be **reopened** before relaunch:

- the phase beads `sase-1d5.1`, `sase-1cx.1`, and `sase-1cj.12.1`;
- the epic beads `sase-1co` and `sase-1ck`, whose land agents closed them.

Relaunching without reopening would start the downstream phases, or finish nothing, with
the core work missing. A closed epic would also make the relaunched land's final `close`
ineligible.

**Why the relaunch cannot happen right away:**

1. The bug is still live. Every relaunched agent would redo sase-core work and fail
   again at commit time. Each runner imports the finalizer from the primary sase
   checkout (editable uv-tool install). That checkout only fast-forwards on
   `sase update`, which in practice has run every few hours. So the fix must be on
   master **and** in the host install before any relaunch.
2. `sase bead work <epic> -Y` force-reuses agent names. The wipe in
   `src/sase/agent/names/_wipe.py` deletes the old owner's artifact directories,
   notifications, and **workspace claims**. The five failed runs' workspaces are still
   claimed and pinned, and they hold the only copy of the unlanded diffs. Once released,
   the next claimant resets them. The diffs must be preserved first.
3. The `sase-1d5.1` and `sase-1ck` land diffs both edit sase-core
   `note_attachment/manifest.rs` and its tests. `sase-1d5`'s later phases rework the
   same Python attachment modules that the `sase-1ck` land diff touches. So `sase-1d5`
   should wait until `sase-1ck` has re-landed.

Also, `just check` currently fails on master for everyone:

- the patch/stitch terminology audit (14 lines in sase-core `at_bearing_notes.jsonl`);
- symvision's `_kitty_graphics_support` private import.

Both fixes are inside the unlanded `sase-1ck` land diff. Until `sase-1ck` re-lands,
treat those two failures as pre-existing and do not fix them in this epic.

## Phase pinned-sibling-bead-action

Fix the regression in `src/sase/finalizers/commit_dispatch.py`:

- For a non-main repo that is in `revision_pins`, replace `bead_action = None` with a
  downgrade to `"keep"` whenever the decision carries a bead action.
  - When the decision has none (no assigned bead), leave it `None`.
  - Update the comment: pinned siblings land first with `-B keep`, and only the main
    stitch applies the declared `close`.
- The fingerprint for identical retries (`stitch_attempt_input_fields`) must use the
  value that is actually passed.
- Check the resume, checkpoint-recovery, and repair paths:
  - `commit_unpushed_resume.py`
  - `commit_checkpoint_recovery.py`
  - `commit_repair_stitch.py`
  - `commit_repair.py`

  Confirm none of them can re-issue a pinned-sibling stitch with no `-B` (or `-B close`)
  for a bead-assigned run. Apply the same rule wherever one could.

Tests (in the existing `tests/test_commit_revision_pin*.py` split files):

- Change the end-to-end dispatch test to assert `bead_actions["sase-core"] == "keep"`,
  while main still gets `"close"`.
- Add a sibling-only case: a bead is assigned, only the pinned sibling is dirty, and it
  declares `keep` (the `sase-1cj.12.1` shape). Assert that the stitch gets `"keep"` and
  dispatch succeeds.
- Add a no-bead case: the decision has no bead fields. Assert that the sibling stitch
  still gets `None`.
- Add a test that runs the real policy through
  `sase.core.bead_action_facade.decide_bead_action`, if its request shape allows it. The
  test should show that a bead-assigned, non-primary repository is accepted with `keep`,
  and rejected when the action is missing. This way the mock-only gap cannot hide a
  regression again.

Verification:

- Read the `lint_and_test` reference memory before finishing.
- Run the focused finalizer tests.
- Run `sase tool run check`. Report the two pre-existing `sase-1ck` failures as
  pre-existing; do not fix them here.

This phase changes only the sase repo. Do not open or edit sase-core, and do not
relaunch anything.

## Phase salvage

Goal: every unlanded diff in the table above survives the relaunch as a bead-note
attachment that the relaunched agent can apply.

1. **Find each workspace** from the table's run ID. Use `sase workspace list` (column
   CLAIMED BY = `ace(run)-<id>`) and `sase workspace path <N>`.
   - Confirm that the dirty file sets still match the failed run's declaration. Use
     `sase final status <agent>`, or the run's `final_submission.json` where one exists.
   - Get the intended commit messages from that declaration. For sase-core, also check
     the preserved `.sase/finalizers/commit-message-*.txt` file named in the failure.
2. **Treat those workspaces as read-only.**
   - Do not run `git add`, `stash`, `reset`, `checkout`, `clean`, or `commit` in them.
   - Do not release or clean their claims. The relaunch phase's `sase bead work` wipe
     does that.
   - Never commit that work from this phase.
3. **Watch this turn's own finalizer context.** Opening another workspace's sase-core
   with `sase repo open sase-core -w <N>` makes that dirty tree show up as _this_ turn's
   commit obligation. This was observed while planning. With the regression fixed, the
   host would then commit another agent's work under this phase's name.
   - After you finish reading the other workspaces, run
     `sase repo open sase-core -r "<why>"` with no `-w`. That points the record back at
     this workspace's own clean checkout.
   - Confirm that `sase final context` shows no foreign sase-core obligation.
   - If one remains, defer it (`belongs_to_another_turn`). Never commit it.
4. **Export one patch per dirty repo per run:** `git diff --binary HEAD`, plus every
   untracked, non-ignored file as a `/dev/null` diff.
   - Record the base `HEAD` SHA.
   - Exclude `.sase/` and build outputs.
   - Verify each patch with `git apply --check` against a pristine checkout of its base
     SHA. Use a throwaway worktree or clone outside all managed workspaces.
   - Check that the patch's file list equals the run's declared paths plus any untracked
     files.
5. **Attach and document** with `sase bead note <bead> "<text> @<patch-path>"` (use
   `sase bead attach` if prose and files must be split). Target beads: `sase-1d5.1`,
   `sase-1cx.1`, `sase-1cj.12.1`, `sase-1co`, and `sase-1ck`. Start each note with
   `UNLANDED PRIOR ATTEMPT:` and state:
   - that the prior agent's completion note describes verified work that **never
     landed**, because of the finalizer regression above;
   - the attached patches, their repos, and their base SHAs;
   - the intended commit messages, and for large phases the approved phase plan
     reference;
   - for the two land beads, that the epic plan's done mark also never landed;
   - instructions to the relaunched agent:
     - apply the patches onto current `origin/master` with `git apply --3way`;
     - resolve conflicts (master has moved; for example,
       `refactor(attachments): split bead upload module into upload package` touches
       files in the `sase-1ck` diff);
     - re-run verification instead of trusting the old note;
     - reuse the approved plan instead of re-planning from scratch.

   If a closed bead refuses a note, record the attachment path and the note text as a
   note on this phase's own bead instead. The relaunch phase will post it after it
   reopens the bead.

6. Finish with a note on this phase's bead that lists, for each target bead, the note
   ordinal, the attachment names, the byte sizes, and the base SHAs.

## Phase relaunch

1. **Make the fix live.**
   - Confirm that the `pinned-sibling-bead-action` commit is on `origin/master`.
   - Check whether the primary checkout that the host runs contains it:
     `git merge-base --is-ancestor <fix-sha> HEAD` in `sase workspace path 0`.
   - If it does not, run `sase update` to fast-forward and reinstall. It may run long;
     hand it to `/sase_monitor` instead of waiting in the foreground.
   - Then confirm with `sase version` that the host `sase` version's git hash contains
     the fix.
   - **Do not relaunch anything until this holds.**
2. **Confirm the salvage.** `sase bead read` each of the five beads and confirm that its
   `UNLANDED PRIOR ATTEMPT:` note and attachment are present. Post any fallback notes
   that the salvage phase left on its own bead, after step 3 reopens the target.
3. **Reopen** with `sase bead open`:
   - `sase-1d5.1`, `sase-1cx.1`, `sase-1cj.12.1`, `sase-1co`, `sase-1ck`.

   Add a short note to each: `REOPENED:`, the reason, and the fix commit SHA.

4. **Dry-run each epic** with `sase bead work <epic> -n`. Check that:
   - the reopened phase is scheduled (for `sase-1co` and `sase-1ck`, only the land
     agent);
   - the only KILL/REMOVE targets are that epic's own WAITING or FAILED agents, such as
     the stuck `sase-1d5.3`-`.8`, `sase-1cx.3`-`.7`, `sase-1cj.12.3`-`.5`, and land
     waiters, plus the failed owners;
   - no RUNNING agent is a target.

   If anything else shows up, stop and record it instead of launching.

5. **Relaunch in this order:**

   ```bash
   sase bead work sase-1ck -Y
   sase bead work sase-1co -Y
   sase bead work sase-1cj.12 -Y
   sase bead work sase-1cx -Y
   sase bead work sase-1d5 -Y -w bead=sase-1ck
   ```

   `sase-1ck` goes first because its re-land clears the `just check` failures for
   everyone. `sase-1d5` waits on the `sase-1ck` bead because of the overlapping diffs.
   `sase-1cx`'s later phases may touch `src/sase/tool/routing.py` near the `sase-1ck`
   diff; ordinary rebase is enough there, so no wait is added.

6. **Verify.**
   - `sase agent list` shows fresh RUNNING or WAITING agents for every epic.
   - No stale WAITING agents or wait-check notifications remain for the old failed
     owners.
   - The old pinned workspaces' claims are released.

   Record the launched agent names on this phase's bead.

## Risks

- If anyone runs `sase bead work` on these epics before the salvage phase finishes, the
  unlanded diffs are lost. The salvage phase has no dependencies, so it starts right
  away.
- A salvaged patch may no longer apply cleanly to today's master. Each relaunched agent
  owns its own 3-way apply and re-verification. The patches are a head start, not a
  guarantee.
