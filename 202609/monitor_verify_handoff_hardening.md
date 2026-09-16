---
tier: epic
title: Harden the monitor verify handoff (sase-11o.1 failure class)
goal: 'A fast-settling verify monitor no longer strands its family: the follow-up
  still dispatches once the starter settles, a not-launchable follow-up durably preserves
  the dirty worktree and names its recovery command, and monitor start never destroys
  command quoting.

  '
phases:
- id: starter-race
  title: Close the monitor-settles-before-starter race
  depends_on: []
  size: medium
  description: 'starter-race: wait bounded for the starter to settle and re-hydrate
    parent nodes before stamping needs_recovery or recording not-launchable.'
- id: recovery-evidence
  title: Preserve worktree evidence on not-launchable follow-ups
  depends_on:
  - starter-race
  size: medium
  description: 'recovery-evidence: snapshot the monitored workspace''s uncommitted
    diff before claim release and add resume-command and snapshot hints to wait_checks
    notifications.'
- id: argv-quoting
  title: Stop monitor start from destroying command quoting
  depends_on: []
  size: small
  description: 'argv-quoting: preserve remainder argv with shlex-aware joining, warn
    on redundant bash -c wrappers, and update the monitor skill''s command-authoring
    guidance.'
proposed_by: bbugyi200.athena.0lx
create_time: 2026-09-16 10:07:16
status: wip
bead_id: sase-11r
---

- **PROMPT:** [prompts/202609/monitor_verify_handoff_hardening.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/monitor_verify_handoff_hardening.md)
- **BEAD:** [sase-11r](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11r/README.md)

# Harden the monitor verify handoff (sase-11o.1 failure class)

## Why (incident summary)

On 2026-09-16, epic phase agent sase-11o.1 finished ~24 minutes of implementation and
handed verification to a monitor
(`sase monitor start --profile verify ... --next '<fix failures and close the bead>'`).
Three independent defects then compounded into a terminal, unrecoverable-by-machine
failure:

1. **Quoting loss**: the recorded monitor command was
   `bash -c just install && just check` (no inner quoting). Under the host's
   `/bin/sh -c` wrapper, `bash -c just install` runs plain `just` (`install` becomes
   `$0`), so `just install` was silently skipped and `just check` started immediately.
2. **Fast failure beat the starter's finalizer**: `just check` failed legitimately in
   ~3s on ruff-format (the worker never ran `just fix`). The monitor settled at 09:42:26
   while its starter agent was still finalizing (until ~09:42:50). Result capture found
   no starter continuation node, stamped
   `continuation_capture_disposition=needs_recovery`, and settlement immediately
   recorded `monitor_followup_outcome=not-launchable` ("monitor result is missing its
   exact starter parent node; automatic dispatch is blocked pending recovery"). The
   `--next` fixer agent that would have run `just fix` and closed the bead never
   launched.
3. **Evidence destroyed**: settlement released the monitor claim, and 15 seconds later a
   subsequent launch reset the workspace to origin/master, wiping the worker's
   uncommitted 12-file diff. The only surviving copy was the gh-workflow diff snapshot
   in the reaper-managed handoff bucket
   (`~/.cache/sase/tmp/gh-diffs/sase-gh-cXG9Ia.diff`, since copied to
   `~/sase-11o.1-recovered-worktree.diff`).

Downstream, waiters sase-11o.2 and sase-11o.land are permanently blocked (`wait_checks`
"Wait dependency can never self-resolve" notifications), and the notification names
neither `sase monitor resume` nor any preserved diff.

Evidence: failed run artifacts under
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/16/20260916094222/`
(`done.json`, `continuation/monitor_result_manifest.json` with `parent_node_ids: []`,
`diagnostics/retained_logs/live_reply.md`), starter run `.../20260916091831/` (its
`agent_meta.json` has
`continuation_node_id=agent-delta:20260916091831:8bedb9f30748eee7`, written after the
monitor had already settled).

## Constraints

- The monitor continuation records path is gated by the `monitor_continuation_records`
  feature flag (retirement tracked by flag bead sase-102). Fix the records path in
  place; do not add new feature flags — these are bug fixes, and the legacy
  (records-disabled) path already routes through `launch_followup_agent`, which has its
  own starter wait.
- No Rust-core changes: monitor orchestration is host-side Python.
- Task bead sase-11q (memory guidance: run `just fix` before a verify handoff) and any
  fix-up of the sase-11o epic itself are out of scope for this plan.
- Workers must follow `PROPOSED FOLLOW-UP:` note discipline (epic phase workers do not
  create beads).

## Phase starter-race (medium)

Goal: a monitor that settles before its starter finishes finalizing must still
auto-dispatch its follow-up once the starter settles (bounded wait), instead of being
permanently stamped `needs_recovery`/`not-launchable`.

Key anchors:

- `src/sase/continuation_capture/monitor.py`: `_hydrate_parent_node_ids` (~line 564)
  falls back to reading `agent_meta.json` from `monitor_starter_artifacts_dir`;
  `_missing_essential_starter_parent` (~line 582) produces the blocking error; the
  publish path (~lines 397, 474–481) stamps the disposition.
- `src/sase/monitor/settlement.py` (~lines 106–130): reads
  `continuation_dispatch_blocked_reason(meta)` and records `not-launchable`, then
  releases the monitor claim — with no starter wait and no re-evaluation.
- Existing starter-wait machinery to reuse: `wait_for_monitor_starter` in
  `src/sase/monitor/followup_persistence.py` (~line 143), used by
  `src/sase/monitor/followup.py` (~lines 147–151) with
  `DEFAULT_STARTER_SETTLE_TIMEOUT_SECONDS = 60.0` (`src/sase/shells/followup.py:34`).
  Note this wait currently runs only _after_ the blocked-reason gate has already
  short-circuited settlement, so it never helps the records path.

Steps:

1. In the result-capture publish path, when parent hydration comes up empty but the meta
   names a starter (`monitor_starter_agent` / `monitor_starter_artifacts_dir`), wait
   (bounded, reusing the starter-settle wait and its default timeout) for the starter's
   terminal marker, then re-run `_hydrate_parent_node_ids` with the starter fallback
   before deciding the disposition. Stamp `needs_recovery` only when hydration still
   fails after the bounded wait.
2. In settlement, before recording `not-launchable` for the missing-starter-parent
   reason, re-evaluate once: re-hydrate parents (the starter may have settled between
   capture and settlement) and, when recovery data now exists, clear the stale
   disposition, backfill the monitor-result node/manifest parent ids (or re-publish the
   capture), and proceed to the normal follow-up launch path instead of
   short-circuiting.
3. Make sure the repaired path updates the already-written
   `continuation/monitor_result_manifest.json` and node record parent ids (today the
   meta can end up self-contradictory: `needs_recovery` + populated
   `continuation_parent_node_ids`, while the node record still says `parent_ids: []`).
4. Tests (e.g. alongside the existing coverage in `tests/monitor/`):
   - monitor settles while starter is still finalizing; starter settles within the wait
     window → follow-up dispatches, disposition `ok`, node record has the starter's
     `agent-delta` parent id.
   - starter never settles → bounded wait expires, `needs_recovery` + `not-launchable`
     recorded exactly as today (no hang).
   - stopped/lost monitors keep bypassing the check (`_missing_essential_starter_parent`
     already exempts them; don't regress that).

## Phase recovery-evidence (medium, deps: starter-race)

Goal: when automatic dispatch still ends `not-launchable`, the failure must be cheap to
recover by hand: the dirty worktree is snapshotted durably before the workspace can be
recycled, and the user-facing notification names the recovery command and the snapshot.

Steps:

1. In the not-launchable settlement path (`src/sase/monitor/settlement.py`, and the
   `_record_not_launchable` helpers in `src/sase/monitor/followup.py` /
   `followup_persistence.py`), before the monitor claim is released, snapshot the
   monitored workspace's uncommitted state from `monitor_cwd` — `git diff HEAD` plus
   untracked files, mirroring the gh-workflow diff step — into the run's durable
   artifacts (e.g. `diagnostics/worktree_recovery.diff`), and record its path in the
   meta/`done.json` and in the recovery prompt text. Skip silently when the tree is
   clean or `monitor_cwd` is gone; snapshot failure must not mask the original error.
2. Enrich the wait_checks terminal-dependency notification
   (`src/sase/scripts/sase_chop_wait_checks.py`): when the blocking run's `done.json`
   shows `monitor_followup_outcome=not-launchable`, add a note naming
   `sase monitor resume <monitor_id>` as the recovery path and attach the
   worktree-recovery diff path when present, alongside the existing "kill and relaunch
   the waiter" guidance.
3. Tests: not-launchable settlement writes the snapshot and references it; wait*checks
   notification includes the resume hint and snapshot file for a blocked terminal
   dependency with a not-launchable follow-up; clean-tree case writes no snapshot.
   Related (not duplicate) prior report: task sase-wp tracks silent _gate* follow-up
   launch failures; link it from this phase's work if useful.

## Phase argv-quoting (small)

Goal: `sase monitor start ... -- bash -c 'just install && just check'` must never
degrade into `bash -c just install && just check`.

Steps:

1. Fix `start_command` in `src/sase/main/monitor/common.py` (~lines 93–102): with a
   single remainder word, return it verbatim (preserves the quoted-single-string idiom
   `-- 'a && b'`); with multiple words, join with `shlex.join` so each argv element
   survives the round trip into the stored `/bin/sh -c` command string.
2. Add a start-time diagnostic: when the resolved command begins with `bash -c` or
   `sh -c`, warn that the host already runs the command under `sh -c` (the wrapper is
   redundant and historically the top misquoting source). Warning only — do not reject.
   If this grows any new CLI option, read `sase/memory/cli_rules.md` first via the
   memory-read skill.
3. Update the generated skill source `src/sase/xprompts/skills/sase_monitor.md`
   (Hazards, ~lines 52–72, and the Canonical Invocation example): state that the command
   after `--` is preserved argument-for-argument, that a single quoted string is run
   verbatim as the shell command, and that `bash -c` wrappers must not be used. Follow
   the generated-skills flow: template change lands with this repo's commit; deployment
   (`sase skill init --force` + `chezmoi apply`) only from the landed tree.
4. Tests: `start_command` unit tests for the three shapes (`-- just check-full`,
   `-- 'just install && just check'`, `-- bash -c 'just install && just check'`), and an
   end-to-end assertion that the stored `monitor_command` for the third shape preserves
   the inner quoting.

## Verification

Each phase runs `just check` (via a verify monitor when slow, per two-speed
verification); the land agent runs `just check-full` before landing the combined tree.
