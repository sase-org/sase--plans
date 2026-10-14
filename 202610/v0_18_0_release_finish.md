---
tier: epic
title: Finish v0.18.0 watcher correctness, visual proof, and publication
parent_bead: sase-1kx
goal: The remaining release watcher defects are fixed, the complete visual lane is
  clean, and sase 0.18.0 is published and installable through the guarded release
  flow.
phases:
- id: remaining-fixes
  title: Repair the watcher edge cases and finish the clean visual proof
  description: 'remaining-fixes: fix the three watcher defects, await the requested
    core release, resolve the remaining visual mismatches, and verify the integrated
    tree.'
  depends_on: []
  size: medium
- id: release-wait
  title: Watch the committed fixes through core publication, CI, and sase publication
  description: 'release-wait: make no edits; preflight and monitor the corrected watcher
    on committed fixes, then close this phase on every outcome.'
  depends_on:
  - remaining-fixes
  size: small
proposed_by: bbugyi200.athena.sase-1kx.land
create_time: 2026-10-14 09:33:38
status: wip
bead_id: sase-1kx.5
---

- **PROMPT:** [prompts/202610/v0_18_0_release_finish.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_0_release_finish.md)
- **PARENT:** [202610/v0_18_0_release.md](https://github.com/sase-org/sase--plans/blob/main/202610/v0_18_0_release.md)
- **BEAD:** [sase-1kx.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kx/sase-1kx.5.md)

# Finish the remaining v0.18.0 release work

This plan is a child of `sase-1kx`. Its parent remains open. Read
`plan:202610/v0_18_0_release.md` with `sase artifact read`, and the parent's landing
note and follow-up dispositions with `sase bead read`. Only the remaining work below is
authorized here; do not repeat the completed gate-reds or A/B/C visual repairs. The
complete audit is `file:explicit:fa8533e2fbc2b13048be7360` (use `sase artifact read` to
consume it).

This needs two phases because the release proof must observe the fix phase's **landed
commit** in GitHub CI. A tale's final coding turn cannot wait for its own future commit
and then close the release. The first phase commits through normal host completion; the
second changes no files and only waits. The child land agent resumes the parent landing
through `parent_bead: sase-1kx` after this child is complete. Parent closure is not a
child implementation phase.

The parent's accepted decisions remain final: `core_release=dispatch` and
`stale_epic_1jz=notes`. Never ask them again, manually merge the release PR, kill the
waiting sase-1jz agents, or force-close a successful nested landing.

## Audited starting point

At the landing audit, local HEAD and fetched `origin/master` were
`508eb963a33d745eba229efc753d5082d58f134a`. Master Gate **38567573367** was green there.
The three completed implementation commits are:

- `7710ce5ffa`: stdlib watcher in `tools/release_watch`, `_release_watch.py`,
  `_release_watch_api.py`, `_release_watch_engine.py`, and its tests/docs.
- `c69997274d`: real startup loader, remote fleet fixtures, live Stitches description
  waits, and three corresponding goldens.
- `9f4e6c4430`: facade-before-registry monkeypatch teardown, bare-git registry reset,
  and archive wait diagnostics.

Concurrent `d265233491` already updated the legacy delivery test when Claude became
explicit. `71116606b9`, `f13280fecc`, and `508eb963a3` split Codex helpers, link-index
tests, and policy-IO tests, respectively; their callers are already integrated. Fetch
and inspect new drift before editing; do not redo those changes.

Full CI **38557185012** runs on pre-fix `8e00f6d211`; its visual job **115834392657**
failed the original nine A/B/C tests. It is not proof for this epic. PR **#299** was
still open with a failing `release-core-floor-smoke`. The approved core dispatch
**38568273048** ran with event `workflow_dispatch` and ended canceled. The subsequent
scheduled core Release-plz run **38568657352** is running its Merge release PR job; core
PR **#328** was refreshed to `9343f50e55d84d042acb5ced1d38586816758ba3` and was
UNSTABLE. Do not duplicate a running dispatch. The parent pin is
`a89eac09c29e2f493ef1152c0d6b0f94e3cce51c`; published core was still 0.39.1 in the last
wait. The originally requested floor-satisfying release is 0.40.0.

## Rules

- Read `lint_and_test.md`, `sase_beads.md`, `symvision.md`; for visual work also read
  `tui.md`, `tui_screenshot.md`, and `tui_perf.md`. Use their audited skill paths.
- Use `sase repo open` before accessing another repo, including `sase-core` or the plans
  sidecar. Shared domain fixes, if truly needed, belong in Rust; no Python core
  fallback. This plan expects watcher and test-fixture work only.
- Preserve assertions, pixel equality, and existing timeouts. Fix semantic waits and
  fixture isolation. Never copy CI candidate PNGs into goldens.
- Run `just fix`, then `sase tool run check` (the wrapped `just check`). Do not run
  `just check-full`. A check-full-only infrastructure failure is not remaining release
  implementation work. Full CI remains the release's remote acceptance signal.
- Use `/sase_monitor` for long commands. Do not end a turn with a command still owned by
  the provider, or leave a phase open waiting for its own future commit. Native helpers,
  if used, never submit root declarations.
- Phase workers file no new tasks: append `PROPOSED FOLLOW-UP:` notes to their phase.
  The parent lander already triaged the historical proposals; do not duplicate them.

## Phase remaining-fixes

### Watcher correctness and core-wait integration

The existing 26 watcher unit tests pass, but the landing probes demonstrated these
uncovered failures:

1. `_is_visual_golden_drift` returns true for the literal sentence
   `Visual screenshot check failed with status=failed (not golden drift)` and for the
   real log of job 115834392657, whose counts were all zero. The matcher accepts the
   generic words `golden drift` and `visual snapshot`, including negations and echoed
   shell source. This makes a test-failure run terminate as deterministic `FULL_CI_RED`,
   bypassing the allowed red retry budget.
   - Classify the visual job's **actual executed report status**, not a search over the
     combined failed logs of every job. Follow the current report contract in
     `.github/workflows/ci.yml` and `tools/render_visual_snapshot_failure_report`. If
     necessary emit a small unambiguous machine marker or read the job artifact. Keep
     the watcher stdlib-only; do not add a dependency on the sase install.
   - Only proven `drift` is deterministic visual failure. `failed`, missing reports, a
     test timeout, report headings, quoted script text, and the phrase
     `not golden drift` must not become drift. Preserve real deterministic lint
     detection and the full-run retry budget.
   - Add regression fixtures for the actual CI negation/echo case, true drift,
     test-failure-only, and a mixed-job log that mentions snapshots elsewhere.
2. `ReleaseWatcher.run` catches `WatcherSignal` in the inner startup `except Exception`
   around `reader.validate`. A SIGTERM at that point produces WATCHER_BROKEN / 16 and a
   canceled close reason naming WATCHER_BROKEN. Route it through the signal handler's
   DEADLINE path instead. Test startup and polling signals, verdict written before
   close, bounded retry, and the restored signal handlers.
3. `parse_run` accepts status=completed with conclusion="", returning None. Treat an
   empty/missing or invalid terminal conclusion as malformed required data. Preserve
   GitHub's valid empty conclusion for in-flight runs. Test that repeated malformed
   completed reads advance the per-signal fail-fast counter and close with
   WATCHER_BROKEN.
4. Integrate the selected core dispatch with the wait. Phase sase-1kx.4 requested the
   core release, then immediately closed on the **old** 0.39.1 floor failure while an
   equivalent scheduled core cut was still running. Add an explicit optional watcher
   input such as `-c/--await-core-version 0.40.0` for an already-requested core release.
   While PyPI's stable core version is below that minimum, report `awaiting_core` and
   suppress CORE_FLOOR_BLOCKED from the old floor evidence; continue normal Full CI work
   and preserve deadline/failed-read/ signal close behavior. Once the minimum is
   published, resume the existing floor ratchet and post-ratchet floor-blocked rules
   without suppression. No core workflow dispatch belongs inside the watcher. Use
   numeric stable-version comparison and validate the option, document it, and test old
   core, reached/newer core, normal no-option behavior, malformed input, and deadline
   while awaiting core. Read `cli_rules.md` before adding the option. Do not hard-code
   0.40.0 into policy.

Keep current publication proof and no-dispatch/no-close behavior for `--once` and
`--dry-run`. Every ordinary exit still writes a verdict and performs the requested phase
close; do not use `--close-bead` on an epic.

### Complete the visual acceptance that phase .2 left unfinished

The last full local TZ=UTC run reported **1366 passing tests but 3 mismatched goldens**:

- `agents_session_panel_turns_gate_90x40`, 14707 pixels, footer `(/) blocks` reflow.
  Start with `test_ace_png_snapshots_agents_agent_session_panel_gate.py` and the
  semantic footer/cleanup waits from `4b99e45907` / sase-1jz.3.
- `link_rail_agents_single_link_60x30`, 1282 pixels: `$0 +1 more` versus the expected
  implemented-by chip. Start with `test_ace_png_snapshots_link_rail.py`, its seeded link
  set, and the current rail layout. Distinguish stale intended copy/width from
  asynchronous link arrival before regenerating.
- `plan_gate_tale_decisions_maxcopy_120x40`, truncation seen only in a full run. Start
  with `test_ace_png_snapshots_plan_gate.py`. Pin the final decision-card layout state,
  reproduce under load, and only regenerate when the visible state is stable.

The first two were also reproduced by phase .2 on clean 8e00f6d211. They are still
in-scope because the approved phase promised a wholly clean visual lane. Do not exclude
them just because they predate A/B/C. Inspect any newly recurring older cases
(tool_runs_session_rail, turns_gate cleanup/scrollbar, renamed generic root, archive
list duration, settled monitor lane badge) if the full lane exposes them; do not
speculatively change currently passing tests.

Run targeted updates through `just fix-tui-screenshots -- <selectors>` and inspect all
resulting changes; require `clean` or `applied`, never accept `partial`. Prove host
independence with targeted checks in the default zone and UTC, then run the full
`TZ=UTC just fix-tui-screenshots --check` through a monitor. Require no failed tests and
`clean`, `created=0 updated=0 stale=0`.

The inherited check run 3001ace890e331c81511759a2d3b49c9 had 26 NEW/1 FLAKY after full
escalation. Representative isolated clean passes do not establish that every loaded
failure is unrelated. Run the integrated `sase tool run check`; inspect exact failures
and obtain clean-base or valid triage evidence for any residual. Repair failures
introduced by this work. Record precise remaining unrelated evidence as phase notes.

Finish by recording source changes, test results, visual report path, and any known
failure evidence. Resolve this phase's epic symbols, close this phase normally, and
submit host completion. Do not dispatch a release against uncommitted fixes.

## Phase release-wait

This phase makes **no file edits**. Its job is to wait on already committed work and
mechanically close on success or failure, so the child land agent always runs.

1. Read this phase and its predecessor's notes. Fetch and fast-forward a clean checkout;
   prove `origin/master` contains remaining-fixes and all three original epic commits.
   Save that SHA as FIXED_SHA. Read PR #299's version (expected 0.18.0).
2. Open `sase-core` with `/sase_repo`. Check the pin, release PR, dispatched workflow,
   and published core. Preserve `core_release=dispatch`: if the required core is not
   published, dispatch Release-plz with `dry_run=false` only when no equivalent urgent
   cut is already running. Read current core repo release instructions; never manually
   change versions or merge its PR. If the pin advanced, determine the new stable core
   version that will contain it and use that as the wait minimum.
3. Run `sase bead epic-symbols <own-phase-id>`. Preflight inline using the corrected
   watcher with `--once --dry-run --close-bead <own-phase-id>` plus
   `--await-core-version <required-core>`. Require exit 0 and one fully parsed line with
   all required signals and the close argv. On failure, close the phase normally with
   resolution canceled, reason `release not confirmed: WATCHER_BROKEN (preflight)`, and
   the output, then submit `/sase_final`; make no repair in this wait phase.
4. Start one monitor with a quoted command, adapting the corrected option spelling:

   ```sh
   sase monitor start \
     --reason 'Watch committed release fixes through core, Full CI, guarded merge, Publish, and PyPI' \
     --timeout 13h -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' \
     -o tail --tail-lines 150 --next "$NEXT" \
     -- 'timeout -k 5m 760m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <required-core> --deadline 12h --close-bead <own-phase-id>'
   ```

   The watcher dispatches Full CI only under its fixed-code/in-flight/cooldown rules,
   lets ci_watch perform the guarded #299 merge, and dispatches Publish only under its
   existing ratchet/post-merge rules. Do not bypass release guards.

   Define NEXT with a quoted heredoc and no prompt directives: the follow-up reads the
   verdict and its own bead, closes it if mechanical close failed (done only for
   RELEASED, canceled with the precise verdict otherwise), then submits `/sase_final`.
   It must not edit code, reopen the phase, or restart the monitor. The land agent
   handles every remaining blocker. If monitor start fails, retry once; if both starts
   fail, close canceled with WATCHER_NOT_STARTED and the errors. Wait for the start
   command itself to exit; do not abandon a yielded live session.

The deadline and cancel outcome do not mean publication succeeded. Closing this wait
phase only releases landing to assess the evidence.

## Landing and parent resumption (not implementation phases)

Do not close this child epic unless the release is confirmed and the implementation
above is complete. Independently verify PyPI lists 0.18.0, GitHub release v0.18.0
exists, Publish build and publish jobs succeeded, and a fresh temporary environment
outside the repo with **unpinned** `pip install sase` resolves 0.18.0 and runs
`sase --version`. Record resolved core, Full CI run/SHA/freshness, Master Gate run/SHA,
#299 merge SHA, and Publish run. A closed canceled wait is never release proof.

If only external waiting remains, use the corrected watcher without --close-bead in a
monitor whose follow-up resumes confirmation. If new code is required, plan only that
work with the same tier rule; do not claim completion or force-close anything.

After this child closes normally, the parent link resumes `sase-1kx` landing. Re-read
all parent descendants and notes, `plan:202610/v0_18_0_release.md`, its prior landing
audit, and post-child drift before closing the parent. Complete the parent's explicit
release closeout: verify and close `sase-16p`, `sase-1gh`, `sase-19d`, `sase-1kg`,
`sase-1ii` only where the new evidence resolves each; apply `stale_epic_1jz=notes`
(close .3 only after verification, then note .4/.5 with the green runs); report whether
ci_watch's heavy dispatcher is live. Never touch the user's primary checkout to deploy
it; use the documented optional gate if needed.

Historical proposal dispositions were completed by the parent lander: new tasks
`sase-1ky` (archive expand flake) and `sase-1kz` (general validated CI waits), duplicate
corroborations `sase-172`, `sase-xo`, `sase-1ic`, `sase-1gn`, causal visual note on
`sase-1jz`, and explicit declines/retained work in the parent's audit note. Preserve
these in the final close note and triage only genuinely new evidence.

Once the parent is fully ready, recheck `sase bead epic-symbols sase-1kx`, resolve or
re-key every entry to a still-open consuming bead, then close it **normally**, run
`just symvision`, and mark `plan:202610/v0_18_0_release.md` frontmatter `status: done`
through the sanctioned plans checkout. Re-read its parent link (none at this audit). If
any parent requirement remains incomplete or ambiguous, record a precise blocker on that
parent and stop the ancestor closure there. Never force a successful landing.
