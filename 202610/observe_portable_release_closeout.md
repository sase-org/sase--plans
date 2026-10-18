---
tier: tale
title: Observe the portable release and finish its audited landing
goal:
  Verify the containing core and sase 0.18.0 publication, close sase-1lh.7.5.4.3.3.4.4,
  and audit its direct plan ancestors.
size: medium
proposed_by: bbugyi200.athena.sase-1lh.7.5.4.3.3.4.4.land
proposed_with: codex/gpt-6-astra
bead: sase-1lh.7.5.4.3.3.4.4
create_time: 2026-10-17 23:20:50
status: wip
---

- **PARENT:**
  [202610/finish_portable_core_release.md](https://github.com/sase-org/sase--plans/blob/main/202610/finish_portable_core_release.md)
- **BEAD:**
  [sase-1lh.7.5.4.3.3.4.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1lh/sase-1lh.7.5.4.3.3.4.4.md)

# Observe the portable release and finish its audited landing

Finish the remaining release acceptance and landing for **sase-1lh.7.5.4.3.3.4.4**,
whose plan is `plan:202610/finish_portable_core_release.md`. Its parent plan bead is
**sase-1lh.7.5.4.3.3.4**, linked to `plan:202610/finish_release_gate_fixtures.md`.

This is one medium tale: the code repair and consumer pin already have their required
host-created commits. Only external release observation and evidence-based closeout
remain. Do not create another repair epic for already-owned external failures. This tale
has no land agent; its worker, including one explicit monitor continuation when needed,
must perform the final closeout below. Nothing here depends on this worker's own future
commit, push, or CI result.

## Verified baseline and ownership

Read the current epic's landing audit note and original linked plan through audited
commands before acting. The lander reviewed every child note and actual implementation:

- Core commit `72bf9203758d7986da1cb6d14b5d13f00327de08` promotes both variadic
  `openat(O_CREAT)` mode operands to `libc::c_uint`; fixed-argument `mkdirat` and
  `fchmod` retain native `mode_t`. Restrictive modes, ownership checks, no-follow opens,
  umask repair and atomic writes remain intact. Core ToolRun
  `aa65464aec20930adefb4ab99a8926a6` passed `just check`.
- Sase commit `ae9f044bb39c22ab5702031b1507249e29a2d4da` pins that core. Sase ToolRun
  `4be2e7f8e57c29b2db0f2c7e2eb1640a` passed `just check`. No binding/wire change
  occurred between those two core revisions.
- Core PR #331 head `e74bb0b320fe554681f72c25608cdbfb7d8457f4` contains the fix and
  proposes 0.42.0. CI run **39089637894**, macOS job **117563429625**, now compiles and
  passes the pairing tests, including permissions, umask, symlinks and hard links. Its
  sole failure is
  `project_stats::snapshot::tests::odd_filenames_and_non_utf8_paths_are_preserved`,
  `crates/sase_core/src/project_stats/snapshot/tests.rs:222`: writing `src/\xff.rs`
  returns OS 92, `Illegal byte sequence`. Result: 5171 passed, 1 failed, 4 ignored.
  Retained log: `file:explicit:cf44a5d9c96251754e7b7e5f`.
- That fixture was introduced by `733256ede964bc99e46f8ce7faffb690df0c6a49`, phase
  **sase-1m3.7.1.2**, before this epic. `/sase_new_task` triage found no task duplicate
  and attached a `DISCOVERED ISSUE` with evidence to causal active epic
  **sase-1m3.7.1**. That epic owns repair, including preserving non-UTF8 Git-tree
  coverage without requiring an unsupported worktree filename. Do not duplicate it.
- Master Gate **39091597244** on `ae9f044bb3` completed failure only in test shard 7,
  job **117571155606**. The exact failure is existing **sase-1lr**:
  `tests/ace/tui/test_link_follow_seam.py::test_beads_sync_reports_visible_and_missing`
  reports PENDING instead of MISSING. The lander recorded +1 with its failed-job log.
  Its approved repair exclusion remains. **sase-1mi** remains the separately tracked
  collision flake, with adjacent **sase-1m9** and **sase-1l8**; no new report was
  needed.
- There were no literal `PROPOSED FOLLOW-UP:` entries on this epic's children. The
  preceding two findings and all declined duplicate/ownership outcomes are recorded on
  the epic. Preserve earlier parent follow-up dispositions.
- Sase PR #299 remains open at `d3f2ba36c7580e1da8ba9a9eaf90016e1da8b1fc`, proposing
  0.18.0 with an obsolete core window `>=0.41.0,<0.42.0`. Let its reconciler advance the
  window after a compatible core is published. GitHub/PyPI still showed sase 0.17.1 and
  core 0.41.0; neither target release existed. Full CI **39085508146** passed on older
  `933970ce72a5b5328499e80751a491c857bfc221`; do not claim it covers the later fixed
  SHA.
- The original watch phase **sase-1lh.7.5.4.3.3.4.4.3** closed **canceled**, without
  starting a monitor. Its closure does not prove release acceptance.
- Both fetched masters had no post-fix/post-pin commits. The sole non-epic sase commit
  since this epic began, `9526d3bae3`, only updates setup/configuration/stats
  documentation and introduces no portability integration gap. Core's release-branch
  delta is only release metadata/changelogs. Epic-symbol inventory was empty.

## Constraints

Preserve inherited decisions **dispatch=yes**, **visual_flakes=recapture**, and
**stale_epics=supersede**. Only the existing release watcher dispatches sase workflows;
ci_watch owns the sase release merge, and core release-plz owns versions and its normal
daily release cut. Do not manually merge, dispatch a core release, hand-edit versions or
dependency windows, weaken gates, fix excluded flakes, or run `just check-full`.

This tale replaces only the canceled phase's observation scheduling. The old
no-successor monitor was appropriate for a phase with a waiting land agent. Here use one
explicit `/sase_monitor` continuation so this tale can perform its own mandatory
closeout. Do not repeatedly spawn watchers or recreate this plan when an unrelated
blocker persists. Preserve the watcher's bounded reruns and red-gate deadlines.

## 1. Refresh committed prerequisites

1. Read `/sase_repo` and open `sase-core`; read its `AGENTS.md`. Fetch both repositories
   and update only clean checkouts through supported fast-forward operations. Inspect
   new commits on master and the release branches, preserving concurrent changes.
2. Verify the committed sase pin includes the portability fix and the complete current
   consumer binding cohort. Review post-pin binding/wire drift. Do not ratchet a pin or
   make source edits merely for a test-only repair in the externally owned stats epic. A
   new genuine compatibility gap must be recorded as remaining work; do not pretend that
   this observation-only plan covers an uncommitted repair.
3. Set `FIXED_SHA` to the concrete committed sase master SHA containing the pin. Resolve
   `CORE` to the containing pending/published compatible release, currently 0.42.0.
   Require containment of the portability fix; do not substitute 0.41.0.
4. Refresh the stats epic, core macOS CI, Master Gate, Full CI, PR #299 and published
   versions. A red stats fixture remains externally owned, not a reason to repeat the
   successful openat repair. The watcher may await its containing release, but final
   acceptance requires an actual green containing macOS job. Record any new failure with
   its actual evidence and use `/sase_new_task` for distinct unrelated work.
5. If the full publication proof in step 3 already exists, collect it and skip the
   monitor. Otherwise continue with one bounded watch.

## 2. Observe once, with a closeout continuation

Read `/sase_monitor` and inspect `tools/_release_watch_api.py` and
`tools/_release_watch_engine.py` if options have changed. Use the existing tool; no
watcher source changes are planned. Do not pass `--close-bead`: an automatic terminal
verdict must not close the epic or this tale before the complete landing audit.

Preflight with concrete `FIXED_SHA` and `CORE` substituted:

```sh
PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --once --dry-run
```

Require exit 0 and a fully parsed state line. If preflight fails, record its failure on
the current epic and keep all incomplete scopes open. Do not call an empty result
success or repeat plan creation to conceal the blocker.

Record the chosen values and exact monitor command on the current epic. Start one
monitor using `/sase_monitor`, with timeout 14h, labels `WATCHING RELEASE` and
`RELEASE WATCHED`, and the following monitored command:

```sh
PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 10m 750m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --deadline 12h
```

Its `--next` must explicitly say: inspect the terminal verdict and retained logs; do not
restart the watcher; on success collect every proof in step 3, complete the canceled
watch phase, and perform step 4 for sase-1lh.7.5.4.3.3.4.4 and its concrete parent
sase-1lh.7.5.4.3.3.4; on failure record the exact blocker and ownership on the current
epic, keep the epic and incomplete ancestors open, and finalize without claiming done.
Include this plan's durable reference in that continuation. Do not add an idle timeout.
Wait for the monitor-start command itself to exit; do not wait on the monitored command
in the provider turn. If start fails twice, record both failures and the incomplete
status rather than claiming a handoff.

## 3. Prove publication, then complete the canceled phase

A `RELEASED` watcher verdict is necessary evidence, not a substitute for the full audit.
Collect and retain concrete SHAs, run/job IDs, URLs and install output proving:

- The containing core release includes `72bf9203` and has a green macOS CI job. Check
  the actual job result, including the previously failing project-stats fixture.
- The published core wheel passes the current complete binding/contract validation in an
  isolated environment; release-floor smoke is green for the release consumer.
- ci_watch merged the correct sase 0.18.0 release PR, GitHub v0.18.0 exists, actual
  Publish **build and publish** jobs passed, and PyPI lists 0.18.0. Skipped jobs do not
  prove publication.
- Release gating used a green Master Gate and sufficiently fresh green Full CI that
  covers `FIXED_SHA`, according to current ci_watch policy.
- A fresh isolated **unpinned** install (`uv pip install --refresh sase` in a fresh
  environment, without a sase/core version constraint or editable checkout) resolves
  sase 0.18.0 and the compatible core. Record both installed versions and successful
  `sase --version`. The watcher's own `sase==0.18.0` install is pinned and does not
  replace this check.

If any proof is absent, append a concrete blocker to the epic and do not close it or
mark its plan done. Do not widen scope to fix externally owned failures.

Once every proof exists, rerun descendant readiness checks. Reopen the canceled watch
phase with `sase bead open sase-1lh.7.5.4.3.3.4.4.3`, preserving its historical canceled
attempt, then close it normally with
`sase bead close sase-1lh.7.5.4.3.3.4.4.3 --note "<the complete publication and macOS proof>"`.
Retire any phase-keyed symbols before this close. This records actual completed
acceptance instead of treating a canceled attempt as success.

## 4. Final step: close this epic and recheck its direct ancestors

This closeout belongs to this tale worker, in the same turn as any final permitted
edits. Never wait for this worker's own commit SHA, push or CI result.

1. Re-read `sase-1lh.7.5.4.3.3.4.4`, all descendants and notes, its linked plan, and
   commits since the landing audit. Confirm the approved decisions and every note are
   addressed, and preserve the recorded follow-up dispositions. Resolve new integration
   gaps before a successful close. For tracked sase-file edits read verification memory,
   run `just fix` and `sase tool run check` (`just check`); never run `just check-full`.
2. Run `sase bead epic-symbols sase-1lh.7.5.4.3.3.4.4`. For every entry, follow
   Symvision policy: wire it, privatize it, add a justified non-test pragma, or delete
   it; re-key only when a specifically identified still-open later bead needs that
   exemption.
3. Close normally:
   `sase bead close sase-1lh.7.5.4.3.3.4.4 --note "<verified source, integration, publication proof, decisions, and every follow-up disposition>"`.
   On rejection, fix the cited readiness or whitelist issue and retry. Never force to
   make a successful nested landing advance. A deliberately canceled/superseded scope is
   a different outcome and must not be represented as completed work.
4. Run `just symvision` (through the required wrapper if guarded). Open the plans repo
   using `/sase_repo`, read the artifact through `sase artifact read`, and set
   `status: done` in `202610/finish_portable_core_release.md` inside the printed plans
   checkout. Do not directly edit bead status files.
5. Read the parent link again. The concrete direct parent is plan bead
   **sase-1lh.7.5.4.3.3.4**, linked plan `plan:202610/finish_release_gate_fixtures.md`.
   Review its previous landing note, **all descendants and every note**, linked plan and
   post-child drift; rerun descendant and linked-plan readiness checks. A canceled
   historical watch can be considered addressed only with the concrete replacement
   acceptance proof and a deliberate note; reopen/complete it if readiness requires. If
   fully complete, retire `sase bead epic-symbols sase-1lh.7.5.4.3.3.4` entries, close
   normally with `sase bead close sase-1lh.7.5.4.3.3.4 --note "<what was rechecked>"`,
   run `just symvision`, and mark `202610/finish_release_gate_fixtures.md` done.
6. Repeat the same full audit through directly parented plan ancestors, obtaining each
   linked plan through its audited bead read: **sase-1lh.7.5.4.3.3**,
   **sase-1lh.7.5.4.3**, **sase-1lh.7.5.4**, **sase-1lh.7.5**, **sase-1lh.7**, then
   **sase-1lh**. Close only fully complete ancestors, retire their symbols first,
   confirm `just symvision` and mark each linked plan done. Stop at the first incomplete
   or ambiguous parent, append a precise blocker note there, and report it. If a
   refreshed parent link instead identifies a phase, verify and close only that phase
   and leave its containing epic to its waiting land agent.

Use `/sase_final` for the normal final declaration. Include all repositories actually
changed, including the plans checkout. Report what closed and any remaining external
blocker; never imply a canceled watch or deadline established success.
