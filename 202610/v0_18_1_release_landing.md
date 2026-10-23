---
tier: epic
title: Finish the sase v0.18.1 release blockers left by sase-1nn and watch it to PyPI
goal: 'Master Gate and Full CI are green on a tip that contains the custom-gate resize
  wait fix, the hermetic TUI profiling env, and an exact sase-core-rs 0.42.3 pin;
  ci_watch merges release PR #321; Publish uploads sase 0.18.1 to PyPI; and the lander
  that observes it records the evidence artifact and closes this epic and sase-1nn.

  '
parent_bead: sase-1nn
phases:
- id: ci-reds
  title: Fix the CI-only modal resize wait and the env-dependent perf golden
  depends_on: []
  size: medium
  description: 'ci-reds: make the custom-gate sizing resize wait prove the screen
    reflowed under Textual 8.2, scrub ambient TUI profiling env in tests, and regenerate
    the one perf golden captured under the leaked env.'
- id: core-cut
  title: Cut sase-core-rs 0.42.3 and raise sase's exact core pin
  depends_on: []
  size: medium
  description: 'core-cut: dispatch the guarded sase-core cut for v0.42.3, wait for
    a complete 0.42.3 on PyPI, ratchet sase''s ==0.42.2 pin, and prove the floor smoke
    locally.'
- id: watch
  title: Watch v0.18.1 through ci_watch, Publish, and PyPI with a self-closing monitor
  depends_on:
  - ci-reds
  - core-cut
  size: small
  description: 'watch: make no repo edits; verify prerequisites, then start one no-follow-up
    monitor whose watcher script always closes this bead and exits 0.'
proposed_by: bbugyi200.athena.sase-1nn.land
proposed_with: claude/opus
create_time: 2026-10-23 18:32:20
status: wip
bead_id: sase-1nn.8
---

- **PROMPT:** [prompts/202610/v0_18_1_release_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_1_release_landing.md)
- **PARENT:** [202610/v0_18_1_release.md](https://github.com/sase-org/sase--plans/blob/main/202610/v0_18_1_release.md)
- **BEAD:** [sase-1nn.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nn/sase-1nn.8.md)

# Finish the sase v0.18.1 release blockers left by sase-1nn

This is a nested landing epic under **sase-1nn** ("Green master's release gates and ship
sase v0.18.1 to PyPI through ci_watch", plan `plan:202610/v0_18_1_release.md`). sase-1nn
fixed the original four blockers, but 0.18.1 is still not on PyPI. Its watch phase
(sase-1nn.7) never ran, and three new blockers were found during its landing. This epic
covers only those remaining blockers, plus a final self-closing watch.

**Target.** The target is sase **0.18.1** through release-please PR **#321**
(`chore(master): release 0.18.1`, branch `release-please--branches--master`). It merges
only through `ci_watch`, the AXE job on athena (config in chezmoi
`home/dot_config/sase/sase_athena.yml`, code in bugyi-chops). Every 5 minutes it merges
a green release-please PR when all of these hold:

1. **Master Gate** is green at the exact master HEAD.
2. The newest completed **Full CI** run on master is green and at most 6h old.
   `ci_watch` dispatches Full CI itself when the newest run is more than 2h old and none
   is in flight.
3. Every check on the release PR is green (SKIPPED counts as green).

After the merge, the next `Publish` run (`publish.yml`) tags, builds, smoke-tests, and
uploads to PyPI. Publish runs on a 3-hourly cron or on
`gh workflow run publish.yml -R sase-org/sase --ref master -f publish_existing=false`.
Never use `release_watch --live` and never dispatch `release.yml` in this epic.

The parent's decisions are final and apply here unchanged: **publish_dispatch = yes**
(the watch dispatches Publish to refresh the release PR and after the merge) and
**gate_rerun = yes** (the watcher reruns a failed Master Gate at master HEAD once per
run).

## Evidence (observed 2026-10-23 by the sase-1nn land agent, master `2d1e7fd486`)

| #   | Blocker                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Evidence                                                                                                                                                                                                                                                                                                                                                          | Phase    |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| 1   | **Master Gate shard 8 and every Full CI test lane are red on one node**: `tests/ace/tui/test_custom_gate_modal_sizing.py::test_compact_scroll_refits_after_resize` (`wait_for(...) timed out after 5.0s` on `scroll.size.height > short_height`). sase-1nn.4's fix (`1a3a7022d1`) passes locally but not in CI. **Root cause:** the fast CI lanes run `uv pip install --no-sources -e ".[dev]"` without the lock, so they resolve **textual 8.2.8**. Workspace venvs and the `visual` extra stay on 8.0.1. In 8.2, `App._on_resize` debounces the screen `Resize` behind a 1/120s timer, and `Screen.size` is derived from `App.size`. The test's `_wait_for_terminal` checks `app.screen.size`, so it passes before the screen reflows. `footer_fits()` already holds at the stale 160x50 layout, so `short_height` reads the full 39-row scroll. The 80x24 `Resize` is then overwritten by the 160x50 one and never reaches the screen. | Master Gate 40021072711 (HEAD 62c5934eb6) job 120645673493. Full CI 40024191990: 3.13 and 3.14 each fail only this node, out of about 57,000 tests. Local repro: 5/5 failures in a fresh `.[dev]` venv (textual 8.2.8) and 6/6 passes on 8.0.1. Lander-verified fix: 20/20 serial on both versions, `tests/ace/tui/test_custom_gate_*` under `-n 8` 3/3 on 8.2.8. | ci-reds  |
| 2   | **Full CI `visual-test` is red on one golden**: `config_center_statistics_perf_degraded_120x40` (`updated`, 100/1,520,532 pixels; the perf panel's scrollbar thumb ends one sub-cell earlier). sase-1nn.5 regenerated it in `62c5934eb6` inside an agent shell that exports `SASE_TUI_PERF=1` and `SASE_TUI_GC_CENSUS=1`. ACE's tmux launcher seeds them through `_PROFILING_ENV_DEFAULTS` in `src/sase/main/ace_tmux_support.py`, and `tests/conftest.py` never scrubs them. CI never sets them, so the perf panel renders an extra "SASE_TUI_PERF off" line and the thumb shrinks. The same leak causes three recurring local KNOWN failures (task **sase-1np**).                                                                                                                                                                                                                                                                       | Full CI 40024191990 job 120662547013: `counts: created=0 updated=1 unchanged=1116 stale=0`. Locally, `TZ=UTC tools/fix_tui_screenshots --check -- <that node>` gives `unchanged=1` with the env exported and `updated=1` with `env -u SASE_TUI_PERF -u SASE_TUI_GC_CENSUS`.                                                                                       | ci-reds  |
| 3   | **Release PR #321 `release-core-floor-smoke` is red again**: `sase_core_rs 0.42.2 is missing 1 of 876 required binding(s)`. Non-epic commit `5d982d2bb9` (feat(beads): collect stranded-work facts) moved `sase-core-revision.txt` to `58aca5fe6915` and calls `require_rust_binding("stranded_work_reduce")`. 0.42.2 lacks it. sase-core release-plz PR **#334** (`chore: release v0.42.3`, CLEAN) contains `58aca5fe`. 0.42.2 is the newest sase-core-rs on PyPI.                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | PR #321 head `c31a6595ba`, floor-smoke job 120664321990.                                                                                                                                                                                                                                                                                                          | core-cut |
| 4   | **sase-1nn.7's watcher never ran**: its wrapper used `timeout -k 5m 10h30m ...`. GNU `timeout` rejects compound durations (`invalid time interval '10h30m'`, exit 125), so the wrapper's fallback canceled the bead after 65s.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Monitor `1hn3r3rdbyxr` output: `RELEASE_WATCH_EXIT=125`.                                                                                                                                                                                                                                                                                                          | watch    |

Already fixed by sase-1nn and verified green in CI: build-core ordering, the Publish
`GH_TOKEN`, the sase-core release-plz jq filter, the 0.42.2 pin, the `PYTEST_ADDOPTS -p`
selector bug, and every other visual golden (1116 unchanged).

## Rules for every phase

- Read `lint_and_test.md` and `sase_beads.md` through `/sase_memory_read`. For TUI or
  visual work, also read `tui.md` and `tui_screenshot.md`. Read `symvision.md` if a
  symvision gate fails.
- Fetch and re-inventory on the current master tip before editing. Other work lands
  continuously, so the Evidence above is a starting point, not a whitelist.
- **Reproduce CI's dependency resolution, not just the workspace venv.** The fast CI
  lanes resolve the newest Textual (task **sase-1cs** tracks that drift). For any TUI
  timing fix, also prove it in a scratch venv outside the repo:
  `uv venv --python 3.12 <tmp>/venv`, then
  `uv pip install --python <tmp>/venv/bin/python --no-sources -e ".[dev]"` from the repo
  root. Run pytest with `<tmp>/venv/bin/python -m pytest` from the repo root. Expect
  about 20 unrelated failures in that venv on a full `tests/ace/tui` run (no required
  plugins, plus the env leak). Judge only the nodes you changed.
- **Fix waits and fixtures, never tolerance.** Do not raise timeouts, add retries around
  assertions, skip, xfail, loosen pixel comparison, or delete assertions. Wait on the
  semantic state the assertion is about, never on a fixed `pilot.pause(<seconds>)`.
- Never copy CI candidate PNGs into the tree. Regenerate a golden only after its test is
  deterministic.
- Shared domain behavior belongs in sase-core (`rust_core_backend_boundary`). None of
  these fixes needs a Rust change. Work in sase-core only through
  `sase repo open sase-core`.
- Verify with `just fix`, then `sase tool run check`, in every checkout you changed. Do
  not run `just check-full`: Full CI on GitHub is this epic's exhaustive signal. The CI
  `lint` job enforces `toobig`, so do not grow a file past its line limit.
- Hand every long command to `/sase_monitor`: full visual runs, verification, CI waits,
  PyPI waits. Never end a turn waiting.
- **No hand merges, no new PRs.** #321 merges only through `ci_watch`, and sase-core
  #334 merges only through its own `Merge release PR` job. Dispatching workflows
  (`gh workflow run`) and rerunning failed jobs (`gh run rerun --failed`) are allowed
  where a phase says so.
- Phase workers create no beads. Record `PROPOSED FOLLOW-UP: <summary — detail>` notes
  on your own phase bead. Name every existing task bead your evidence resolves, so the
  land agent can close it. Before closing, run `sase bead epic-symbols <own-phase-id>`
  (it must list nothing). Then close with `sase bead close <id> --note '<evidence>'` and
  land through `/sase_final`. If you cannot finish, still close the bead with
  `-R canceled` and the evidence, so the land agent always runs.

## Phase ci-reds

1. **Resize wait.** In `tests/ace/tui/test_custom_gate_modal_sizing.py`,
   `_wait_for_terminal` must prove that the active screen has laid out at the new size,
   not only that `App.size` changed. Replace `pilot.app.screen.size` with
   `pilot.app.screen.outer_size` in `terminal_matches`. `Widget.outer_size` is `_size`,
   which only the screen's own `_refresh_layout` updates. Rewrite the docstring to
   explain this: Textual holds the App-level `Resize` before forwarding it to the screen
   (until `_on_idle` in 8.0, behind a 1/120s debounce timer since 8.2), and since 8.2
   `Screen.size` derives from `App.size`. Neither size proves the screen reflowed, so a
   geometry predicate that already holds at the stale layout passes before the resize
   lands. The lander verified exactly this one-line change: 20/20 serial on textual
   8.0.1 and 8.2.8, and 3/3 `-n 8` runs of `tests/ace/tui/test_custom_gate_*` on 8.2.8.
   Keep every other assertion as is. No product change is needed: the modal's
   `on_resize` refit runs once the screen actually receives `Resize`. Audit
   `tests/ace/tui/test_custom_gate_*` and the other sizing tests that use a
   `screen.size`-based terminal wait for the same vacuous wait, and fix only real
   instances.
2. **Hermetic profiling env.** Add an autouse fixture in `tests/conftest.py`, next to
   the existing ambient-env scrubbing, that `monkeypatch.delenv(..., raising=False)`s
   `SASE_TUI_PERF` and `SASE_TUI_GC_CENSUS`. Decide whether `SASE_TUI_TRACE` (also
   seeded by `_PROFILING_ENV_DEFAULTS`) and `SASE_TUI_PERF_PATH` belong in the same
   list, and include them only if a test reads them ambiently. Tests and benches that
   need these variables already `monkeypatch.setenv` them (for example
   `tests/ace/tui/_bench_tui_jk_helpers.py`). Grep for any test that relies on an
   ambient value and convert it to an explicit `setenv`. Prove that, with both variables
   exported, these nodes pass:
   - `tests/ace/tui/util/test_gc_policy.py::test_freeze_happens_once`
   - `tests/ace/tui/util/test_gc_policy.py::test_census_off_by_default`
   - `tests/ace/tui/test_statistics_perf.py::test_populated_perf_renderable_covers_every_panel`

   This resolves task **sase-1np**; name it in your close note. `test_app_import_budget`
   is not part of this phase; it belongs to sase-1ic.

3. **Perf golden.** With the scrub in place, regenerate only
   `tests/ace/tui/visual/snapshots/png/config_center_statistics_perf_degraded_120x40.png`.
   Run the targeted `just fix-tui-screenshots` on
   `tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py::test_config_center_statistics_perf_degraded_png_snapshot`
   with `TZ=UTC`. Inspect the update: the only expected difference is the perf panel's
   scrollbar thumb length. If any other golden changes, stop and trace it before
   accepting it.
4. **Proof.**
   - Run the resize node 20 times serially in both the workspace venv and the CI-like
     scratch venv.
   - Run `tests/ace/tui/test_custom_gate_*` 3 times under `-n 8` in the scratch venv.
   - Run the full `just fix-tui-screenshots --check` with `TZ=UTC` through
     `/sase_monitor`, with `SASE_TUI_PERF=1 SASE_TUI_GC_CENSUS=1` exported to prove the
     scrub covers the visual harness. It must report `clean` with
     `created=0 updated=0 stale=0` and no failed tests.
   - Run `sase tool run check`.

   Record the pass counts and the check report.

## Phase core-cut

Goal: a complete `sase-core-rs` release whose tag contains master's
`sase-core-revision.txt` is on PyPI, and sase master pins it exactly. At planning time
that is **0.42.3** (sase-core PR #334, containing `58aca5fe`).

1. Open sase-core with `sase repo open sase-core -r '<why>'` and fetch. Read master's
   current `sase-core-revision.txt` in sase. If PyPI already serves a sase-core-rs
   version whose tag contains that revision, with its full file set, skip to step 4. The
   full file set is the `EXPECTED_DIST_SUFFIXES` in that tag's `release-plz.yml`:
   manylinux_2_28 x86_64 and aarch64, universal2, win_amd64, and the sdist.
2. Confirm the open release-plz PR. At planning time it is #334
   (`chore: release v0.42.3`, branch `release-plz-2026-10-23T19-04-34Z`). It must be
   open and green, and it must contain the source revision. If release-plz has since
   retargeted it, use the version it now names everywhere below. Then dispatch the
   guarded cut, which is the same argv the steward's `dispatch_core_cut` uses:
   `gh workflow run release-plz.yml -R sase-org/sase-core --ref master -f dry_run=false`.
   The `Merge release PR` job waits for checks (up to 40 minutes) and merges. The
   resulting push runs `Release-plz`, which tags, builds the wheel matrix, and
   publishes.
3. Wait through `/sase_monitor`, with `--next` and a timeout of about 3h. Poll until
   `https://pypi.org/pypi/sase-core-rs/<version>/json` exists and lists every expected
   suffix, or until a release-plz run for that cut fails. In the follow-up:
   - If the merge job failed on a small, clear script or workflow bug, fix it in
     sase-core, land it, and record a `PROPOSED FOLLOW-UP` asking the land agent to
     re-dispatch the cut. You cannot re-dispatch on your own unlanded fix.
   - If a wheel leg failed, or the file set is incomplete, record the job IDs and logs
     as `PROPOSED FOLLOW-UP`.
   - In either case, close `-R canceled` with the evidence and run `/sase_final`.
4. In sase, run `python tools/ratchet_core_pin` with the flags its `--help` documents
   for a local master ratchet, then `uv lock` if the tool did not refresh the lock. It
   must rewrite the pyproject dependency to the new exact pin and refresh `uv.lock`.
   Keep `sase-core-revision.txt` unchanged.
5. If master's `sase-core-revision.txt` moved again past the cut while you waited, and
   the new revision needs bindings the cut lacks, run steps 2-4 once more for the next
   release-plz PR. If it is still behind after that, close `-R canceled` with the
   evidence.
6. Reproduce the floor smoke locally. Do it exactly as the `release-core-floor-smoke`
   job in `.github/workflows/ci.yml` does: a fresh `uv venv --python 3.12` outside the
   repo, `uv pip install -e '.[dev]' "sase-core-rs==<version>"`,
   `tools/check_sase_core_rs_bindings`, then `tools/validate_sase_core_rs`. It must
   report 0 missing bindings. Paste the output into your close note.
7. Run `just fix` and `sase tool run check`, then land.

## Phase watch

This phase makes **no repo edits**. It ends by starting one monitor with **no
`--next`**, so no LLM turn runs after it and nothing can hang. The watcher script closes
this bead itself, and the monitored command always exits 0. A monitor member that
settles `completed` or `stopped` with no follow-up satisfies the land agent's
agent-session wait.

1. **Idempotent start.** `sase bead read` this bead. If it is already closed (a relaunch
   after recovery), run `/sase_final` and stop.
2. **Already released?** If `curl -fsS https://pypi.org/pypi/sase/0.18.1/json` succeeds,
   close this bead `--note` with that evidence and run `/sase_final`.
3. **Prerequisites.** Fetch. For each sibling phase, read its bead.
   - A phase closed `canceled` means a prerequisite failed. Close this bead
     `-R canceled -r "release watch not started: <phase> canceled"` with its close note,
     then run `/sase_final`.
   - Confirm each `done` sibling's commit is on `origin/master`
     (`git log origin/master --grep 'SASE_BEAD=\[<phase-id>\]'`). A phase whose note
     says no commit was needed is fine. Re-fetch for up to about 5 minutes before
     declaring a commit missing.
   - Confirm master's `pyproject.toml` pins a `sase-core-rs` that is on PyPI, and that
     `tools/check_sase_core_rs_bindings` passes against it. Reuse core-cut's
     temporary-venv recipe outside the repo.
   - Confirm `gh pr view 321 -R sase-org/sase --json title,state` is still the open
     `chore(master): release 0.18.1`. If release-please retargeted the version or
     replaced the PR, close
     `-R canceled -r "release watch not started: release target changed"` with the
     evidence.
   - Confirm the release path has not been cut over. Read `sase-1n6.9` ("Cut releases
     over to release.yml and hand merges to the steward"). If it is closed `done`,
     `ci_watch` and `publish.yml` may no longer merge and publish #321. In that case,
     close `-R canceled -r "release watch not started: release path cut over"` with the
     evidence.

   Any failure closes this bead `-R canceled` with the evidence, followed by
   `/sase_final`. Do not repair code in this phase.

4. **Epic-symbol guard.** `sase bead epic-symbols <this-phase-id>` must list nothing.
   Otherwise, close `-R canceled -r "release watch not started: epic-symbol entries"`
   and run `/sase_final`.
5. **Refresh the release PR.** If #321's head commit predates master's pin commit from
   core-cut, or its `release-core-floor-smoke` check is red, run
   `gh workflow run publish.yml -R sase-org/sase --ref master -f publish_existing=false`
   once. Release-please then rebuilds the release branch from master, and
   `sync-release-metadata` reconciles it. Do not wait for the result here; the watcher
   observes it.
6. **Write the watcher outside the repo.** Use
   `W=/tmp/sase-release-watch-<this-phase-id>` and `mkdir -p "$W"`. Write `$W/watch.sh`
   with a quoted heredoc (`<<'EOF'`), so nothing expands at write time:

   ```sh
   #!/bin/sh
   # sase v0.18.1 release watcher: sh watch.sh BEAD LOG [preflight]
   # Never edits files. In watch mode it closes BEAD itself and always exits 0.
   BEAD=$1 LOG=$2 MODE=${3:-watch}
   REPO=sase-org/sase VERSION=0.18.1
   TITLE="chore(master): release $VERSION"
   unset GH_FORCE_TTY
   export NO_COLOR=1 CLICOLOR=0 CLICOLOR_FORCE=0 FORCE_COLOR=0
   START=$(date -u +%Y-%m-%dT%H:%M:%SZ)
   END=$(( $(date +%s) + 36000 ))
   outcome=DEADLINE detail= dispatched= reruns=
   say() { printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*" | tee -a "$LOG"; }
   while :; do
     if curl -fsS -o /dev/null "https://pypi.org/pypi/sase/$VERSION/json" 2>/dev/null; then
       outcome=RELEASED; break
     fi
     merged=$(gh pr list -R "$REPO" --base master --state merged \
       --search "in:title \"release $VERSION\"" --json title,mergedAt \
       --jq "[.[] | select(.title == \"$TITLE\")][0].mergedAt // \"\"" 2>>"$LOG")
     pub=none
     if [ -n "$merged" ]; then
       pub=$(gh run list -R "$REPO" --workflow publish.yml --limit 10 \
         --json databaseId,createdAt,status,conclusion \
         --jq "[.[] | select(.createdAt > \"$merged\")] | sort_by(.createdAt) | last
               | if . == null then \"none\" else \"\(.databaseId):\(.status):\(.conclusion)\" end" 2>>"$LOG")
       if [ "$pub" = none ] && [ -z "$dispatched" ] && [ "$MODE" = watch ]; then
         gh workflow run publish.yml -R "$REPO" --ref master -f publish_existing=false >>"$LOG" 2>&1 && dispatched=yes
       fi
       case "$pub" in
         *:completed:failure|*:completed:cancelled|*:completed:timed_out|*:completed:startup_failure)
           outcome=PUBLISH_RED detail="Publish run $pub"; break ;;
       esac
     fi
     head=$(gh api "repos/$REPO/commits/master" --jq .sha 2>>"$LOG")
     gate=$(gh run list -R "$REPO" --workflow master-gate.yml --commit "$head" --limit 1 \
       --json databaseId,status,conclusion,attempt \
       --jq '.[0] | "\(.databaseId):\(.status):\(.conclusion):\(.attempt)"' 2>>"$LOG")
     case "$gate" in
       *:completed:failure:1)
         id=${gate%%:*}
         case " $reruns " in *" $id "*) ;; *)
           reruns="$reruns $id"
           [ "$MODE" = watch ] && gh run rerun "$id" --failed -R "$REPO" >>"$LOG" 2>&1 ;;
         esac ;;
     esac
     full=$(gh run list -R "$REPO" --workflow full.yml --branch master --limit 10 \
       --json databaseId,status,conclusion,createdAt,headSha \
       --jq "[.[] | select(.status == \"completed\" and .createdAt > \"$START\")][0:2]
             | map(\"\(.databaseId):\(.conclusion):\(.headSha[0:10])\") | join(\",\")" 2>>"$LOG")
     case "$full" in *:failure:*,*:failure:*) outcome=FULL_CI_RED detail="Full CI $full"; break ;; esac
     say "state merged=${merged:-no} publish=$pub head=${head:-?} gate=${gate:-?} full_since_start=${full:-none}"
     [ "$MODE" = preflight ] && exit 0
     [ "$(date +%s)" -ge "$END" ] && break
     sleep 300
   done
   say "outcome=$outcome $detail"
   [ "$MODE" = preflight ] && exit 0
   if [ "$outcome" = RELEASED ]; then res=done reason="sase $VERSION confirmed on PyPI"
   else res=canceled reason="release not confirmed: $outcome${detail:+ ($detail)}"; fi
   tail -n 60 "$LOG" > "$LOG.note"
   i=0
   while [ "$i" -lt 3 ]; do
     sase bead close "$BEAD" -R "$res" -r "$reason" -n "@$LOG.note" && break
     i=$((i + 1)); sleep 30
   done
   exit 0
   ```

   The script has four outcomes:
   - `RELEASED`: PyPI serves 0.18.1. The bead closes `done`.
   - `PUBLISH_RED`: a post-merge Publish run failed.
   - `FULL_CI_RED`: two consecutive Full CI runs since the start failed. That releases
     the land agent after about 4h instead of 10h.
   - `DEADLINE`: 10h passed.

   Every outcome except `RELEASED` closes the bead `canceled`, with the last 60 log
   lines as the note.

7. **Preflight inline.** Run all three of these:
   - `sh -n "$W/watch.sh"`.
   - `timeout -k 5m 630m sh -n "$W/watch.sh"`, which proves the exact `timeout`
     durations parse. sase-1nn.7 died on `10h30m`, which GNU `timeout` rejects.
   - `sh "$W/watch.sh" <this-phase-id> "$W/watch.log" preflight`.

   Each must exit 0, and the last must print one fully populated `state` line (or
   `outcome=RELEASED`). Preflight never dispatches, reruns, or closes. If the line has
   `?` fields or a command fails, fix the script's gh or jq invocation only, without
   touching the repo, and retry once. If it still fails, close
   `-R canceled -r "release not confirmed: WATCHER_BROKEN (preflight)"` with the output,
   and run `/sase_final`.

8. **Record, then hand off.** Append a phase note with:
   - the master HEAD;
   - the core pin;
   - the preflight line;
   - whether Publish was dispatched in step 5;
   - the exact monitor command.

   Then, from the workspace root, run this in the foreground until it exits:

   ```sh
   sase monitor start \
     -C "$PWD" \
     -r 'Watch sase v0.18.1 through ci_watch, Publish, and PyPI; the watcher closes <this-phase-id>' \
     -t 11h \
     -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' \
     -L 'sase 0.18.1 watch' \
     -- 'timeout -k 5m 630m sh /tmp/sase-release-watch-<this-phase-id>/watch.sh <this-phase-id> /tmp/sase-release-watch-<this-phase-id>/watch.log; rc=$?; echo "RELEASE_WATCH_EXIT=$rc"; sleep 60; sase bead close <this-phase-id> -R canceled -r "release not confirmed: watcher exit $rc left the bead open" || true; exit 0'
   ```

   Pass the command as one quoted string with no `bash -c` wrapper. Do **not** pass
   `--next`, `-m`, `-p`, or `--idle-timeout`. A successful start ends this turn, and
   that handoff is exempt from `/sase_final`. If `sase monitor start` exits nonzero
   twice, close `-R canceled -r "release not confirmed: WATCHER_NOT_STARTED"` with both
   errors, then run `/sase_final`.

### Why the land agent always runs

Each layer ends inside the next one out:

1. The watcher stops at 10h.
2. The outer `timeout` stops at 630 minutes (10.5h), plus 5 minutes of kill grace.
3. The fallback close runs after a 60s pause.
4. The monitor's own 11h timeout never fires on a live host.

A fallback close after a successful close is refused as a conflicting re-close, and
`|| true` absorbs that.

| Path                                                   | Who closes the bead                               | What settles the session             |
| ------------------------------------------------------ | ------------------------------------------------- | ------------------------------------ |
| `RELEASED`, `PUBLISH_RED`, `FULL_CI_RED`, `DEADLINE`   | the watcher (3 tries)                             | wrapper exits 0, monitor `completed` |
| watcher hang, or outer `timeout` fires                 | the wrapper's fallback close                      | wrapper exits 0                      |
| watcher crash, or its close fails                      | the wrapper's fallback close                      | wrapper exits 0                      |
| prerequisite, epic-symbol, preflight, or start failure | this agent, inline                                | this agent's `/sase_final`           |
| a human stops the monitor                              | nobody automatically; the land agent can close it | monitor `stopped` counts as success  |
| reboot (`lost`) or an external SIGKILL                 | nobody                                            | a human: `sase bead work <epic-id>`  |

Step 1 makes recovery safe. `sase bead work <epic-id>` relaunches only the land agent if
this bead is closed, and relaunches the watch if it is open.

## Parent handoff

This epic's `parent_bead` is **sase-1nn**, a plan bead. After this epic closes, its land
agent resumes sase-1nn's interrupted landing as its lander prompt directs. Re-verify
sase-1nn's descendants and plan, retire any `--epic-symbol` entries keyed to it, close
it with a note naming this epic and the evidence artifact, run `just symvision`, and set
`status: done` in `plan:202610/v0_18_1_release.md`. The task beads to close with
evidence once their fixes land are **sase-1np** (resolved by ci-reds) and any bead a
phase names as resolved. **sase-1cs** (the Textual install drift) stays open: this epic
hardens one test, not the drift.

## Landing

The land agent confirms the release before it lands anything as done.

1. **If sase 0.18.1 is on PyPI,** verify each item and record it:
   - PyPI (`https://pypi.org/pypi/sase/0.18.1/json`) lists a wheel and an sdist; record
     the filenames, SHA-256 values, and upload times.
   - `gh release view v0.18.1 -R sase-org/sase` exists, along with its tag commit.
   - The merged release PR shows the ci_watch merge
     (`gh pr view <n> --json mergedBy,mergedAt,mergeCommit`).
   - The green Full CI run (ID and SHA) and the green Master Gate run at the merged
     HEAD.
   - The Publish run ID, with every job's conclusion.
   - **Clean install proof.** Outside any repo, with an isolated `HOME`, create
     `uv venv --python 3.12` and run `uv pip install sase==0.18.1` with `UV_NO_CACHE=1`.
     The installed `sase` must report 0.18.1. The `sase-core-rs` version must equal the
     pin. `sase core health --json` must pass.

   Write these into `sase_v0_18_1_release_evidence.md` in a temporary directory outside
   the repo. Register it with
   `sase artifact create -p <file> -l "sase v0.18.1 PyPI release evidence" --bead <this-epic-id>`.
   If this epic is nested, also add a `sase bead note` on the root epic bead that names
   the artifact ref. Close the task beads that phases named as resolved, each with its
   evidence. Then close this epic.

2. **If 0.18.1 is not on PyPI,** read every phase's close note and the watcher log
   (`/tmp/sase-release-watch-<watch-id>/watch.log`, if present), and fix the blockers:
   - Fix small, clearly scoped blockers yourself, under the phase rules above.
   - For anything that needs CI round-trips, a core release, or waiting, author a
     **nested epic** with `/sase_plan`. It covers only the remaining blockers, plus a
     final self-closing watch phase built exactly like the watch phase above:
     prerequisites, preflight, a no-`--next` monitor, and a wrapper fallback close.
   - The nested epic's Landing section must repeat this whole Landing section verbatim,
     so every lander keeps creating nested epics until 0.18.1 is observed on PyPI.
   - Only the lander that actually observes 0.18.1 on PyPI creates the evidence
     artifact.
   - Turn the phases' `PROPOSED FOLLOW-UP` notes into task beads with `/sase_new_task`
     only when they are not release blockers.
