---
tier: tale
title: 'Land E1 Installable family: fix epic-caused canary, sase-core CI and doc defects,
  then close sase-1my'
goal: The canary alert fails closed and reaches an owner, sase-core CI is green, the
  safety docs are accurate, and epic sase-1my is closed.
size: medium
decisions:
  drill_tooling:
    ask: Should the landed TestPyPI drill tooling stay in the repo, dormant?
    choices:
      keep: Keep the dispatch-only workflow, checker and tests; the doc marks the
        drill optional and not run
      remove: Delete the drill workflow, checker, their tests and the doc section;
        the drill is gone for good
    default: keep
    why: It is already built and tested, and it costs nothing until someone sets up
      the TestPyPI project
    answer: keep
proposed_by: bbugyi200.athena.sase-1my.land
proposed_with: claude/opus
bead: sase-1my
decided_by: auto
status: done
---

- **PARENT:**
  [202610/e1_installable_family.md](https://github.com/sase-org/sase--plans/blob/main/202610/e1_installable_family.md)
- **BEAD:**
  [sase-1my](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1my/README.md)

# Land epic sase-1my (E1 Installable family): fix what the epic broke, then close it

## Context

Epic `sase-1my` ("E1: Installable family", plan `plan:202610/e1_installable_family.md`)
has all six phases closed.

- **Owner's close of the exit phase.** The owner closed exit phase `sase-1my.6` with the
  note "I don't think we need to use a test PyPI package for this, right?". The TestPyPI
  drill and the 7-day canary soak are therefore not part of this landing.
- **Live proofs already recorded.**
  - **V-E1a (retention):** `sase-1my.6` note #1.
  - **V-E1c (floor check):** sase-github run `39292291601`, a probe that was red with
    `DEPENDENCY_UNPUBLISHED sase>=99: newest published is 0.18.0`, and run
    `39292296600`, a normal run that was green.
  - **V-E1b (canary):** proved by the land agent on master `a9046529ca`. Dispatch run
    `39298640599` was green on ubuntu-latest and macos-latest. Probe run `39298650862`
    (`probe_expect=sase==0.0.0`) was red on both legs.
- **Follow-up triage is done.** It is recorded as an epic note on `sase-1my`:
  - +1 on the flake tasks `sase-1gz` and `sase-15h`;
  - new bug task `sase-1n0`;
  - the soak and the drill were declined.

  Do not file any of these again.

The land review found defects **caused by this epic** that must be fixed before the epic
closes. They are listed below. Each one is verified, with file and line.

Ground rules carried over from the epic plan:

- **Fail closed.** Missing, partial or unreadable evidence is never green.
- **Detectors, not gates.** Do not make the canary or headroom alert block anything.
- **Leave E2/E3 files alone.** Do not touch:
  - sase: `publish.yml`, `ci.yml`, `build-core.yml`, `master-gate.yml`, `full.yml`,
    `tools/release_watch*`, `tools/ratchet_core_*`, `release-please-config.json`;
  - sase-core: the `release-plz.yml` jobs.
- **Human-only actions stay human.** Never yank, delete, or set up PyPI/TestPyPI
  projects.
- **Do not edit any memory note.** The epic decided `decision_record = no`.

## Part A — sase-core

Open the repo with `sase repo open sase-core -r "<why>"` and work only in the printed
path. Read its `AGENTS.md` first.

1. **CI is red on master because of a nonexistent action tag.**
   - The problem:
     - `.github/workflows/ci.yml:83` (the `release-scripts` job) uses
       `astral-sh/setup-uv@v10`.
     - So does `.github/workflows/pypi-headroom.yml:35`.
     - That tag does not exist. Upstream tags run from `v3` to `v7`, and sase pins `@v4`
       everywhere.
     - Every sase-core CI run since `9876b3ae` fails with
       `Unable to resolve action astral-sh/setup-uv@v10`.
     - The weekly headroom workflow would fail before its first step, which would be a
       permanent false alert.
   - The fix: change both to `astral-sh/setup-uv@v4`, matching sase.
   - Confirm the tag exists with
     `git ls-remote --tags https://github.com/astral-sh/setup-uv`.
2. **The delete guard can be bypassed by spelling the package name differently.**
   - `.github/scripts/pypi_delete.py:500` gates the protected-set check on
     `args.package == retention.PACKAGE`. That is an exact string match, so
     `--package sase_core_rs` or `SASE-Core-RS` skips the guard.
   - Compare names with `packaging.utils.canonicalize_name` instead. The retention
     module already depends on `packaging`.
   - Add a test proving that a non-canonical spelling is still refused before any
     credential is read.
3. **The backfill commands in the docs fail as written.**
   - In `docs/pypi-retention.md`, "Core backfill" steps 1 and 6 run
     `python3 .github/scripts/pypi_release_files.py status <version>` without setting
     `EXPECTED_DIST_SUFFIXES`. As written that exits 1 with "EXPECTED_DIST_SUFFIXES is
     not set".
   - Use the env-prefixed form that the heal section already shows, a few lines earlier
     (it reads `EXPECTED_DIST_SUFFIXES` from `release-plz.yml`), or point to it
     explicitly.
4. **The headroom alert text hard-codes the threshold.**
   - `.github/workflows/pypi-headroom.yml` hard-codes "30 days" in:
     - the issue title (line ~54);
     - the label description (line ~66);
     - the issue body (line ~75).
   - The `min_runway_days` dispatch input can differ.
   - Keep one fixed, threshold-free title so that dedupe stays stable, for example
     `[pypi-headroom] sase-core-rs runway below threshold`. Put the actual threshold
     value in the body.
5. **Verify:** run `sase tool run check` in the sase-core checkout. Two gateway flakes
   are known: `sase-1gz` and `sase-15h`. If one of them fails, rerun once and note it.
   Do not treat it as yours.

## Part B — sase: the family install canary

The files are `.github/workflows/install-smoke.yml`, `tools/check_family_install`,
`tests/test_github_actions_install_smoke.py` and `tests/test_check_family_install.py`.
Keep `tests/test_render_applicability.py` green.

1. **The alert job has no repo context (blocking).**
   - The problem:
     - Job `install-canary-alert` has no checkout, no `GH_REPO` and no `--repo`.
     - The live run `39298650862` shows
       `failed to run git: fatal: not a git repository`.
     - So `gh issue list` always yields nothing (its error is swallowed by `|| true`), a
       red run's `gh issue create` fails, and a green run never closes the issue.
   - The fix: add `GH_REPO: ${{ github.repository }}` to that step's `env`, next to
     `GH_TOKEN`.
2. **A missing leg can read as green (fail-closed violation).**
   - The problem: the alert verdict is red only when there are _no_ artifacts. One
     missing leg plus one green leg reads as green.
   - The fix, so the alert step is green only when all of these hold:
     - **Both artifacts are present.** `install-canary-ubuntu-latest.json` and
       `install-canary-macos-latest.json` both exist, parse, and say `green`. Hardcode
       the expected set with a comment that it mirrors the matrix, and add a contract
       test that keeps the set equal to `jobs.install-smoke.strategy.matrix.os`.
     - **The smoke job succeeded.** `needs.install-smoke.result == 'success'`. Pass it
       in through `env`.
3. **A red job can upload a green artifact.**
   - The problem: the "Merge the canary result contract" step reads only the two helper
     JSON files. A failure in any of these steps leaves both files green:
     - the CLI smoke;
     - `sase core health`;
     - doctor;
     - completion;
     - the plugin proof.

     The artifact then says `green`, and the alert would close the issue.

   - The fix:
     - Give every step that runs before the merge step an `id`.
     - Pass `${{ toJSON(steps) }}` to the merge step through `env`.
     - In the merge step, walk an explicit ordered list of those step ids. If any has
       `outcome == 'failure'`, force the merged verdict to red, set `failing_step` to
       the first failed id, and give a reason that names it.

4. **Early failures produce a weak fallback artifact.**
   - The problem: when the documented install, the floor check, the family install or
     the exact reinstall fails, the helper never runs. The fallback dict then lacks
     `os`, `run_id`, `attempt` and `sha`, and it reports
     `failing_step: read-result-artifact`.
   - The fix:
     - Fill the identity fields from the environment: `matrix.os`, `GITHUB_RUN_ID`,
       `GITHUB_RUN_ATTEMPT` and `GITHUB_SHA`.
     - Prefer the first failed step id from item 3 as `failing_step`.
5. **The `os` field disagrees with the contract.**
   - The problem: the header contract example says `"os": "ubuntu-latest"`, but the
     helper defaults `--os` to `RUNNER_OS`, which is `Linux`.
   - The fix: pass `--os "${{ matrix.os }}"` to both assertion calls, so that an
     artifact's name and its content agree for E2.6.
6. **The probe input is pasted straight into the shell.**
   - The problem: `${{ inputs.probe_expect }}` is expanded inside `run:` scripts.
   - The fix: move it into a step `env:` variable (for example `PROBE_EXPECT`) and quote
     it in the shell.
   - Add a contract test that no `run:` block contains `${{ inputs.`.
7. **The core expectation goes stale on re-fetch.**
   - The problem: `_refetch` in `tools/check_family_install` (around line 600) returns
     the old `core_expected`. A core release that lands mid-run is therefore never
     absorbed.
   - The fix: recompute it with
     `core_expectation(installed.get("sase"), fresh["sase"], _fetch_json)`.
   - Add a unit test.
8. **The exact assertion re-fetches newest instead of checking the pins.**
   - The problem: the "Reinstall the exact current release" step prints pins, but the
     exact assertion step re-fetches the newest versions. A release that lands between
     the two steps reads as a mismatch.
   - The fix: hand the pins to the assertion step, for example through `$GITHUB_ENV` or
     a file, and pass them as `--expect NAME==VERSION`. The helper already supports
     `--expect`, and it switches the rule to `exact`.
9. **The plugin proof never loads anything.**
   - The problem:
     - `check_plugins_loaded` passes whenever `plugin_signals` is non-empty.
     - `src/sase/version/_plugins.py:49-50` always adds `distribution_name:<dist>` for
       any installed `sase-*` distribution, without loading anything.
     - So the check proves "installed", not "entry points loaded".
   - The fix:
     - Run a small load script under the **installed tool's own interpreter**. Resolve
       it from the installed `sase` executable's shebang, or use
       `$UV_TOOL_DIR/sase/bin/python`. Do not use uv's interpreter.
     - For each plugin in `LOAD_PROOF_PLUGINS`, the script enumerates that
       distribution's entry points in the groups sase recognises
       (`sase.plugins.inventory.ENTRY_POINT_GROUPS`) and calls `.load()` on each.
     - Red when a plugin has no such entry point, or any `.load()` raises. Name the
       plugin, the group and the entry point.
   - Keep the verdict logic in a pure function over the load report, and unit-test it.
   - Wire it into the "Prove each plugin's entry points loaded" step.
10. **Tests.** Extend the workflow contract tests and the helper unit tests to cover
    items 1–9:
    - `GH_REPO` on the alert step;
    - the alert reads `needs.install-smoke.result`;
    - the expected artifact set equals the matrix;
    - no `${{ inputs.` inside `run:` blocks;
    - every pre-merge step has an `id`;
    - re-fetch recomputes the core expectation;
    - the load-report verdict;
    - fallback identity fields.

## Part C — sase docs (`docs/contribute/release-safety.md`)

Read the `docs_writing` memory with `/sase_memory_read` before editing.

1. **The safety-net bullet for the canary is wrong in two places.**
   - Its contract list omits `attempt` and `sha`. Add them.
   - Re-read the sentences about the alert issue and "a missing artifact counts as red".
     They must be true after Part B, and they must say that _both_ OS artifacts are
     required.
2. **The floor-check bullet says "hourly".** It should say: hourly for the first day,
   then every four hours, stopping at GitHub's 50-attempt rerun cap. That matches the
   shared `release-dependency-floor.yml`.
3. **The "Related" section claims "`INSTALL.md` shows the family form", which is
   false.** The canary's family command is `INSTALL.md`'s
   `--with sase-github --with sase-telegram` form plus `--with sase-research-artifacts`,
   per the epic's `research_artifacts = yes` decision. Do not change `INSTALL.md`.
4. **"Practice drill"** follows `drill_tooling` (below).
5. **Run the docs checks** that the `lint_and_test` memory names, including the docs
   map, if they apply.

> [!decision] drill_tooling = keep
>
> - Leave `.github/workflows/testpypi-drill.yml`, `tools/check_testpypi_drill` and their
>   tests in place.
> - Reword "Practice drill" to make three points:
>   - the drill is an optional rehearsal;
>   - it was not run when E1 landed;
>   - it needs the human-only TestPyPI project and pending-publisher setup first.

> [!decision] drill_tooling = remove
>
> - Delete these files:
>   - `.github/workflows/testpypi-drill.yml`;
>   - `tools/check_testpypi_drill`;
>   - `tests/test_check_testpypi_drill.py`;
>   - `tests/test_github_actions_testpypi_drill.py`.
> - Delete the `_load_testpypi_drill_workflow` helper in
>   `tests/_github_actions_ci_helpers.py`.
> - Delete `test_ci_and_publish_never_invoke_the_testpypi_drill` in
>   `tests/test_github_actions_ci_workflow.py`.
> - Delete the "Practice drill" section, and any other drill mention in
>   `release-safety.md`.
> - `grep -rn -i 'testpypi\|check_testpypi_drill' .` must find nothing left in tracked
>   files. Plan and research archives are the exception.

## Part D — verify sase

1. Read the `lint_and_test` memory with `/sase_memory_read`, then run what it requires
   (`just check` through `sase tool run`). Do **not** run `just check-full`.
2. Do a local live proof on this host:
   1. Install the family into isolated `UV_TOOL_DIR`, `UV_TOOL_BIN_DIR` and `HOME`
      directories, using the command printed by
      `python3 tools/render_applicability --install-family-command`.
   2. Run `uv run --no-project --with packaging tools/check_family_install` in newest
      mode. It must be green.
   3. Run the new plugin load proof against that install. It must be green.
   4. Rerun with `--force-expect sase==0.0.0`. It must be red.
   5. Record the outputs for the close note.

   You cannot observe CI for your own changes. The host commits after your turn, so do
   not wait for CI.

## Part E — close the epic (final step, same turn)

Do this in the same turn as the code. Nothing resumes this landing afterwards.

1. **Epic symbols.** Run `sase bead epic-symbols sase-1my`.
   - It printed no entries at land time. If entries appear now, resolve each one per the
     Symvision epic-whitelist policy (read the `symvision` memory): wire it up,
     privatize it, add a non-test pragma, or delete it.
   - No later bead stays open to re-key to.
2. **Close.** Run `sase bead close sase-1my --note "<verification>"`. The note must
   state:
   - **The live proofs:**
     - V-E1a: `sase-1my.6` note #1, with `plan --keep 30` protecting 0.42.0 and 0.42.1
       "by sase 0.18.0".
     - V-E1b: runs `39298640599` (green, both OSes) and `39298650862` (probe red).
     - V-E1c: sase-github runs `39292291601` (probe red) and `39292296600` (green).
   - **The fixes this tale landed:**
     - Part A: the setup-uv pin, the canonical-name guard, the backfill commands and the
       headroom wording.
     - Part B: items 1–9.
     - Part C: the docs.
     - The `drill_tooling` outcome.
   - **The local proof results** from Part D.
   - **The follow-up outcomes** already in the epic note:
     - `sase-1gz` and `sase-15h` got +1s;
     - `sase-1n0` was filed;
     - the soak and the drill were declined, per the owner's `sase-1my.6` close;
     - the minor items were declined.
   - **The soak caveat:** the 7-day scheduled-canary soak was **not** completed. The
     fixed canary's `install-canary` alert issue, mirrored into a task bead, is the
     ongoing detector.
   - **Integration:** none of the non-epic commits since the epic began touch its
     surface.

   Never pass `--force`. If the close is refused, fix the cause and close again.

3. **Symvision.** Run `just symvision` in sase and confirm it is clean.
4. **Plan status.** Set `status: done` in the frontmatter of the epic's plan file, the
   PLAN path printed by `sase bead read sase-1my -r "<why>"`. It currently reads
   `status: wip`.
5. **Parent bead.** `sase-1my` has none, so stop here.
