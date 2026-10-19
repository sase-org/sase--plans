---
tier: epic
title: 'E1: Installable family — retention by need, published floors, family canary'
goal: 'A clean `uv tool install` of sase, alone or with the family plugins, installs
  the newest family release on Linux and macOS, and that release works. Retention
  can never delete, and no family release can declare, a dependency that a supported
  release cannot resolve. A broken install is caught within a day and reaches an owner.

  '
decisions:
  floor_check_home:
    ask: Where should the release-dependency-floor check logic live?
    choices:
      shared: One script and reusable workflow in sase; each plugin calls it at @master
      vendored: Each plugin carries its own copy and tests; no cross-repo workflow
        coupling
    default: shared
    why: One implementation to fix, and plugin CI already checks out sase master
    answer: shared
  research_artifacts:
    ask: Should the family canary and floor check also cover sase-research-artifacts?
    default: true
    why: It is on PyPI with a sase floor, and its master adds a direct sase-core-rs
      floor
    answer: true
  yank_0_1_0:
    ask: How should E1 handle sase 0.1.0, the silent resolver fallback when every
      newer window breaks?
    choices:
      propose: The runbook phase raises a gate with the exact PyPI yank steps for
        you to perform
      document: The runbook records the trade-off only; 0.1.0 stays the silent fallback
    default: propose
    why: A broken window then fails loudly instead of installing February's build
    answer: propose
  testpypi_drill:
    ask: Should E1 run a TestPyPI yank-and-fix-forward drill (you create one project
      and publisher)?
    default: true
    why: It rehearses the human-only recovery steps before a real outage needs them
    answer: true
  decision_record:
    ask: Add the "a published release stays installable" record to the decisions web?
    memory:
    - decisions:installable-releases
    requested: Can you now help me implement the E1 epic described by the pypi_release_reliability_design_and_work_split.md
      file in the research sidecar repo?
    default: true
    answer: false
phases:
- id: retention
  title: Protected-set retention in sase-core, failing closed
  depends_on: []
  size: medium
  description: 'retention: derive a consumer-driven protected set in pypi_retention.py,
    add a delete-list validator, enforce it in pypi_delete.py before any login, and
    make the script tests run with packaging.'
- id: canary
  title: Family install canary on Linux and macOS
  depends_on: []
  size: medium
  description: 'canary: extend install-smoke.yml to install sase plus the plugins
    on ubuntu and macOS, assert newest and exact versions, run the first commands,
    publish a result artifact, and open or close one alert issue.'
- id: floor-check
  title: release-dependency-floor check on plugin release PRs
  depends_on: []
  size: medium
  description: 'floor-check: add a sibling-floor satisfiability and joint-resolve
    check, wire it into each plugin''s release PRs so it is never skipped, and rerun
    it on a schedule so it clears once the sibling publishes.'
- id: headroom
  title: Core storage runway line and headroom alert
  depends_on:
  - retention
  size: small
  description: 'headroom: record the measured runway and the open size request in
    docs/pypi-retention.md, add a runway threshold to verify, and alert weekly when
    it is breached.'
- id: runbook
  title: Recovery runbook, release-safety docs, and the TestPyPI drill
  depends_on:
  - canary
  - floor-check
  - headroom
  size: medium
  description: 'runbook: document yank and fix-forward, core backfill, and the sase-side
    safety nets, prepare the drill, and raise one gate for the human-only steps.'
- id: exit
  title: Live acceptance, decision record, and seven-day soak
  depends_on:
  - retention
  - canary
  - floor-check
  - headroom
  - runbook
  size: small
  description: 'exit: prove V-E1a to V-E1c live, run the drill if it is set up, write
    the decision record, and hold the epic open until the canary has been green for
    seven consecutive days.'
proposed_by: bbugyi200.athena.11s
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 10:16:24
status: wip
bead_id: sase-1my
---

- **PROMPT:** [prompts/202610/e1_installable_family.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/e1_installable_family.md)
- **BEAD:** [sase-1my](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1my/README.md)

# Plan: E1 Installable family

This is epic **E1** of the converged PyPI release reliability design. Every phase agent
should first read the design's "The installable promise", "E1 Installable family" and
"Acceptance cases" sections:

```bash
sase artifact read research:202610/pypi_release_reliability_design_and_work_split/pypi_release_reliability_design_and_work_split.md "<why>"
```

Track 0, which shipped sase 0.18.0, is finished: epic `sase-1mk` closed on 2026-10-19.
Its report is `research:202610/pypi_release_reliability_track0_remaining_work.md`. E2
(release train and steward) and E3 (trustworthy signal) are separate epics that may run
in parallel with this one. Stay out of the files they own (see
[Boundaries](#boundaries-shared-by-every-phase)).

## Where things stand (measured 2026-10-19)

- **PyPI `sase`:** 0.18.0 requires `sase-core-rs>=0.42.0,<0.43.0`. Of the 30 published
  releases, only **0.18.0 and 0.1.0 resolve today**.
  - Every release from 0.1.5 through 0.17.1 pins a core window that retention has
    already deleted.
  - 0.1.0 has no core dependency, so it is the resolver's silent fallback.
- **PyPI `sase-core-rs`:** 0.42.1 is the newest. There are 53 versions (oldest 0.34.23),
  every one complete with five files, totalling 4.09 GiB of the 10 GiB limit.
  - The 50 GiB size request
    [pypi/support#12339](https://github.com/pypi/support/issues/12339) has been open
    since 2026-09-19. It is already filed, so E1.4 files nothing new.
- **Plugins on PyPI:**
  - `sase-github` 0.2.24 requires `sase>=0.18.0`. It shipped while 0.18.0 was
    unpublished.
  - `sase-telegram` 0.4.29 requires `sase>=0.17.0`.
  - `sase-research-artifacts` 0.2.0 requires `sase>=0.17.0`. Its master already declares
    `sase>=0.18.0` and `sase-core-rs>=0.35.0`, the first direct plugin→core floor, which
    has no upper bound.
  - `sase-nvim` is not on PyPI.
- **Retention:** sase-core's `.github/scripts/pypi_retention.py plan` keeps the newest
  30 versions plus `--keep-version` entries. It is consumer-blind, and it has no tests.
  `pypi_delete.py` only refuses lists that reach a `--keep-version` floor.
- **Script tests:** sase-core's `scripts/check.sh script-test` runs a bare
  `python3 -m unittest discover -s .github/scripts`, and the CI `release-scripts` job
  installs nothing. **athena's system `python3` has no `packaging`**, which
  `pypi_retention.py` imports.
- **`install-smoke.yml`:** a daily job on `ubuntu-latest` only. It runs the
  single-sourced documented command (`tools/render_applicability --install-command`),
  checks the version against the `>=0.18` floor, and runs `sase core health` and
  `sase doctor`. It installs no plugins and makes no newest-version assertion.
- **`ci_watch`** (bugyi-chops `src/bugyi_chops/ci_watch.py`, config in chezmoi
  `sase_athena.yml`):
  - It merges release-please PRs for sase, sase-github and sase-telegram, in the order
    sase-github, sase-telegram, sase.
  - `plan_release_merge` refuses with `release_pr_checks_not_green` unless every rollup
    entry is `COMPLETED` with conclusion `SUCCESS`, `SKIPPED` or `NEUTRAL`. Two
    consequences:
    - **a SKIPPED check counts as green;**
    - commit-status contexts have no `status`/`conclusion` fields, so they read as never
      green.
  - So a red check run blocks a plugin release with **no ci_watch change**.
- **Owner path:** only `sase` is a SASE project. The `external_issue_mirror` AXE job
  mirrors sase-org/sase GitHub issues into task beads every 15 minutes, with task type
  `github`. Issues on sase-core or plugin repos never become beads.

## Boundaries shared by every phase

1. **Fail closed.** Unreadable, malformed, partial or missing evidence never yields
   "safe to delete", "canary green" or "floor satisfied".
2. **Human-only actions stay human.** That covers PyPI deletion, yanks, PyPI and
   TestPyPI project or publisher setup, and size requests. Propose them through
   `/sase_gate`, and never script around a missing credential.
   - Agents never create PRs, branches or commits (`host-owned-completion`). So live
     proofs use `workflow_dispatch` probes and local runs, never planted PRs.
3. **Detectors, not gates.** A red canary or headroom alert blocks nothing in E1. A
   recovery release must never be held by an install outage. The floor check is E1's
   only new merge blocker, and it applies only to plugin release PRs.
4. **Single sources stay single.**
   - The complete core file set is `EXPECTED_DIST_SUFFIXES` in sase-core's
     `release-plz.yml`; read it, don't copy it.
   - The project limit stays the existing pair that a unit test keeps equal.
   - The documented install command stays in `tools/render_applicability`.
5. **Do not touch E2/E3 files:**
   - sase: `publish.yml`, `ci.yml`, `build-core.yml`, `master-gate.yml`, `full.yml`,
     `tools/release_watch*`, `tools/ratchet_core_*`, `release-please-config.json`;
   - sase-core: the `release-plz.yml` jobs;
   - the chezmoi `ci_watch` config.
6. **Repos.**
   - Open every non-sase repo with `sase repo open <name>` and work only in the printed
     path. Each modified repo is a commit obligation of that phase's declaration.
   - Verify sase-core with `sase tool run check` in its checkout (see its `AGENTS.md`),
     and sase per the `lint_and_test` memory. Verify plugins with the checks their own
     `AGENTS.md`/`CLAUDE.md` name.
   - Read the `docs_writing` memory before touching sase `docs/`.

## retention

**Repo:** sase-core.

**Files:**

- `.github/scripts/pypi_retention.py` and `pypi_delete.py`;
- a new `.github/scripts/test_pypi_retention.py`, plus delete-guard tests;
- `scripts/check.sh` (`cmd_script_test` only);
- `.github/workflows/ci.yml`, the `release-scripts` job only;
- `docs/pypi-retention.md` (the "Tools" and "Procedure" sections).

**Protected set.** It is a union of named sources. Every entry records _why_ it is
protected, for example "sase 0.18.0 requires sase-core-rs<0.43.0,>=0.42.0":

1. **Supported consumer releases.** Use an explicit consumer list: `sase`,
   `sase-github`, `sase-telegram`, `sase-research-artifacts`. For each consumer, take
   the newest `SUPPORT_HORIZON = 10` non-yanked releases. A CLI flag may raise the
   horizon but never lower it.
   - Read each release's own `requires_dist` from the per-version JSON
     (`/pypi/<pkg>/<version>/json`). The project-level `info.requires_dist` describes
     only the newest release.
2. **Window rule.** For every requirement on `sase-core-rs`, whatever its extras or
   markers (be conservative):
   - for a bounded specifier (an upper bound, or `==`), protect every published version
     that satisfies it;
   - for a specifier unbounded above, protect the newest _complete_ version that
     satisfies it.

   E2.2's `==X.Y.Z` pins then fall out as one-version windows, with no special case.

3. **Confirmed-working baseline.** Protect the windows of the newest two sase releases,
   of any age, whose window currently holds a complete core version. They are the
   rollback and recovery baseline even when they fall outside the horizon.
4. **Unchanged sources:** the newest 30 (`DEFAULT_KEEP`) and `--keep-version`.
5. **Seam for E2.** Leave a documented seam where E2 adds "every version a candidate
   manifest says was qualified". Do not build that source here.

**Complete file sets.**

- A version is _complete_ when it holds every `EXPECTED_DIST_SUFFIXES` file and none of
  them is yanked.
- Read the set from `release-plz.yml` and reuse `pypi_release_files.py`'s parsing.
- Protecting a version protects every one of its files.
- Warn about any supported release whose window holds no complete version.

**Already-broken releases.** Releases 0.1.5 through 0.17.1 already have empty windows.

- Print them in a loud `ALREADY UNRESOLVABLE` block that points to the core-backfill
  runbook.
- Do **not** refuse a plan because of them. Nothing of theirs is left to delete, and a
  refusal would block every reclaim until they age out.

**Fail closed.** Exit non-zero and write no list when:

- the core JSON, or any listed consumer's JSON, cannot be read, returns a non-200
  (including a 404 for a listed consumer), or is malformed;
- any per-version fetch fails (partial enumeration);
- a core requirement cannot be parsed;
- the expected-file set cannot be read.

**`check --list FILE`.** A new subcommand validates any delete list against the live
protected set (or a fixture).

- An entry is a version or a single distribution filename.
- On a violation it exits 1 and names the consumer release, the requirement, and the
  version or file the list would break.
- `plan` runs the same check on its own output before writing it.
- Extend `--input-json` (or add a fixture-bundle flag) so that tests and offline dry
  runs can supply the core, consumer and per-version JSON.

**`pypi_delete.py`.**

- Before reading any credential or logging in, compute the protected set live. Refuse
  with exit 2 and a new `error_kind`, such as `protected`, when the list touches the set
  or the set cannot be computed.
- Import the retention module from its sibling path rather than forking its logic. Keep
  the existing `--keep-version` guard.

**Script-test runner.** Make the script tests run with `packaging` available, both in
`just check` on athena and in the CI `release-scripts` job. For example, use
`uv run --no-project --with packaging --with requests python -m unittest discover ...`
plus `astral-sh/setup-uv` in that job. Never use a skip-if-missing guard. The existing
stdlib tests must keep passing offline.

**Tests** (fixtures, no network):

- a planted list containing `0.42.0` is refused;
- a planted list naming only `0.42.0`'s `win_amd64` wheel is refused;
- a URLError, a 5xx, malformed JSON, or a 404 for a listed consumer each fails closed;
- a failed per-version fetch fails closed;
- an already-broken release is reported, not refused;
- an `==` window, and an unbounded window, are handled correctly;
- a horizon below 10 is refused;
- `pypi_delete.py` refuses a protected list before asking for credentials.

**Docs.**

- Replace Procedure step 1's manual consumer audit with the automatic protected set and
  `check`.
- Add `check` to the Tools table.
- State that the driver refuses protected lists.

**Done when:**

- against live PyPI (read-only), `plan --keep 30` shows 0.42.0 and 0.42.1 protected "by
  sase 0.18.0", each with its full five-file set, and keeps both out of the delete list;
- both planted `check` runs refuse;
- forcing PyPI to be unreachable makes `plan` fail closed;
- `sase tool run check` passes in sase-core.

## canary

**Repo:** sase.

**Files:**

- `.github/workflows/install-smoke.yml` (keep the file and its name; the schedule and
  `workflow_dispatch` stay);
- a new helper, for example `tools/check_family_install`;
- workflow contract tests, for example `tests/test_github_actions_install_smoke.py`.
  Keep `tests/test_render_applicability.py` green.

**Prose.** Limit prose to the workflow header and the tool docstrings. The runbook phase
owns the docs.

**Job shape.** Use a matrix `os: [ubuntu-latest, macos-latest]` with `fail-fast: false`.
Each leg uses an isolated `UV_TOOL_DIR`, `UV_TOOL_BIN_DIR` and `HOME`, with uv's cache
off (`UV_NO_CACHE=1`, refresh). Each leg runs:

1. **The documented command and floor check, as today.**
2. **The documented family command:**
   `uv tool install "sase>=0.18" --with sase-github --with sase-telegram`, which is the
   `INSTALL.md` form.
   - Single-source it next to `INSTALL_COMMAND` if practical, and keep
     `render_applicability --check` green.
3. **The newest assertion.** Read installed versions from `sase version -j`, whose
   `packages` list already includes plugins.
   - sase and each plugin must equal the newest non-yanked, non-pre-release PyPI
     version.
   - `sase-core-rs` must equal the newest version that satisfies **the installed sase's
     own requirement**. A core minor beyond sase's window is normal between releases and
     must not read as red.
   - On a mismatch, re-fetch PyPI once, to absorb CDN lag or a release that lands
     mid-run.
4. **Exact current release.** In a fresh tool directory, install `sase==<newest>` plus
   each plugin `==<newest>`, then run the same checks.
5. **Health and the documented first commands:**
   - `sase core health --json` and `sase version -j`;
   - `sase doctor` in an isolated home, keeping the existing step that accepts the known
     "no usable default LLM provider" CI diagnostic;
   - `sase completion install` inside the isolated `HOME`;
   - proof that each plugin's entry points loaded.

**Result contract (for E2.6's `INSTALL_BROKEN`).** Each leg uploads an
`install-canary-<os>.json` artifact. It records:

- `schema_version`, os, run ID and attempt, and SHA;
- `verdict`: `green` or `red`;
- per package: name, expected version, installed version, and the rule applied
  (`newest`, `window` or `exact`);
- the failing step and reason.

Document this contract in the workflow header.

**Alert.** A final job (`needs` both legs, `if: always()`) holds the workflow's only
`issues: write` permission.

- On a red scheduled or normal run, it opens one sase-org/sase issue with a fixed title
  and an `install-canary` label (create the label if it is missing), or it comments on
  the issue that is already open.
- A green run closes that issue with the run link.
- A missing or unreadable result artifact counts as red.
- The issue mirror turns the issue into one task bead (type `github`). Accept that type;
  don't change the mirror.

**Probe.** Add a `workflow_dispatch` input that forces an impossible expectation, to
prove the red path. Probe runs never open or close the issue unless a separate `alert`
input asks for it.

**Helper.**

- Use the stdlib plus `packaging`, run in the workflow through
  `uv run --no-project --with packaging`. Reuse `tools/_sase_install_pypi.py`'s fetch if
  it fits.
- Unit-test the pure functions:
  - sase 0.1.0 installed while 0.18.0 is newest → red, naming
    `sase 0.1.0 (expected 0.18.0)`;
  - core 0.42.1 under a `<0.43.0` window while 0.43.0 exists → green;
  - a yanked newest version is ignored;
  - a backtracked plugin → red.

> [!decision] research_artifacts The family command and the assertions add
> `--with sase-research-artifacts`.

> [!decision] research_artifacts = no The canary covers sase, sase-github and
> sase-telegram only.

**Done when:**

- the unit and contract tests pass. The contract tests cover: both OSes,
  `fail-fast: false`, the cache off, the scoped `issues: write`, the probe input, the
  artifact upload, and the documented command still read from `render_applicability`;
- a local run of the helper on athena, against a real isolated family install, is green,
  and a local probe run is red.

The first live run happens in **exit**, after this phase lands.

## floor-check

**Predicate.** On a family release PR, check every runtime dependency on a family
sibling (normalized name `sase` or `sase-*`; no list to maintain):

- at least one published, non-yanked, non-pre-release version must satisfy the
  specifier;
- the PR's full runtime dependency list must resolve jointly from public PyPI
  (`uv pip compile`, Python 3.12, no cache).

A red check names the sibling, the specifier and the newest published version, for
example `DEPENDENCY_UNPUBLISHED sase>=0.19.0: newest published is 0.18.0`. Unreachable
PyPI is red (fail closed); the scheduled rerun recovers it. Optional and dev
dependencies, and non-family names, are ignored by the per-sibling rule.

**Wiring.**

- **Runs on every PR to `master`, enforces only on release PRs.** The job is never
  SKIPPED, because ci_watch counts SKIPPED as green. It enforces only when the head ref
  starts with `release-please--`; elsewhere it passes with a notice.
- **Each plugin** (sase-github and sase-telegram, plus per `research_artifacts`) gets a
  caller workflow `.github/workflows/release-dependency-floor.yml`. Its triggers:
  - `pull_request` on `master`;
  - an hourly `schedule`;
  - `workflow_dispatch` with a probe input, for example an extra requirement such as
    `sase>=99`.
- **Each plugin also gets a contract test** in its own test suite. It asserts the
  triggers, the enforcement condition and the job's target.
- **Liveness.** A red floor check must turn green after the sibling publishes, without a
  new push.
  - On `schedule`, find the open release-please PR and the latest floor-check run for
    its current head SHA. If that run failed, run `gh run rerun --failed` on it.
    `actions: write` goes on that job only.
  - Bound the reruns: confirm GitHub's per-run attempt cap, then back off (for example
    hourly for the first day, then every few hours).
  - Do not use commit statuses. ci_watch would read them as never green.
- **sase's own release PR** keeps `release-core-floor-smoke`, which E2 owns. Do not add
  this check to sase.

> [!decision] floor_check_home = shared
>
> - The logic lives in sase: `tools/check_release_dependency_floor`, with
>   `tests/test_release_dependency_floor.py`.
> - It runs through a reusable workflow,
>   `.github/workflows/release-dependency-floor.yml` (`on: workflow_call`). The workflow
>   checks out the caller repo at the PR head, and the script at the called workflow's
>   own commit (for example `job.workflow_sha`), so the script and the workflow never
>   skew.
> - Each plugin's caller workflow is a thin
>   `uses: sase-org/sase/.github/workflows/release-dependency-floor.yml@master`. All
>   four repos are public.

> [!decision] floor_check_home = vendored
>
> - Each plugin gets its own `tools/check_release_dependency_floor` and tests, plus a
>   self-contained workflow.
> - Keep the copies identical, and say in each header that they are copies to change
>   together.

> [!decision] research_artifacts = no Wire only sase-github and sase-telegram.

**Tests** (fixtures, no network):

- an unpublished floor → red, with the reason;
- a satisfiable floor → green;
- only pre-release or only yanked matches → red;
- a joint conflict → red;
- a PyPI error → red;
- non-family deps are ignored;
- a non-release head ref → pass with a notice.

**Done when:** all tests pass in every touched repo; a local run against each plugin's
current `pyproject.toml` is green, and one with a planted `sase>=99` is red. The
workflow header cites ci_watch's `plan_release_merge` refusal semantics. The live probe
happens in **exit**.

## headroom

**Repo:** sase-core.

**Files:** `pypi_retention.py` (`verify`), a new `.github/workflows/pypi-headroom.yml`,
`docs/pypi-retention.md`, and tests.

- **Runway line.** Add a dated line to `docs/pypi-retention.md`, taken from `verify`'s
  output: the measured use (4.09 GiB of 10 GiB, 53 releases, about 80 MB each) and the
  runway at one release a day. It also records the open size request and what to change
  if PyPI grants it: both limit constants, which the existing test keeps equal.
- **Threshold.** Add `verify --min-runway-days D` (default 30), with the cadence
  defaulting to one release a day. It exits non-zero when the projected runway falls
  below D.
- **Alert.** A weekly scheduled workflow runs that check.
  - On a breach it opens or updates one sase-org/sase issue, since only sase issues are
    mirrored into beads. Use `SASE_RELEASE_TOKEN`, which exists in sase-core, if it has
    issue scope there.
  - Otherwise the red run itself is the alert, because ci_watch watches sase-core.
    Record which path is live.
  - It never deletes anything, and never proposes a deletion.

**Done when:** the unit tests cover the threshold math; a local run with a forced high
threshold exits non-zero and a normal run exits zero; `sase tool run check` passes.

## runbook

**Repos:**

- **sase-core:** `docs/pypi-retention.md` gets a new "Core backfill" section.
- **sase:** a release-safety and recovery doc. Choose the placement and nav per the
  `docs_writing` memory, for example beside the release sections of
  `docs/contribute/rust-backend.md`.
- **Drill tooling,** per `testpypi_drill`.

**Content:**

- **Yank, then fix forward.** PyPI's web UI is the only way to yank. Exact pins still
  install yanked files. Never move a tag or reuse a version.
- **Core backfill.** Give the exact steps to cut a new patch from an old core tag (for
  example 0.32.17 from `v0.32.16`) through `release-plz.yml`'s guarded manual route.
  - Re-verify them against the current workflow: the `manual-version` label,
    `expected_version`, the quota cost, and the side effect that `dry_run=false` also
    merges an open release PR.
  - Backfill is human-gated; this phase documents it and does not perform it.
- **Recovery is never blocked.** An install outage holds unrelated packaging churn,
  never a separately qualified recovery release.
- **sase-side safety nets:**
  - what a red canary means, its artifact contract, and the alert issue;
  - what the floor check blocks, and how it clears;
  - the retention protected set, linking the sase-core doc.
- **Related bead.** `sase-1m4` (blog posts that still say a bare `uv tool install sase`)
  relates to the 0.1.0 fallback. Cross-reference it; don't fix it here.

> [!decision] yank_0_1_0 = propose The doc records why yanking 0.1.0 makes a broken
> window fail loudly, and how to undo it. The phase's gate carries the exact PyPI UI
> steps for yanking `sase` 0.1.0, which you perform.

> [!decision] yank_0_1_0 = document The doc records the trade-off and leaves 0.1.0
> published, with no gate step for it.

> [!decision] testpypi_drill **Drill setup:**
>
> - A tiny throwaway package is published to TestPyPI by a `workflow_dispatch` workflow,
>   through a `testpypi` environment's trusted publisher.
> - A verification tool asserts each step of the drill.
>
> **Drill steps:**
>
> 1. publish a good 0.0.1;
> 2. publish 0.0.2 with a broken dependency;
> 3. you yank 0.0.2 in the TestPyPI UI;
> 4. verify that resolution skips 0.0.2 and that `==0.0.2` still resolves;
> 5. publish 0.0.3 as the fix-forward;
> 6. verify that 0.0.3 wins.
>
> The gate lists creating the TestPyPI project and its pending publisher.

> [!decision] testpypi_drill = no No drill workflow is written. The runbook is checked
> by review only.

**The gate.** Raise one `/sase_gate` listing every human step the decisions select.
Raise it as the phase's **last** action, because creating a gate ends the turn. If no
decision selects a human step, raise no gate.

## exit

1. **V-E1a (retention), live and read-only:**
   - `plan` protects 0.18.0's core and its full file set;
   - both planted `check` lists refuse;
   - unreachable PyPI fails closed.
2. **V-E1b (canary):** dispatch `install-smoke.yml` on `master`.
   - Both OS legs must be green.
   - A probe dispatch must be red with the expected reason, with the alert off.
   - Confirm that the next scheduled run is green and that its artifacts parse.
3. **V-E1c (floor check):**
   - In one plugin, a probe dispatch with `sase>=99` is red with a named reason, and a
     normal dispatch is green.
   - Record that a FAILURE conclusion maps to ci_watch's `release_pr_checks_not_green`.
4. **The drill,** if the TestPyPI setup is done. If it isn't, note it as pending on the
   phase bead and leave the gate open.
5. **The decision record,** per `decision_record`, written through `/sase_memory_write`
   and followed by `sase memory init`.
6. **The soak.**
   - **What counts:** seven consecutive UTC days of _scheduled_ `master` Install smoke
     runs with both OS legs green. Count from the first scheduled run that uses the new
     matrix. Probes and dispatches don't count.
   - **If the seven days aren't in yet:** hand off to `/sase_monitor` with a deadline of
     about 10 days. The poll command exits 0 once the condition holds, and non-zero on
     the first red scheduled run.
   - **On a red run,** the follow-up triages it:
     - a canary defect: fix it and restart the count once;
     - a real outage: follow the runbook and record it through `/sase_new_task`.
7. **The evidence:** a bead note listing the run URLs, the plan outputs and the soak
   window.

> [!decision] decision_record Add `decisions:installable-releases`, titled "A Published
> Release Stays Installable":
>
> - **Claim:** retention never deletes, and no family release declares, a dependency
>   that a supported release cannot resolve. The installable promise has its own fence
>   (the protected set and published floors), its own detector (the family canary) and
>   its own owner path (the mirrored alert issue).
> - **Why:** it was chosen over consumer-blind newest-N retention, over widening core
>   windows, and over relying on manual consumer audits.
> - **Cost:** about 80–100 MB of quota per protected release.
> - **Reopens when:** quota pressure would force deleting a protected version, or E2's
>   exact pins make the window rule obsolete.

## Handoffs to E2 and E3

- **The canary artifact contract** is the input to E2.6's `INSTALL_BROKEN`, at severity
  1.
- **The protected-set seam** is where E2's candidate manifests plug in. `==` pins
  already work.
- **The floor check** is the plugin instance of "published floors". sase keeps
  `release-core-floor-smoke`, which E2.2 collapses.
- **E3's bounded-rerun helper** may later wrap the canary. E1 adds only the single PyPI
  re-fetch.

## Acceptance

These are the design's acceptance cases 6, 7 and 14.

- **Case 6, retention regression:** a plan that deletes the only compatible core, or one
  required platform wheel, is refused. Unavailable PyPI also refuses.
- **Case 7, unpublished sibling floor:** a plugin release PR that names an unreleased
  sase stays red until the sibling publishes. It then turns green without a new push.
- **Case 14, public recovery:** nothing E1 adds blocks a recovery release while the
  canary is red.

**Epic exit:**

- E1.1–E1.3 are verified live;
- the runbook is published;
- the canary has been green on both OSes for 7 consecutive days.

## Non-goals

- Deleting or yanking anything on PyPI by agent.
- Backfilling the broken 0.1.5–0.17.1 windows. The backfill is documented, not
  performed.
- Exact core pins, `release.yml`, the steward, and changes to ci_watch code or config,
  none of which E1 needs.
- A Windows canary.
- Changing the issue mirror's task type.
