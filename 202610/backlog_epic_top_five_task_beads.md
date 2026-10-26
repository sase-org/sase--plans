---
tier: epic
title: 'Task-bead backlog, epic 1: fix the five most important open task beads'
goal: 'The five most important open task beads in the sase project are resolved on
  master and closed with recorded evidence: sase-1of (Full CI visual-test red), sase-14r
  (the shard-timings table is about to turn Master Gate red), sase-1nw (the stuck
  bead read model burns CPU and disk on athena), sase-yo (closed-as-done beads strand
  unlanded work), and sase-1dp (agent transcripts reach the public sidecar unscanned).
  The land agent then starts the next backlog epic, five beads at a time, until no
  workable task bead remains.

  '
decisions:
  secret_scan:
    ask: Build the pre-publish secret scan for agent transcripts and prompt files
      (sase-1dp)?
    default: build
    choices:
      build: Withhold scanner hits, handle GH013, and gate a push-protection change
      cancel: No code change; sase-1dp is closed as canceled with your opt-out recorded
    why: 17 of 21,467 published sidecar files had scanner hits; new transcripts go
      public unscanned
    answer: build
  dormant_flakes:
    ask: Should the lander close flakes with zero failures in the last 30 days as
      canceled?
    default: close
    choices:
      close: Lander re-checks the 30-day store and closes still-quiet flakes; a later
        +1 reopens them
      keep: Dormant flakes stay open and compete for later epics like any other bead
    why: About 28 flakes show no failure in 1,037 runs, and +1 reopens any that recur
    answer: close
phases:
- id: stats-goldens
  title: Green Full CI's visual-test after the Statistics language breakdown (sase-1of)
  depends_on: []
  size: small
  description: 'stats-goldens: confirm the 7 drifting config_center_statistics PNG
    goldens reflect the intended language-breakdown change, regenerate only those,
    prove the visual check is clean, and close sase-1of.'
- id: shard-drift
  title: Refresh the shard-timings table and make the ratchet catch drift (sase-14r)
  depends_on: []
  size: medium
  description: 'shard-drift: find why the weekly shard-timings ratchet never refreshed
    the 23-day-old table, fix it, add a file-count drift trigger well below the 20%
    test limit, refresh tests/shard_timings.json, and close sase-14r.'
- id: read-model
  title: Make a crate-version mismatch converge in the bead read model (sase-1nw)
  depends_on: []
  size: large
  description: 'read-model: reproduce the stale crate_version read-model cache that
    every reader rebuilds and rolls back, fix the sase-core rebuild/commit path so
    it converges, move the sase core pin, and close sase-1nw.'
- id: stranded-close
  title: Never leave a bead closed as done when its turn's commit never lands (sase-yo)
  depends_on: []
  size: large
  description: 'stranded-close: when the host commit finalizer fails to land the work
    of a turn whose bead closed in that turn, reopen the bead with a durable STITCH
    FAILED note naming the stranded workspace, so epics stop advancing past unlanded
    phases; close sase-yo.'
- id: sidecar-secret-scan
  title: Scan agents-sidecar transcripts before publication and survive GH013 (sase-1dp)
  depends_on: []
  size: large
  description: 'sidecar-secret-scan: run the core attachment secret scanner over agent
    transcripts and prompt files before they reach the public sase--agents sidecar,
    withhold hits, add GH013 handling, propose push protection through a gate, and
    close sase-1dp (or close it canceled if the owner opted out).'
proposed_by: bbugyi200.athena.14m
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 14:14:57
status: wip
bead_id: sase-1oh
---

- **PROMPT:** [prompts/202610/backlog_epic_top_five_task_beads.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/backlog_epic_top_five_task_beads.md)
- **BEAD:** [sase-1oh](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1oh/README.md)

# Plan: Task-bead backlog, epic 1: the five most important open task beads

## Background

On 2026-10-26 the sase project had **555 open task beads**. A read-only triage (sase
master `60832648be`, sase-core `4121de03`) checked every one against the current code.
Seven helpers each took one slice and ran single test nodes, checked git history and
read the local failure records. Results:

- **156 beads were closed**, each with its evidence as a note: 143 verified fixed, 2
  obsolete, and 11 duplicates folded into their keepers.
- **Two new `ci` beads were filed** for red gates no bead tracked: `sase-1of` and
  `sase-1og`.
- **401 task beads remain open.** Verdicts for each, with importance, estimated size,
  evidence and shared-cause clusters, are in the artifact
  `file:explicit:accb34387e533e1c0cc7aca8`. Read it with
  `sase artifact read file:explicit:accb34387e533e1c0cc7aca8 "<why>"`. It is a snapshot:
  treat every row as a lead and re-verify before acting.

This epic owns exactly five of those beads, the most important by impact. The project
owner's standing rules for the backlog chain are:

- Each epic owns **at most five task beads** and has **at most five phases**.
- While workable task beads remain, the land agent starts the next epic under the same
  rules (see **Landing**).

## Global constraints for every phase

- **Own your bead.** Each phase names the task bead it owns. Read it first with
  `sase bead read <id> -r "<why>"`; its description, `task_type_fields`, notes and +1
  evidence hold the repro. Re-verify the defect still reproduces at your base before
  changing anything. If other work already fixed it, close it `done`, citing the fixing
  commit and your reproduction.
- **Close on evidence.** Close the owned bead with
  `sase bead close <id> --note "<evidence>"` and the default `done` resolution. Close it
  only after the fix is verified. The note names your phase bead, your verification
  commands and their results. The one exception is `secret_scan = cancel` (see that
  phase).
- **No shortcuts.** Never weaken an assertion, raise a timeout, cap or budget just to
  get green, add a symvision pragma or suppression, or `xfail`/skip a node.
- **Rust-core boundary.** Shared backend behavior belongs in sase-core's `sase_core`
  crate. Open it with `sase repo open sase-core` and read its `AGENTS.md`. When sase
  must build a newer core, move `sase-core-revision.txt` past that commit with
  `just ratchet-core-revision` (see `docs/rust_backend.md`). Never edit sase-core
  versions or changelogs.
  - Active epic `sase-1oe` (phase `sase-1oe.3`) is also moving master's core pin. Rebase
    before you ratchet, and re-ratchet if the pin moved under you.
- **Verification.** Run `sase tool run check` in every repo you change. Run targeted or
  repeated nodes directly with `.venv/bin/python -m pytest`. Use `/sase_monitor` for
  anything long-running; never end a turn to wait.
- **Human-only and outward-facing actions.**
  - Never run `just install`, `just install-dev`, raw `sudo`, or any GitHub repository
    settings change. Propose them through `/sase_gate` (and `/sase_sudo` for privileged
    commands).
  - Never print, note or commit a secret value.
- **Memory.** Read the notes your work touches before changing code:
  - `tui` before TUI or visual-snapshot work;
  - `sase_beads` before closing beads;
  - `lint_and_test` before finishing.

  Do not edit `sase/memory/**`. If a note becomes inaccurate, append a
  `PROPOSED FOLLOW-UP:` note to your phase bead.

- **Beads.** Phase workers never create beads. Record discovered work as
  `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own phase bead.

---

## Phase `stats-goldens`: green Full CI's visual-test (sase-1of)

**State at planning time.**

- **The failure.** Full CI run `40347322042` at `60832648be` fails only its
  `full / visual-test` job. `just fix-tui-screenshots` drift reports
  `created=0 updated=7 unchanged=1119 stale=0`.
- **Which goldens.** All seven are Statistics-pane goldens from
  `tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py`:
  - `projects`, `projects_narrow`, `help`;
  - the `projects_model`, `projects_repo`, `projects_patch` and `projects_single`
    scenes.
- **Last green.** Full CI at `7423fa8380` (2026-10-25).
- **Likely cause.** `ba59799ce9` ("add project language breakdown with rows card and
  repo LANGUAGE column"), with follow-ups `ce75abd76f` and `49b1b9088b`. They changed
  `src/sase/stats/render/*` without regenerating these goldens.

**Steps.**

1. **Check whether it is already fixed.** Look at the newest Full CI runs and the git
   log of `tests/ace/tui/visual/snapshots/png/config_center_statistics_*`. Phase
   `sase-1oe.4` of an active epic is also chasing a green master. If a newer Full CI
   `visual-test` is already green on a SHA that contains a golden refresh, close
   `sase-1of` `done` with that run as evidence.
2. **Reproduce.** Run
   `TZ=UTC .venv/bin/python -m pytest -m visual tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py`.
   Then inspect the actual, expected and diff images the visual harness writes (the
   `tui` memory note says where).
3. **Judge the drift.** Every changed region must be the intended change: the language
   breakdown card, the repo LANGUAGE column, the Swift labels or the uncounted-files
   surfacing. If anything looks like a layout regression (clipping, overlap, misaligned
   columns or truncated legends), fix the renderer instead of accepting the golden.
4. **Regenerate** only the affected goldens through the repo's visual-maintenance path
   (`just fix-tui-screenshots`, run under UTC as the `tui` note prescribes). Do not
   touch unrelated goldens.
5. **Verify.**
   - The Statistics visual file passes.
   - `just fix-tui-screenshots --check` reports no updates.
   - `sase tool run check` passes.
6. **Close `sase-1of`.** The next scheduled Full CI run (every 2 hours) is the lander's
   final proof.

## Phase `shard-drift`: refresh the shard-timings table and make the ratchet catch drift (sase-14r)

**State at planning time.**

- **The table.** `tests/shard_timings.json` has `measured_file_count` 4757 and
  `generated_at` 2026-10-03 (host athena).
- **The tree.** It now holds about 5,600 test files (the triager counted 5,590
  discovered; `git ls-files` shows 5,630). That is roughly 18% drift against the hard
  20% limit in
  `tests/test_test_shards.py::test_committed_table_measured_count_has_not_drifted_too_far`.
- **The deadline.** At about 36 new files a day, that node turns every scoped lane and
  Master Gate red within days.
- **History.** This bead has already gone red once: there were eight +1s in September,
  followed by two manual refreshes.
- **The ratchet.** `.github/workflows/shard-timings-ratchet.yml` runs weekly
  (`43 4 * * 1`). It downloads the newest _successful_ Full CI `shard-timings` artifact
  and refreshes the table when the shard assignment would change or the table is older
  than 14 days, then opens a PR on branch `shard-timings-ratchet-<stamp>`. The triager
  saw it report success without refreshing a 23-day-old table.

**Steps.**

1. **Diagnose the ratchet.** Find why it did not refresh. Use
   `gh run list --workflow shard-timings-ratchet.yml` and the run logs, and check for an
   existing `shard-timings-ratchet-*` branch or PR. Candidates:
   - the "branch already exists; a refresh is already proposed" short-circuit stuck on a
     PR nobody merged;
   - a stale or missing artifact on the newest successful Full CI run;
   - the `--max-age 14` check in `tools/refresh_shard_timings` not firing.

   Fix the actual cause, and add a regression test for the tool logic you change.

2. **Add a drift trigger.** The ratchet must also refresh when the discovered test-file
   count drifts at least 10% from `measured_file_count`. Implement it in
   `tools/refresh_shard_timings` so the workflow and local runs share one rule, and test
   it.
3. **Refresh the committed table now.** Prefer a CI-measured payload: the newest
   successful Full CI `shard-timings` artifact via
   `tools/refresh_shard_timings --from-payload`. Use `just refresh-shard-timings` on a
   quiet host only if no payload is usable, and say which you used in the close note.
4. **Verify.**
   - `tests/test_test_shards.py` passes.
   - The new trigger tests pass.
   - `sase tool run check` passes.
5. **Close `sase-14r`.**

## Phase `read-model`: make a crate-version mismatch converge in the bead read model (sase-1nw)

**State at planning time.**

- **The stuck cache.** On athena, the primary beads clone's read-model cache
  (`<beads clone>/.git/sase/bead-read-model/bead-read-model-<key>.sqlite`) still has
  `crate_version=0.40.0` and a frontier and mtime of 2026-10-16. The installed
  `sase_core_rs` is 0.42.6.
- **The cost.** Every reader rebuilds into the WAL and rolls back. A parked runner wrote
  about 67 MB/min, nearly all of it cancelled. On 2026-10-24, about 20 runners together
  used about 2.2 cores and about 20 MB/s, during a swap and jbd2 stall.
- **The code.** sase-core `crates/sase_core/src/bead/read_model/store.rs`:
  `open_and_serve` (about L495), `rebuild_from_replay` (about L992), the crate-version
  check (about L1196), and `drop_cache_file` / `invalidate_cache_in_place`.
- **The bead's hypothesis (unverified).** After a version mismatch the rebuild loses the
  generation CAS or hits a write fault and rolls back. The 0.40.0 meta is never
  overwritten, so every reader repeats the cycle.

**Steps.**

1. **Reproduce in a sase-core test.** Seed a cache whose meta carries an old
   `crate_version`, then drive the normal read path, including concurrent readers. The
   test must show the first reader commits a fresh rebuild and later reads are cache
   hits.
   - If a synthetic fixture does not reproduce it, copy athena's stuck cache file
     read-only into a temp directory and reproduce against the copy.
   - Never modify the live cache file.
2. **Fix the rebuild/commit path** so a version mismatch converges in one rebuild under
   contention. Keep the generation CAS's protection against lost updates.
3. **Integrate.**
   - Run `sase tool run check` in sase-core.
   - Move sase's `sase-core-revision.txt` pin past the fix (rebase first because of
     `sase-1oe`), then run `sase tool run check` in sase.
   - Add or adjust a sase-side test only if sase's binding usage changes.
4. **Live acceptance needs a human install.** Agents cannot reinstall the global sase on
   athena (decision `global-install-is-human-only`). Raise a `/sase_gate` that proposes
   the reinstall once the fix is on master. Its follow-up checks:
   - the cache meta's `crate_version` updates after the first read;
   - a parked runner no longer churns the `-wal` file every minute.
5. **Close `sase-1nw`** on the regression-test evidence, and name the gate in the note.

## Phase `stranded-close`: never leave a bead closed as done when its turn's commit never lands (sase-yo)

**State at planning time.**

- **What happened (2026-09-08).** Phase beads closed `done` inside their turns. The host
  commit finalizer's before-commit hook (`just fix`) then failed (ENOSPC). Nothing
  reopened or marked the beads, and the epic advanced. The verified work was left
  uncommitted in an ephemeral workspace; it was lost once and recovered by hand once.
- **What has changed since.** The finalizer system has changed a lot:
  - `src/sase/finalizers/commit_*.py`, including the repair, checkpoint-recovery and
    repair-handoff modules;
  - prepared completion in `src/sase/monitor/host_completion_state.py`;
  - the stranded-work classifier `5d982d2bb9`
    (`docs/reference/work-units/bead-lifecycles.md` "Stranded beads").

  The triager found that the classifier covers only `in_progress` beads, and that
  nothing reopens or durably marks a bead that closed in a turn whose commit never
  landed.

- **The prompts.** `bd/work_phase_bead` and `bd/work_task` in
  `src/sase/default_config.yml` still tell agents to run `sase bead close` inside their
  turn.
- **Merged duplicate.** `sase-193` (the failed-declaration-recovery variant of the same
  gap) was closed into this bead.
- **Binding decisions.** `host-owned-completion`, `single-turn-agents` and
  `receipts-prove-before-they-skip` apply. Agents close in-turn and the host commits
  afterwards, so the repair belongs on the failure side, never on the close side. Read
  them with `sase memory read decisions:<keyword> -r "<why>"`.

**Steps.**

1. **Re-verify current behavior.** Write a failing test, at the finalizer level or as a
   fakey e2e, that covers both triggers:
   - a turn that closes its bead and then fails its before-commit hook;
   - a turn whose declaration recovery fails.

   If today's host already reopens or durably marks the bead and surfaces the stranded
   workspace, close `sase-yo` `done` with that test as evidence.

2. **Otherwise implement the host-side reaction.** When the commit finalizer ends
   without landing the declared commit of a turn that closed beads during that turn, the
   host:
   - reopens each such bead through the bead store mutation path, so epic runners and
     land agents stop treating the phase as landed;
   - appends a `STITCH FAILED:` note naming the workspace, the preserved commit-message
     path and the tail of the hook output.

   Make sure the stranded-work facts, the `sase doctor` WARNING and ACE's stranded flag
   see the reopened bead. Do not change the agent-side close instructions.

3. **Verify.** Run the new regression tests, the existing finalizer and stranded-work
   tests, and `sase tool run check`.
4. **Close `sase-yo`.**

## Phase `sidecar-secret-scan`: scan agents-sidecar transcripts before publication (sase-1dp)

> [!decision] secret_scan = cancel Make no code change. Close `sase-1dp` with
> `sase bead close sase-1dp -R canceled -r "Owner opted out via epic decision secret_scan"`
> and a note naming this epic. Then finish the phase.

> [!decision] secret_scan = build Do the steps below.

**State at planning time.**

- **The scanner.** The core attachment scanner (sase-core `note_attachment/scanner.rs`,
  rules v1: known values, credential patterns and env dumps) is reached through the
  `attachment_scan_file` binding. Only `src/sase/bead/attachments/audience.py:276` uses
  it.
- **GH013 handling.** It exists only in
  `src/sase/bead/attachments/upload/secret_scan.py`, as the outbox's permanent `blocked`
  state plus a hidden-clone reset.
- **The gap.** The agents-sidecar publishers (`src/sase/agents_sync/`) push transcripts
  and prompt files to the public `sase--agents` sidecar with no scan.
- **Evidence.** A count-only scan found 17 of 21,467 published files with hits (12
  known-value, 2 credential-pattern, 3 env-dump).

**Steps.**

1. **Scan before publishing.** Scan every transcript and prompt file before it is staged
   for publication. On a hit, withhold the file and keep a durable local record (path
   and rule ids, never values) that `sase doctor` surfaces.
   - Keep scanning logic in sase-core. Add a core API only if the existing binding is
     not enough.
2. **Handle GH013.** Teach the publishers to recognize "push declined due to repository
   rule violations", modeled on the attachments outbox. Mark the offending item
   permanently blocked, reset the hidden clone, and keep publishing everything else, so
   that push protection can never stall publication.
3. **Already-published hits.** Run the scanner over a local checkout of the published
   sidecar. Attach a **private** report (`-K`; paths and rule ids only, never values) to
   your phase bead. Raise a `/sase_gate` asking the owner whether to rotate credentials
   and purge history. Do not rewrite sidecar history yourself.
4. **Push protection.** Enable GitHub secret scanning and push protection on the
   existing public sidecars **only by proposing it through `/sase_gate`**. Use
   sase-github's `sdd_secret_scanning` path or the `gh api` command in sase-github
   `docs/configuration.md`. Raise this gate only after steps 1–2 are on master.
5. **Verify.** Run unit tests for withhold-on-hit, clean-file publication and GH013
   classification, then `sase tool run check`.
6. **Close `sase-1dp`.**

---

## Landing (epic land agent)

Do not close the epic bead until steps 1–4 pass on the landed master tip.

1. **Status audit.** Run
   `sase bead read sase-1of sase-14r sase-1nw sase-yo sase-1dp -r "<why>"`. Each must be
   closed `done` with a note citing verification evidence. The exception is `sase-1dp`
   under `secret_scan = cancel`, which is closed `canceled`.
2. **Re-verify on master** with quick checks:
   - **sase-1of:** a scheduled Full CI run on a SHA that contains the golden refresh has
     a green `visual-test` job. Wait for it with `/sase_monitor`.
   - **sase-14r:** `tests/test_test_shards.py` passes, and the drift-trigger tests pass.
   - **sase-1nw:** the sase-core regression test passes at the pinned revision.
   - **sase-yo:** the stranded-close regression tests pass.
   - **sase-1dp** (`build` only): the withhold and GH013 tests pass.
3. **If a bead is open or its check fails,** reopen it and its phase bead with
   `sase bead open`. Finish small gaps in this turn, or rerun
   `sase bead work <epic-id>`. Never close one of the five as `canceled` or
   `superseded`, except `sase-1dp` under `secret_scan = cancel`.
4. **Wrap up.** Run `sase tool run check`. Turn warranted `PROPOSED FOLLOW-UP:` notes
   into task beads through `/sase_new_task`. Then close the epic bead.
5. **Start the next backlog epic** while workable task beads remain. "Workable"
   excludes:
   - snoozed beads;
   - beads owned by another active epic;
   - `in_progress` beads with a live holder;
   - flag beads whose removal condition is not yet met.

   Do this as the last action of the landing turn: `sase plan propose` ends the turn.
   1. **Re-triage.** List open task beads with `sase bead list -t task -n 0`. Use the
      verdict artifact `file:explicit:accb34387e533e1c0cc7aca8` as leads. Close every
      bead you can **verify** is a duplicate (resolution `superseded`, naming the
      keeper) or obsolete or already fixed (`canceled` or `done`), each with evidence.
      Add a pointer note to the keeper. Fan out read-only helpers if the set is large.
   2. > [!decision] dormant_flakes = close Re-check the selection-health store
      > (`~/.sase/test-selection/<project>`, 30-day window) for each flake bead marked
      > dormant in the artifact, and for UNSURE `sase-n6`, `sase-nf` and `sase-oh`.
      > Close each one that still shows zero failures as `canceled`, with a "not
      > reproducing in N runs over 30 days; +1 to reopen" note.

      > [!decision] dormant_flakes = keep Leave dormant flakes open; they compete like
      > any other bead.

   3. **Rank and plan.** Rank the remaining beads by impact: red gates and release
      blockers, then data loss or security, then host health, then recurring CI flakes,
      then everything else. Write the next epic with `/sase_plan`. It owns the **five**
      most important beads in **at most five phases**, and one phase may own several
      related beads.
      - Leads from this triage, in rough order: `sase-1og` (xsmall; the check-full
        flake-baseline gate exits 2), `sase-18t`, `sase-1a7`, `sase-12f`, `sase-1cb`,
        `sase-1cs` with `sase-vr`, `sase-1nh`, `sase-1mu`, `sase-13a`, `sase-120`,
        `sase-o7`, `sase-j0`.
   4. **Decisions.** Keep them minimal. Use them to let the owner opt out of feature
      work, memory changes, or anything else whose value is questionable.
   5. **Pass the chain on.** Copy this landing section's chaining rules into the new
      plan's landing section, so every epic hands off to the next until no workable task
      bead remains. Do not carry the `dormant_flakes` decision forward unless dormant
      flakes remain.
   6. If no workable task bead remains, do not propose a plan. Report that the backlog
      is clear, listing the beads left only because they are snoozed, owned by another
      epic, or not-yet-due flags.
