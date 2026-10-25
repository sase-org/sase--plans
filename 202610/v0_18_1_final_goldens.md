---
tier: epic
title: Repair the remaining release goldens and publish sase 0.18.1
goal: The Python 3.12 help and Archive help goldens match current behavior, ci_watch
  merges release PR 321 after green release gates, and Publish uploads sase 0.18.1
  with retained clean-install evidence.
parent_bead: sase-1nn.8
phases:
- id: goldens
  title: Repair the Python 3.12 CLI and Archive help goldens
  description: 'goldens: integrate the stats and stranded-work changes by repairing
    their two deterministic golden failures, then verify and land the repair.'
  depends_on: []
  size: medium
- id: watch
  title: Watch the committed repairs through release gates and PyPI
  description: 'watch: verify the repair commit reached master, refresh the release
    PR when needed, and start a bounded self-closing monitor that releases the land
    agent on success or failure.'
  depends_on:
  - goldens
  size: small
proposed_by: bbugyi200.athena.sase-1nn.8.land
proposed_with: codex/gpt-6-astra
create_time: 2026-10-24 02:46:43
status: done
bead_id: sase-1nn.8.4
---

- **PROMPT:** [prompts/202610/v0_18_1_final_goldens.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_1_final_goldens.md)
- **PARENT:** [202610/v0_18_1_release_landing.md](https://github.com/sase-org/sase--plans/blob/main/202610/v0_18_1_release_landing.md)
- **BEAD:** [sase-1nn.8.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nn/sase-1nn.8.4.md)

# Repair the remaining release goldens and publish sase 0.18.1

This is the remaining work from the **sase-1nn.8** landing audit, not a restart of its
completed implementation. Its parent is **sase-1nn**. Read the retained audit with
`sase artifact read file:explicit:8e0989eee98b211b259bfe9b "Need the landing evidence"`,
and read sase-1nn.8's LANDING AUDIT note. The prior plan is
`plan:202610/v0_18_1_release_landing.md`; the root plan is
`plan:202610/v0_18_1_release.md`.

**Why two phases:** the bounded golden edits fit one coding agent, but the release watch
must run after that agent's end-of-turn commit reaches master. A tale cannot wait for
its own future commit or CI result. The repair phase therefore lands first; the
dependent watch and this child epic's lander finish the operational acceptance. No phase
closes either ancestor epic.

## Verified baseline and remaining work

At planning time, fetched master is `102c6ee847215d0e813532329fb8af48a0ad29b7`. PR
**#321**, `chore(master): release 0.18.1`, remains OPEN at
`d1c884a13c96983aa09b670e4184ca4202128e69`. PyPI sase/0.18.1 returns HTTP 404. The prior
watch **sase-1nn.8.3** closed **canceled**, not successful, after two consecutive failed
Full CI runs. Its timeout and self-closing monitor worked.

Completed and independently verified:

- `43314db7ad`: screen.outer_size waits in both terminal-sizing helpers, a
  PERF/CENSUS/TRACE scrub in tests/conftest.py, and the corrected perf golden.
  Current-master focused verification with all flags exported: 26 passed, including all
  six modal-sizing tests, the three environment nodes, project-label audits, and parser
  tests on Python 3.14. The three env nodes also pass unset.
- `dd0c6d8d19`: pyproject.toml and uv.lock pin **sase-core-rs==0.42.3**. Its tag
  `f726ca1e` contains source revision `58aca5fe69158286092e3a33685687a52f8d2b8f`; all
  five expected distributions are on PyPI. An isolated published-wheel probe finds all
  876 bindings and tools/validate_sase_core_rs exits 0. No new core cut is currently
  needed.
- `a7996c3b77` already fixed the stranded_project_key_for_agent path-label audit with a
  justified identity-only exemption. No remaining repair there.
- Both sase-1nn.8 and sase-1nn currently have no epic-symbol entries.

Two deterministic blockers remain:

1. **Python 3.12 CLI golden**, tracked by **sase-1nt**. Non-epic commit `102c6ee847`
   added the `By repo` definition twice and the repos[] jq example twice in
   `tests/stats/golden/cli/project_help.txt`. Production emits each once;
   `project_help_py313.txt` is already correct. Master Gate **40058175796** attempt 2
   shard 4 (job **120769167380**) and Full CI **40063272183** Python 3.12 (job
   **120783473758**) fail only
   `tests/main/test_parser_stats.py::test_project_help_golden` for this reason. Python
   3.13 and 3.14 Full CI pass.
2. **Archive help PNG**. `bb81495765` (sase-1no.3) added three stranded-work Help legend
   rows but left `agents_archive_help_120x40.png` stale. Latest Full CI visual job
   **120783473777** reports exactly **52,757/1,520,532** changed pixels, matching
   sase-1nn.8.1 note 1 and earlier Full CI **40054807788**. The latest full visual run
   has created=0, updated=1, unchanged=1119, stale=0. The earlier run also reported
   `agents_renamed_generic_session_root_120x40`; that did not recur on the latest run,
   so do not blindly regenerate it.

Non-epic drift since the first parent-epic commit also includes `a2a38005df` (finalizer
split) and `1a5e01adba` (doctor test split); neither conflicts with the release work.
The stats change affects Projects visuals, and its new goldens pass the latest visual
lane. Keep the perf golden repair intact.

## Decisions and constraints

The root's final decisions remain **publish_dispatch=yes**, **gate_rerun=yes**. No new
decision is needed. Release merges belong exclusively to **ci_watch** on athena: Master
Gate green at the exact master HEAD, newest completed Full CI green and at most six
hours old, and all PR checks green (skipped is acceptable). ci_watch dispatches Full CI
if the latest is over two hours old and none is in flight. Publish (`publish.yml`) runs
every three hours or by the authorized dispatch below.

- Never hand-merge #321, create a new PR, dispatch release.yml, run the release steward
  with --live, or change merge authority. Read sase-1n6.9 before watching; it was
  in_progress at planning time. If its cutover has completed, stop the old watch path
  and return that evidence to the lander.
- Fetch and re-inventory master before editing. Other agents can fix these files first.
  If already correct, verify and document the existing commit.
- Read lint_and_test.md and sase_beads.md through /sase_memory_read. Read tui.md and
  tui_screenshot.md before visual work. No memory changes are part of this plan.
- Keep assertions, strict pixels, and timeouts. No skips, xfails, retrying assertions,
  tolerant comparison, or fixed sleep as a layout wait. If a render is unstable,
  establish its semantic ready state before accepting any golden.
- Run `just fix`, then `sase tool run check` for tracked file changes. This is the
  `just check` gate. **Never run just check-full**; GitHub Full CI is the release
  signal. An unrelated check-full-only defect is test infrastructure, not epic work.
- Use /sase_monitor for long commands, full visual runs, or CI/PyPI waits. Do not end a
  turn with an exec session running. Host finalization creates commits; phases do not
  manually commit, branch, or merge.
- Use /sase_repo before any linked repo access, and audited artifact reads for plans.
  Shared backend behavior belongs in Rust; these two repairs need none.
- Phase workers create no task beads. Record PROPOSED FOLLOW-UP notes on the assigned
  phase, and name resolved task IDs. Run `sase bead epic-symbols <own-phase-id>` before
  closing. Close successful work with evidence; if unable to finish, close the phase
  deliberately as canceled with the actual blocker so the lander runs. Never use --force
  to advance this successful nested landing.

## Phase goldens

1. Read your own bead, sase-1nn.8, sase-1nt, and the audit artifact. Fetch and verify
   current failure evidence. Preserve already-landed resize/env/core fixes.
2. Repair only the duplicate expected lines in
   `tests/stats/golden/cli/project_help.txt`. Prefer regenerating with Python 3.12 via
   the existing `SASE_UPDATE_PROJECT_RENDER_GOLDENS=1` mode and
   `tests/main/test_parser_stats.py::test_project_help_golden`, then rerun without
   update mode. It should remove exactly one `By repo` row and one repos[] jq example.
   Preserve the correct production definitions and py313 golden. Use an isolated scratch
   venv if the checkout uses Python 3.14; install the checkout's .[dev] with the exact
   published core as CI does. Run the parser file on 3.12 and on 3.13 or 3.14. Do not
   mistake a 3.14-only pass for 3.12 proof.
3. Inspect `src/sase/ace/tui/modals/help_modal/agents_reference_sections.py` and
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_archive_states.py::test_archive_help_png_snapshot`.
   Regenerate its PNG locally with
   `TZ=UTC just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_archive_states.py::test_archive_help_png_snapshot`.
   Inspect old/new images and the retained report. Expected change follows the three
   inserted Help legend rows. Confirm the intended Archive help context is still
   visible, rather than accepting a loading frame or a different scroll position. The
   test currently scrolls until `Archive (read-only)` appears and asserts that text
   before capture; retain those assertions and strengthen a semantic visibility/settle
   wait only if current evidence requires it. Never copy a CI candidate PNG into the
   repo.
4. Prove that node with check-only captures in both the host zone and TZ=UTC, requiring
   no drift and no unstable recapture. Run a full
   `TZ=UTC just fix-tui-screenshots --check` through /sase_monitor to confirm the
   current tip has no additional deterministic golden gaps. If a different golden fails,
   inspect and reproduce it before changing it; preserve unrelated good images. Report
   infrastructure-only or nonreproducing failures with evidence.
5. Run just fix and sase tool run check. Retain the reports and record what is resolved
   (sase-1nt, plus the archive-help DISCOVERED ISSUE on sase-1no). Check your
   epic-symbols, close your phase with the exact tests and outcomes, and submit
   /sase_final so the host lands the edits. Do not wait for this turn's future commit or
   CI. The dependent watch verifies those.

## Completed follow-up triage

The sase-1nn.8 lander already used /sase_new_task, searched existing and recent tasks,
inspected active epics, and recorded every outcome on sase-1nn.8:

- sase-1nn.8.1 note 1: archive-help drift causally belongs to still-open sase-1no, phase
  .3; a DISCOVERED ISSUE note is filed there, no duplicate task. This plan repairs it as
  release integration and the lander reports the resulting commit.
- sase-1nn.8.2 note 3: import budget stays on sase-1ic with a supplementary note
  relaying phase check 981738e311517a34324ee4002d5bd789 (3532 vs strict 3527, KNOWN
  witness 0565caed0d1ee997557a16ec01dc1ecb). No duplicate or extra +1; this known local
  count does not fail the release CI lanes.
- sase-1nn.8.2 note 4: project-key audit declined as already repaired by a7996c3b77 and
  independently passing. sase-1ly concerns a different older site.
- Lander-found stats duplicate: **sase-1nt**, small ci task, dependency on sase-1nn.8 to
  prevent duplicate execution. Close it once this repair is verified. sase-14p is a
  different argparse spelling issue and remains open.
- **sase-1np and sase-1nq are closed done**, independently verified as resolved by the
  profiling-env scrub. No need to reopen or re-file them. sase-1cs (Textual dependency
  drift) remains open; no dependency-resolution redesign is in scope.

If any triage item gains new evidence, append it to its existing owner. The goldens
phase should not create a second task for either repair in this plan.

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
3. **Prerequisites.** Fetch. Read the goldens sibling bead, whose commit must land
   before this phase proceeds.
   - A phase closed `canceled` means a prerequisite failed. Close this bead
     `-R canceled -r "release watch not started: <phase> canceled"` with its close note,
     then run `/sase_final`.
   - Confirm each `done` sibling's commit is on `origin/master`
     (`git log origin/master --grep 'SASE_BEAD=\[<phase-id>\]'`). A phase whose note
     says no commit was needed is fine. If the commit is not present yet, use a short
     /sase_monitor wait with a follow-up to re-fetch before declaring it missing; do not
     sleep in the provider turn.
   - Confirm master's `pyproject.toml` pins a `sase-core-rs` that is on PyPI, and that
     `tools/check_sase_core_rs_bindings` passes against it. Use an isolated environment
     outside the repo with the exact published dependency from pyproject.toml, then run
     tools/check_sase_core_rs_bindings and tools/validate_sase_core_rs. All 876 bindings
     existed at planning time.
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
5. **Refresh the release PR.** If #321's head does not contain the goldens repair
   commit, or its `release-core-floor-smoke` check is red, run
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
   export NO_COLOR=1 GH_COLOR=never CLICOLOR=0 CLICOLOR_FORCE=0 FORCE_COLOR=0
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

## Landing and ancestor handoff

This child's `parent_bead` is **sase-1nn.8**, a plan bead, whose direct parent is
**sase-1nn**, also a plan bead. The lander handles closure, never a child phase.

1. Re-read every child and note, inspect implementation commits and source, and review
   post-start drift. A canceled watch is not proof of publication. Resolve epic-caused
   issues; triage other PROPOSED FOLLOW-UP entries with /sase_new_task.
2. Observe actual **sase 0.18.1 on PyPI** before calling the release complete. Record
   wheel/sdist filenames, SHA-256 hashes and upload times; GitHub release v0.18.1 and
   tag commit; merged PR #321's mergedBy, mergedAt and mergeCommit; qualifying green
   Full CI ID/SHA and Master Gate at the eligible merged head; Publish run ID and every
   job conclusion. Do not infer ci_watch authorship from the merge account alone: retain
   the watcher/ci_watch evidence as available.
3. Prove a clean install outside the repository, with an isolated home directory
   supplied only to the child process (never repurpose the shell's HOME variable).
   Create a Python 3.12 venv and install sase==0.18.1 from PyPI with UV_NO_CACHE=1.
   Verify installed sase is 0.18.1, its core version matches the exact dependency, and
   `sase core health --json` succeeds. Do not replace a global install.
4. Write `sase_v0_18_1_release_evidence.md` in a temporary directory and register
   `sase artifact create -p <file> -l "sase v0.18.1 PyPI release evidence" --bead <this-child-epic-id>`.
   Add the artifact ref in notes on sase-1nn.8 and sase-1nn. Close sase-1nt with the
   repair and both-interpreter evidence. Note the resolved archive golden on sase-1no
   without closing that unrelated epic.
5. For this child epic, run `sase bead epic-symbols <this-child-epic-id>`; resolve each
   symbol or re-key only to a still-open later bead needing it. Close normally with
   `sase bead close <this-child-epic-id> --note "<verification and evidence>"`, run
   `just symvision`, open the plans repo with /sase_repo and set status: done in this
   child's linked plan. No --force on a successful nested landing.
6. Resume **sase-1nn.8**: read its prior landing note, every descendant and note,
   `plan:202610/v0_18_1_release_landing.md`, and post-child drift; rerun descendant and
   linked-plan readiness checks. If its named canceled watch phase .3 must be completed
   for readiness, reopen **sase-1nn.8.3** and close it normally using the newly observed
   release evidence, preserving its historical cancellation notes. Retire every
   `sase bead epic-symbols sase-1nn.8` entry; close with
   `sase bead close sase-1nn.8 --note "<rechecked scope, integration, follow-up outcomes, and release artifact>"`.
   Run just symvision and set status: done in **plan:202610/v0_18_1_release_landing.md**
   using the plans checkout opened by /sase_repo. If any requirement is incomplete,
   leave it open and record why.
7. Then resume **sase-1nn**: review its prior landing notes, all descendants and notes,
   **plan:202610/v0_18_1_release.md**, and drift. Recheck descendant and linked-plan
   readiness. If the old canceled watch **sase-1nn.7** must be completed, reopen and
   normally close it with actual release evidence. Resolve
   `sase bead epic-symbols sase-1nn`, close with
   `sase bead close sase-1nn --note "<full parent recheck and release artifact>"`, run
   just symvision, and mark **plan:202610/v0_18_1_release.md** status: done. This root
   plan currently has no parent; inspect the actual parent link before stopping.
   Continue only through fully complete direct plan ancestors, stopping and recording
   the first incomplete or ambiguous ancestor. Do not force-close.

If the release is still absent, leave the ancestor epics and their plans open. Read the
watcher outcome and diagnose the concrete remaining blocker. Use /sase_plan and its
validate/revalidate/propose loop for remaining work only; prefer a tale if one coder can
directly finish, including the necessary closeout, without waiting for that turn's own
future commit. Use a nested child epic when separate committed code, CI/publication, and
landing phases are genuinely needed. Its parent_bead is this child's bead, and its
lander resumes this exact ancestor chain. Carry forward the clean-install/evidence
requirements and the bounded self-closing watcher. Do not close any release epic done
merely because its descendant beads are closed.
