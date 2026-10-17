---
tier: epic
title: Finish visual subprocess deadlines and restore the 0.18.0 release target
goal:
  Visual retries cannot escape their wall-clock deadline, and the approved sase 0.18.0
  release is published and verified through ci_watch.
parent_bead: sase-1lh.7.5
phases:
  - id: deadline-repair
    title: Bound every subprocess exit path and declare the intended release version
    depends_on: []
    description:
      "deadline-repair: fix early-EOF waits and surviving descendants, add bounded
      regressions, and include Release-As: 0.18.0 in the host commit declaration."
    size: medium
  - id: release-watch
    title: Restore the generated release PR target and verify publication
    depends_on:
      - deadline-repair
    description:
      "release-watch: make no file edits, verify the committed repair and Release-As
      trailer, regenerate the release PR through Publish, then run a bounded watch with
      an honest close."
    size: small
proposed_by: bbugyi200.athena.sase-1lh.7.5.land
create_time: 2026-10-16 23:11:53
status: wip
---

- **PROMPT:**
  [prompts/202610/finish_deadline_release_target.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/finish_deadline_release_target.md)
- **PARENT:**
  [202610/finish_recapture_release.md](https://github.com/sase-org/sase--plans/blob/main/202610/finish_recapture_release.md)

# Finish the remaining deadline and release work

This child resumes **sase-1lh.7.5**, whose plan is
`plan:202610/finish_recapture_release.md`. Its direct plan ancestors are **sase-1lh.7**,
`plan:202610/sase_1lh_landing_repair.md`, and **sase-1lh**,
`plan:202610/v0_18_0_green_ci_release.md`. Read the beads and these artifacts through
the audited commands before implementing. Their accepted decisions remain
**dispatch=yes**, **visual_flakes=recapture**, **stale_epics=supersede**.

This is an epic because actual publication must include the repair's host-created commit
and its one-time release-version declaration. One coder can implement the repair, but
cannot observe its own post-turn commit in CI. The second, no-edit phase owns that
observation. No closeout phase is needed: this child's `parent_bead` link returns its
land agent to the interrupted landing.

## Audited baseline

The parent lander reviewed all three children and their notes, commits 499c825cb9 and
f564f0f996, actual visual maintenance and launch integration code, all intervening
master changes through **4cb33feae9f69561640ea96856f54be00f9856c9**, and the fetched
base branch. Existing focused tests pass: 58 recapture/timeout/launch tests and 118
watcher-decision/project-audit/base/verify/salvage tests. The split watcher test facade
`tests/test_release_watch_decisions.py` sets `__test__ = False`; select its three
`_dispatch`, `_evaluate`, and `_stale` modules to actually exercise these tests.

The strict recovery fixes, targeted-scope treatment, Python help assertion, timestamp
transport, identity-only audit exemption, and four inspected goldens are present. The
docs follow-up from sase-1lh.7.5.2 note 2 was independently fixed by 08d9b66154;
`tools/check_docs` now has zero errors and only the three known Contribute warnings. The
launch follow-up from sase-1lh.7.5.1 note 1 is fixed by f564f0f996; the parent lander
closed sase-1lw and the verified identity-audit tasks sase-1lx/sase-1ly. Do not recreate
those tasks or repeat those repairs. The parent has the complete follow-up triage note
and evidence references. Read the landing audit `file:explicit:6342ab3533376d4bbfa3d04f`
and bounded reproduction script `file:explicit:68a2c07fd2b673bf2fa2cc23` through
`sase artifact read`.

The release-watch child sase-1lh.7.5.3 closed **canceled**, not successful. At audit
time PR #299 is OPEN with title **chore(master): release 0.17.2** and head
043f8a95eebbc72ef26fcbbb6553192b1af4bc64. PyPI's current sase is 0.17.1; 0.18.0 is
absent. sase-core-rs is 0.40.1. Publish 38996094567 regenerated the PR successfully but
skipped build/publish. The watch correctly returned RELEASE_PR_GONE because no PR title
carries its exact requested version. Do not broaden that match to accept 0.17.2 as
success.

`release-please-config.json` sets `bump-patch-for-minor-pre-major: true`. Its normal
feature bump is 0.17.2. Upstream release-please explicitly supports a one-time
**Release-As: 0.18.0** commit-body declaration. The lander opened
`gh:googleapis/release-please` through `/sase_repo` and verified its README section "How
do I change the version number?" plus `test/strategies/base.ts` at
45e6e20d824a94b51478ea0b51c01ab6352eb25b. Reopen through `/sase_repo` if needed; do not
web-fetch repository files. A one-time commit trailer avoids changing the project's
future pre-1.0 versioning policy or leaving a persistent release-as override.

Master Gate 38996356739 has one independent configure/review readiness failure; its
exact node passes locally on the same SHA. Task **sase-1m9** owns that bounded
semantic-readiness fix, with evidence `file:explicit:cb9fb741595e074b5783a449`. Full CI
38996026102 at 3e950ebdbd was in flight. Inspect current results before acting; neither
observation proves release readiness.

## Rules

- Fetch and fast-forward at phase start. Recheck concurrent changes before editing.
- No manual commits, branches, PR merges, or release-branch pushes. Host completion
  commits the repair. Only the release-watch phase may dispatch workflows.
- Keep ci_watch eligibility and configuration intact. Do not publish or accept the wrong
  version, bump the core floor without binding evidence, or weaken pixel equality.
- No skips, xfails, larger sleep/wait budgets, or copying CI goldens. Keep update-mode
  salvage and the single shared recapture driver. This is test harness work, with no new
  shared backend feature or memory edit.
- Read the tools, TUI/screenshot, and lint/test memories before editing those areas.
  Read ACE memory for any ACE change and docs-writing memory for any docs change.
- For file changes run `just fix` and `sase tool run check`, plus the focused cases
  below. Do not run `just check-full`. A check-full-only failure is test infrastructure
  follow-up, not remaining epic work.
- Phase agents file no task beads. Record genuinely new discoveries in
  `PROPOSED FOLLOW-UP:` notes. The child lander triages them.

## Phase deadline-repair

Work primarily in `tests/ace/tui/visual/_visual_maintenance_exec.py` and
`tests/test_fix_tui_screenshots_exec_timeout.py`, with integration coverage in the
existing check-recapture tests. The current helper starts an owned process session and
uses selectors, but only bounds reading from its output pipe.

### Fix both reproduced escape paths

1. **EOF before process exit.** `_drain_child_output` breaks on an empty read and ends
   with unbounded `proc.wait()`. A child that closes fd 1 and 2, then sleeps 1.2
   seconds, returns **0 after 1.216 seconds with a 0.15-second budget**. With an
   indefinitely sleeping child it hangs indefinitely. Retain the same absolute deadline
   after EOF, bound the wait by its remaining time, and use the same timeout cleanup and
   return code 124 when the child remains alive. Do not reset the budget on EOF.
   Preserve the no-deadline behavior for update mode.
2. **Leader exits but a descendant ignores SIGTERM.** `_terminate_process_tree` sends
   SIGKILL only if `proc.wait(timeout=grace)` times out. A normal leader exits on
   SIGTERM while its SIGTERM-ignoring descendant stays alive. Reproduction with
   timeout=0.5/grace=0.1 returned 124, leader=-15, descendant state=S and alive. Decide
   escalation from the owned process group's remaining processes, not only the leader's
   exit. After the bounded grace, kill remaining group members even if the leader has
   already been reaped. Bound all waits, release inherited pipe/lock holders, and handle
   an already-gone group without errors. Do not kill unrelated process groups or treat
   zombie processes as live workloads.

Add deterministic, self-cleaning subprocess regressions for early stdout/stderr EOF
while alive, a normally exiting leader with a TERM-ignoring descendant, and an already
exited leader with a live descendant retaining the pipe. Establish child readiness via a
handshake rather than hoping its signal handler has installed. Ensure the tests have an
outer safety bound and cleanup in `finally`, so a broken implementation cannot hang the
test runner or leak children. Keep existing silent-child, TERM-ignoring-leader,
partial-log, fast-child, and no-timeout tests.

Also prove the governed check path converts this timeout to a readable manifest with
budget exhaustion and the correct unrecovered first-pass outcome; check mode must never
write golden/index bytes. Preserve strict first-pass/retry error handling and
update-mode agreement semantics. Inspect any other blocking wait in this same execution
path rather than fixing just the two literal examples. No full visual recapture is
needed solely for process lifecycle logic.

Run the timeout/check-recapture, base/verify/salvage/apply maintenance modules and
workflow tests, then the required `sase tool run check`. Verify the help assertion on
supported Python 3.12 if changing it; it already passed Master Gate after the prior fix.

### Declare the approved release target in host completion

The primary repository's `/sase_final` commit message must describe the repair and
contain this separate body/footer line:

```text
Release-As: 0.18.0
```

This is part of the authorized 0.18.0 release, not a version-policy change. Do not
manually commit or push, hand-edit generated package versions, add a permanent
release-as config, or wait for this turn's own SHA. Record the intended trailer in the
phase note so the following phase can verify the actual host-created commit before
dispatching. If 0.18.0 was independently published before this phase starts, do not
redeclare an already-published version; record that evidence and repair the deadline
normally, with release acceptance reassessed by the lander.

## Release-watch preparation

The following no-edit preparation belongs to release-watch, before step 3 of the
inherited lifecycle below. It addresses the exact reason the prior watch stopped.

After the sibling repair is committed, verify its SHA on origin/master and inspect the
actual commit body for `Release-As: 0.18.0` (unless that version already exists). Do not
start another version-filtered watch while #299 still targets 0.17.2.

If the target is still wrong, first observe any in-flight Publish rather than duplicate
it. If no Publish is in flight and no Publish on a commit containing the declaration has
completed, this phase is authorized to dispatch **one** normal regeneration:

```sh
gh workflow run publish.yml --repo sase-org/sase --ref master -f publish_existing=false
```

Observe the run and PR metadata through a bounded `/sase_monitor` preparation command
when it needs waiting, with a maximum 20-minute preparation deadline and a follow-up
that resumes this same no-edit phase. Record the dispatched run ID so a resumed turn
does not dispatch again. Before launching the ordinary release watcher, require a
successfully completed regeneration and an open/merged PR carrying 0.18.0, or verified
0.18.0 on PyPI. A successful Publish workflow with build/publish skipped proves only
regeneration, not publication. If the target still cannot be restored, close this phase
canceled with the observed title, run URL, commit trailer evidence, and
`release not confirmed: RELEASE_TARGET_NOT_RESTORED`; let the lander diagnose it. Do not
change watcher matching or report this as RELEASED.

The ordinary release watch below remains one bounded watch. The optional preparation
monitor has its own bounded lifetime and resumes this phase before that watch starts.

## Phase release-watch

This phase makes no file edits and always closes its bead with an honest verdict.

1. **Idempotent start.** Read this phase bead. If it is already closed, because this is
   a relaunch after recovery, run `/sase_final` and stop.
2. Fetch and fast-forward. Confirm that every sibling phase's commit is on
   `origin/master`, and use the newest `origin/master` SHA as FIXED_SHA. Pick CORE as
   the newest sase-core-rs on PyPI that passes `tools/check_sase_core_rs_bindings`
   against FIXED_SHA (0.40.1 at planning time). Inspect current Full CI, Master Gate,
   and PR #299, and do not duplicate an in-flight workflow.
3. **Preflight inline.** Append `--no-dispatch` to both commands when
   `dispatch = false`.

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --once --dry-run --close-bead <phase-id>
   ```

   It must exit 0 and print a fully parsed state line. On failure, close this phase with
   `-R canceled -r "release not confirmed: WATCHER_BROKEN (preflight)"` and the output,
   then run `/sase_final`. Do not repair code in this phase.

4. **Start one `/sase_monitor`.** Use timeout `14h`, statuses `WATCHING RELEASE` /
   `RELEASE WATCHED`, and `--tail-lines 200`, around:

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 10m 750m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --deadline 12h --close-bead <phase-id>
   ```

   The `--next` follow-up instruction must say all of the following:
   - Make no edits.
   - Read `sase monitor show <id>`, the verdict file, and the phase bead.
   - If the bead is still open, close it. Use `--note @<verdict file>` for `RELEASED`.
     Otherwise use `-R canceled -r "release not confirmed: <VERDICT>"`, or
     `UNKNOWN (<monitor state>, exit <code>)` when no verdict exists.
   - Never reopen the bead, restart the watch, merge, dispatch, or create beads.
   - Finish with `/sase_final`.

   If `sase monitor start` exits nonzero twice, close canceled with
   `release not confirmed: WATCHER_NOT_STARTED` and both errors, then run `/sase_final`.
   Run the start command in the foreground until it exits.

### Why this bead always closes

Each layer has its own deadline, and each deadline finishes inside the next layer out:
the watcher stops at 12 h, the outer `timeout` at 12.5 h (plus a 10-minute SIGTERM
grace), and the monitor at 14 h.

| Path                                                 | Who closes the bead                                               | What releases the land agent              |
| ---------------------------------------------------- | ----------------------------------------------------------------- | ----------------------------------------- |
| Any verdict, including the watcher's 12 h `DEADLINE` | Watcher, in-process retries                                       | Monitor follow-up runs `/sase_final`      |
| Outer `timeout` at 12.5 h                            | Watcher's signal path, a detached closer, inside the 10 min grace | Follow-up (monitor `failed`)              |
| Monitor timeout at 14 h (SIGTERM, then SIGKILL)      | Detached closer, or the follow-up if the close never ran          | Follow-up (monitor `timeout`)             |
| Watcher crash or hang                                | Outer `timeout` path above, or the follow-up                      | Follow-up                                 |
| Preflight or monitor-start failure                   | The phase agent, inline                                           | The phase agent's own `/sase_final`       |
| A human stops the monitor                            | Detached closer (`STOPPED`)                                       | Needs a human: `sase bead work <epic-id>` |
| Reboot (monitor `lost`) or a SIGKILL from outside    | Nobody; the bead stays open                                       | Needs a human: `sase bead work <epic-id>` |

The last two rows can only come from a human or the host, and SASE surfaces both.
`wait_checks` raises "Wait dependency can never self-resolve" for the parked land agent.
The recovery is safe because of step 1:

- `sase bead work <epic-id>` relaunches only the land agent when this bead is closed.
- It relaunches this phase when the bead is open, and the phase then watches again with
  a fresh preflight.

## Landing and parent continuation

The lander must verify both repaired process cases, all child notes, the committed
release declaration, later integration drift, and the actual release outcome. A closed
canceled watch is not success. For a positive release require all six inherited facts:
ci_watch merged PR #299; GitHub release v0.18.0 exists; Publish build and publish jobs
succeeded; PyPI lists 0.18.0; an unpinned fresh isolated install resolves 0.18.0 and
reports it with its resolved sase-core-rs version; and the relevant Master Gate and Full
CI are green. Read the successful Full CI visual manifest, retaining every unstable node
for the original parent lander's triage. If only external waiting remains use
/sase_monitor, not an ordinary final response promising continuation. If repair remains,
follow the user-directed tier-aware plan loop. Never close on a negative verdict.

After this child lands, its parent_bead resumes **sase-1lh.7.5**. Its closeout is a
land-agent duty, not a phase in this child:

1. Re-read sase-1lh.7.5, every descendant and note, and
   plan:202610/finish_recapture_release.md; recheck all named-phase and linked-plan
   readiness, the saved follow-up outcomes, release acceptance, and post-child drift.
2. Run `sase bead epic-symbols sase-1lh.7.5`. Resolve every entry or re-key it only to a
   still-open later bead that truly needs the exemption. Read Symvision memory before
   this cleanup. Close normally using
   `sase bead close sase-1lh.7.5 --note "<verified scope, integration, release evidence, and all follow-up dispositions>"`.
   Then run `just symvision` through the permitted tool wrapper if required and set
   `status: done` in its plan file, **202610/finish_recapture_release.md** in the plans
   repo opened through /sase_repo. Do not force merely to succeed; never force a
   successful nested landing.
3. Its parent is the plan bead **sase-1lh.7**, not a phase. Review its previous landing
   note, every descendant and note, plan:202610/sase_1lh_landing_repair.md, and
   post-child drift. Rerun descendant and linked-plan readiness checks. If fully
   complete, clear `sase bead epic-symbols sase-1lh.7`, close it normally with the
   evidence, run `just symvision`, and mark **202610/sase_1lh_landing_repair.md** done.
4. Repeat the same full readiness audit for directly parented **sase-1lh** and
   plan:202610/v0_18_0_green_ci_release.md. Complete its still-open post-release duties:
   triage every unstable node from the Full CI run relied on by ci_watch, verify Install
   smoke healed, and honor its approved stale_epics=supersede disposition for sase-1kx
   and, only if its own goal is proven, sase-1jz. Read the original plan's exact rules
   and the sase-1kx task-closeout requirements first. That deliberately authorized
   supersession of obsolete competing plans is distinct from normal successful
   parent-chain closure. Clear this parent's epic-symbols, close normally, run
   symvision, and mark **202610/v0_18_0_green_ci_release.md** done only when all its
   scope is complete.
5. Stop at the first incomplete or ambiguous parent. Append a note to that concrete
   parent naming the blocker and report it; do not mark its plan done or force it shut.
