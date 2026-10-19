---
tier: epic
title:
  Repair the marker-audit catalog drift from e5c02dc8a4 and ship sase v0.18.0 through
  ci_watch
goal: "sase 0.18.0 is on PyPI, published from release-please PR #299 after ci_watch
  merged it on a green Master Gate and a fresh green Full CI, with sase-core-rs 0.42.0
  published as its core floor. The release is confirmed before landing and recorded in
  an evidence artifact by the final epic lander, and ancestor epics sase-1mk.9 and
  sase-1mk are resumed and closed.

  "
parent_bead: sase-1mk.9
phases:
  - id: marker-audit
    title:
      Bring the agent-artifact marker audits in line with the atomic done.json writers
    depends_on: []
    size: small
    description:
      "marker-audit: teach the marker mutation and path-passing audits about
      write_json_marker_atomic and review the two done.json writers e5c02dc8a4 moved
      onto it (task sase-1mt), so all four audit nodes pass."
  - id: watch
    title: Watch sase 0.18.0 to PyPI and always close
    depends_on:
      - marker-audit
    size: small
    description:
      "watch: run one bounded tools/release_watch monitor that closes this phase bead on
      every outcome, so the lander always runs."
proposed_by: bbugyi200.athena.sase-1mk.9.land
proposed_with: claude/opus
create_time: 2026-10-19 00:00:07
status: wip
---

- **PROMPT:**
  [prompts/202610/ship_v0_18_0_marker_audit_repair.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/ship_v0_18_0_marker_audit_repair.md)
- **PARENT:**
  [202610/ship_v0_18_0_post_repair_reds.md](https://github.com/sase-org/sase--plans/blob/main/202610/ship_v0_18_0_post_repair_reds.md)

# Plan: Repair the marker-audit catalog drift from e5c02dc8a4 and ship sase v0.18.0 through ci_watch

## Why this nested epic exists

Parent epic **sase-1mk.9** landed all five of its repair phases on sase master: research
README `43e9066b` (research sidecar), JUMP golden `b68b5166ad`, New Project owner switch
`7f3c8d0965`, runtime flakes `e5c02dc8a4`, and link-follow kind `f457166f4a`. Its land
agent verified that each change is on `origin/master` (`dd963c0fa6`) and that none was
reverted. Full CI 39187446189 at `dd963c0fa6` shows the repairs held. `lint` passed, so
the research README is healed. `visual-test` passed, so the JUMP golden is healed.
`perf-floors`, `tutorial-tests`, `ace-page-group-isolation` and `build-core` passed too.
No link-follow, New Project, known-continuation or provider-drain node failed there or
in any later Master Gate.

The parent's watch phase sase-1mk.9.6 still ended **FULL_CI_RED** (watcher exit 10, at
2026-10-19T03:36Z). PyPI still lists sase 0.17.1 and has no 0.18.0. sase-core-rs
**0.42.0** is on PyPI (published 2026-10-18T17:18:53Z). PR #299 is `OPEN`, `MERGEABLE`
and `CLEAN` at head `d797ec0233`, and `release-core-floor-smoke` passed there.

**This epic owns every remaining release blocker outright.** No phase or lander may
label a red "externally owned" and wait.

### The one blocker as of 2026-10-19 ~03:55Z (re-derive before acting)

**Marker-audit catalog drift. The parent's own phase sase-1mk.9.5 caused it; task
sase-1mt tracks it.** Commit `e5c02dc8a4` ("fix(runtime): harden triage and provider
drain handling") changed two writers:

- `src/sase/axe/run_agent_exec_markers.py:write_done_marker_and_update_index`
- `src/sase/axe/run_agent_runner_finalize.py:write_error_done_marker`

Both used to write `done.json` with `open(...)` plus `json.dump(...)`. They now write it
with `sase.core.atomic_json.write_json_marker_atomic`, followed by
`update_agent_artifact_index_for_marker_mutation`. The static AST audits do not count
`write_json_marker_atomic` as a mutation call. `_MUTATION_CALL_NAMES` in
`tests/_agent_artifact_marker_audit_helpers.py` lists `dump`, `write_json_file`,
`mkstemp` and so on, but not this helper. So four nodes fail deterministically:

1. `tests/test_agent_artifact_marker_mutation_audit.py::test_tracked_marker_mutation_sites_are_reviewed`:
   the reviewed set still names both contexts, but the AST no longer finds them.
2. `tests/test_agent_artifact_marker_mutation_audit.py::test_reviewed_marker_mutation_sites_match_expected_mutations`:
   both reviewed entries expect `("open", "dump")`.
3. `tests/test_agent_artifact_marker_mutation_audit.py::test_reviewed_marker_mutation_sites_declare_lifecycle_coverage`:
   `KeyError: 'src/sase/axe/run_agent_exec_markers.py:write_done_marker_and_update_index'`.
4. `tests/test_agent_artifact_marker_path_passing_audit.py::test_tracked_marker_path_passing_sites_are_reviewed`:
   `write_error_done_marker` is now an unreviewed path-passing site, because it passes
   `Path(current_artifacts_dir) / "done.json"` straight to the helper.

These four nodes fail in four places:

- every Master Gate since `e5c02dc8a4`: 39169961708, 39170326756, 39171797772,
  39174391280, 39176179423 and 39185847786, in shards `test (6)` (node 4) and `test (7)`
  (nodes 1-3);
- every Full CI test leg since: 39172267490, 39178872128 and 39187446189, each with
  `4 failed, ~56349 passed` on 3.12, 3.13 and 3.14;
- the parent land agent's local run at `dd963c0fa6`;
- unrelated agents' `just check` runs (sase-1mt).

They are the only failures in those runs. Master Gate was last green at `7f3c8d0965`
(run 39168441419).

`e5c02dc8a4`'s own scoped `sase tool run check` passed because it never selected these
source-scanning audits. That selection gap predates this epic. It is filed as task
**sase-1mv**, and it is **not** this epic's work.

### Integration notes from the parent landing

- Concurrent commits since sase-1mk.9 started are `fa1b12db27` (agent-cli shadowed
  installs), `5fe54b5974` and `dd963c0fa6` (core pin to `6a626d10`; the stats project
  counter now defaults to `sloc`) and `1276b73b8b` (procs gear hue). None of them
  conflicts with or duplicates the release work.
  - The core pin bump stays compatible with the 0.42.0 floor. `v0.42.0` already has the
    `sloc` counter wire value, and the core diff `v0.42.0..6a626d10` adds no pyfunctions
    and no wire fields that Python consumes (only `PROJECT_STATS_CLASSIFIER_VERSION`,
    which sase never reads).
  - If PR #299's `release-core-floor-smoke` turns red after release-please refreshes the
    PR, that is CORE_FLOOR_BLOCKED for this epic's lander. It is not a reason to change
    the floor by hand.
- Concurrent commit `457dd728d5` changed Publish's `build` checkout to release-please's
  `sha` output. Publish's job names are unchanged. The Publish run after ci_watch's
  merge must still show `release`, `build`, `install-smoke`, `install-smoke-core-floor`
  and `publish` succeeding.
- The parent lander triaged every sase-1mk.9 `PROPOSED FOLLOW-UP:`. It corroborated
  sase-1fi, sase-xo, sase-1kf and sase-1k8, and filed sase-1mu and sase-1mv. This epic's
  Landing triages only its own phases' proposals.
- Neither ancestor has `--epic-symbol` entries. sase-1mk.9's `parent_bead` is sase-1mk,
  and sase-1mk has no parent. sase-1mk.9's Landing resumes at step 4 once this epic
  confirms the release. sase-1mk's Landing then resumes the same way.
- The Full CI 3.12 test leg ran 83-93 min against its 120-min job timeout. Watch it, but
  it is not a blocker (sase-v8).

### What ci_watch needs, all in the same 5-minute tick

The logic and config are unchanged from the parent. To read the logic, open
`gh:bbugyi200/bugyi-chops` with `/sase_repo`. All three of these must hold together:

1. **Master Gate** has completed green at the exact master HEAD SHA.
2. The **newest completed Full CI** run on master is green and finished within
   `heavy_max_age_hours: 6`. A newer red run hides an older green one.
3. **PR #299** is the only open release-please PR. It is `MERGEABLE`/`CLEAN` with every
   check SUCCESS, SKIPPED or NEUTRAL, and no release-please or Publish run is in flight.

### Concurrent work this epic must respect

- Other agents push to sase master. Every phase starts from a fetched, fast-forwarded
  `origin/master` and preserves concurrent changes.
- The stale release chains (**sase-1lh**, **sase-1kx**, **sase-1c1**) have no live
  agents. Do not relaunch them. Only the final lander touches them.

## Rules for every phase

- Re-derive your area's current failures from fresh CI (`gh run list`,
  `gh run view --log-failed`) before editing. The snapshot above is a starting point.
- Fix root causes. Never skip, xfail, quarantine or delete a test or assertion. Never
  add rerun-on-failure, and never widen ci_watch or watcher gates. Never loosen an audit
  to make it pass. A new indirect writer must become visible to the audit, not
  invisible.
- A failure in your area that fresh CI shows and the snapshot lacks is yours. For a
  failure outside every phase's area, append a `PROPOSED FOLLOW-UP:` note with run and
  job IDs to your phase bead. Phase workers never create beads and never close task
  beads. The watch phase is the only phase that closes a bead itself: its own.
- After editing sase files, read the lint_and_test memory, run `just fix`, then run
  `sase tool run check`. Never run `just check-full`. The scoped check does not select
  the marker audits (sase-1mv), so also run both audit modules directly, as phase
  marker-audit requires.
- Never merge a PR by hand, edit versions, changelogs or dependency windows, or change
  ci_watch's code or config. Dispatch release workflows only where this plan says so.
- End with a phase note that names the task beads your fix addresses and the
  verification evidence, so the lander can close those beads with release evidence.

## Phase marker-audit

1. Fetch and fast-forward `origin/master`. Reproduce the failures with
   `.venv/bin/python -m pytest -q -p no:cacheprovider tests/test_agent_artifact_marker_mutation_audit.py tests/test_agent_artifact_marker_path_passing_audit.py`.
   Expect exactly the four nodes above. Confirm from the newest Master Gate and Full CI
   runs that nothing else is red. If another node is red, it is yours too.
2. Read `tests/_agent_artifact_marker_audit_helpers.py`, both audit modules, and
   `src/sase/core/atomic_json.py`. `write_json_marker_atomic` is a real marker mutation:
   it writes a temp file, fsyncs it, then calls `os.replace` into the marker path. The
   audit must keep seeing these writers, so:
   - Add `write_json_marker_atomic` to `_MUTATION_CALL_NAMES`, the set of calls the
     scanner treats as direct mutations.
   - Update both reviewed `_REVIEWED_MARKER_MUTATION_CONTEXTS` entries
     (`write_done_marker_and_update_index` and `write_error_done_marker`) to their
     actual mutation-call tuple (`("write_json_marker_atomic",)`). Keep
     `lifecycle_calls=(_UPDATE_INDEX,)`; both still call
     `update_agent_artifact_index_for_marker_mutation` right after the write.
   - Rerun the path-passing audit. Recognizing the helper as a mutation call should
     clear `write_error_done_marker` from the path-passing set. If it does not, add a
     reviewed path-passing entry with an accurate justification. Do not use a broad
     exemption.
   - Recognizing the helper may surface other `src/sase` functions that pass a tracked
     marker literal (`done.json`, `agent_meta.json`, `running.json`, `waiting.json`,
     `pending_question.json`, `workflow_state.json`, `plan_path.json`, `prompt_step_`)
     to `write_json_marker_atomic`. Today its other callers write launch-admission,
     hold, monitor and diagnostics files. Review every newly surfaced context the same
     way, with real lifecycle coverage or an accurate exemption. If one is a genuine
     Tier 1 index gap, fix it in product code.
3. Keep `e5c02dc8a4`'s runtime behavior. Do not revert the atomic `done.json` writes;
   they are the sase-1k2 provider-drain fix. Confirm that nothing reading `done.json`
   depends on key order, since `write_json_marker_atomic` uses `sort_keys=True`.
4. Run both audit modules plus `tests/test_run_agent_runner_completion_helpers.py` and
   `tests/fakey/test_provider_drain_e2e.py`. Run `just fix`, then `sase tool run check`.
   The audits are deterministic AST checks, so no stress loop is needed.
5. Append a phase note. It names sase-1mt as resolved by this change, lists every
   reviewed context you added or changed, and gives the pass output of both audit
   modules and the check's ToolRun ID.

## Phase watch

This is the release-watch section. Nested epics copy it verbatim, close table included,
substituting only their own bead IDs. Make no source edits, and read `/sase_monitor`
first. **Every path through this phase ends with this phase bead closed**, so the lander
always runs.

1. If this phase bead is already closed, use `/sase_final` and stop.
2. Fetch and fast-forward the sase checkout. Confirm that every sibling repair phase's
   change is on `origin/master`, using its closing note and `git log`. If a change is
   missing, start one `/sase_monitor` 10-minute `sleep` whose `--next` resumes this
   step. If the change is still missing after that, close this bead with
   `sase bead close <watch-bead> -R canceled -r "WATCH_PRECONDITION: <what is missing>" --note "<evidence>"`
   and finalize. Otherwise set `FIXED_SHA` to the fetched `origin/master` tip.
3. Open sase-core with `/sase_repo` and fetch it. Set `CORE` as follows:
   - If the containing release is already published, use that version.
   - Otherwise use the version the open release-plz PR proposes (currently 0.42.0).
   - The chosen version must contain the core-fixture change and `mobile_pairing_issue`.
     Never use 0.41.0.

   If `CORE` is not yet published, confirm the open release PR's head contains the
   core-fixture commit. If every check on that head has completed and one failed, close
   this bead canceled with `-r "CORE_CI_RED: <failing checks>"` and the job URLs as the
   note, then finalize.

4. Run `sase bead epic-symbols <watch-bead>`; it must be empty. Never key an exemption
   to a watch bead. If an entry needs a Justfile edit, close this bead canceled with
   `-r "WATCH_PRECONDITION: epic symbols"` and an explanatory note, then finalize; the
   lander handles it.
5. If PyPI already lists sase 0.18.0, prove it yourself: install `sase==0.18.0` in a
   fresh temporary venv and run `sase --version`. Close this bead normally with that
   output and finalize.
6. Preflight with the concrete values:

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --dispatch-core-cut --once --dry-run --close-bead <watch-bead>
   ```

   The preflight must exit 0 and print a fully parsed state line. If it does not, close
   this bead canceled with `-r "WATCHER_BROKEN (preflight)"` and the output as the note,
   then finalize. Record `FIXED_SHA`, `CORE`, the preflight output and the exact monitor
   command in a note on this bead.

7. Start exactly one monitor, with placeholders substituted:

   ```sh
   sase monitor start -C "$PWD" -r 'Watch sase-core <CORE> and sase 0.18.0 to PyPI; the watcher closes this phase bead' -t 14h -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' -L 'sase 0.18.0 release watch' -m '@small' -n '<NEXT>' -- 'PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 10m 750m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --dispatch-core-cut --deadline 12h --close-bead <watch-bead>; rc=$?; echo "RELEASE_WATCH_EXIT=$rc"; sleep 120; sase bead close <watch-bead> -R canceled -r "release not confirmed: watcher exit $rc left the bead open" || true; exit 0'
   ```

   `<NEXT>` is this text, with its IDs substituted: "Release-watch closeout for
   `<watch-bead>` of epic `<epic-id>`. Do not restart the watcher, dispatch anything,
   merge, or edit files. Read the monitor result and the verdict file it names. Run
   `sase bead read <watch-bead>`. If the bead is not closed, close it: normally with the
   verdict file as the note only if the verdict is RELEASED; otherwise with
   `-R canceled -r "release not confirmed: <verdict or monitor outcome>"`. Append a
   one-paragraph summary to `<epic-id>` with `sase bead note`, giving the verdict, exit
   code, verdict-file path and last state line. Then use /sase_final."

   Do not pass `--idle-timeout`. Let `sase monitor start` itself run to completion. If
   it fails twice, close this bead canceled with `-r "WATCHER_NOT_STARTED"` and both
   errors as the note, then finalize.

Three layers close this bead: the watcher's own `--close-bead` on every verdict,
including signals; the shell fallback after the watcher exits; and the follow-up's
check. The fallback's canceled close is refused harmlessly when the watcher already
closed the bead as done. A canceled watch is not a release, and the lander decides what
follows.

| Watcher verdict                                                                                                | How this bead closes        | What the lander does                                     |
| -------------------------------------------------------------------------------------------------------------- | --------------------------- | -------------------------------------------------------- |
| RELEASED                                                                                                       | done, verdict file as note  | Confirms the full proof, then lands                      |
| FULL_CI_RED, MASTER_GATE_RED, CORE_CI_RED, CORE_FLOOR_BLOCKED, INSTALL_FAILED, PUBLISH_FAILED, RELEASE_PR_GONE | canceled, verdict as reason | Repairs through a nested epic                            |
| DEADLINE, STOPPED, WATCHER_BROKEN, WATCH_PRECONDITION, WATCHER_NOT_STARTED                                     | canceled, verdict as reason | Runs its own watcher if only waiting remains, else nests |

## Landing

The land agent confirms the release before landing. Closed phases are not proof. Nested
epics copy this section verbatim.

1. **Integration audit.** Read this epic, every phase and note, and this plan. Fetch
   sase and sase-core. Verify each repair phase's change is on master and that no
   concurrent commit undid it. Run `sase bead epic-symbols <epic>` and retire each entry
   under Symvision policy: wire it, privatize it, add a justified pragma, or delete it.
   Re-key an entry only to a still-open nested child bead. If you edit sase files, run
   `just fix` and `sase tool run check`. Never run `just check-full`.
2. **Confirm the release.** Every one of these must hold:
   - sase-core-rs `<CORE>` is on PyPI with every expected distribution. Its `v<CORE>`
     tag contains the core-fixture change, and the macOS CI job for the released commit
     passed.
   - PR #299 was merged by ci_watch. Show the merge commit, `mergedBy`, and the matching
     row in `~/.sase/axe/lumberjacks/ci_watch/ci_watch_releases.json`.
   - The GitHub release v0.18.0 exists.
   - The Publish run after the merge succeeded in `release`, `build`, `install-smoke`,
     `install-smoke-core-floor` and `publish`. A skipped job is not proof.
   - `https://pypi.org/pypi/sase/json` lists 0.18.0.
   - In a fresh temporary directory outside every repo, `uv venv` followed by an
     unpinned `uv pip install --refresh sase` resolves sase 0.18.0, and `sase --version`
     prints it. Record the resolved sase-core-rs version, which must be at least `CORE`.
   - Master Gate was green at the tip ci_watch merged, and the Full CI run it relied on
     was green and less than 6 h old.
3. **If the release is not confirmed,** read the watcher verdict and its evidence, then:
   - **Only external waiting remains** (core wheels still publishing, Full CI in flight,
     Publish not yet run, ci_watch between ticks): start your own `/sase_monitor`
     running the watch phase's watcher command **without** `--close-bead`, with `-t 14h`
     and a `--next` that resumes this Landing at step 2. Do this at most once for an
     unchanged state.
   - **GitHub release v0.18.0 exists but PyPI lacks 0.18.0,** and Publish's build or
     publish failed transiently: run
     `gh workflow run publish.yml -R sase-org/sase --ref master -f publish_existing=true`
     once, then take the waiting path.
   - **Repair is needed** (FULL_CI_RED, MASTER_GATE_RED, CORE_CI_RED,
     CORE_FLOOR_BLOCKED, INSTALL_FAILED, PUBLISH_FAILED, RELEASE_PR_GONE, or a waiting
     path that already ran once without progress): create a nested child epic with
     `/sase_plan`, setting `parent_bead` to this epic. The child gets:
     - one repair phase for every observed blocker, owned outright and never deferred to
       another epic;
     - a watch phase that copies "Phase watch" and its close table verbatim;
     - this Landing section verbatim, so its lander keeps nesting until the release is
       observed, creates the artifact, and resumes its ancestors.

     Record on this epic what the child owns, then propose the plan. That ends your
     turn, and the child's lander resumes this landing through `parent_bead`.

   - **ci_watch is not ticking, or it refuses to merge while all three gates are
     green:** ask through `/sase_questions`. Never merge by hand.
   - Never close this epic, or force-close it, without a confirmed release.

4. **Evidence artifact.** The first lander to confirm the release is the final epic
   lander. It writes `sase_v0_18_0_release_evidence.md` in a fresh temporary directory,
   not inside a repo. The file carries every proof from step 2:
   - run, job and PR IDs, SHAs, URLs and times;
   - PyPI file names with their sha256 for sase 0.18.0 and core `<CORE>`;
   - the unpinned install output;
   - the watcher verdict file's content;
   - the flake beads closed in step 5.

   Then run
   `sase artifact create -p <file> -l "sase v0.18.0 PyPI release evidence" --bead <this-epic>`.
   For each ancestor epic in this chain, add
   `sase artifact link add <artifact> related bead:<ancestor> "v0.18.0 release evidence"`.
   Ancestor landings reuse this artifact and never create a second one.

5. **Follow-ups.**
   - Triage every `PROPOSED FOLLOW-UP:` note through `/sase_new_task`, corroborating
     existing tasks instead of duplicating them.
   - Close each flake task bead that a phase note says it fixed (sase-1a7, sase-1mj,
     sase-1k9, sase-1j4, sase-1ht, sase-1a2, sase-1lr, sase-1mi, sase-1m9, sase-1l7,
     sase-1l8, sase-1ky, plus every task bead a phase of this epic names, including
     sase-1mm and sase-1k2) only when the release's green Full CI and Master Gate runs
     plus the phase's stress evidence support it. Otherwise append the evidence as a
     note.
   - Note the outcome on sase-1m3.7.1. Close sase-10d if the evidence proves it.
   - Triage Install smoke, which should heal after the release, and the 3.12
     coverage-leg time margin (sase-v8).
6. **Stale epics.**

   Parent epic sase-1mk decided `stale_epics = supersede`; that choice is final here.

   After confirming the release, close the **sase-1lh** chain (deepest open descendant
   first, from sase-1lh.7.5.4.3.3.4.4 up to sase-1lh), the **sase-1kx** chain (from
   sase-1kx.5.3.3.3 up to sase-1kx) and **sase-1c1**. Close each with
   `-R superseded --force -r "superseded by <this-epic>: sase v0.18.0 released"`, with
   the artifact ref and a one-line summary as the note. **sase-1i5.9.1** and
   **sase-1jz** still have non-release scope (plugin floors and goldens), so give them
   an evidence note only.

7. **Close and resume.**
   - Close this epic with
     `sase bead close <epic> --note "<proof summary, artifact ref, follow-up dispositions>"`.
     If the close is refused, fix the cited readiness issue and retry. Never use
     `--force` for a successful landing.
   - Run `just symvision`, through the tool wrapper if it is guarded.
   - Open the plans repo with `/sase_repo`, read this plan with `sase artifact read`,
     and set `status: done` in it.
   - If this epic has a `parent_bead`, resume that parent's Landing at step 4 and reuse
     the existing artifact. Repeat up the chain, and stop with a blocker note at the
     first ancestor that is incomplete.
   - Use `/sase_final`, and include every repository you changed.

## Out of scope

- Changing ci_watch's logic or config, or merging any PR by hand.
- Releasing sase-github or sase-telegram, and raising plugin floors (sase-1i5.9.1.4).
- CI Telemetry's leak gate, Deploy Docs, and feature work beyond the listed repairs.
- The scoped-check selection gap for source-scanning audits (task sase-1mv).
