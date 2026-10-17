---
tier: epic
title: Repair portable core publication and the remaining sase release gates
goal:
  Repair the core macOS compile failure and two Master Gate blockers, observe a
  compatible published core and sase 0.18.0, and return the incomplete landing to
  sase-1lh.7.5.4.3.3.
parent_bead: sase-1lh.7.5.4.3.3
phases:
  - id: core
    title: Make mobile pairing file modes portable to macOS
    description:
      "core: repair the two libc mode_t boundary mismatches blocking core 0.42.0,
      preserve permission protections, and verify the linked core."
    depends_on: []
    size: small
  - id: gates
    title: Repair the launch timestamp fixture and rescue audit integration
    description:
      "gates: make the deferred-workspace test independent of ambient launch timestamps,
      review the renamed rescue deletion site, finish the prior bounded fixture sweep
      and tutorial proof, and verify current gates."
    depends_on:
      - core
    size: medium
  - id: watch
    title: Observe the compatible core and guarded sase 0.18.0 publication
    description:
      "watch: verify the committed fixes, choose a containing core release target, and
      run one self-closing release watch whose monitor has no follow-up and always
      settles."
    depends_on:
      - gates
    size: small
proposed_by: bbugyi200.athena.sase-1lh.7.5.4.3.3.land
proposed_with: codex/gpt-6-astra
create_time: 2026-10-17 17:10:43
status: wip
---

- **PROMPT:**
  [prompts/202610/finish_release_gate_fixtures.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/finish_release_gate_fixtures.md)
- **PARENT:**
  [202610/green_gates_ship_0_18_0.md](https://github.com/sase-org/sase--plans/blob/main/202610/green_gates_ship_0_18_0.md)

# Finish the remaining release work

This is remaining work from the landing of **sase-1lh.7.5.4.3.3**, whose plan is
`plan:202610/green_gates_ship_0_18_0.md`. Its clock and tutorial fixes are committed;
its watch phase closed **canceled**, with `CORE_FLOOR_BLOCKED`. Neither that parent nor
any ancestor is complete. This plan does not repeat the earlier watcher repairs, release
declaration, or completed fixture changes.

Three phases are necessary: core's portability fix receives its host-created commit, the
sase repair agent pins that committed core and fixes the Python gate fixtures, then the
watcher observes both committed fixes through CI and publication. No coder waits for its
own commit, SHA, push, or CI result. A tale cannot complete this release proof and
closeout in its one coding turn.

Inherited decisions remain **dispatch=yes**, **visual_flakes=recapture**, and
**stale_epics=supersede**. No new decisions or memory changes. Only the existing watcher
dispatches sase workflows; ci_watch owns the sase release merge. Do not manually merge
either release PR, edit release versions, or force a successful nested landing. Core
release-plz owns its versions and daily release cut.

## Verified baseline

At fetched master `2dd7fe7f8e204d874166ec222199383b81a58c16`:

- `f2a7b802f9` derives healer fixture row, ledger, and bundle timestamps from the test
  clock and the seven-day lookback. `60718e5b1b` waits for new tutorial agents in
  STARTING or DONE as well as RUNNING, includes completed rows, falls back to on-disk
  metadata, and adds bounded failure diagnostics. Landing reran 22 healer and tutorial
  unit tests successfully; the slow tutorial e2e was excluded.
- Master Gate **39067090508** fails the deferred-workspace node, the directory operation
  audit, and the separately tracked `sase-1lr` link-follow flake. Retained complete log:
  `file:explicit:c50fd282ffcf1c8eb77aaf14`.
- The deferred test's fake bead claim asserts a nonempty `submitted_at`, but its launch
  fixture supplies none. CI receives `None`. A local run passed with the lander's
  ambient `SASE_LAUNCH_SUBMITTED_AT`; unsetting precisely that variable reproduces the
  CI failure in isolation (1 failed, 5.62 s). The production path reads it in
  `run_agent_runner.py` and persists it through bootstrap and
  `run_agent_runner_launch.py:_promote_bead_claim`.
- The initial local three-node audit (deferred pass with ambient timestamp, rescue audit
  failure, monitor fixture pass) is retained as
  `file:explicit:cafab83a28176d7d9320e9a4`. Do not mistake the first result for a
  clean-environment deferred-workspace pass.
- Concurrent `e19e0403f5` moved rescue deletion into `_rescue_git_repo_body` without
  updating `tests/test_agent_artifact_directory_operation_audit.py`. The audit fails
  independently in isolation with exactly that renamed context.
- Other post-start commits reviewed: `958835abda` (onboarding facade split),
  `dc864a1d70` (SIGTERM handling), `2cae86a1aa` (statistics loader), and `2dd7fe7f8e`
  (mobile test split). Statistics adds `project_stats_query`; release compatibility must
  cover the complete current binding inventory, not just the earlier plan.
- PR **sase#299** remains OPEN, targeting 0.18.0, head
  `40f7c193fee068303782a256d4439c289f867552`. Its release metadata already correctly
  ratcheted to **core 0.41.0**. CI **39067706323** fails because that published wheel
  lacks **mobile_pairing_issue** (1 of 866 required bindings at that PR head). Evidence:
  `file:explicit:f68e1caacef1146e1e83c99a`.
- Core pin **916d5e3a64d9a7f7877170a0087fd53c456e306a** contains that binding and its
  registration. It is after core tag v0.41.0. Core PR **#331** now proposes **0.42.0**;
  CI **39068106880** passes Linux and wheel import smoke but **fails macOS
  compilation**: `crates/sase_core/src/mobile_pairing/fs.rs:441` passes `mode: u32` to
  `libc::mkdirat`, and line 517 does the same to `libc::fchmod`; macOS `mode_t` is
  `u16`. Both produce E0308. Job **117499868212** is the direct evidence. Core's normal
  release cut is daily **07:41 UTC**. Push/heal runs regenerate its PR but do not merge
  it.
- PyPI serves sase **0.17.1**, core **0.41.0**, and no sase 0.18.0. A successful Publish
  run with skipped build/publish jobs is not release evidence. Full CI **39058491506**
  still ran against pre-fix `80f3dc7572` at audit time.

Refresh these facts before acting; other agents may have fixed or published them. The
master dependency window `>=0.35.0,<0.36.0` is intentionally ratcheted on the release
branch by `tools/ratchet_core_window`. Do not change it just because an editable
checkout is newer. The observed blocker is an unpublished capability, not a failure of
the release-branch reconciler.

## Phase core

1. Open `sase-core` with `/sase_repo`, read its `AGENTS.md`, and check current master
   and release PR #331 for a concurrent fix. Read the macOS log for run **39068106880**,
   job **117499868212**. This is a deterministic compiler error, not a flaky test or
   permission-policy choice.
2. Repair the Unix syscall boundary in `crates/sase_core/src/mobile_pairing/fs.rs` so
   `mkdirat` and `fchmod_fd` pass `libc::mode_t` correctly on Linux and macOS. Trace all
   callers/constants and keep exact owner-only directory/file mode bits, no-follow
   checks, ownership checks, and fail-closed error handling. Use native mode types or a
   justified checked conversion at the boundary; do not add an unchecked panic or
   broadly silence compiler/lint errors. Review adjacent Unix mode arguments for the
   same issue.
3. Run the existing focused mobile-pairing tests through the core recipes, formatting,
   and `sase tool run check` from that checkout. Use the repository's check scripts,
   never bare cargo. Confirm supported target compilation where available; do not claim
   a local Linux check proves macOS. The subsequent watch verifies macOS CI.
4. Do not edit Cargo/package versions, changelogs, release tags or the release PR.
   release-plz regenerates it. Record what changed and verified, then finalize so the
   host commits the linked-core fix. Do not wait for this turn's own SHA. No sase source
   pin edit belongs in this phase, since its core commit is not yet available. New
   unrelated findings go in phase notes.

## Phase gates

1. Fetch and fast-forward the clean workspace; read this phase and parent notes. Review
   the latest completed Master Gate and Full CI failures. Preserve fixes that landed
   concurrently. Use `/sase_repo` for every linked checkout. Verify the core phase's
   host-created commit exists on core master, then move `sase-core-revision.txt` past
   that fix with the normal ratchet and validate the resulting binding/contract cohort.
   Do not pin an uncommitted core tree.
2. In
   `tests/test_axe_run_agent_runner_deferred_workspace_flow.py::TestDeferredWorkspaceFlow::test_deferred_wait_gates_before_claim_and_prepares_claimed_workspace`,
   provide an explicit valid submission timestamp through the real fixture launch
   environment and assert its exact propagation. Review the common runner fixture and
   `tests/_conftest_environment.py` to remove reliance on ambient submission state. Test
   both unset and hostile ambient values. Preserve the production legacy no-stamp
   semantics; do not make the fake accept `None` or invent a production submission time
   merely to satisfy the test.
3. Read `_rescue_git_repo_body` in `src/sase/workspace_provider/rescue.py`, confirm what
   its deletion targets and protection rules are, and move the reviewed-site entry from
   `_rescue_git_repo` to `_rescue_git_repo_body`. Retain or correct the exemption
   explanation based on the actual operation. Keep the audit strict.
4. Finish the bounded proof gaps in the previous plan: sweep literal calendar stamps
   under `tests/` that feed wall-clock-relative age/TTL/lookback comparisons for
   fixtures already aged out or expiring before 2026-11-17. Leave parsing-only and
   explicitly frozen-clock fixtures alone. The earlier clock agent recorded only
   adjacent auto-restart coverage. Record the sweep and any fixes. Run the tutorial e2e
   on Python 3.12 with `CI=1`, fresh HOME and a published/pinned-compatible core, at
   least ten times as the previous plan required; retain diagnostic output if it fails.
   Six prior local runs were useful but did not complete that proof.
5. Fix any other confirmed deterministic current release gate introduced by drift,
   within this bounded repair. Separate unrelated local-only failures and flakes. Do not
   edit `sase-1ht` or `sase-1lr`, add skips/xfails, weaken assertions, increase test
   timeouts, or retry a failing tutorial step. If new work exceeds this phase, plan it
   honestly rather than closing it unfinished.
6. Run the focused modules and the clean-environment deferred reproduction, then
   `just fix` and **`sase tool run check`** (the wrapper for `just check`). Never run
   `just check-full`. Read `lint_and_test.md` and other applicable memory first. Use
   `/sase_monitor` for commands that cannot fit the turn. Record the result, any known
   unrelated failures, and all new discoveries as `PROPOSED FOLLOW-UP:` notes. Finish
   the phase normally so host completion commits the repairs.

## Phase watch

Make no source edits. Read `/sase_monitor` before starting the monitor. The sole allowed
Justfile change is the epic-symbol fallback below; if used, finish that turn without
starting the watch so the host can commit it. The phase uses the normal size-derived
provider routing. Foreground handoff completion and the absence of a follow-up are
mandatory regardless of provider.

1. Idempotence: if this phase is already closed, use `/sase_final` and finish. Otherwise
   fetch/fast-forward and confirm the gates phase's commit is in `origin/master` (or its
   explicit no-change proof). Set **FIXED_SHA** to the current tip. If the host push is
   briefly pending, use the monitor workflow to wait; do not end a turn with a live
   yielded command. Missing prerequisites must produce an explicit canceled phase with
   evidence, never a false success.
2. Resolve the compatible core target. Open `sase-core` through `/sase_repo` and check
   that the pending release contains the pin and every new binding needed by FIXED_SHA.
   At baseline the target is **0.42.0**, core PR #331; 0.41.0 is demonstrably
   insufficient. If the containing release is published, validate its downloaded wheel
   against the full required-binding and contract checks in isolation. If still pending,
   use its verified release version as **CORE** for `--await-core-version` and let the
   daily release-plz cut publish it. Do not choose the newest incompatible published
   wheel, manually bump versions, or dispatch a core release. If a different core
   release is required after drift, document the containing commit/version proof. The
   release-floor smoke remains authoritative once that version is on PyPI.
3. Run `sase bead epic-symbols <watch-bead>`. It must be empty. If any entries exist,
   re-key only those lines to this child epic's still-open bead, cancel this phase with
   an explanatory note, and finalize the edit; the lander restarts the watch. Never key
   a whitelist entry to a watch bead.
4. If 0.18.0 is already published, verify the release proof below and close this phase
   with it. Otherwise run an inline preflight:

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --once --dry-run --close-bead <watch-bead>
   ```

   Require exit 0 and a parsed state line. Failure means cancel the phase as
   `WATCHER_BROKEN (preflight)` with the evidence and finalize, not launch blind.

5. Append the chosen SHA, CORE, preflight output and full command to the phase. Start
   exactly one monitor with no successor:

   ```sh
   sase monitor start -C "$PWD" -r 'Watch compatible core and sase 0.18.0; watcher closes this watch phase' -t 14h -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' -L 'sase 0.18.0 compatible-core watch' -- 'PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 10m 750m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --deadline 12h --close-bead <watch-bead>; rc=$?; echo "RELEASE_WATCH_EXIT=$rc"; sleep 120; sase bead close <watch-bead> -R canceled -r "release not confirmed: watcher exit $rc left the bead open" || true; exit 0'
   ```

   Replace every placeholder before launch. Do not pass `--next`, `-m`, `-p`, or
   `--idle-timeout`. Run `sase monitor start` in the foreground until it exits; poll any
   yielded session to its exit code. A successful handoff ends this turn mechanically
   and needs no final declaration. If start fails twice, cancel the phase with
   `WATCHER_NOT_STARTED`, retain both errors, and finalize.

The watcher closes this phase on every verdict; the wrapper closes it if the watcher
crashes or fails to close, then exits 0 so the monitor session settles and the lander
runs. The 12 h watcher deadline, 12.5 h outer timeout with 10 min grace, fallback pause,
and 14 h monitor timeout preserve the existing ordering. A stopped monitor settles
through the existing stop path. Host loss/SIGKILL needs explicit relaunch; do not claim
that can be guaranteed away. A deadline or canceled watch is not a successful release.

## Landing and continuation

The child lander must confirm: ci_watch merged sase PR #299; GitHub release v0.18.0
exists; actual Publish build and publish jobs passed; PyPI lists 0.18.0; a fresh
isolated **unpinned** install resolves sase 0.18.0 and records its core version; and the
Master Gate and recent Full CI relied on for release are green. Retain the watch
verdict, job IDs, versions, and install output. Recheck core compatibility against any
post-repair additions. A pre-fix Full CI or skipped publish is insufficient.

Do not close on `CORE_FLOOR_BLOCKED`, a merely closed watch bead, or old success
evidence. A transient wait with no edits may be resumed through `/sase_monitor`;
remaining code repair requires the normal plan workflow. Honor all follow-up
dispositions already recorded on **sase-1lh.7.5.4.3.3** and triage only new evidence.

This plan intentionally has no parent-close, Symvision, or parent-plan-status phase. Its
`parent_bead: sase-1lh.7.5.4.3.3` hands the successful child landing back to that
interrupted landing. The child lander follows the standard closeout for its own epic,
then fully rechecks the parent and directly parented plan ancestors before any normal
close, removing or re-keying whitelist entries first and marking each linked plan done
only after its scope is complete. The parent plan is
`plan:202610/green_gates_ship_0_18_0.md`; its direct parent is **sase-1lh.7.5.4.3**,
then **sase-1lh.7.5.4**, **sase-1lh.7.5**, **sase-1lh.7**, and **sase-1lh**. Read their
previous landing notes, descendants and linked plans and rerun readiness checks.
Preserve the root's saved Install smoke heal/visual triage and approved supersession of
`sase-1kx` and `sase-1jz`. Never force a successful landing; stop and note the first
incomplete or ambiguous ancestor.
