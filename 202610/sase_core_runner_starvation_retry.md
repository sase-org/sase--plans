---
tier: tale
title: Rerun sase-core jobs GitHub never assigned a runner
goal:
  sase-core runs whose only red jobs never got a hosted runner rerun automatically, and
  v0.42.7 is complete on PyPI.
size: medium
decisions:
  scope:
    ask: Should the automatic rerun also watch the CI workflow, not just Release-plz?
    choices:
      both:
        Watch Release-plz and CI, so a starved macos-latest CI leg on master or a
        release PR is rerun too
      release:
        Watch Release-plz only, so a starved CI leg stays red until someone reruns it by
        hand
    default: both
    why: CI's macos-latest leg has the same exposure, and the detector is identical
    answer: both
  retries:
    ask: Should a run get a second automatic rerun when the first rerun is also starved?
    choices:
      two:
        Rerun attempts 1 and 2; needs SASE_RELEASE_TOKEN so a rerun can re-trigger the
        retry workflow
      one:
        Rerun attempt 1 only, with least-privilege GITHUB_TOKEN; the scheduled heal
        stays the backstop
    default: one
    why:
      matches release_plz_merge.py's rerun-once rule and keeps the PAT out of a
      workflow_run job
    answer: one
proposed_by: bbugyi200.athena.14n
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 16:11:42
status: wip
---

# Rerun sase-core jobs that GitHub never gave a runner

All code changes are in the linked **sase-core** repo. Open it with
`sase repo open sase-core -r "<why>"` and work only in the path it prints. Read that
repo's `AGENTS.md` first. Verification there is `sase tool run check`, not bare
`just check`.

## Diagnosis (already established)

`actstat --repo sase-org/sase-core` reports one red commit:
`4121de0 chore: release v0.42.7 (#338)`, workflow **Release-plz #1787** (run
`40346929251`).

- The only red job is `macos universal2` (job `121656624358`, `runs-on: macos-latest`).
  It was queued at 15:12:31Z and cancelled at 15:27:34Z **without ever starting**:
  `runner_id: 0`, `runner_name: ""`, `steps: []`. Its check-run annotations say
  `The job was not acquired by Runner of type hosted even after multiple attempts`
  (failure) and
  `Due to capacity constraints, jobs targeting macOS arm64 runners may experience longer queue times.`
  The run concluded `failure`, while the job itself concluded `cancelled`.
- The other four builds (linux x86_64, linux aarch64, windows x86_64, sdist) succeeded.
  `twine check` and `publish to PyPI` were skipped because they need the macOS job. So
  `v0.42.7` was tagged and its GitHub release created, but **nothing reached PyPI**.
  PyPI's newest `sase-core-rs` is still 0.42.6.
- **This is not a code defect.** No step of the job ran. GitHub's hosted macOS arm64
  pool had no capacity. In the last 200 Release-plz runs this is the only failure of
  this kind. The 17 earlier failures (Oct 9–24) were all in `Merge release PR`, an
  unrelated, already-fixed problem.
- The workflow's publish gate is self-healing: the next push or six-hourly cron
  (`23 */6 * * *`) sees that the tagged workspace version is not complete on PyPI and
  rebuilds it. But GitHub delays scheduled runs heavily: no scheduled run had fired by
  19:40Z, even though 18:23Z was due. So nothing healed v0.42.7 until an unrelated push
  (`4f17b72`) started Release-plz run `40388871469` at 19:40Z. That run did get a macOS
  runner and was building wheels when this plan was written. The version was unpublished
  for 4.5+ hours, and the commit stayed red the whole time.

**The gap this plan closes:** nothing reruns a job GitHub never scheduled. Recovery
waits for an unrelated push or a late cron, and the red run remains the commit's visible
status. `ci.yml`'s `macos-latest` matrix leg has the same exposure. An
infrastructure-only failure (every red job never got a runner) is always safe to rerun:
no step ran, so nothing was uploaded, tagged, or published.

## Step 1: Confirm the v0.42.7 heal and test the rerun mechanism live

1. Check the outcome of Release-plz run `40388871469`
   (`gh run view 40388871469 -R sase-org/sase-core`) and the PyPI state:

   ```bash
   EXPECTED_DIST_SUFFIXES="$(python3 -c "import yaml; print(yaml.safe_load(open('.github/workflows/release-plz.yml'))['env']['EXPECTED_DIST_SUFFIXES'])")" \
     python3 .github/scripts/pypi_release_files.py status 0.42.7
   ```

   `complete` is the goal. PyPI's JSON API is CDN-cached briefly after an upload, so
   re-query once before you conclude otherwise.
   - If that run is still in progress, wait for it with `/sase_monitor` (for example,
     monitor `gh run watch 40388871469 -R sase-org/sase-core --exit-status`). Never end
     the turn to wait.
   - If a build job in it was starved again (same signature: `cancelled`, `runner_name`
     empty, no steps), run `gh run rerun 40388871469 -R sase-org/sase-core --failed`,
     monitor it, and use that rerun as the live test in item 2 below instead.
   - If it failed for a real reason (a step ran and failed), stop. Report the failing
     step and its log excerpt in the final summary instead of guessing a fix.

2. **Live test of the two GitHub behaviours the helper depends on.** Rerun the original
   failed run: `gh run rerun 40346929251 -R sase-org/sase-core --failed`. Then monitor
   it with `/sase_monitor` (macOS build plus publish, roughly 15–25 minutes). Start it,
   and do Steps 2–4 while it runs. It is safe:
   - The rerun reuses attempt 1's `publish-plan` outputs (version 0.42.7,
     `build_ref=v0.42.7`).
   - `publish` passes `skip-existing: true`, and the quota guard only charges files PyPI
     lacks. If 0.42.7 is already complete, the upload is a no-op.
   - If attempt 1's artifacts are not visible to attempt 2, the `publish` guard
     (`pypi_release_files.py dist`) fails before uploading anything.

   Record what it shows:
   - (a) Does `--failed` rerun a job whose conclusion was `cancelled`, not `failure`?
     Check that attempt 2 contains a `macos universal2` job that actually ran.
   - (b) Does `metadata-check` in attempt 2 see the four artifacts uploaded in attempt
     1, so `dist/` holds all five files?

   This rerun also turns `4121de0` green in `actstat`. If (a) is false, have the helper
   rerun individual jobs instead, as described in Step 2. If (b) is false, say so in the
   docs (Step 4) and in the final summary. A Release-plz rerun would then still heal
   through the next push or scheduled run, but not by itself.

## Step 2: Add the helper `.github/scripts/rerun_starved_jobs.py`

Stdlib only. Model its shape on `.github/scripts/release_plz_merge.py`:

- a `Protocol` for the GitHub calls
- a `GitHubCLI` implementation over `gh`, with errors raised as one exception type
- a pure decision function
- `main()` returning an exit code

Module docstring: "Rerun jobs GitHub never assigned a runner to, once per workflow run".
Also explain why, citing run `40346929251`.

- **Inputs (env):**
  - `GH_REPO` and `GH_TOKEN`, consumed by `gh`
  - `RUN_ID`
  - `RUN_ATTEMPT` (int)
  - `RUN_CONCLUSION`, so the script can also refuse anything but `failure`, independent
    of the workflow `if`
- **Fetch the jobs of exactly that attempt:**
  `gh api --paginate "repos/{owner}/{repo}/actions/runs/$RUN_ID/attempts/$RUN_ATTEMPT/jobs?per_page=100" --jq '.jobs[]'`.
  This returns one JSON object per line. Malformed output raises the module's error.
- **Starved job:** conclusion in `{"failure", "cancelled", "timed_out"}` **and** `steps`
  empty **and** `runner_id` falsy (`0`/`null`) **and** `runner_name` empty. All four
  must hold. A job that failed in "Set up job" (for example, the floating setup-uv ref
  that `test_workflow_action_pins.py` documents) had a runner and has steps, so it is
  **not** starved.
- **Decision**, returned as a small frozen dataclass with `action` (`rerun`/`skip`),
  `reason`, and the starved job names:
  - `skip` if `RUN_CONCLUSION != "failure"`. A `cancelled` run means a person or a
    concurrency group cancelled it.
  - `skip` if `RUN_ATTEMPT >= MAX_ATTEMPTS`.
  - `skip` if there are no red jobs, or none of them is starved.
  - `skip` if any red job is **not** starved. Mixed failures are not
    infrastructure-only, and a rerun would also rerun a real failure that may already
    have uploaded an artifact. A v4 artifact name cannot be uploaded twice in one run.
  - Otherwise `rerun`. Skipped dependents (`twine check`, `publish`) are not red; ignore
    them.
- **Rerun:** `gh run rerun "$RUN_ID" --failed`. This reruns all red jobs plus their
  dependents in a single new attempt. Rerunning several single jobs one at a time does
  not work, because each call opens a new attempt and GitHub then refuses jobs from
  older attempts. Fallback, only if Step 1(a) showed that `--failed` skips `cancelled`
  jobs: when exactly one job is starved, use `gh run rerun --job <job id>` (that job
  plus its dependents); when several are starved, use `--failed` and log a
  `::warning::`.
- **Output:**
  - Always append one line to `$GITHUB_STEP_SUMMARY`: the decision, the reason, the
    starved job names, and the run URL
    `https://github.com/$GH_REPO/actions/runs/$RUN_ID`.
  - Print a `::notice::` on rerun.
  - A `skip` exits 0, because "nothing to rerun" must never turn the retry workflow red.
  - A `gh` failure, malformed JSON, or a missing or invalid env value prints `::error::`
    and exits 1.
- `MAX_ATTEMPTS = 2` (only attempt 1 is rerun). Add a comment saying the cap prevents
  rerun loops.

> [!decision] retries = two Set `MAX_ATTEMPTS = 3` and use
> `GH_TOKEN: ${{ secrets.SASE_RELEASE_TOKEN || secrets.GITHUB_TOKEN }}` in Step 3. A
> rerun requested with `GITHUB_TOKEN` does not reliably trigger `workflow_run` again
> when it completes, so a second retry needs the PAT. Add a comment explaining that.

## Step 3: Add `.github/workflows/runner-starvation-retry.yml`

```yaml
name: Runner starvation retry

# <header comment: the 40346929251 incident in two sentences; the rule "rerun only
#  when every red job never got a runner, once per run"; that the publish gate's
#  push/six-hourly heal remains the backstop; and that names in `workflows:` must
#  equal the watched workflows' `name:` lines (a test pins that).>
on:
  workflow_run:
    workflows: [Release-plz, CI]
    types: [completed]

permissions: {}

jobs:
  rerun-starved:
    name: Rerun runner-starved jobs
    runs-on: ubuntu-latest
    timeout-minutes: 10
    if:
      ${{ github.repository_owner == 'sase-org' && github.event.workflow_run.conclusion
      == 'failure' }}
    permissions:
      actions: write
      contents: read
    concurrency:
      group: runner-starvation-retry-${{ github.event.workflow_run.id }}
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false

      - name: Rerun jobs no runner picked up
        env:
          GH_TOKEN: ${{ github.token }}
          GH_REPO: ${{ github.repository }}
          RUN_ID: ${{ github.event.workflow_run.id }}
          RUN_ATTEMPT: ${{ github.event.workflow_run.run_attempt }}
          RUN_CONCLUSION: ${{ github.event.workflow_run.conclusion }}
        run: python3 .github/scripts/rerun_starved_jobs.py
```

- The script owns the attempt cap. Do not copy it into the `if`, because two copies
  drift.
- `workflow_run` always runs the default branch's copy of this workflow and checks out
  the default branch, so PR code is never executed with `actions: write`.

> [!decision] scope = release Use `workflows: [Release-plz]`. In the header comment, and
> in Step 4's docs, note that CI is deliberately not watched.

## Step 4: Tests, comments, docs

- **New file: `.github/scripts/test_rerun_starved_jobs.py`.** Use `unittest` with a fake
  GitHub object, like `test_release_plz_merge.py`. Build the fixtures from the real job
  above:
  `{"name": "macos universal2", "conclusion": "cancelled", "runner_id": 0, "runner_name": "", "steps": []}`.
  Cases:
  - Starved-only attempt 1 → exactly one `rerun_failed`.
  - A starved job plus a job that ran and failed → no rerun, and the reason names the
    real failure.
  - A real failure only → no rerun.
  - A "Set up job" failure (runner assigned, one failed step) → not starved.
  - Attempt `MAX_ATTEMPTS` → no rerun.
  - Run conclusion `cancelled` → no rerun.
  - Skipped dependents only next to a starved job → rerun.
  - No red jobs → skip.
  - `gh` rerun failure → exit 1 with an `::error::` line.
  - `main()` writes the summary line.
- **Workflow-text checks**, stdlib only like `test_release_cadence.py`:
  - The new workflow triggers on `workflow_run` with `types: [completed]`.
  - Every name in `workflows:` equals the `name:` line of an existing file in
    `.github/workflows/`. Read those files, so renaming `CI` or `Release-plz` breaks the
    test.
  - The job gates on `github.event.workflow_run.conclusion == 'failure'` and has
    `actions: write`.
  - Top-level `permissions: {}` is present.
  - The job runs `python3 .github/scripts/rerun_starved_jobs.py`.
  - The token expression matches the chosen `retries` branch.
- **`.github/workflows/release-plz.yml` header:** the paragraph that lists "runner
  starvation" among the self-healing causes gets one sentence. It says that a run whose
  only red jobs never got a runner is rerun once, within minutes, by
  `runner-starvation-retry.yml`, and that the publish gate stays the backstop. Do not
  change any job logic in this file.
- **`docs/pypi-retention.md`:** add a short `## Runner starvation` section after
  "Healing a partial release". Cover:
  - the symptom: the annotation text, a `cancelled` job with no steps, and a red run
    with `twine check`/`publish` skipped
  - the automatic once-per-run rerun and its "every red job starved" rule
  - the backstop: the next push or six-hourly heal
  - the manual remedy, `gh run rerun <run id> --repo sase-org/sase-core --failed`
  - what Step 1(a)/(b) showed about cancelled-job reruns and cross-attempt artifacts
  - the 4.5-hour v0.42.7 gap as the motivating incident
- No `version` field, `CHANGELOG.md`, or `release-plz.toml` change. release-plz owns
  those.

## Step 5: Verify and land

- `sase tool run check` in sase-core must pass. Use an explicit tool timeout of at least
  10 minutes; it runs the script tests via `./scripts/check.sh script-test`. While
  iterating, the fast loop is
  `uv run --no-project --with packaging --with requests python -m unittest discover -s .github/scripts -t .github/scripts`.
- Commit subject, Conventional Commits:
  `fix(ci): rerun jobs GitHub never assigned a runner`. Landing is host-owned through
  the final declaration. Do not push or tag by hand.
- The final summary must state:
  - the v0.42.7 PyPI status
  - Step 1's two findings (a) and (b)
  - whether `actstat --repo sase-org/sase-core` shows the newest settled commit green

## Out of scope

- Changing the macOS runner label, or cross-compiling universal2 off macOS. Neither
  removes the capacity risk, and sase-14h already tracks the arm64-only wheel trade-off.
- Teaching `release_plz_merge.py` to treat a starved, `cancelled` release-PR check as
  rerunnable. It refuses cancelled checks today. With `scope = both`, the retry workflow
  reruns that check, but the daily cut may already have refused. If that matters,
  capture it as a task bead through `/sase_new_task`; do not widen this change.
