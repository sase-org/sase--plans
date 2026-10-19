---
tier: epic
title: "E2: Exact-candidate release train and steward"
goal: "Every sase release is a sealed candidate built once at its tag: the wheel and
  sdist plus an exact `sase-core-rs==X.Y.Z` with its full file set and digests. It is
  qualified against the published core, exercised on Linux and macOS, published from the
  retained bytes with digest-checked resume, and confirmed from PyPI before it is
  un-drafted. One stateless steward decides release merges and the core cut, and every
  blocked release names a typed reason and its owner.

  "
decisions:
  steward_host:
    ask: Where should the release steward's loop run?
    choices:
      workflow:
        A scheduled sase workflow runs master's steward; escalations become mirrored
        issues
      chop:
        An athena axe chop runs it via a chezmoi wrapper; escalations become beads and
        notifications
    default: workflow
    why:
      Always runs master's own steward code, needs no athena checkout, and survives
      athena downtime
    answer: workflow
  release_cadence:
    ask: How often may the steward merge a sase release PR?
    choices:
      daily:
        At most one release per UTC day, after the daily core cut; a burned candidate
        frees the slot
      eligible:
        Whenever eligible and no train is in flight; several releases a day are possible
    default: daily
    why:
      Matches the daily core cut and keeps the ten supported releases spanning about ten
      days
    answer: daily
  memory_release_record:
    ask:
      Add decisions:exact-release-pair and mark ci-two-speed-split and
      rust-core-required superseded in part?
    memory:
      - decisions:exact-release-pair
      - decisions:ci-two-speed-split
      - decisions:rust-core-required
    default: true
    corrects:
      "`ci_watch` gates release on both signals: `gating_workflows` proves this commit's
      fast gate is green, and `heavy_max_age_hours` requires a recent green `Full CI`
      run."
    answer: true
phases:
  - id: core-release
    title: Core-cut resilience and the in-flight retention source
    depends_on: []
    size: medium
    description:
      "core-release: in sase-core, the release-plz merge job reruns failed checks once
      and emits a typed CORE_CUT_RED, and retention also protects the exact core pins of
      sase tags not yet on PyPI."
  - id: exact-build
    title: release.yml builds once at the tag and seals the candidate manifest
    depends_on: []
    size: medium
    description:
      "exact-build: a new tag-dispatched release.yml with an identity assertion, one
      build, a sealed candidate manifest stashed on the draft release, a rehearse mode,
      and the gate for the PyPI publisher and tag policy."
  - id: core-pin
    title: Exact core pin and the shared core-ensure rule
    depends_on: []
    size: medium
    description:
      "core-pin: ratchet_core_pin writes sase-core-rs==X.Y.Z chosen by one shared
      core-ensure rule, master converts to the exact form, and every tool that assumed a
      window accepts the pin."
  - id: exercise
    title: Exercise the exact files on Linux and macOS
    depends_on:
      - exact-build
    size: medium
    description:
      "exercise: release.yml installs the stashed wheel and sdist into clean
      environments on Linux and macOS and runs a no-LLM fakey workflow, an upgrade from
      N-1, and pip check."
  - id: qualify
    title: Qualify the candidate against the published core
    depends_on:
      - exact-build
      - core-pin
      - exercise
    size: large
    description:
      "qualify: ci.yml gains ref and core-source inputs, a versioned N1 inventory
      decides what blocks, and release.yml qualifies against the exact published core
      with fail-closed evidence."
  - id: publish
    title: Digest-checked publish, PyPI confirmation, and un-draft
    depends_on:
      - qualify
    size: medium
    description:
      "publish: release.yml uploads only missing files after digest checks, confirms
      every file and an exact install from PyPI, runs the family canary, then un-drafts;
      a resume mode re-enters at publish."
  - id: steward
    title: Stateless release steward in observe mode
    depends_on:
      - exact-build
      - core-pin
      - core-release
    size: large
    description:
      "steward: tools/release_watch becomes a stateless reconciler with typed states,
      one idempotent action per tick, core-ensure, bounded reruns, abandonment, and 24h
      escalation, running observe-only."
  - id: metrics
    title: Release-green record and release metrics
    depends_on:
      - steward
    size: medium
    description:
      "metrics: a read-only tools/release_metrics derives release-green per SHA and the
      release figures, each with numerator, denominator and window, and publishes a
      pre-train baseline."
  - id: handover
    title: Cut releases over to release.yml and hand merges to the steward
    depends_on:
      - publish
      - steward
      - metrics
    size: medium
    description:
      "handover: release-please goes draft with forced tags, publish.yml keeps only
      release-please and metadata sync, the steward goes live, ci_watch drops sase
      releases, and a gate hands over merge authority."
  - id: exit
    title: First steward release, acceptance evidence, and the decision record
    depends_on:
      - handover
    size: medium
    description:
      "exit: confirm the human handover steps, watch the first real release through the
      steward, prove digests and attestation match the manifest, record acceptance
      evidence, and apply the memory decisions."
proposed_by: bbugyi200.athena.11y
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 16:40:05
status: wip
---

- **PROMPT:**
  [prompts/202610/e2_release_train_steward.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/e2_release_train_steward.md)

# Plan: E2 Exact-candidate release train and steward

This is epic **E2** of the converged PyPI release reliability design. Every phase agent
should first read the design's "The releasable promise", "E2 Exact-candidate release
train and steward", "File ownership" and "Acceptance cases" sections:

```bash
sase artifact read research:202610/pypi_release_reliability_design_and_work_split/pypi_release_reliability_design_and_work_split.md "<why>"
```

The check inventory this epic implements is N1 in
`research:202610/merge_queue_fence_reassessment_and_epic_split/merge_queue_fence_reassessment_and_epic_split.md`.

E1 (`sase-1my`, installable family) closed on 2026-10-19. Its canary artifact contract,
floor check and retention seam are inputs here. E3 (trustworthy signal) has **no epic
bead yet**, so this plan does not wait for E3.1's node-level rerun helper: the steward
supplies one bounded job-level rerun, and qualification inherits E3.1's node-level
reruns automatically once E3 adopts them in `ci.yml`'s test steps.

## Where things stand (measured 2026-10-19)

- **PyPI:** `sase` 0.18.0 requires `sase-core-rs>=0.42.0,<0.43.0`; `uv.lock` locks
  0.42.0. `sase-core-rs` 0.42.1 is newest. `sase-core-revision.txt` is `f0f2275f…`.
- **`publish.yml`** (cron `17 */3 * * *`, plus `workflow_dispatch` with
  `publish_existing`): jobs `release` (three release-please retries; outputs
  `release_created` and `sha`) → `sync-release-metadata` (runs
  `tools/ratchet_core_window` on the release-please branch and pushes "chore: sync
  release metadata") → `build` (already checks out `needs.release.outputs.sha`) →
  `install-smoke` and `install-smoke-core-floor` (Linux, wheel only) → `publish`
  (`environment: pypi`, `skip-existing: true`, no digest check, no PyPI confirmation).
- **release-please:** `release-please-config.json` has no `draft` and no
  `force-tag-creation`; the only extra file is `src/sase/__init__.py`. Tags are
  `vX.Y.Z`. **No workflow or tool touches GitHub releases today** (no `gh release`
  anywhere).
- **`ci.yml`:** triggers `pull_request` and `workflow_call` with **no inputs**; bare
  checkouts; `concurrency: ci-${{ github.ref }}` with `cancel-in-progress: true`. Core
  comes from `build-core.yml` (source at `sase-core-revision.txt`) through the
  `setup-sase` action, which needs exactly one wheel, one `sase-core-sha.txt` and one
  `sase-macro-lsp` binary. On the release PR only `release-core-floor-smoke` runs (it
  installs `sase-core-rs==<floor>` from PyPI, runs the bindings check, seven core smokes
  and the contract set).
- **No release-blocking inventory exists.** There is no `release_blocking` marker and no
  N1 file. Release gating lives in `ci_watch`.
- **GitHub:** the `pypi` environment allows **only the `master` branch** (one custom
  branch policy). There is no `testpypi` environment. `SASE_RELEASE_TOKEN` exists in
  sase and sase-core. The repo allows merge commits only.
- **`tools/release_watch`** (about 2.8k lines, 87 tests): one long-running stateful
  watcher with verdicts `RELEASED`, `FULL_CI_RED`, `PUBLISH_FAILED`,
  `CORE_FLOOR_BLOCKED`, `RELEASE_PR_GONE`, `INSTALL_FAILED`, `DEADLINE`,
  `WATCHER_BROKEN`, `MASTER_GATE_RED`, `STOPPED`, `CORE_CI_RED`. It dispatches Full CI,
  Publish and the core cut, reruns Master Gate shards, closes a bead, keeps cooldowns in
  memory, and **never merges**. Nothing schedules it.
- **Exact pin today breaks four places:** `validate_sase_core_rs_version` classifies an
  `==` mismatch as generic exit 1, so `tools/_sase_install_core.check_version_window`
  fails dev installs, and the Justfile `rust-install`/`rust-dev-install` and
  `validate_test_environment` lose their "behind" guard;
  `smoke_sase_core_rs_telemetry --print-minimum` requires `>=`; and two tests
  (`test_powerful_variables_landing.py`, `test_queue_directive.py`) regex-match a
  window. Runtime never compares versions, and dev installs already lift the requirement
  through overrides.
- **sase-core `release-plz.yml`:** the daily cut (`41 7 * * *`) and a dispatch with
  `dry_run=false` run `release-plz-merge`, which waits on
  `gh pr checks --watch --fail-fast` and squash-merges. It never reruns, and a failure
  is an untyped red step. `EXPECTED_DIST_SUFFIXES` (five files) is the single source of
  core completeness. `pypi_retention.py compute_protected_set()` has a documented E2
  seam comment.
- **ci_watch** (bugyi-chops, config in chezmoi `home/dot_config/sase/sase_athena.yml`):
  merges sase's release PR when Master Gate is green, Full CI is green within 6 hours
  (`heavy_max_age_hours`) and every rollup check is green, where **SKIPPED counts as
  green**. `merge_enabled` is **one global boolean**; removing `sase-org/sase` from
  `release_repositories` stops both its sase merges and its sase Full CI dispatch
  (`full.yml` keeps its own two-hour cron).
- **Deterministic provider:** the bundled `fakey` provider (`docs/contribute/fakey.md`).

## Shared contract (fixed here so parallel phases agree)

**`release.yml` modes** (a `workflow_dispatch` input `mode`):

- `release`: dispatched at a `v*` tag. Runs every job, stashes on the draft GitHub
  release, publishes and confirms.
- `resume`: dispatched at a `v*` tag that already has a sealed generation whose required
  evidence passed. Runs only `publish` and `confirm`, from the stashed bytes.
- `rehearse`: dispatched at a branch (normally `master`). It builds, seals, qualifies
  and exercises, but stores everything as workflow artifacts, never touches a GitHub
  release, and runs `publish` as a verify-only plan with no upload.

**Job ids:** `identity`, `build`, `stash`, `qualify`, `exercise`, `publish`, `confirm`.
`publish` needs `qualify` and `exercise`. Concurrency is `release-${{ github.ref }}`
with no cancel; the qualify call uses `qualify-<sha>`, never cancelled.

**Candidate manifest** (owned by `tools/release_candidate`, created in `exact-build`):

- **Sealed at build, immutable:** the repository, version, tag (or the rehearsal ref),
  peeled SHA and tree; the candidate ID and build generation; the builder workflow
  revision, run ID and attempt; the core source pin; the exact core version with its
  full file set and PyPI digests; and every dist's filename, size and SHA-256 plus its
  asset locator.
- **Candidate ID:** derived only from sealed fields. A rebuild at the same tag is a new
  generation that must be qualified again. Tags never move.
- **Observations, append-only:** each one is its own asset named by generation, kind,
  run ID and attempt, so a re-append is idempotent and nothing is rewritten. The kinds
  are `qualification`, `exercise`, `publication`, `confirmation`, `abandoned` and
  `veto`. `exact-build` defines all of them, and later phases only fill them in.
- **Storage:** in `release` and `resume` modes, assets on the draft GitHub release; in
  `rehearse` mode, workflow artifacts.

**Promotion predicate** (one implementation, `tools/release_candidate evaluate`, used by
the `publish` job and the steward):

```text
publishable(candidate) =
    source_and_version_identity_match
    AND exact_expected_files_retained_and_hashed
    AND every_required_check_has_acceptable_evidence_for_this_candidate
    AND public_dependency_contract_satisfied   # exact core pin on PyPI, complete, contains the source pin
    AND no_confirmed_release_veto
```

**Core-ensure rule** (one module, `tools/_release_core.py`, created in `core-pin`): pick
X.Y.Z as the newest _complete_ published `sase-core-rs` whose tag `vX.Y.Z` contains the
candidate's `sase-core-revision.txt` commit. Complete means every
`EXPECTED_DIST_SUFFIXES` file is present and none is yanked, with the suffix set read
from sase-core's `release-plz.yml` at that tag and never copied. If no version
qualifies, the result is `CORE_LAG`. An unreadable input is an error, never a pick.

## Boundaries shared by every phase

1. **Missing evidence is red.** Skipped, cancelled, missing, stale, zero-collected or
   unreadable evidence never satisfies a gate. Unlike ci_watch, **SKIPPED is red**
   wherever E2 evaluates a required check.
2. **Human-only actions stay human.** They are: PyPI trusted publishers, GitHub
   environment and ruleset settings, repository variables, `chezmoi update` on athena,
   yanks and deletions. Propose them through `/sase_gate` as the phase's **last**
   action, because creating a gate ends the turn. Record the gate ID in a bead note so
   that the next phase can check `sase gate show`.
3. **Agents never create PRs, branches, tags or commits** (`host-owned-completion`).
   Live proofs use `workflow_dispatch` rehearsals at `master`, read-only `--once`
   steward runs and fixtures. Destructive cases run on fixtures only.
4. **A workflow dispatched at a tag runs that tag's own workflow files.** A tag cut
   before `release.yml` existed can never run it, and a `release.yml` fix reaches only
   later tags. Rehearse at `master`. A candidate broken by its own `release.yml` is
   burned, never patched in place.
5. **Single sources stay single:**
   - the version lives in `pyproject.toml` and `src/sase/__init__.py`, bumped by
     release-please;
   - the core completeness set lives in sase-core's `release-plz.yml`;
   - the N1 inventory is one versioned file;
   - the documented install command stays in `tools/render_applicability`.
6. **Ownership:**
   - **E2 owns:** `publish.yml`, the new `release.yml` and `release-steward.yml`,
     `ci.yml` **inputs, checkouts, concurrency, core source and evidence-reporting
     steps**, `build-core.yml` ref plumbing, `tools/release_watch*`,
     `tools/ratchet_core_*`, `tools/release_candidate*`, `tools/_release_core.py`,
     `tools/release_metrics*`, `release-please-config.json`, sase-core's
     `release-plz.yml` merge job, and the chezmoi ci_watch config.
   - **E3 owns** the test steps, the quarantine file and `culprit-check.yml`.
   - **E1's surfaces are read-only here.** That covers `install-smoke.yml`,
     `release-dependency-floor.yml`, `tools/check_family_install` and the retention
     rules; reuse them by import or call. The one exception is the retention seam in
     `core-release`.
   - Two agents must never rewrite the same workflow at once. The phase graph serializes
     every `release.yml` and `ci.yml` editor. If an E3 phase bead is in progress on
     `ci.yml` when `qualify` starts, rebase onto its landed work and keep your diff to
     the E2-owned parts.
7. **Repos:**
   - Open sase-core and chezmoi with `sase repo open <name> -r "<why>"` and work only in
     the printed path. Each modified repo is a commit obligation of that phase's
     declaration.
   - Do not modify bugyi-chops; ci_watch needs only a config change.
   - Verify sase-core with `sase tool run check` in its checkout (see its `AGENTS.md`).
     Verify sase per the `lint_and_test` memory.
   - New contract tests join `tests/contract_manifest.txt` through
     `just refresh-contract-manifest`, within its budget rules.
8. **Docs:** read the `docs_writing` memory first. Release internals and runbooks live
   in `contribute/`. `exact-build` creates `docs/contribute/release-train.md`.
   - `exercise`, `qualify` and `publish` extend it in turn (they are serialized).
   - `steward` and `metrics` document themselves only in their tool help and docstrings.
   - `handover` writes the steward and metrics sections, so no two parallel phases edit
     the same page.

## core-release

**Repo:** sase-core. **Files:** `.github/workflows/release-plz.yml`, the
`release-plz-merge` job only; a new `.github/scripts/release_plz_merge.py` with tests;
`.github/scripts/pypi_retention.py` at the E2 seam; `docs/pypi-retention.md`; and the
existing `test_release_cadence.py` contract.

**Merge-job resilience:**

- Move the "wait for checks, then merge" logic out of inline shell into
  `release_plz_merge.py`. Keep the PR identification rules exactly as they are: the
  author, the `release-plz-<timestamp>` branch, the `chore: release vX.Y.Z` title and
  the body footer.
- On the first red: rerun the failed jobs of each failing check's run **once**
  (`gh run rerun <id> --failed`), wait again, then merge on green.
- **Never rerun** a check that never registered, a cancelled run, or a second failure.
- On a refusal, exit non-zero. Write one typed line to the step summary and to
  `::error::`, for example `CORE_CUT_RED check=<name> run=<url> attempt=<n>`, plus job
  outputs (`outcome`, `failed_check`, `run_url`). The outcomes are `MERGED`,
  `NO_RELEASE_PR` (a notice and success, as today), `CHECKS_NEVER_REGISTERED` and
  `CORE_CUT_RED`.
- Keep the `workflow_dispatch` entry point and its inputs unchanged. The steward calls
  it.

**In-flight retention source:**

- At the seam in `compute_protected_set()`, add one source: the exact core pins declared
  in `pyproject.toml` at every sase `v*` tag newer than the newest sase release on PyPI.
  These are trains in flight whose tag exists but whose files are not yet public.
- Read the tags with `git ls-remote --tags`, and each `pyproject.toml` through a public,
  unauthenticated fetch.
- Record why each entry is protected ("sase tag v0.19.0 (not yet on PyPI) pins
  sase-core-rs==0.43.2"). Fail closed on any unreadable tag or file.
- A window specifier at a tag (pre-E2 tags) is ignored by this source; E1's rules cover
  it.
- Update the seam comment and the doc's protected-set list.

**Tests** (fixtures, no network): a flaky check is rerun once and merges; a second red
refuses with `CORE_CUT_RED` naming the check; a never-registered check refuses without a
rerun; no PR is a notice; an in-flight tag's pin is protected; an unreadable tag fails
closed; a window at a tag is ignored by the new source.

**Done when:** `sase tool run check` passes in sase-core. A live read-only
`plan --keep 30` still matches E1's V-E1a output plus an empty in-flight source. The
next scheduled daily cut's run log shows the new script (record the run URL in a bead
note, or note it as pending if no cut ran before the phase ends).

## exact-build

**Repo:** sase. **Files:** a new `.github/workflows/release.yml`;
`tools/release_candidate` plus `tools/_release_candidate*.py`; tests
(`tests/test_release_candidate*.py`, and `tests/test_github_actions_release.py` with a
new loader in `tests/_github_actions_ci_helpers.py`); and a new
`docs/contribute/release-train.md`.

**`release.yml`, the jobs this phase owns:**

- **Triggers:** `workflow_dispatch` only, with inputs `mode` (`release` | `resume` |
  `rehearse`) and `core_version` (rehearse only: an override, sealed into the manifest
  as `override`). Add no schedule and no push trigger.
- **`identity`:**
  - in `release` and `resume` modes, require `github.ref == refs/tags/vX.Y.Z` and that
    `GITHUB_SHA`, the peeled tag SHA, `pyproject.toml`'s version and `__version__` all
    agree;
  - in `rehearse` mode, require a branch ref, and assert that the ref SHA, the checkout
    SHA, `pyproject.toml` and `__version__` all agree;
  - output the version, SHA, tree and mode.
- **`build`:**
  - check out exactly the identity SHA, then run `uv build` and `twine check`;
  - verify that the dist set is exactly one wheel plus one sdist named for the version;
  - read the exact core pin: from `==X.Y.Z` in `pyproject.toml` at the ref, else (pre-E2
    windows, rehearsals) from `uv.lock`'s locked version, or from the rehearse override;
  - fetch that core's PyPI file list and digests, and cross-check them against
    `uv.lock`'s hashes where the lock covers that version;
  - run `tools/release_candidate seal`.
- **`stash`** (`release` mode only): upload the dists and the sealed manifest as assets
  on the draft GitHub release for the tag. Refuse if that release is missing or is not a
  draft. Refuse a second sealed generation unless the input `rebuild: true` is given, so
  that a duplicate dispatch can never double-build.
- Leave `qualify`, `exercise`, `publish` and `confirm` to their phases. Until they
  exist, `release` mode must stop red after `stash` with "not yet qualified". It must
  never succeed.

**`tools/release_candidate`:**

- subcommands `seal`, `verify`, `observe`, `show`, `evaluate` (the promotion predicate,
  with qualification and exercise clauses that read "required evidence missing" until
  those phases fill them) and `download`;
- a schema version, and the observation kinds from the shared contract;
- stdlib plus `packaging` only, run through `uv run --no-project --with packaging`.

**release-please draft support, verify only (do not flip it here):**

- Open `googleapis/release-please-action` and `googleapis/release-please` as external
  repos with `sase repo open`.
- Confirm from source that the release-please version bundled in our pinned
  `release-please-action@v5` honors `draft: true` together with
  `force-tag-creation: true`. In particular, confirm that the tag is created at once and
  that the next run still finds the previous release.
- Record the evidence (versions and file/line) in a bead note.
- If the option is unsupported, name in that note the action version that supports it,
  or the fallback: non-draft releases whose title is marked qualifying until `confirm`.
  `handover` applies whichever you record.

**Docs:** `docs/contribute/release-train.md` covers the topology, the modes, the
identity rule, the manifest schema and the tag-frozen-workflow rule. Add it to the nav
per the `docs_writing` memory.

**Tests** (fixtures, no network):

- each identity mismatch refuses: tag versus SHA, pyproject versus `__version__`, a
  branch ref in `release` mode;
- the candidate ID is stable across observations and changes with the generation;
- an extra or missing dist refuses;
- an incomplete core file set refuses to seal (acceptance case 4);
- a second `stash` without `rebuild` refuses;
- a workflow contract test asserts the triggers, the job graph, the `release-<ref>`
  no-cancel concurrency, the absence of push or schedule triggers, and that `release`
  mode can't succeed without later evidence.

**Live proof:** dispatch `release.yml -f mode=rehearse` at `master`. The sealed
rehearsal manifest names the dispatched SHA, the wheel's METADATA names the version, and
the core file set equals PyPI's. Record the run URL in a bead note.

**Gate (last action):** raise one `/sase_gate` for the human with two steps:

1. on PyPI, add a trusted publisher for `sase` (owner `sase-org`, repository `sase`,
   workflow `release.yml`, environment `pypi`);
2. in GitHub, add a `v*` **tag** rule to the `pypi` environment, keeping the existing
   `master` branch rule until `handover`.

The gate's body explains that the legacy `publish.yml` path is unaffected.

## core-pin

**Repo:** sase. **Files:**

- `tools/ratchet_core_window` renamed to `tools/ratchet_core_pin`, and the new
  `tools/_release_core.py`;
- `tools/validate_sase_core_rs_version`, `tools/validate_test_environment`,
  `tools/_sase_install_core.py`, `tools/smoke_sase_core_rs_telemetry`;
- the Justfile recipes `ratchet-core-window` (renamed), `rust-install` and
  `rust-dev-install`;
- `publish.yml`, the `sync-release-metadata` job only;
- `ci.yml`, the `release-core-floor-smoke` job's comments and assertions only;
- `pyproject.toml`, `uv.lock` and tests;
- `docs/contribute/rust-backend.md`, the "Who owns the published version window"
  section.

**Work:**

1. **Core-ensure** (`tools/_release_core.py`): implement the shared-contract rule.
   - Test containment through the GitHub compare API
     (`repos/sase-org/sase-core/compare/<pin>...vX.Y.Z` is `ahead` or `identical`), with
     a local-git fallback for offline fixtures.
   - Read the completeness set from `release-plz.yml` at the tag.
   - Return a typed result: `PINNED(X.Y.Z, files)` or `CORE_LAG(pin, newest_complete)`,
     or raise an error.
2. **`ratchet_core_pin`:**
   - write `sase-core-rs==X.Y.Z` using core-ensure's pick, keeping the existing strict
     `pyproject`/`uv.lock` diff checks and the rollback;
   - refuse to move the pin backwards; exit codes stay 0/2/3;
   - on `CORE_LAG`, change nothing and print the typed reason;
   - switch `sync-release-metadata` to it and rename the Justfile recipe.
3. **Exact-pin compatibility:**
   - `validate_sase_core_rs_version` classifies `==` as behind (3) or ahead (4), never
     generic 1, so `check_version_window`, `rust-install`, `rust-dev-install` and
     `validate_test_environment` keep their behind guard;
   - `--print-minimum` returns the pinned version for `==`;
   - fix the two regex tests.
   - Then grep for every other `>=`-only parse of the core requirement (tools, tests,
     `src/sase/dev_update`, docs) and fix it.
4. **The two floor smokes collapse.** Under an exact pin, the floor _is_ the pin and
   _is_ the newest in the window. `release-core-floor-smoke` keeps its name, because
   consumers parse it, but its comments and assertions now describe the pinned-core
   smoke. `publish.yml`'s two install smokes stay working until `handover` deletes them;
   do not restructure them.
5. **Spike, then convert master.** Change `master`'s requirement to
   `sase-core-rs==<the version uv.lock locks>`, with only the specifier changing in
   `uv.lock`. Prove that `uv lock --check`, `just install-venv` with and without
   `SASE_CORE_WHEEL`, and a `tools/sase_install` dev **plan** (not a global install; see
   `global-install-is-human-only`) all work. Release-please's python strategy bumps only
   `version`; confirm that it leaves the dependency line alone.
6. **Docs:** rewrite "Who owns the published version window" for the exact pin, the
   core-ensure rule and `CORE_LAG`.

**Tests:**

- core-ensure picks the newest complete version containing the pin;
- an incomplete newest version is skipped (acceptance case 4);
- a yanked file makes the version incomplete;
- no containing version gives `CORE_LAG`;
- an unreadable compare or workflow file raises;
- the ratchet writes `==` and refuses a backwards move;
- the validator classifies behind and ahead under `==`;
- `--print-minimum` handles `==`.

**Done when:** a report-only `ratchet_core_pin` against live PyPI and GitHub prints a
typed pick or `CORE_LAG` naming the pin; `master` builds and tests under the exact form
per the `lint_and_test` memory; the existing publish contract test still passes after
renaming.

## exercise

**Repo:** sase. **Files:** `release.yml` (the `exercise` job), a new
`tools/exercise_release` with tests, the contract test, and the "Exercise" section of
`docs/contribute/release-train.md`.

**Job:** a matrix of `os` [`ubuntu-latest`, `macos-latest`] × `artifact` [`wheel`,
`sdist`] with `fail-fast: false`. Each leg:

1. downloads the stashed dists (the draft release, or the rehearsal artifact) and
   verifies their digests against the sealed manifest;
2. creates a clean Python 3.12 environment with an isolated `HOME`, `XDG_*`,
   `SASE_HOME`, `UV_TOOL_DIR` and `UV_TOOL_BIN_DIR`, and `UV_NO_CACHE=1`, then installs
   the exact file with dependencies from public PyPI. With the exact pin this resolves
   the manifest's core, and the leg asserts that;
3. runs `tools/exercise_release`, a scripted no-LLM workflow using the `fakey` provider.
   The tool uses commands documented in `docs/contribute/fakey.md` and the CLI
   reference:
   - create durable work, then reload it in a new process;
   - cross the host/worker boundary once, through a real agent launch with fakey;
   - render one headless TUI frame, using the visual-snapshot tooling's headless path;
   - shut down cleanly with no orphan processes;
4. runs `uv pip check`, `sase core health --json`, and `sase version -j`, whose
   `packages` list must name the manifest's core version;
5. on the `wheel` legs only, runs an **upgrade from N−1**: install the previous PyPI
   release, create durable state with it, upgrade to the candidate wheel, then reload
   the state and rerun health.

**Evidence:** each leg uploads `exercise-<os>-<artifact>.json` with the verdict, the
first failed step and its reason. A final `exercise` summary step merges the four legs
into one `exercise` observation. A missing or unreadable leg makes the observation red.

**Probe:** add a rehearse-only input `probe: break-macos`. It deletes one imported
module from the installed package on the macOS legs only, so those legs must go red and
the evaluator must report `EXERCISE_RED(macos)` (acceptance case 8). Probes are refused
in `release` and `resume` modes.

**Tests:** the tool's step runner and verdict merge are tested with fixtures; a contract
test asserts the matrix, `fail-fast: false`, the isolation variables, the digest
verification before install, and that the probe is refused outside `rehearse`.

**Done when:** a rehearsal at `master` has four green legs, and a rehearsal with
`probe: break-macos` has red macOS legs and green Linux legs. Record both run URLs. A
macOS-only failure must withhold publication; this proof completes in `publish`, when
`publish` exists.

## qualify

**Repo:** sase. **Files:**

- `ci.yml`: inputs, checkouts, concurrency, core source and evidence-reporting steps;
  never test commands;
- `build-core.yml`: ref plumbing;
- the `setup-sase` action, only for accepting a published wheel;
- a new versioned N1 inventory file, for example `.github/release-qualification.json`;
- `release.yml`, the `qualify` job;
- `tools/release_candidate`'s qualification clause;
- tests, and the "Qualify" section of `release-train.md`.

**`ci.yml` as a qualifier:**

- **`workflow_call` inputs:** `ref` (default: the caller's ref), `core-source` (`source`
  | `published`, default `source`) and `core-version` (required for `published`).
  - Thread `ref` into every checkout, including the core-revision read in
    `build-core.yml`.
  - The defaults must keep `pull_request` runs and Full CI byte-for-byte equivalent in
    behavior. Contract-test that.
- **`core-source: published`:**
  - download the exact `sase_core_rs-<version>-…-manylinux_2_28_x86_64.whl` from PyPI
    and verify its SHA-256 against the sealed manifest;
  - build only the `sase-macro-lsp` binary from sase-core at tag `v<version>`, because
    the LSP is not in the wheel;
  - write `sase-core-sha.txt` as the tag's commit;
  - publish the same artifact names that `setup-sase` already expects;
  - assert that the installed core version equals the manifest's, and fail fast on a
    mismatch.
- **Concurrency:** the workflow-level group becomes input-aware: `qualify-<sha>` with no
  cancel when it is called for qualification, unchanged otherwise. A qualify run and a
  Full CI run on master never cancel each other and never share artifacts.
- **Evidence:**
  - Each blocking test job uploads `qualify-evidence-<job>.json` with its collected,
    passed, failed and error counts and whether collection errored, using pytest's
    machine-readable output.
  - This is an `if: always()` reporting step. Do not change how tests are selected or
    run.
  - E3.1's `FLAKY:` lines, once they exist, are copied through verbatim.

**N1 inventory** (one versioned file; its digest goes into the `qualification`
observation):

- **Release-blocking:** `lint`, `test (3.12)`, `test (3.13)`, `test (3.14)`, and the
  `exercise` legs. The pinned-core smoke on the release PR is the pre-merge instance.
- **Advisory:** `visual-test`, `perf-floors`, `tutorial-tests` and
  `ace-page-group-isolation`. They run on every candidate and never block. A failure is
  reported for the steward to escalate.
- The admission class (Master Gate) is unchanged and is not re-run here.

**Evaluation** (`release_candidate evaluate`, the qualification clause):

- every blocking job must be `success` with evidence showing at least one collected test
  and no collection error;
- `skipped`, `cancelled`, missing, zero-collected or unreadable evidence is red, and the
  reason is typed `QUALIFY_RED(<signature>)`;
- the signature names the job, its failure class (`tests` | `collection` | `zero` |
  `harness` | `missing`) and the first failing test IDs;
- write the `qualification` observation with run IDs, attempts, counts, the inventory
  digest and any `FLAKY:` lines.

**Bounded rerun contract, which the steward implements:**

- A failed qualification on attempt 1, whose every failing blocking job has class
  `tests`, gets **one** `gh run rerun --failed`.
- Any other class, or a second red, is final.
- The evaluator exposes a `rerun_eligible` boolean so that the steward does not
  reimplement the rule.

**Probe:** a rehearse-only input `probe_core_version` qualifies against an older
complete core (for example the oldest retained one). The bindings check in `lint` must
then fail against the published core while the source-built Full CI stays green, and the
evaluator must report `QUALIFY_RED` naming `lint` and the missing bindings (acceptance
case 3).

**Tests:**

- every red class above, using fixture run and job JSON;
- a contract test for the inputs, their defaults, the input-aware concurrency, and the
  rule that a published-core run never invokes the source build of the wheel;
- the inventory file is the only list of blocking job names (the evaluator, the steward
  and the docs read it).

**Done when:** a rehearsal at `master` qualifies green against the exact published core,
and the probe rehearsal is red with the typed signature.

- If `master` currently needs an unpublished binding, the honest outcome is a typed
  core-containment refusal at seal time.
- In that case, hand the retry to `/sase_monitor`: poll for the next daily core cut,
  with about a 36-hour deadline, and let its follow-up rehearse again.

## publish

**Repo:** sase. **Files:** `release.yml` (the `publish` and `confirm` jobs and `resume`
mode), `tools/release_candidate` (`publish-plan` and `confirm`), tests, and the "Publish
and confirm" section of `release-train.md`.

**`publish`** (`environment: pypi`, `permissions: id-token: write`, concurrency
`publish-<tag>` with no cancel):

1. Run the promotion predicate. In `resume` mode, also require that the generation's
   `qualification` and `exercise` observations already pass.
2. Download the stashed assets and verify every digest against the sealed manifest.
3. Run `publish-plan` against the PyPI JSON for the version:
   - an identical file already on PyPI is skipped;
   - an existing filename with a different digest refuses with `PUBLISH_CONFLICT`;
   - a PyPI file that is not in the manifest refuses;
   - a missing stashed asset refuses.
4. Upload only the planned subset with `pypa/gh-action-pypi-publish@release/v1` and
   `skip-existing: false`. Keep the default attestations on.
5. Write a `publication` observation that lists each file as uploaded or already
   present.

In `rehearse` mode, `publish` runs steps 1–3 only, as a verify-only plan with no
environment and no upload.

**`confirm`:**

1. Poll the PyPI JSON with a bounded wait for CDN lag, until every manifest file is
   listed with a matching SHA-256 and nothing extra is listed. Anything else is
   `PUBLISH_PARTIAL` or `PUBLISH_CONFLICT`.
2. On Linux and macOS, install `sase==<version>` from public PyPI into an isolated tool
   directory. Then require that `sase version -j` reports the manifest's core and that
   `sase core health --json` passes.
3. Run the family canary logic exactly: reuse `tools/check_family_install`'s exact-rule
   checks. Do not modify `install-smoke.yml`.
4. Un-draft with `gh release edit v<version> --draft=false --latest`, then write a
   `confirmation` observation.

**Tests** (fixtures, acceptance case 5):

- interrupt after one upload, then resume: identical files are skipped and the rest are
  uploaded;
- a different hash refuses safely;
- an extra PyPI file refuses;
- a missing stashed asset refuses;
- two concurrent resumes for one tag serialize through the concurrency group (a contract
  test);
- `confirm` never un-drafts before every check passes;
- `resume` refuses without passing evidence.

**Done when:** a full rehearsal at `master` is green through the verify-only `publish`.
Because `master`'s version equals the last release and its bytes differ, that plan must
report `PUBLISH_CONFLICT` for every file. This is the live proof of the different-hash
refusal; record it as expected. The `probe: break-macos` rehearsal shows `publish`
skipped or refused.

## steward

**Repo:** sase. **Files:**

- `tools/release_watch`, rewritten. Keep the CLI name, and replace the stateful watcher
  and its in-memory cooldowns.
- `tools/_release_watch*.py`, reorganized as fact reader, decider, actions and sinks.
- The release_watch tests, replaced or migrated: delete tests for removed behavior and
  keep the parser and fixture helpers that still apply.
- Per `steward_host`, either a new `.github/workflows/release-steward.yml` or the
  chezmoi wrapper.

**Facts, read every tick and never cached across ticks:**

- PyPI JSON for `sase` and `sase-core-rs`;
- `v*` tags and GitHub releases, including drafts, with their manifest and observation
  assets;
- the release-please PR, its head, checks, labels and pinned core;
- `master`'s HEAD and its Master Gate run;
- `release.yml` runs, keyed by tag and SHA, with attempts and jobs;
- `publish.yml` runs;
- sase-core's open release PR, `release-plz.yml` runs and their typed outcomes
  (`core-release`), and its `master` CI;
- the newest scheduled install-canary artifacts (E1's contract);
- the steward's own open escalation records.

**Typed states** (one per tick, with an owner action each):

- `RELEASED(v)` / `RELEASABLE` / `NOTHING_TO_RELEASE`.
- `CORE_LAG(pin)`: dispatch the core cut.
- `CORE_CI_RED` / `CORE_CUT_RED(check)`: the core cut reruns once (`core-release`); then
  escalate.
- `CORE_INCOMPATIBLE`: the release PR's pinned-core smoke is red; refresh the PR, else
  escalate.
- `GATE_RED(sha)`: rerun failed Master Gate test shards once per HEAD; E4 adds the
  revert later.
- `QUALIFY_RED(signature, recovered?)` / `EXERCISE_RED(platform)`: one rerun if
  `rerun_eligible`, else abandon.
- `DEPENDENCY_UNPUBLISHED(pkg)`: the pinned core is gone or incomplete; escalate and
  never publish.
- `PUBLISH_PARTIAL`: dispatch `release.yml -f mode=resume`.
- `PUBLISH_CONFLICT`: escalate at once; never retry.
- `INSTALL_BROKEN(member)`: severity 1, escalated at once. It **never** blocks
  publishing a separately qualified candidate (acceptance case 14).
- `VETOED`: a human `release-hold` label on the release PR, or a `veto` observation.
- `WATCHER_BROKEN` / unknown: fail closed, take no action, exit non-zero.

**Actions** (at most **one** train action per tick; each one idempotent, with its key
derived from facts):

- merge the release PR with `gh pr merge --merge --match-head-commit`, only when:
  - the merge authority is on;
  - no train is in flight (meaning a newer tag whose newest generation is neither
    confirmed nor abandoned);
  - Master Gate is green at HEAD;
  - every required PR check is green, with SKIPPED and missing treated as red;
  - the PR's pin equals core-ensure's pick;
  - there is no veto;
  - the cadence rule allows it;
- dispatch `publish.yml`, to make release-please tag and draft after a merge, or to
  refresh a stale PR or pin;
- dispatch `release.yml` at the tag (`release` or `resume`);
- rerun once;
- dispatch the core cut (`release-plz.yml -f dry_run=false`);
- mark a candidate abandoned: write the observation and prefix the draft's title. Never
  abandon a version that has any file on PyPI.

**Supporting rules:**

- **Escalation bookkeeping** (open, +1 or close) may also happen once per tick, keyed by
  state.
- **No cooldowns kept in memory.** Cooldowns and "already done" come from run and merge
  timestamps.
- **Concurrency:** the tick runs under a single concurrency group with no cancel, so a
  restart mid-train finds the earlier action's fact and does nothing new (acceptance
  case 10).

> [!decision] release_cadence = daily A merge is allowed only when no release PR was
> merged earlier the same UTC day, except when that day's candidate was abandoned. It is
> also allowed only after 09:00 UTC, which leaves the 07:41 core cut time to publish.

> [!decision] release_cadence = eligible A merge is allowed whenever the other merge
> conditions hold. The only rate limit is the one-train-in-flight rule.

**Escalation:** after 24 hours continuously in one blocked state, measured from the
oldest fact that has held it, escalate once and then +1 on later ticks. Close the record
when the state clears. `INSTALL_BROKEN`, `PUBLISH_CONFLICT` and `DEPENDENCY_UNPUBLISHED`
escalate at once. Advisory qualification failures escalate once per candidate as `ci`
work. Never launch repair agents.

> [!decision] steward_host = workflow
>
> - `.github/workflows/release-steward.yml`:
>   - triggers: `schedule` every 15 minutes, plus `workflow_dispatch` with a `live`
>     input;
>   - `concurrency: release-steward`, with no cancel;
>   - it checks out `master` and runs the steward with `SASE_RELEASE_TOKEN`, so that
>     merges trigger push workflows and the core cut can be dispatched cross-repo;
>   - verify the token's sase-core `actions` scope with a read-only call. If it is
>     missing, report `WATCHER_BROKEN` and list it in `handover`'s gate.
> - The escalation sink is one sase-org/sase issue per state family:
>   - a fixed title, plus the label `release-steward` (and `ci` for advisory items);
>   - comments serve as the +1s, and the issue closes on recovery;
>   - the issue mirror turns each issue into one task bead (type `github`).
> - The run's step summary carries the typed state.
> - This phase ships the workflow **observe-only**: the schedule runs `--tick` without
>   `--live`, so it prints its decision and takes no action.

> [!decision] steward_host = chop
>
> - The escalation sink is direct: `sase bead create` (task type `ci`, high severity,
>   via the `/sase_new_task` rules) or `+1` on the existing bead, plus one `sase notify`
>   thread per state family.
> - In chezmoi, add an axe routine `release_steward` (interval 900 s, `job_timeout` 300
>   s) next to `ci_watch`, and a wrapper script in a configured `axe.job_script_dirs`
>   directory:
>   - the wrapper keeps a dedicated clean worktree of `origin/master` (fetch, then
>     detach) and runs `tools/release_watch` from it under `uv run --no-project`;
>   - it honors `SASE_JOB_DRY_RUN`, and writes the structured result and the report.
> - This phase ships the routine with `dry_run` on.

**CLI:**

- `--once` prints the decision read-only;
- `--tick [--live]` decides and, when live, acts;
- `--json` emits a machine-readable state;
- `--until-released [VERSION] --deadline D` repeats read-only ticks, for `/sase_monitor`
  polls;
- exit 0 for every well-formed state; non-zero only for `WATCHER_BROKEN`, an argument
  error, or an `--until-released` timeout.

Drop `--close-bead`, `--fixed-sha` and the in-memory cooldowns, and remove their tests.

**Tests** (fixture fact sets, no network):

- every typed state and its single action;
- precedence when several states hold;
- two consecutive ticks over facts updated by the first tick's action take no second
  action (the restart case);
- SKIPPED or missing required checks block a merge;
- an incomplete core blocks the pin;
- `CORE_LAG` dispatches exactly one cut, and a cut in flight blocks another;
- escalation fires once at 24 hours, then +1, then closes;
- `INSTALL_BROKEN` does not block a qualified candidate;
- a version with any file on PyPI is never abandoned;
- both cadence branches.

**Done when:** live `--once` against real GitHub and PyPI prints a typed state with its
evidence. The observe-only loop has completed at least two scheduled ticks; record their
URLs. The repo checks per `lint_and_test` pass.

## metrics

**Repo:** sase. **Files:** a new `tools/release_metrics` plus modules and tests. It
reuses the steward's fact reader, read-only. Do not edit workflows or docs pages.

**Figures:** each one has its numerator, its denominator and its window (default 28
days), as JSON and as Markdown:

- days since the last confirmed release;
- landing to PyPI lead time, p50 and p90: from each commit's arrival on `master` to the
  first PyPI release containing it;
- time to release-green, p50 and p90, against the < 6 h target: from each `master` SHA
  to the first SHA at or after it that is release-green;
- core-lag hours: how long `sase-core-revision.txt` pointed at a commit that no complete
  published core contained;
- burned versions per week: abandoned drafts, or tags with no PyPI files;
- change-fail rate: confirmed releases followed within 7 days by a yank, an
  `INSTALL_BROKEN`, or a superseding release whose notes reference a fix to it.

**Release-green record:** a SHA is release-green when its Master Gate run is green and a
Full CI run for that exact SHA is green on every N1 blocking job.

- Release-green is derived from runs every time and never stored.
- Expose `release_metrics green --json` (per-SHA evidence) for E3.5's culprit search and
  E4.
- It never authorizes an ancestor, because green is not hereditary.

**Tests:** fixture histories with known answers for each figure, including empty
denominators, which print "n/a (0 of 0)", never 0%.

**Done when:** a live run prints every figure with its numerator and denominator over
the real history. Record that output as the pre-train **baseline** in a bead note.

## handover

**Repos:** sase and chezmoi. **Start by confirming the prerequisites:**

- `exact-build`'s gate is approved (`sase gate show <id>`, from its bead note);
- `gh api repos/sase-org/sase/environments/pypi/deployment-branch-policies` lists a `v*`
  tag rule.

If either is missing, hand the wait to `/sase_monitor`: poll both with about a 7-day
deadline, and let the follow-up continue. Land nothing that removes the legacy publisher
first.

**Work:**

1. **release-please:** set `draft: true` and `force-tag-creation: true` in
   `release-please-config.json`, or apply the fallback that `exact-build` recorded.
2. **`publish.yml`:** keep only `release` and `sync-release-metadata`. Delete `build`,
   both install smokes, `publish` and the `publish_existing` input. Rewrite
   `tests/test_github_actions_publish.py`, including the release_watch dispatch argv
   pins. Recovering an old version is now `release.yml -f mode=resume` at its tag.
3. **The steward goes live:**
   - actions and escalation are on;
   - merges are additionally gated by a merge-authority switch that starts **off**:
     - workflow branch: the repository variable `RELEASE_STEWARD_MERGE`;
     - chop branch: the routine var `merge_enabled`;
   - the steward reports `RELEASABLE (merge authority off)` until a human flips the
     switch;
   - add a once-daily metrics block to the steward's report.
4. **chezmoi ci_watch config** (`home/dot_config/sase/sase_athena.yml`):
   - remove `sase-org/sase` from `release_repositories`, `merge_order`, `merge_method`,
     `gating_workflows`, `heavy_workflows`, `heavy_max_age_hours` and
     `heavy_dispatch_every_hours`;
   - keep it in `repos`, so its CI health is still watched;
   - update the routine description;
   - this relaxes the Full CI freshness rule only now, after qualification and
     confirmation are deployed. Full CI keeps its own two-hour cron.
   - On the chop branch, also flip the steward routine's `dry_run` off.
5. **Docs:**
   - `release-train.md` gains "The steward" (the states, the owner actions, the merge
     authority and the cadence) and "Release metrics";
   - `docs/contribute/rust-backend.md` "Watching a release" points to the steward and
     drops the old verdict table;
   - `docs/contribute/release-safety.md`'s recovery uses `mode=resume` and never
     `publish_existing`;
   - fix stale references to `publish_existing` and to `ratchet_core_window` anywhere in
     `docs/`.
6. **A final rehearsal:** a `master` rehearsal of `release.yml` and a live `--once` both
   match expectations. Record the URLs.

**Gate (last action):** one `/sase_gate` listing these human steps in order:

1. apply chezmoi on athena with `chezmoi update -a --force`. ci_watch then stops merging
   sase release PRs;
2. turn the steward's merge authority on:
   `gh variable set RELEASE_STEWARD_MERGE --body true -R sase-org/sase` on the workflow
   branch, or set the chop var through chezmoi;
3. remove the `master` branch rule from the `pypi` environment, leaving only `v*` tags;
4. optionally, remove `publish.yml`'s trusted publisher on PyPI.

Steps 1 and 2 belong together. Until they are both done, ci_watch may still merge a sase
release PR. That is harmless: the steward picks up the resulting tag and drives
`release.yml`.

**Done when:** contract tests and the repo checks pass, the rehearsal is green, and the
gate is raised.

## exit

1. **Confirm the human steps.**
   - The handover gate is approved.
   - The `pypi` environment has `v*` tags only.
   - The merge switch is on, and the applied `~/.config/sase/sase_athena.yml` no longer
     lists sase as a release repo.
   - If anything is pending, monitor it with `/sase_monitor`, with about a 7-day
     deadline.
2. **The first steward release.** Hand the wait to `/sase_monitor`, with
   `tools/release_watch --until-released` and about a 4-day deadline. The poll exits 0
   once a version newer than the one current at handover is confirmed and un-drafted,
   and non-zero on `WATCHER_BROKEN`.
   - If a candidate is burned on the way, that is expected. Keep watching the next one.
   - On a typed blocked state past 24 hours, triage it through the owner action the
     state names.
3. **Prove the release.** For that version:
   - every PyPI file's SHA-256 equals the sealed manifest's;
   - PyPI's provenance (the Integrity API) names `refs/tags/v<version>`, a source digest
     equal to the tag SHA and the manifest SHA, and `release.yml` as the build config;
   - `master` had advanced past the release commit before the build (acceptance case 1);
   - the core digests equal PyPI's;
   - no human acted beyond the pre-authorized steps.
4. **The single decider (acceptance case 10).**
   - The ci_watch run ledger shows no sase release merge after the apply.
   - During that release's train, a manual `release-steward` dispatch, or a
     `--tick --live` on the chop branch, takes no second dispatch or upload.
5. **Refusals.** List every refusal the steward reported during the soak, each with its
   typed reason and owner.
6. **The decision record and the supersession marks,** per the memory decisions below.
   Write them with `/sase_memory_write`, then run `sase memory init`. Regenerate the
   design-decisions page with `just fix`.
7. **Evidence:** one bead note with:
   - the run URLs;
   - the manifest, attestation and digest comparison;
   - post-release `release_metrics` figures next to the `metrics` baseline;
   - the acceptance-case mapping below.

> [!decision] memory_release_record Add the strand `decisions:exact-release-pair`,
> titled "PyPI Publishes Exactly The Qualified Release Pair":
>
> - **Claim:**
>   - A sase release is a sealed candidate: the tag-built wheel and sdist plus an exact
>     `sase-core-rs==X.Y.Z` with its full file set.
>   - It is qualified against the published core, exercised on Linux and macOS,
>     published from retained bytes with digest-checked resume, and confirmed from PyPI
>     before un-drafting.
>   - One stateless steward decides release merges and the core cut, and publishing a
>     qualified candidate is pre-authorized. Yanks stay human.
>   - ci_watch no longer merges sase's release PR, and Full CI freshness no longer gates
>     a release.
> - **Why:** it was chosen over HEAD-plus-bump releases gated on fresh Full CI (the
>   50-day stall), over core windows (tested ≠ shipped, and consumer-blind retention),
>   and over qualifying inside the merge queue.
> - **Cost:**
>   - a failed candidate burns a version;
>   - every release needs a core release that contains its source pin;
>   - a core fix waits for the next sase release.
> - **Reopens when:**
>   - burned versions exceed about one a week;
>   - `CORE_INCOMPATIBLE` recurs;
>   - a plugin depends on `sase-core-rs` directly and conflicts with the pin;
>   - or fewer than 2 of 3 daily candidates qualify over two weeks.
>
> Mark two older strands `metadata.status: superseded-in-part` with `superseded_by`
> pointing to the new strand. Give each one `[[...]]` back-link stating what was
> retired, and leave their bodies otherwise untouched:
>
> - `decisions:ci-two-speed-split`: its release clause (the ci_watch release gate) is
>   retired;
> - `decisions:rust-core-required`: its core-window cost clause ("a pinned dependency
>   window (`sase-core-rs>=0.31.0,<0.32.0`) that must be ratcheted forward
>   deliberately") is now an exact pin ratcheted at release time.

## Acceptance mapping

The design's acceptance cases for E2:

| Case                  | Where it is proven                                                                       |
| --------------------- | ---------------------------------------------------------------------------------------- |
| 1 Wrong source        | `exact-build` identity tests; exit attestation and manifest SHA with `master` advanced   |
| 2 Incomplete evidence | `qualify` evaluator fixtures (skipped, cancelled, zero, missing); `exercise` missing leg |
| 3 Wrong public core   | `qualify` `probe_core_version` rehearsal → `QUALIFY_RED` naming the bindings             |
| 4 Incomplete core     | `core-pin` core-ensure fixtures; `exact-build` seal refusal                              |
| 5 Partial publication | `publish` resume fixtures; rehearsal verify-only `PUBLISH_CONFLICT`                      |
| 8 Real-use packaging  | `exercise` `probe: break-macos` withholds `publish`                                      |
| 10 Single decider     | `steward` restart fixtures; exit ledger and manual-dispatch check                        |
| 14 Public recovery    | `steward` fixture: `INSTALL_BROKEN` never blocks a qualified candidate                   |

**Epic exit:**

- at least one real release went through the steward with no human action beyond the
  pre-authorized ones;
- its PyPI digests and attestation SHA equal the manifest's;
- every refusal named a typed reason and its owner.

## Handoffs to E3, E4 and E6

- **E3.1** adopts its node-level rerun in `ci.yml`'s test steps. Qualification inherits
  it with no change. The steward's job-level rerun then fires rarely, and stays as a
  backstop. E3 then owns the `FLAKY:` format, which is copied verbatim here.
- **E3.5 and E4** consume `release_metrics green --json` and the steward's `GATE_RED`
  and `QUALIFY_RED` signatures. E4 adds the revert action behind `GATE_RED`.
- **E3.7**, the visual promotion study, may move `visual-test` into the blocking class
  only by editing the N1 inventory file.
- **E6** takes the release PR into the queue. It uses the steward's merge authority, and
  it holds packaging landings while `INSTALL_BROKEN`.

## Non-goals

- Node-level reruns, quarantine, drift canaries, culprit attribution and reverts (E3,
  E4).
- Moving the plugins' release merges off ci_watch. They stay protected by E1's floor
  check.
- Changing ci_watch code, the issue mirror's task type, or `install-smoke.yml`.
- A TestPyPI interrupt drill. There is no `testpypi` environment; the resume logic is
  proven on fixtures and the live verify-only plan.
- Automatic repair agents, yanks and PyPI deletions.
- Moving release promotion into `sase_core`. Release promotion is tooling.
