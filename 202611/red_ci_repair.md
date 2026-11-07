---
tier: epic
title: Repair red GitHub Actions CI across five projects
goal: "Every project whose GitHub Actions CI went red in the ~12 hours before 2026-11-07
  07:30 EST (sase-core, sase, bob-cli, bob-mac-capture, dotfiles) is green at its
  default-branch tip, with machine-readable green evidence attached to the phase's final
  turn, or, after a very large and well-documented effort, carries a research-sidecar
  report that tells Bryan exactly what is still needed to finish.

  "
decisions:
  test_quarantine:
    ask:
      May a phase skip a failing test or relax a CI gate when that is the only way to
      reach green?
    choices:
      root_cause_only:
        Real fixes only; anything unfixable goes into the report and stays red
      quarantine_flakes:
        Proven flakes may be skipped, each with a flake bead; gates stay as they are
      relax_gates:
        Also allows raising timeouts or dropping a job, each with a tracking task bead
    default: quarantine_flakes
    why:
      Runner noise shouldn't block green, but deterministic failures must not be hidden
    answer: quarantine_flakes
  release_merge:
    ask: When downstream CI needs a release PR merged and published, who merges it?
    choices:
      gate:
        The phase proposes the merge through /sase_gate and you approve before it
        publishes
      agent:
        The phase merges the release PR itself once its CI is green, publishing the
        release
      human:
        The phase stops at a green release PR and lists the merge as your manual step
    default: gate
    why:
      Publishing can't be undone; a gate keeps you in the loop without a report
      round-trip
    answer: gate
phases:
  - id: sase-core
    title: Turn sase-core CI green and unblock its release
    depends_on: []
    size: xlarge
    description:
      "sase-core: fix the macOS-only agent_artifact_run_trash test failure that keeps
      master CI and the v0.44.0 release-plz PR red, then get a sase-core-rs release
      containing sase's core pin published under the release_merge decision."
  - id: sase
    title: Turn every red sase workflow green
    depends_on:
      - sase-core
    size: xlarge
    description:
      "sase: keep Master Gate green and repair Full CI, CI Telemetry, Publish
      (CORE_LAG), and the release-please PR's release-core-floor-smoke, starting from
      the in-flight d2aa3ef Full CI result."
  - id: bob-cli
    title: Turn bob-cli CI green
    depends_on: []
    size: xlarge
    description:
      "bob-cli: clear the clippy lint failure around the oversized CaptureError result
      type and the macOS test job that is cancelled at its ~45 minute limit on every
      master push."
  - id: bob-mac-capture
    title: Turn bob-mac-capture CI green
    depends_on: []
    size: xlarge
    description:
      "bob-mac-capture: find and fix the hang that trips the 15 minute swift test
      watchdog in the macOS 26 SwiftPM job on master and on the ci/precommit snapshot
      branches."
  - id: dotfiles
    title: Turn dotfiles CI green
    depends_on: []
    size: xlarge
    description:
      'dotfiles: remove the ripgrep dependency (or provision it) behind the "rg: command
      not found" failure in tests/bash/doctl_login_test.sh that has kept the test job
      red since 2026-11-05.'
proposed_by: bbugyi200.athena.19h
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-07 07:46:40
status: wip
---

- **PROMPT:**
  [prompts/202611/red_ci_repair.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/red_ci_repair.md)

# Plan: Repair red GitHub Actions CI across five projects

## Source Of This Plan

The planner searched Bryan's Gmail for GitHub Actions failure notifications
(`from:notifications@github.com`) received in the ~12 hours before 2026-11-07 07:30 EST.
It found 24 failure emails spread across five repositories. It then used `gh run list` /
`gh run view` to confirm that each repository was **still red at its default-branch
tip** when planning finished, and pulled failing-job log excerpts. The repositories:

| Project         | GitHub repo                | Failure emails in window               | Red at tip when planned                               |
| --------------- | -------------------------- | -------------------------------------- | ----------------------------------------------------- |
| sase-core       | `sase-org/sase-core`       | 1 (release PR); master red since 11-06 | `CI` on master `c8adc87` and release-plz PR `d8c2bfc` |
| sase            | `sase-org/sase`            | 11                                     | Full CI, CI Telemetry, Publish, release-please PR CI  |
| bob-cli         | `bobs-org/bob-cli`         | 4                                      | `CI` on master `dbd30ec`                              |
| bob-mac-capture | `bobs-org/bob-mac-capture` | 3                                      | `CI` on master `1e0a092`                              |
| dotfiles        | `bbugyi200/dotfiles`       | 5                                      | `CI` on master `e931695`                              |

The per-phase sections below record what the planner saw. Treat these as **leads, not
conclusions**: re-read the current runs first, because other agents keep landing commits
on these repositories.

## Contract Shared By Every Phase

### Scope

A phase owns its project's failures listed below, plus anything its own changes break.
If an unrelated commit by someone else turns the tip red for a _new_ reason while you
work, fix it when the fix is quick. Otherwise record it in your evidence and file a `ci`
task bead through `/sase_new_task`. Don't chase other people's regressions indefinitely.

"Green" means: the latest run of every in-scope workflow listed in your section is
`completed` / `success` on a head SHA at or after your last fix commit, with no in-scope
job `cancelled` or `skipped` because something failed upstream. A first-attempt green is
the goal. If a run went green only on a rerun attempt, say so in the evidence. Under the
`test_quarantine` decision, that also means filing a `flake` bead for each test
involved.

### Working Rules

- Open the project's repository with `/sase_repo` and work only in the path it prints.
  Never edit another checkout, and never web-fetch repository files.
- Read the project's own `AGENTS.md` / `CLAUDE.md` and follow its lint and test recipes
  before you land anything.
- Prefer root-cause fixes. The `test_quarantine` decision bounds what else is allowed:
  - `root_cause_only`: never skip a test or weaken a gate.
  - `quarantine_flakes`: a test proven flaky on an unchanged tree may be skipped, with a
    `flake` task bead per test. Timeouts, job lists, and lint levels stay as they are.
  - `relax_gates`: you may also raise a timeout or drop or soften a job, with a task
    bead per relaxation explaining how to restore it.
- Never change repository variables, secrets, branch protection, or workflow
  permissions. For example, sase's `RELEASE_STEWARD_MERGE_ENABLED` is human-owned. Never
  force-push or rewrite history.
- The runner and athena are Linux. For macOS-only failures, reproduce on the `mac`
  tailnet machine: read the `tailnet` reference memory with `/sase_memory_read` first.
  Or use the repository's own existing macOS pre-commit verification flow when it has
  one.

### Landing And Verifying (Single-Turn Agents)

Agents never commit or push. Host completion lands your work after your turn ends, so
you can only see CI on your fix in a **later** turn. The loop:

1. Make the fix and run the project's local checks.
2. Submit an explicit commit decision for the target repository through `/sase_final`.
3. Before ending the turn, start a zero-weight wait with `/sase_monitor`: `-w 0`, labels
   `WATCHING CI` → `WATCHED CI`, polling `gh run list` / `gh run watch` for the new tip
   SHA. Give it a `--next` that tells the follow-up turn to inspect the result and
   either iterate or finish.
4. Some in-scope workflows only run on a schedule (sase's Full CI, CI Telemetry,
   Publish). Trigger those with `gh workflow run` when they accept `workflow_dispatch`
   rather than waiting for the schedule. First read each workflow file to confirm a
   dispatch cannot publish a release beyond what the `release_merge` decision allows.

> [!decision] release_merge = gate When a phase needs a release PR merged so downstream
> CI can go green, it waits for that PR's CI to pass. It then proposes the exact
> `gh pr merge …` command through `/sase_gate` and continues from the gate's follow-up.

> [!decision] release_merge = agent When a phase needs a release PR merged so downstream
> CI can go green, it merges that PR itself once the PR's CI is green. It then watches
> the release workflow until the package is published.

> [!decision] release_merge = human The phase stops when the release PR's CI is green.
> It lists the merge as Bryan's manual step in its final response and evidence, and
> doesn't merge.

### Evidence Required Before A Phase Is Done

The phase's final turn must attach evidence before declaring the work done. That is the
turn that confirms green, usually a monitor follow-up.

1. Publish a `ci_evidence` variable through `/sase_var` (JSON) with: `repo`,
   `default_branch`, `tip_sha`, `runs` (for each in-scope workflow: `workflow`,
   `run_id`, `url`, `head_sha`, `event`, `attempt`, `conclusion`, `completed_at`),
   `fixes` (landed commit SHAs with one-line summaries), `quarantined` (tests or gates
   and their bead ids, or empty), and `notes`.
2. Repeat the same data as a short Markdown table in the final response. Include the
   `gh run list` command that reproduces it.

### Fallback: Research Report When CI Cannot Be Fixed

Use this only after a **very large** effort: several distinct, evidence-backed
hypotheses, each tried and checked against real CI runs. Then you may stop and write a
report instead of evidence.

- Store it in the **sase project's `research` sidecar** (`sase-org/sase--research`), so
  every report lands in one place even for non-sase projects. Open the sidecar with
  `/sase_repo`, follow its existing directory and naming conventions, and land the
  report through host completion like any other change.
- Contents: the failing workflows and jobs with run URLs; every attempt (hypothesis,
  change, CI result, run link); why the attempts failed; why you believe further agent
  effort has no realistic chance; and a numbered list of exactly what Bryan must do to
  finish (commands, settings, credentials, hardware, or decisions). Also state what you
  landed and what you deliberately left out.
- Still land any partial fixes that are strict improvements. Never leave the tip redder
  than you found it.
- Publish `ci_report` through `/sase_var` with the report path and its
  `@research:<path>` reference, and cite that reference in the final response.

A phase is complete when it ends with either green evidence or a report. Dependent
phases start either way.

## Phase: sase-core

Repository: the linked `sase-core` repo (`sase repo open sase-core`).

Observed when planned:

- `CI` on master at `c8adc87` (run `42294092992`, first attempt) fails only in
  `cargo fmt + clippy + test (macos-latest)`, with `5901 passed; 1 failed`. The failing
  test is
  `agent_artifact_run_trash::tests::quarantine_moves_to_mirrored_layout_and_restore_is_byte_identical`,
  which panics at `crates/sase_core/src/agent_artifact_run_trash/tests.rs:276`. The
  ubuntu, maturin, and release-script jobs pass.
- `CI` on the release-plz PR `chore: release v0.44.0` (branch
  `release-plz-2026-10-31T14-34-59Z`, head `d8c2bfc`, run `42346248489`) fails in the
  same macOS job. This PR's CI has failed repeatedly since 2026-11-06.
- Downstream impact: sase's `Publish` and release-PR CI fail with
  `CORE_LAG(pin=c8adc878…, newest_complete=0.43.3)`. No published `sase-core-rs` release
  contains sase's pinned core SHA, because this release is blocked.

Leads: the failure is macOS-only, so suspect path canonicalization (macOS temp dirs
resolve `/var` → `/private/var`), case-insensitive APFS, or metadata (mode, mtime,
xattrs) leaking into the byte-identical restore check. Recent agent-hijack work touched
this area, so diff the module's recent history first. Keep the fix in the Rust core. Per
the Rust Core Backend Boundary note, update sase's `sase-core-revision.txt` pin only if
sase must consume a new binding.

In scope for green: `CI` on master at the tip, and `CI` on the open release-plz PR at
its head. Then, under `release_merge`, a published `sase-core-rs` release that contains
sase's current core pin. Note in the evidence which release version unblocks the `sase`
phase.

## Phase: sase

Repository: the sase primary repo (this project). Read the `lint_and_test` reference
memory before landing, the `tui` memory before touching visual snapshots, and the
`ci-two-speed-split` and `check-full-is-explicit` decision strands. This phase exists to
repair CI failures, so `just check-full` is explicitly authorized here. Run it through
`sase tool run` / `/sase_monitor` as those skills describe, never inline.

Observed when planned (latest run of each workflow):

- `Master Gate` (push) was red on six consecutive master pushes, from `e764cd9` through
  `8a9a993` (lint plus test-shard failures). It went **green at `d2aa3ef`**. Keep it
  green.
- `Full CI` (scheduled) was red at `8dce35a` (run `42357202695`) and `e764cd9`:
  - `visual-test`: 110 first-pass failures exceeded the 25-node recapture cap. Examples
    include PNG pixel diffs like `agents_archive_shelf_plain_160x40` and a `wait_for()`
    timeout in `test_ace_png_snapshots_agents_metadata_search`.
  - `test (3.12/3.13/3.14)`: failures include
    `tests/test_pytest_isolation_guards.py::test_session_sandbox_seal_is_not_first_test`,
    `::test_no_undo_on_shared_monkeypatch_fixture`, and
    `tests/ace/tui/test_proc_producer_inventory.py::test_inventory_matches_live_production_source`.
  - A dispatched `Full CI` on `d2aa3ef` ("fix(ci): repair Full CI first-attempt reds at
    master tip", run `42370668331`) was still running at planning time. **Start by
    reading its result.**
- `CI Telemetry` (scheduled) was red at `8dce35a` (run `42347892210`):
  - `coverage-contexts` hit a pytest-xdist `INTERNALERROR` after
    `AssertionError: in-flight ,X must not touch disk or the proc store` in
    `tests/ace/tui/test_kill_and_edit_last_launch_dispatch.py::test_inflight_handler_does_no_synchronous_disk_or_proc_store_work`.
  - `contention-test` failed.
  - `test-cost` was cancelled; the whole run lasted 3h16m.
- `Publish` (scheduled) was red at `8dce35a` (run `42344932501`).
  `sync-release-metadata` exits 3 with `CORE_LAG`, which is blocked on the `sase-core`
  phase's release.
- `CI` on the release-please PR `chore(master): release 0.19.0` (head `fd6f010`, run
  `42345185819`) fails in `release-core-floor-smoke`. The floor binding lacks newer core
  symbols, the same `CORE_LAG` root cause.

In scope for green:

- On master at the tip: `Master Gate`, `Full CI`, `CI Telemetry`, and `Publish`.
- `CI` on the open release-please PR at its head.

If the `sase-core` phase ended in a report rather than a release, still make every
workflow green that isn't blocked by `CORE_LAG`, and cite that report for the rest.
Merging sase's own release PR is not required for this phase.

## Phase: bob-cli

Repository: the `bob-cli` SASE project (open it with `/sase_repo`, for example
`sase repo open bob-cli`).

Observed when planned:

- `CI` on master has been red on every completed push since at least 2026-11-06 07:48
  EST. The latest is `dbd30ec` (run `42364277532`). Many intermediate pushes were
  `cancelled` by concurrency.
- `lint` fails: `cargo clippy --workspace --all-targets --all-features -- -D …` flags
  `clippy::result_large_err` on `native::capture::output::CaptureError`. Dozens of
  functions return `Result<_, CaptureError>`, and clippy suggests boxing large variants.
  There is also a diagnostic at `src/native/task_cancel/prune.rs:119`.
- `test (macos-latest)` is `cancelled` at about 45 minutes on every run, which looks
  like a job timeout or a hung test. `test (ubuntu-latest)` and `package` pass.

Leads: check whether a toolchain or clippy bump introduced the lint, or whether
`CaptureError` gained a large variant. Box the large payloads rather than `allow`-ing
the lint, unless the repo's lint conventions say otherwise. For macOS, find the hanging
test on the `mac` host using per-test timeouts. Compare notes with the `bob-mac-capture`
phase, which also has a macOS hang, in case the cause is shared.

In scope for green: `CI` on master at the tip, with all four jobs succeeded and none
cancelled.

## Phase: bob-mac-capture

Repository: `bob-mac-capture` is a linked repo of the bob-cli project. Open it with
`/sase_repo`; from a sase workspace it may open as the external repo
`gh:bobs-org/bob-mac-capture`.

Observed when planned:

- `CI` (one job, `macOS 26 SwiftPM`) has been red on master since at least 2026-11-06
  07:36 EST. The latest is `1e0a092` (run `42337582432`), "fix(mac-capture): repair fast
  == escape …".
- The `Test` step trips the workflow watchdog:
  `swift test ran longer than 15 minutes; sampling xctest`. The XCTest sample in the log
  sits around the `BobRefsFetcher` and argv-record tests. The suite hangs; it doesn't
  fail an assertion.
- Many `ci/precommit/*` runs titled "macos-ci-check verification snapshot" are also red.
  That is an existing pre-land macOS verification flow. Reuse it to check a fix before
  landing, instead of inventing a new one.

Leads: read the xctest sample in the failing log to find the blocked thread: subprocess
wait, main-thread run loop, or a semaphore or expectation that never fulfils. Reproduce
on the `mac` host with a test filter and a timeout. Also check whether recent `bob` CLI
JSON or argv changes (for example `ref list -R all -A -f json`) desynchronized a fake.

In scope for green: `CI` on master at the tip.

## Phase: dotfiles

Repository: `bbugyi200/dotfiles` should be the `chezmoi` linked repo. Open it with
`sase repo open chezmoi` and confirm `git remote -v` points at `bbugyi200/dotfiles`. If
it doesn't, open `gh:bbugyi200/dotfiles` as an external repo. Never run `chezmoi apply`
or otherwise touch Bryan's live home directory.

Observed when planned:

- `CI` job `test (ubuntu-latest)` has failed on every master push since `9d4011a`
  ("feat(doctl): add guided Apollo recovery login", 2026-11-05 23:35 UTC). The previous
  push, `af323fb`, was green. The latest is `e931695` (run `42368492034`). `lint`
  passes.
- The failing case is "Help and usage errors":
  `./tests/bash/doctl_login_test.sh: line 195: rg: command not found`. The runner has no
  ripgrep.

Leads: if only the test uses `rg`, switch it to portable `grep`. If the script under
test also calls `rg` at runtime, that is a real portability bug: make the script
portable, or declare and provision the dependency in CI the way the repo provisions its
other tools. This looks small, so the worker may plan the smallest tier that fits.

In scope for green: `CI` on master at the tip, with both jobs succeeded.
