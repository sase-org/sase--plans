---
tier: epic
title:
  Finish recapture correctness and release gate integration, then confirm sase v0.18.0
goal:
  Check-mode recapture preserves execution and completeness failures and obeys its
  wall-clock budget. Current release gates and visual checks pass, ci_watch merges PR
  299, and sase 0.18.0 installs from PyPI, allowing sase-1lh.7 and its parent sase-1lh
  to finish their landing audits.
parent_bead: sase-1lh.7
phases:
  - id: recapture-contract
    title: Finish strict check-mode recovery and enforce the retry deadline
    size: medium
    depends_on: []
    description:
      "recapture-contract: repair internal-error masking, targeted-run false failures,
      unbounded retry subprocesses, and the Python 3.12 help assertion without changing
      update-mode salvage or exact golden matching."
  - id: release-integration
    title: Reconcile current release-blocking tests, docs anchors, and four visual diffs
    size: medium
    depends_on:
      - recapture-contract
    description:
      "release-integration: integrate the launch submission timestamp and logical-plan
      repair changes with their audits, reconcile docs-owned gate drift, inspect and
      repair the four persistent visual differences, and verify the release floor
      contract."
  - id: release-watch
    title: Watch v0.18.0 to PyPI after the repair commits land
    size: small
    depends_on:
      - release-integration
    description:
      "release-watch: make no file edits, verify the committed repair baseline,
      preflight the watcher, monitor one bounded watch, and always close with an honest
      verdict."
proposed_by: bbugyi200.athena.sase-1lh.7.land
create_time: 2026-10-16 20:23:03
status: wip
---

- **PROMPT:**
  [prompts/202610/finish_recapture_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/finish_recapture_release.md)
- **PARENT:**
  [202610/sase_1lh_landing_repair.md](https://github.com/sase-org/sase--plans/blob/main/202610/sase_1lh_landing_repair.md)

# Finish only the remaining release work

The parent is **sase-1lh.7**, linked to `plan:202610/sase_1lh_landing_repair.md`; its
parent is **sase-1lh**, linked to `plan:202610/v0_18_0_green_ci_release.md`. Read those
artifacts with `sase artifact read`, and read both beads before implementation.

The landing audit reviewed all four phase beads and every note, actual code, the three
implementation commits, and intervening master commits through
`f6ff20ef2a68185187e3fcceab2b95529054a9d6`. The explicit nested-drop anchor fix
`e41b209b24` is complete. The watcher repair `ec7e86e690` implements the approved
staleness policy, fresh-red dispatch accounting, and detached close recovery. Recapture
repair `2eb0ca791a` still has the concrete gaps below. Existing focused coverage passes
(149 tests); those tests do not cover the newly reproduced cases. No `--epic-symbol`
entries remain on sase-1lh.7 as of this audit. The detailed evidence is
`file:explicit:7452d1d4b006dacd914725b4`; read it with
`sase artifact read file:explicit:7452d1d4b006dacd914725b4 "Need the landing audit"`.

The release phase sase-1lh.7.4 closed **canceled**, with CORE_FLOOR_BLOCKED. PR #299 is
still open; PyPI still lists 0.17.1 and has no 0.18.0. Its floor-smoke run 38982644527
has 831 passes and one static project-identity audit failure, not an established missing
core binding. The approved watcher correctly treats this same-base PR as non-stale. Do
not ratchet the core just to silence that assertion.

This is an epic because the repair must first be committed by host completion, then a
distinct no-edit agent can watch CI and publication at that committed SHA. A tale cannot
satisfy release acceptance before its own post-turn commit. There is no closeout phase:
the child land agent resumes the parent through parent_bead.

## Rules and already-settled decisions

- Inherit **dispatch=yes**, **visual_flakes=recapture**, and **stale_epics=supersede**
  from sase-1lh. Do not ask these again.
- The authorized zero-unstable host proof is already in tui_screenshot.md. This plan
  needs no memory edits. Use the memory skills if a new memory issue arises.
- Start each phase from fetched, fast-forwarded origin/master; inspect new commits and
  current CI before making an already-landed repair again.
- Run `just fix` and `sase tool run check` for file changes. No `just check-full`. A
  check-full-only failure is test infrastructure work, not a release repair here.
- No skips, xfails, raised test-wait timeouts, relaxed pixel equality, copied CI
  goldens, removed audit coverage, or weakened ci_watch eligibility gates.
- Only release-watch may dispatch workflows. No manual release PR merges or pushes to
  the release-please branch. Do not change ci_watch gates or configuration.
- Read the applicable tools, TUI, ACE, docs-writing, and lint/test reference memories
  before editing those areas. Use `/sase_repo` for other repositories. Shared backend
  behavior belongs in sase-core; the planned changes are test harness and frontend
  integration work and do not call for a new backend.
- Phase workers create no task beads; put genuinely new discoveries in
  `PROPOSED FOLLOW-UP:` notes. Existing tracked issues below may be closed only once
  their exact scope is verified, with a note naming the repair evidence.

## Phase recapture-contract

Work in `tests/ace/tui/visual/_visual_maintenance_{run,verify,exec,types}.py`, focused
maintenance tests/helpers, and the existing help test. Keep the shared attempt loop; do
not build another check-only retry scheduler.

### 1. Restore strict check semantics with scope-aware recovery

The lander reproduced all of these with `_pair`, `FakeRunner`, and `AttemptScript` from
`tests/test_fix_tui_screenshots_check_recapture.py` and
`tests/_fix_tui_screenshots_helpers.py`, without changing tracked files:

1. First-pass ACE capture differs from its golden. First-pass child exit and
   `session_exitstatus` are both 3 (pytest internal error). Retry matches exactly.
   Current result: exit 0, status clean, `complete=false`, no errors.
2. Same recovery, first-pass child exit 3 but inventory session exit 0. Current result:
   exit 0 clean.
3. A targeted ACE-only invocation (`--check -- ACE_NODE`) drifts then matches. Current
   result: exit 3 due to `visual_root_unexecuted:pager`, despite a complete targeted
   inventory. The missing other root is legitimate for targeted scope.

`check_mode_strict_errors` only screens selected reason prefixes. It ignores child exit,
non-test-failure session statuses, collection mismatch, unexpected workers, and
full-inventory evidence; `_run_locked_check` skips its old strict checks when any node
is recaptured. Make one scope-aware policy account for those facts after recovery.
Ignore only ordinary pytest test-failure status and node failure reasons that are
actually cured by trusted exact-matching recaptures. Internal errors, interrupts, usage
failures, unexplained exit disagreement, protocol errors, collect-only/mismatched
collection, missing workers, skipped nodes, and structural full-scope omissions must
remain failures. Do not demand both roots for targeted runs. Preserve a meaningful
failure reason and affected node IDs in the manifest.

Audit retry acceptance too: `_run_one_verify_attempt` records child exit, but the check
loop currently trusts node data without validating abnormal retry exits or global
protocol errors. A matching image from a broken retry cannot cure the first pass.
Preserve independent valid node recovery and update-mode salvage semantics.

Tests must cover the three reproductions, full versus targeted scope, ordinary
failed-node recovery, unrecovered failures, skipped/uninventoried nodes, collection
mismatch, and retry exit/protocol errors. Verify that check mode never changes any
golden/index bytes and that update-mode agreement tests remain green.

### 2. Enforce the wall-clock budget inside the subprocess

`_AttemptDriver` checks its deadline only before launching a batch containing every
remaining node. `run_governed_visual_pytest` then blocks in stdout.read/proc.wait
without a timeout. A batch can run past 600 seconds and consume the 45-minute CI job's
report-upload margin. Checking time between the two attempts is insufficient.

Carry the remaining deadline into governed execution. Ensure a quiet/hung child cannot
block the deadline check. On expiration, terminate and reap the owned process tree with
a bounded grace and kill fallback; do not leave xdist children or a suite admission
lease behind. Preserve partial logs and a trustworthy inventory when available; nodes
not proved recovered keep their first-pass outcome. Always publish a manifest/report and
record budget exhaustion. Update mode with no deadline must retain its existing
behavior. Keep the CI 600-second value unless new measured evidence warrants a different
budget; extending the job or per-test waits is not the fix.

Add meaningful small subprocess tests for a silent child and descendants, plus
fake-clock boundary tests for exhaustion before the first retry and mid-batch. Prove the
budget limits elapsed execution and leaves a readable failure report.

### 3. Repair this epic's Python-version-sensitive test

Master Gate 38983979564 shard 7 fails
`tests/test_fix_tui_screenshots.py::test_help_lists_sorted_check_flag_and_exit_codes` on
Python 3.12. The test added in 2eb0ca791a assumes the literal substring
`-b, --recapture-budget-seconds`; Python 3.12 repeats `SECONDS` after the short alias.
Assert the aliases, destination/metavar, order, and semantics without an
interpreter-specific adjacency assumption. Verify on supported 3.12 and the local
interpreter. Do not rewrite unrelated generated CLI pages to work around it.

Run the check-recapture, verify, salvage, apply, and base maintenance test modules plus
CI workflow tests, then the required `sase tool run check`.

## Phase release-integration

First read current master and CI. Every issue below is already classified; record which
fixes landed independently and avoid duplicating them. This phase owns the bounded
release integration if the issues are still present. Add resolution notes to the
existing owner instead of inventing a new parallel tracking item.

### 1. Clear the deterministic floor and fast-gate regressions

- **sase-1lx, small CI task:**
  `tests/test_project_display_presentation_audit.py::test_project_labels_never_derive_from_paths`
  rejects `repair_archived_prompt_path` in
  `src/sase/ace/tui/actions/hints/_view_target_repair.py` after a9701c36e6. The
  `Path(project_file).parent.name` result is a storage identity passed to
  `hidden_sidecar_clone_dir`, not a label. Review the call chain and either add the
  precise reasoned identity exemption in
  `tests/_project_display_presentation_audit_path_labels.py` or reuse the canonical
  identity seam with its explicit audit coverage. Never substitute a display name into a
  storage path. Run the audit and logical-plan/archive repair tests.
- **sase-1lw, small CI task:** fae5cf5076 carries the true submission timestamp through
  extra_env. Update explicit launch fakes and exact environment assertions in
  `tests/test_launch_approval.py`, `tests/test_launch_approval_tui.py`, and
  `tests/test_multi_prompt_launcher_macro_groups.py`. Assert the trusted timestamp and
  original SLOT/force-reuse behavior; freeze time where exact dictionaries are intended.
  Preserve one shared stamp per fanout and fresh stamps on resubmission. Master Gate
  38983979564 shards 2/3/4 identify the five failing nodes. Run those files and
  launch-submission regressions, then close these two tasks with evidence.
- **Active owner sase-1le.20.1 (DISCOVERED ISSUE notes):** 367562ba52 changed the
  compact memory help in `src/sase/main/parser_root_help.py` without updating
  `tests/main/test_parser_root_help.py::test_root_help_renders_compact_help`. Reconcile
  the current wording. Separately, docs-check on the same Master Gate fails five
  `docs/_meta/moves.yml` destinations under `reference/work-units/bead-cli/`:
  `sase-bead-create-1`, `sase-bead-dep-1`, `sase-bead-list-1`, `sase-bead-sync-1`,
  `sase-bead-task-type-1`. Fix historical redirects/anchor destinations to the correct
  retained content, preserving historical URLs rather than deleting the URL contract.
  Run strict `just docs-check`, docs-move regressions, and compact root-help tests.
  Leave Deploy Docs's separate covers/mkdocs-exporter issue to its owner.

Recheck the actual release contract set used by `release-core-floor-smoke` with the
currently published supported floor. All 831 other tests passed at the observed PR head.
Do not treat the old CORE_FLOOR_BLOCKED label as authorization to modify core APIs or
bump the floor unnecessarily.

### 2. Inspect and repair the four remaining visual differences

Full CI **38980994819**, SHA **fae5cf5076**, visual job **117244671206**, completed in
33m21s. It includes every sase-1lh.7 repair and 552b38f7ed instruction goldens. Its
final manifest run ID is `e9e359ecae434504b64c2fe9eaa22cdc`: created=0, updated=4,
unchanged=1096, stale=0. Remaining images:

- `agents_retry_e2e_completed_chain_120x40.png`
- `agents_retry_e2e_countdown_120x40.png`
- `agents_retry_e2e_running_fallback_120x40.png`
- `agents_archive_help_120x40.png`

The corresponding nodes are in `test_ace_png_snapshots_agents_retry_e2e.py` (three
`test_real_fakey_*` nodes) and
`test_ace_png_snapshots_agents_archive_states.py::test_archive_help_png_snapshot`. All
four pytest nodes pass during both serial retries; their pictures still differ.

Read the latest report and inspect expected/actual/diff images. The audit could read
logs but both artifact download methods returned invalid ZIPs. If that still happens,
generate targeted local candidates and inspect those; never claim the old CI pixels were
reviewed. Reproduce actual differences before deciding a fix.

Archive-help drift is routed to active epic **sase-1lj** because 552b38f7ed changed help
commands. Confirm the intended content and update only its affected golden if needed, or
take the E4 owner's landed repair. The retry images share nodes with **sase-173**
(random hex artifact IDs); **sase-13v** closed older timeout/491px bugs. Do not assume
either cause from the node names. Check new launch timestamps, artifact-ID sources,
locale/timezone, and loaded state. Pin the actual nondeterminism or correct a newly
intended frame. Record the root cause on the existing task if it matches; otherwise
propose the distinct follow-up to this child lander.

For each changed golden, inspect the image, then prove two targeted checks (one with
`TZ=UTC`) with **zero unstable nodes**. Preserve no-edit exact equality in CI. Do not
regenerate unaffected goldens or repeat a full screenshot suite routinely.

The old run's eight retry-only passes must remain visible in evidence, not treated as
stable fixtures: renamed generic agent-session root, swarm clan panel, clan fold levels,
node-finder no results, artifacts split narrow-size3, persistent-filter small terminal,
copy-as-stitches dark, pager history past-frame-True-size1. The parent sase-1lh lander
owes per-node triage for the successful release's actual Full CI run; this old list is
context, not a replacement for that later evidence.

Run `just fix` and `sase tool run check`. Record targeted proof and owner/task outcomes
on this phase. Do not dispatch Full CI or wait for your own commit in this coding phase;
host completion creates the commit, then release-watch starts.

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

Closed phases, including a canceled release-watch phase, do not prove success. The child
lander must independently confirm all six release criteria from
`plan:202610/sase_1lh_landing_repair.md`: ci_watch merged PR #299; GitHub v0.18.0
release exists; Publish build/publish jobs succeeded; PyPI lists 0.18.0; an unpinned
fresh temporary install resolves 0.18.0 and reports it with its resolved core version;
and the relevant Master Gate and Full CI are green. Waiting belongs in `/sase_monitor`,
repairs belong in the user-directed tier-aware plan workflow. Never force an ordinary
successful nested landing.

Then land this child normally and follow its parent_bead to **sase-1lh.7**. Re-read all
descendants, all notes, the linked plan, this audit, and post-child drift. Retire or
legitimately re-key every exemption from `sase bead epic-symbols sase-1lh.7`, close
normally with a concrete verification note, run `just symvision`, and set status done in
`plan:202610/sase_1lh_landing_repair.md`. These are land-agent duties, not a phase.

Repeat readiness checks for **sase-1lh** and `plan:202610/v0_18_0_green_ci_release.md`,
including its previous landing notes. It still owes post-release Install-smoke and
per-unstable-node triage, and the accepted `stale_epics=supersede` work for the sase-1kx
tree and sase-1jz, with the specifically named task evidence closeout in its plan. Do
not force any successful nested closure. If fully complete, retire exemptions, normally
close sase-1lh, run symvision, and mark its plan done. At the first incomplete or
ambiguous ancestor, record the blocker there and report it rather than claiming success.

## Completed follow-up triage from the interrupted landing

Carry the parent lander's saved triage note into eventual close notes:

- sase-1lh.7.1 #1 and .3 #1: one +1 on **sase-1lm**; no duplicate CLI-doc task.
- .2 #2: the requested later Full CI ran; instruction-bundle-specific goldens were not
  among its remaining differences. Archive-help drift is recorded on **sase-1lj**. No
  task merely to wait for that already-completed visual job.
- .2 #4: current local memory check passes; historical projection drift was routed to
  **sase-1lj**. No new task or +1 on **sase-vu**, whose live-write isolation bug was not
  reproduced here.
- Independent launch transport and project-identity failures: **sase-1lw** and
  **sase-1lx**, both small CI tasks, created after all-status duplicate search,
  last-week sweep, and causal-epic review.
- Root-help and historical bead anchors: notes on **sase-1le.20.1**, no new tasks.
- Retry-image observation: supplementary note on **sase-173**; no +1 or reopening of
  **sase-13v** without confirming the actual diff signature.

The repaired explicit nested anchor_target logic, authorized host-proof memory sentence,
fresh-red budget, and signal-during-close behavior need no redesign.
