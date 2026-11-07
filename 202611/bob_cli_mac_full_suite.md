---
tier: tale
title: Complete bob-cli macOS CI coverage and timing audit
goal:
  All macOS CI commands pass with complete target evidence and comfortable headroom
  under the unchanged 45-minute timeout, completing sase-1rq.3.1.3.
size: medium
proposed_by: bbugyi200.athena.sase-1rq.3.1.3
proposed_with: codex/gpt-6-astra
bead: sase-1rq.3.1.3
create_time: 2026-11-07 09:56:04
status: wip
---

- **PARENT:**
  [202611/bob_cli_ci_green.md](https://github.com/sase-org/sase--plans/blob/main/202611/bob_cli_ci_green.md)
- **BEAD:** sase-1rq.3.1.3

# Complete bob-cli's macOS CI coverage and timing audit

## Objective and ownership

Complete assigned phase `sase-1rq.3.1.3` (`mac-full-suite`) of
`plan:202611/bob_cli_ci_green.md`, under the shared contract in
`plan:202611/red_ci_repair.md`. Run all three macOS CI check commands on a tree
containing the preceding fixes, repair remaining failures or hangs, and publish a
complete test-target inventory and a credible runtime budget below the unchanged
45-minute job limit. Only close `sase-1rq.3.1.3` after its acceptance criteria hold. The
later phase `sase-1rq.3.1.4` owns the final master-tip CI confirmation and aggregate
`ci_evidence`; do not close it or any ancestor.

This is a medium tale because the known failure families have landed, this phase has no
earlier unfinished implementation notes, and one coder can own the remaining
verification and bounded repairs. Long builds and tests use `/sase_monitor`
continuations. There is no evidence yet that another decomposition is needed, and no new
reviewer decision is necessary. Do not create task beads; discovered follow-up work
belongs in `PROPOSED FOLLOW-UP:` notes on `sase-1rq.3.1.3`.

## Verified starting point

Planning checked these facts on 2026-11-07; refresh changing facts before execution:

- GitHub `master` and the opened checkout's `origin/master` both point to
  `41a2de71e669905adfef834fb3a0dbe8f9bfb43f`, whose commit is
  `fix(tests): make macOS CI suites deterministic and green`.
- That commit contains the prerequisite phase's APFS fixture fix, canonical vault-path
  handling in `src/native/gmail/link.rs`, canonical worker-test expectations, clipboard
  diagnostic-row lookup, explicit manual login tests, and guarded Google test harnesses.
  Those harnesses use `tests/support-bin/open` and `xdg-open`, scrub ambient desktop/SSH
  variables, and bound accidental consent waits. Preserve those fixes.
- `b8c417f0ba1995ee24c6e75a35de61128b0028c9` also landed: the preceding lint phase made
  the EarnedEta test independent of `CARGO_INCREMENTAL`.
- The local checkout can lag its freshly fetched remote ref: the planner's worktree was
  still at `e9284fbf`. Do not mistake that old worktree for current master or repeat
  fixes because they are absent there. Advance the clean opened checkout before testing.
- CI run [42382951164](https://github.com/bobs-org/bob-cli/actions/runs/42382951164) was
  in progress on `41a2de71`; no full green result was claimed. Inspect its outcome
  before inventing new work. The older run on `b8c417f0` was cancelled.
- The prerequisite bead reports passing macOS filters plus `gcal_auth` (55 tests, 7.74
  s), `gmail_shell` (52, 11.06 s), and `gkeep_auth` (14, 1.99 s). It did not run the
  entire macOS suite. It also records one clean-base Linux `check-scripts` failure:
  `test_real_cargo_install_transcript_uses_correct_phases_and_hides_chatter`, a
  `StopIteration` finding the compiling-dependencies frame on base `dbd30ec0`.
- `ssh -x mac` succeeds. The host is Darwin arm64 and has about 96 GiB free on
  `/System/Volumes/Data`. The noninteractive SSH PATH contains the Cargo bin directory
  and system directories, but misses Homebrew Bash 5 and `uv`. Executables were found at
  `/opt/homebrew/opt/bash/bin/bash`, `/opt/homebrew/bin/uv`, and
  `/Users/bbugyi/.local/bin/uv`. Python is 3.9.6. Outside the repo rustup defaults to
  stable; the repo explicitly pins Rust 1.95.0 in `rust-toolchain.toml`.
- `sase bead epic-symbols sase-1rq.3.1.3` currently reports no allowances. Check again
  before closure.

## Constraints inherited from the approved epic

The final decisions are `test_quarantine = quarantine_flakes` and
`release_merge = gate`. Prefer root-cause fixes. Only a test proven flaky by failing and
passing on an unchanged tree can be quarantined; record the exact evidence and a
`PROPOSED FOLLOW-UP: flake — ...` note. A deterministic macOS failure may not be ignored
or hidden behind a platform skip.

Do not edit `.github/workflows/*`, job timeouts, job lists, the `clippy_gate`,
variables, secrets, branch protection, Cargo package versions, or CHANGELOG.md. Never
dispatch `release.yml`. The existing advisory clippy warnings, including
`result_large_err`, are outside this repair. No memory edits are authorized or needed by
this tale.

Use `/sase_repo` to open `bob-cli`, read its `AGENTS.md`, and make all source changes in
the returned checkout. Do not commit or push master manually; declare the target repo's
changes through `/sase_final`. The existing `scripts/dev/macos-ci-check`
throwaway-branch verification is the parent-approved exception for pre-land CI.

## 1. Refresh the baseline and prepare the Mac

1. Read the assigned bead and the prerequisite `sase-1rq.3.1.2` through audited bead
   reads. Read the two plan artifacts through `sase artifact read`. Preserve the
   assigned phase's runtime-owned status; do not reopen it or set status by hand.
2. Open `bob-cli` with `/sase_repo`, inspect status, fetch `origin/master`, and advance
   the clean checkout without discarding existing work. Confirm the fixes above are
   ancestors of the test baseline. If earlier work already supplied complete evidence,
   inspect it and run only what remains instead of repeating repairs.
3. Read `.github/workflows/ci.yml`, `justfile`, and `rust-toolchain.toml` at that
   baseline. Inspect the current master CI jobs and failures through connected GitHub
   tools or `gh run list` / `gh run view`. Read current code locally, not through web
   repository-file endpoints.
4. Read `tailnet.md` with `/sase_memory_read` before SSH. Use a new, uniquely named
   `/tmp/bob-cli-sase-1rq.3.1.3-<run-id>` directory on the Mac, populated from the
   authorized local checkout by archive/rsync. Exclude `.git`, `target`, and
   `sase/repos`. Record the baseline SHA and any patch digest so results identify the
   exact source tree. The scratch tree is a disposable test copy; author repairs in the
   opened local checkout and resync them deliberately.
5. Recheck disk space; stop building below approximately 12 GiB free. Use a target
   directory inside this scratch directory and `nice -n 10` for builds/tests. Do not
   touch live `~/projects` checkouts or another agent's scratch directories.
6. Verify the pinned toolchain, clippy/rustfmt, just, Python, Bash 5, and uv. Put only
   the Bash 5 directory ahead of system Bash, as CI does; avoid broadly prepending
   Homebrew's bin directory because it changes BSD utility selection. Expose uv through
   a narrow executable shim or its own directory after inspecting what it contains.
   Record tool versions and the effective PATH.
7. Use `CARGO_INCREMENTAL=0` and unset `SSH_CONNECTION`, `SSH_TTY`, and `SSH_CLIENT` for
   test processes. Keep authentication fakes and the committed opener guards intact. Do
   not use real credentials, a real vault, or the laptop's browser to satisfy tests. Any
   additional diagnostic opener shim must not be necessary for the final passing run;
   fix isolation in the relevant test harness if a gap appears.

If the Mac is unreachable or becomes unavailable, use the existing
`scripts/dev/macos-ci-check` under `/sase_monitor` and inspect the exact-SHA macOS job.
Record the deviation and obtain complete target/timing evidence from that run. Do not
claim Linux execution as macOS evidence.

## 2. Measure all targets with bounded execution

Run the exact macOS workflow commands in order, with the environment described above:

```sh
just lint
cargo test --locked --workspace --no-fail-fast
cargo test --locked --workspace --lib archive_move_presenter -- --nocapture --test-threads=1
```

Keep diagnostic orchestration in scratch files unless a discovered defect justifies
maintained tooling. It must preserve exit status, save raw logs, and distinguish a
timeout, a compile failure, a failed test, a conditional skip, and an unrun target. Use
a per-binary watchdog and a bounded outer command. A temporary Python driver can watch
Cargo's target-start output, reset a deadline at each target, record elapsed time, and
kill its owned process group on timeout. Account for `Doc-tests` and build time
separately; a single timer for the whole suite is insufficient. Ensure silence from a
running test cannot block timeout enforcement. A local dummy sleeping process can verify
watchdog termination without exercising product behavior.

Use explicit limits generous enough for real work: the previous runner took roughly 90 s
for the lib target and 439 s for `cli`; for example, 15 minutes per large target and 3
minutes per smaller integration target are diagnostic ceilings. Build/lint may need
longer on the first cold run. Record chosen limits. Never turn a timeout into success,
and never change CI's 45-minute job limit to accommodate it. If a hang aborts the Cargo
invocation, retain its last test output, identify the culprit with a targeted rerun, and
continue coverage of the other binaries with targeted commands. After fixes, the exact
complete command sequence above must succeed.

Derive target coverage from Cargo's workspace/target metadata and actual execution logs,
not a hardcoded list that can omit new targets. Include both workspace crates
(`bob-obsidian` and `bob-obsidian-assets`), library and binary unit-test targets, `cli`,
`dataview_parity`, all six `gcal_*`, all eight `gkeep_*`, all six `gmail_*`,
`randomize`, `tasks_parity`, `tasks_real_vault_parity`, every doctest group, and the
separate archive-move presenter invocation. Show zero-test targets explicitly.
Distinguish libtest ignored counts from tests that return early when optional tools are
absent.

For each target capture package/target, executed/passed/failed/ignored counts, result,
wall time, watchdog limit, and log location. Also record lint, build, complete
test-step, presenter-step, and total elapsed times. Assert that no discovered target is
silently missing from the final inventory.

Use `/sase_monitor` before starting work that can outlast the provider turn. The
continuation must retain the exact tree identity, scratch path, commands, coverage
inventory, and next action. Read and follow the monitor skill, wait for the staging
command to exit, then finalize; do not leave a foreground build running at turn end.
Build/test monitors must carry normal capacity weight. Pure external CI waits may use
weight zero. One monitor can run a batch that collects all required logs.

## 3. Repair only evidence-backed failures and budget problems

Prioritize the newly exercised Gmail targets, particularly `tests/gmail_support/mod.rs`,
`tests/gmail_support/fake_gog.py`, and the individual Gmail integration tests. Other
likely mechanisms are APFS case collisions, `/var` versus `/private/var` prefix
comparisons, subprocess/environment isolation, PATH differences, and accidental
real-network or consent waits. Inspect a failing assertion before selecting a mechanism.

Fix production behavior when the contract is wrong; adjust host-dependent fixtures or
test harnesses when production is correct. Add or strengthen focused regression coverage
for actual behavior fixes. Re-run affected targets on macOS between edits, and their
counterparts on Linux. Do not repeatedly run every target while diagnosing one failure.
After the final change, run the full macOS sequence once on the final source tree.

Compare measured times to the prior runner's approximately 5-minute lint and 18-minute
cold test build, with 90-second lib and 439-second CLI execution. The sum before the
remaining tests was already about 32 minutes, so a warm local run alone is not proof of
adequate runner headroom. Separate compilation from tests and include setup/presenter
overhead and uncertainty. If the projected job reaches about 35 minutes, investigate
slow tests and remove unnecessary sleeps, real I/O, or waiting through root-cause fixes.
Use actual current-runner step timings or `scripts/dev/macos-ci-check` when
local-to-runner extrapolation cannot establish the roughly ten-minute margin. Never
obtain headroom by dropping coverage or relaxing gates.

Before landing source changes, run the parent's Linux checks with `CARGO_INCREMENTAL=0`:
`just fmt`, `just lint`, `just check-scripts`, `just check-release-tools`, and affected
Cargo test targets (or `just check` for broad changes). Record an incidental failure
that reproduces identically on the clean base as a `PROPOSED FOLLOW-UP:` note with
commands, baseline, and any existing tracking bead; the user explicitly permits closure
despite such unrelated baseline failures. Cite the prerequisite bead's existing PTY-test
finding rather than creating a duplicate. This exception does not turn a missing or hung
in-scope macOS target into a passing run.

## 4. Deliver evidence and close the assigned phase

Create a durable Markdown evidence file with the tested SHA and patch identity, Mac
OS/architecture/tool versions, environment, exact commands, complete inventory, raw-log
references, repairs, Linux validation results, and the runner-time budget. Register it
with `sase artifact create ... --bead sase-1rq.3.1.3`; copy any needed Mac-only logs
back before deleting scratch data. Keep the report factual about local versus Actions
evidence. Include actual CI run URLs and attempts when used, plus
`gh run list --repo bobs-org/bob-cli --workflow ci.yml --branch master --limit 10` for
the final CI phase to refresh run state.

Acceptance requires all three macOS commands passing on the final source tree, every
workspace target and doctest accounted for, no unexplained hang or missing result, Linux
checks passing or independently documented unrelated baseline failures, and evidence of
comfortable headroom under 45 minutes. Explain any proven-flake quarantine and link its
phase note. Record concrete remaining external blockers instead of claiming completion
if these criteria are not met; further implementation must remain mechanically scheduled
through the appropriate SASE continuation.

Remove only this phase's Mac scratch directory and target data after saving evidence.
Append a concise evidence note to `sase-1rq.3.1.3`, run
`sase bead epic-symbols sase-1rq.3.1.3`, and resolve any newly introduced entries or
re-key them to an appropriate still-open parent/later phase. Then close only the
assigned phase using `sase bead close sase-1rq.3.1.3 --note '<verification evidence>'`.
Use `/sase_final` for all changed repositories and do not mutate after declaration.
Leave aggregate master-tip confirmation and ancestor closure to their assigned agents.
