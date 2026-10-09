---
tier: epic
title: Clear the last Full CI reds and ship sase v0.18.0
goal: 'Full CI is green on a sase master tip that also has a green Master Gate, release
  PR 299 merges, and `pip install sase==0.18.0` works from PyPI, so the interrupted
  landing of epic sase-1io.7 can close.

  '
parent_bead: sase-1io.7
decisions:
  ready_gate:
    ask: How should the bead-scale gate stop failing Full CI on ratio:ready?
    choices:
      known_miss: Record ratio:ready as a known miss owned by sase-1j5; quick and
        unblocks the release
      fix_measure: Hold the active set fixed in the gate corpora so ready stays blocking;
        more work
    default: known_miss
    why: The gate never passed on CI; sase-1j5 owns the measurement fix without blocking
      the release
    answer: known_miss
phases:
- id: ready-gate
  title: Make the bead-scale perf gate measure what it claims
  depends_on: []
  size: small
  description: 'ready-gate: stop Full CI perf-floors failing on the bead-scale gate''s
    ratio:ready criterion, which measures active-set growth rather than closed history,
    and correct the false pass verdict in the perf runbook.'
- id: reply-card-race
  title: Fix the dropped Reply-card switch in the Agents deck visual tests
  depends_on: []
  size: medium
  description: 'reply-card-race: find and fix why ctrl+j sometimes never switches
    the Agents deck Main panel to the Reply card under parallel load, and fix any
    other Full CI-only or Master Gate red on the starting tip.'
- id: ship
  title: Prove the release gates green, merge PR 299, and publish v0.18.0
  depends_on:
  - ready-gate
  - reply-card-race
  size: medium
  description: 'ship: drive Master Gate, a fresh Full CI, and PR 299''s checks green
    on a tip with both fixes, merge PR 299, publish, and verify the PyPI install.'
proposed_by: bbugyi200.athena.sase-1io.7.land
decided_by: reviewer
decided_via: tui
create_time: 2026-10-09 14:56:39
status: wip
bead_id: sase-1io.7.6
---

- **PROMPT:** [prompts/202610/ship_v0_18_0_after_full_ci_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/ship_v0_18_0_after_full_ci_fixes.md)
- **BEAD:** [sase-1io.7.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1io/sase-1io.7.6.md)

# Plan: Clear the last Full CI reds and ship sase v0.18.0

This child epic finishes the interrupted landing of epic `sase-1io.7` ("Fix the
read-model cache race, cut sase-core-rs, and ship sase v0.18.0"). Every phase of that
epic closed, but its `ship` phase recorded `RELEASE NOT SHIPPED`. The user's original
request, quoted exactly: "Can you help me do whatever needs to be done to get all tests
green and release version v0.18.0 of the sase package to PyPI?" Make reasonable calls
yourself and record them in bead notes. Do not stop to ask.

## Escalation Rule: Hand Off To `opus/opus@xhigh`

This rule carries over from the parent epics, where the user explicitly asked for it. If
you get stuck, or for any reason think you cannot close your assigned phase bead, do not
end your turn with the bead open. Hand off with the `/sase_handoff` skill to the model
`opus/opus@xhigh`. The user's instruction is your explicit authorization to use that
skill. Write a self-contained successor prompt that includes the phase id, what you
found, what you tried, the current evidence (run ids, PR numbers, failing tests), and
what is left.

- If `sase pipe` rejects that model spec, retry with `--model 'claude/opus@xhigh'`. A
  non-zero exit means no handoff happened, so you are still running.
- Fence any literal percent-sign or hash-sign text in the successor prompt. Both are
  live syntax for the successor.
- A successor already on opus at xhigh hands off again only when its context window is
  spent, using `--fresh`. Otherwise it keeps working.
- Waiting on CI is **not** being stuck. Wait with `/sase_monitor`.

## State Established By The sase-1io.7 Land Agent (2026-10-09 ~18:40 UTC)

Re-verify everything at the current `origin/master` before you act. Other agents push to
master all day.

- **PyPI.** `sase` is still 0.17.1. `sase-core-rs` 0.37.2 is complete: five unyanked
  distributions, including the five bindings PR 299 needed. The core side of the release
  is finished. Do not cut another core release unless a fix here needs a new binding.
- **PR 299.** `chore(master): release 0.18.0` is OPEN and mergeable (`CLEAN`), head
  `5089955038`. Its `sase-core-rs` floor was ratcheted to `>=0.37.2` by publish dispatch
  37965566735, and `release-core-floor-smoke` passes.
- **Master Gate** is green on tip `73c128074d` (run 37972110667).
- **Full CI** has not been green in its last 20 runs. On the newest completed runs only
  two causes remain:
  1. **`perf-floors`** fails deterministically on the `just bead-perf-scale-gate` step:
     `ratio:ready FAIL` (2.849 in run 37947270619 at `73f593a3a5`; 3.329 in run
     37963193136 at `4dec143bc2`). Every other perf floor in that job passes.
  2. **`visual-test`** fails every run on one Agents-deck Reply-card wait. It was
     `test_agents_decks_single_main_paged_png_snapshot` in 37947270619 and
     `test_agents_deck_blocks_arrival_dot_png_snapshot` in 37963193136. Phase
     `sase-1io.7.3` also saw `test_agents_deck_view_split_narrow` and
     `test_agents_deck_blocks_paged_newest`/`older` fail the same way locally under
     parallel load, and all of them pass on serial retry.
- The test-leg failures in run 37963193136 (both artifact-marker audit tests and
  `test_launch_followup_agent_reauthors_auto_prefix`) already pass at the tip.
- Full CI run 37965698802 (dispatched on `63a8f7a62e`) was still building when this plan
  was written. Its result is the newest evidence for the phases below.
- **Overlap.** Phase `sase-1i5.9.1.2.1.7` ("Prove Master Gate and a fresh Full CI on the
  release tip") is still in progress but has been silent since 2026-10-09 02:44 UTC. Its
  goal is a subset of this epic's. Do not close it; `ship` records evidence on it.

Release mechanics are unchanged from the parent plans:

- `publish.yml` runs on cron (`17 */3 * * *`) or by `workflow_dispatch`. With
  `publish_existing=false` it regenerates the release PR from current master and
  ratchets the `sase-core-rs` window to the newest published core.
- After PR 299 merges, the next generation run creates the tag and the GitHub release.
  It then runs `build`, `install-smoke`, `install-smoke-core-floor`, and `publish`.
- The `ci_watch` AXE routine merges a fully green release-please PR (merge method
  `merge`) when three things hold: Master Gate is green for the master tip, Full CI was
  green within the last 6 hours, and the PR's own checks are green.

## Guardrails For Every Phase

- Fix root causes. Never weaken an assertion, skip a test, add a `xfail`/retry, or raise
  a timeout to get green. Update an expectation only after you confirm the product
  change behind it was intentional (`git log -S`, the responsible commit). Then make the
  stale side match current truth. The `ready_gate` decision below is the one sanctioned
  exception, and only in its default branch.
- Never hand-edit release-owned files: `CHANGELOG.md`, any `version` field,
  `.release-please-manifest.json`, or the `sase-core-rs` window in `pyproject.toml`.
- Do not run `just install` or `just install-dev`; use `just install-venv`.
- Verify with `sase tool run check` after `just fix`. Only `reply-card-race` may run
  `just check-full`, through `/sase_monitor` with the `verify` profile.
- Wait for CI and PyPI only through `/sase_monitor`, with a bounded timeout and a
  `--next` that says exactly what to do with the result. Full CI often waits about an
  hour in the runner queue and then runs for about an hour, so use timeouts of at least
  three hours for it. Never end a turn promising to come back.
- The only PR you may merge is sase PR 299 (in `ship`). Leave every other open PR alone.
- Before fixing a failure, look for an existing bead (`sase bead search`,
  `sase bead list -T ci`, `sase bead list -T flake`). Note on that bead that you are
  fixing it rather than duplicating the work.
- Land code only through the host finalizer at the end of your turn. A phase cannot
  watch CI for its own commit, which is why the fix phases and `ship` are separate.
- Phase workers never create beads. Record `PROPOSED FOLLOW-UP:` notes on your own phase
  bead instead.

## Phase ready-gate: Make the bead-scale perf gate measure what it claims

Background: epic `sase-1h8` (phase `sase-1h8.14`, commit `10385fe3c3`) added
`just bead-perf-scale-gate` to Full CI `perf-floors`. It runs
`tests/perf/bench_bead_scale.py` at 1x and 4x and blocks only on `ratio:ready`, the p95
ratio of `bead_read_facade.ready` (ceiling 1.5). Every other criterion is listed in
`--gate-allow` as a known miss with an owning bead (`sase-1iu`, `sase-1iv`, `sase-1iw`,
`sase-1ix`). The gate has failed on every Full CI run since it landed. Task bead
**`sase-1j5`** holds the full analysis. Read it first with `sase bead read sase-1j5`.

- The scaled synthetic corpus (`tests/perf/_bead_corpus.py`) grows the active set along
  with closed history. `ready` returns 41 rows at 1x and 171 at 4x, so its per-row cost
  grows about 4.2x.
- Locally (sase-core-rs 0.37.2) a warm `ready` call takes a 5.7 ms median at 1x and 13.9
  ms at 4x. strace shows SQLite page reads rising from 694 to 2,784 and no stat sweep.
- The cause is not core drift and not the sase-1io.7 cache-race fix. The gate already
  failed at core pin `01b0ad73`, which predates that fix, and no read-model code changed
  between `5c4033f6` and `01b0ad73`.
- The "`ratio:ready` passes (1.03)" verdict in `docs/perf_runbook.md` came from load
  noise. Its own artifact (`sdd/plans/202610/perf_artifacts/bead_perf_gate.json`, load
  ~27) is non-monotonic: 34.9 / 59.5 / 62.5 / 38.5 ms median at 1x / 2x / 4x / 8x.

Steps:

1. Confirm the failure at the tip. Read the `perf-floors` job of the newest completed
   Full CI run and reproduce locally with `just bead-perf-scale-gate`, or with the ready
   subset:
   `.venv/bin/python tests/perf/bench_bead_scale.py --scale 1 --scale 4 --runs 3 --only ready`.
   Confirm no other `perf-floors` step is red. If another one is, fix it here too.
2. Apply the reviewer's `ready_gate` answer.

> [!decision] ready_gate = known_miss

- Add `ratio:ready` to the `--gate-allow` list of the `bead-perf-scale-gate` recipe in
  the `Justfile`. Keep the 1x+4x scales, the tolerance, and every other criterion
  unchanged, so the gate still measures and reports `ratio:ready` on every run.
- Rewrite the "Bead history-independence gate" section of `docs/perf_runbook.md`:
  - CI no longer blocks on any criterion. Every A1 criterion runs as a recorded known
    miss, and the strict local `just bead-perf-scale -- --check-gate` still enforces
    everything.
  - Replace the "`ratio:ready` passes (1.03)" verdict with the measured truth: the CI
    and local ratios above, the 41 → 171 row growth, and the load-noise artifact.
  - Add a known-miss bullet owned by `sase-1j5`, in the same style as the other bullets.
- Update any comment that still says the gate "blocks on ready ratio", such as the CI
  step in `.github/workflows/ci.yml` and the module comment in
  `tests/perf/_bead_scale_gate.py`.
- Do not change `tests/perf/test_bead_scale_gate.py` expectations. They test the
  evaluator on synthetic numbers and stay correct.

> [!decision] ready_gate = fix_measure

- Make the gate measure closed-history independence. The ratio corpora keep a fixed
  active set (the 1x active rows) while closed history scales, through an explicit
  corpus option the gate recipe passes. The default `bench_bead_scale.py` output and the
  other ops' corpora stay unchanged, so the `sase-1iu`..`sase-1ix` numbers remain
  comparable.
- Unit-test the new corpus option in `tests/perf/` next to the existing corpus tests.
- `ratio:ready` must pass at 1x→4x locally with the CI tolerance, and the strict
  `--check-gate` run must still report the other criteria honestly. If it does not pass,
  stop and apply the known-miss branch instead, and say so in your bead note.
- Correct the runbook verdict the same way, with before and after numbers, and record on
  `sase-1j5` what changed and what it still owns (for example, the per-row `prepare` in
  sase-core `has_active_blocker_in`).

3. Note on `sase-1j5` that `ready-gate` handled the CI side and which branch was taken.
4. Run `just bead-perf-scale-gate` (exit 0),
   `pytest tests/perf/test_bead_scale_gate.py`, and `sase tool run check`.

**Done when** `just bead-perf-scale-gate` exits 0 locally, the runbook states the
measured truth, and `sase tool run check` passes.

## Phase reply-card-race: Fix the dropped Reply-card switch in the Agents deck visual tests

Evidence: in Full CI run 37963193136, `test_agents_deck_blocks_arrival_dot_png_snapshot`
timed out in `_show_reply`
(`tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_blocks.py`). It waited 15 s
for "Main deck shows the Reply card" after pressing `ctrl+j`. The timeout's final frame
(the `last_frame_svg` in the run's `ace-visual-artifacts` capture.log) shows the Main
panel still on the **Context** card ("MAIN cards · auto · Context ‣ Reply 1/2", AGENT
PROMPT visible). So the key was dropped, or a later rebuild undid it. This is not
slowness, and a longer timeout would not help.

1. `just install-venv`, `git fetch`, and list the red jobs and failures of the newest
   completed Full CI run and the newest Master Gate run on the tip. If the newest Full
   CI run is older than the tip, dispatch one with
   `gh workflow run full.yml --repo sase-org/sase` and wait through `/sase_monitor`.
   - Failures `ready-gate` owns count as KNOWN here.
   - Every other red, Full CI-only or Master Gate, is yours. That includes new golden
     drift: regenerate an intended golden only after confirming the responsible commit,
     and fix real regressions in the product.
   - For known flakes (`sase-1ib`, `sase-1g8`, `sase-1j4`, and others), fix the root
     cause when you can reproduce it under load. Otherwise add the CI evidence to the
     bead and leave it.
2. Read the `tui` memory note first (`sase memory read tui.md`) for screenshot and
   visual-test guidance.
3. Reproduce under load. Repeat the Agents-deck visual files
   (`test_ace_png_snapshots_agents_deck_blocks.py`,
   `test_ace_png_snapshots_agents_decks.py`, and every other file whose helper presses
   `ctrl+j` and then waits for `active_card_id == "reply"`) with
   `pytest -m visual -n 8`. Run them in a loop, ideally two copies at once to mimic
   runner contention, until one fails. Record the fail rate.
4. Find the mechanism. Instrument the path from the `ctrl+j` binding
   (`action_next_deck_card` in `src/sase/ace/tui/actions/agents/_panel_detail.py`) to
   `cycle_focused_deck_card` (`src/sase/ace/tui/widgets/_agent_detail_deck_show.py`) and
   the panel's `cycle_card`. Several `except Exception: return None` branches there
   swallow the press silently. Then decide which of these happens:
   - the press arrives before the focused panel can cycle (a deferred view transition,
     the panel not yet mounted, or focus elsewhere), or
   - the press works, but a later refresh or re-show (for example the `auto` view, or a
     background detail load re-showing the subject) resets the active card to Context.
5. Fix it at the right layer:
   - If a real user could lose a `ctrl+j` the same way (press right after selecting an
     agent, or during a background refresh), fix the product. A press must either cycle
     or be applied once the panel is ready, and a background re-show must keep the
     user's chosen card (the deck's preferred-card stickiness).
   - If only the test reads too early, make its readiness wait match exactly what
     `cycle_card` needs before pressing. Apply that in one shared helper used by every
     affected file.
   - Never add press retries, sleeps, or longer timeouts.
   - Add a deterministic regression test for the mechanism, beside the existing deck
     tests. It should not need timing luck.
6. Prove it with the same loaded repetition used to reproduce it: 0 failures in at least
   the number of runs that failed before. Record the before and after counts.
7. Run the affected visual tests with `just test-visual --check` and inspect any changed
   PNG. Commit every dirty golden per `/sase_final`'s screenshot rule.
8. This bead authorizes `just check-full` through `/sase_monitor` with the `verify`
   profile as final verification. Otherwise use `sase tool run check`.

**Done when** the Reply-card switch can no longer be dropped, every other Full CI-only
or Master Gate red on the starting tip passes locally, and the fixes are ready to land.

## Phase ship: Prove the release gates green, merge PR 299, and publish v0.18.0

1. **Preconditions.** `git fetch` and confirm that the `ready-gate` and
   `reply-card-race` commits are on `origin/master`. Confirm that `sase-core-rs` 0.37.2
   (or newer) is still complete on PyPI.
2. **Start in parallel.**
   - `gh workflow run full.yml --repo sase-org/sase` on the current tip, unless a run
     already covers a tip containing both fixes.
   - `gh workflow run publish.yml --repo sase-org/sase -f publish_existing=false`. This
     regenerates PR 299 from current master and keeps its floor ratcheted.
   - Confirm PR 299 is still titled `chore(master): release 0.18.0`, its floor is the
     newest published core, and `release-core-floor-smoke` passes.
3. **Monitor** Master Gate for the tip, the Full CI run, and PR 299's checks until they
   settle.
   - A red that recurs and is a known flake can be retried with
     `gh run rerun <run-id> --failed` once per run. Record the flake evidence on its
     bead, and never rerun a deterministic failure.
   - If anything else is red, reproduce it at `origin/master`. If it needs code, land
     the fix and close this bead with a note that starts `RELEASE NOT SHIPPED:`. Give
     the failures fixed and the runs to re-dispatch; this epic's land agent finishes the
     release.
   - Hand off only if you cannot produce a fix.
   - Master moves while you wait. `ci_watch` needs Master Gate green on the **current**
     tip, so recheck the newest tip's Master Gate before merging.
4. **Merge.** When all three `ci_watch` conditions hold, let `ci_watch` merge PR 299. It
   runs one merge per tick. If it has not merged within about 30 minutes, merge it
   yourself with `gh pr merge 299 --repo sase-org/sase --merge`. The user explicitly
   asked for this release, and that is the method `ci_watch` uses.
5. **Publish.** Run
   `gh workflow run publish.yml --repo sase-org/sase -f publish_existing=false` so
   release-please creates the `v0.18.0` tag and GitHub release. Monitor `build`,
   `install-smoke`, `install-smoke-core-floor`, and `publish`. If the tag exists but
   publishing failed, fix the cause and use the `-f publish_existing=true` dispatch; its
   upload uses `skip-existing`.
6. **Verify.** `https://pypi.org/pypi/sase/json` reports `0.18.0` with a wheel and an
   sdist. In a fresh `uv venv`, run `uv pip install sase==0.18.0`, then confirm that
   `sase version` and `sase core health --json` both succeed.
7. **Record and notify.**
   - Note the green Master Gate and Full CI run ids on `sase-1i5.9.1.2.1.7`, whose goal
     they satisfy. Do not close it.
   - Send a `sase notify create` summary for the user: the PyPI URL, the core version,
     and the failures that were fixed.

**Done when** `sase==0.18.0` installs from PyPI and passes the health check, or the bead
closes with a `RELEASE NOT SHIPPED:` note after landing a further fix.

## Landing

The land agent confirms that `sase==0.18.0` is live on PyPI and that the final Master
Gate and Full CI runs were green. If `ship` closed with `RELEASE NOT SHIPPED` and the
release is still not on PyPI, finishing the release **is** the landing work. Follow the
`ship` procedure and the escalation rule. Then triage the phases' `PROPOSED FOLLOW-UP:`
notes as usual.

After this epic closes, resume the interrupted landing of its parent epic `sase-1io.7`
as the land prompt describes. The parent's own verification and follow-up triage are
already recorded in its bead notes ("LAND TRIAGE"), and it has no `--epic-symbol`
entries. Close it, run `just symvision`, and mark its plan done. Then continue with its
parent `sase-1io`.
