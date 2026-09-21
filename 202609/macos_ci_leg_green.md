---
tier: epic
title: Make the required sase-core macOS CI leg actually run and pass
goal: 'The sase-core rust-checks macOS leg installs its toolchain, runs fmt, clippy,
  and the full test suite, and a master CI run is green on both ubuntu-latest and
  macos-latest with the macOS leg blocking.

  '
phases:
- id: ci-toolchain-parse
  title: Make the CI toolchain-parse step portable to the macOS runner
  depends_on: []
  size: small
  description: 'ci-toolchain-parse: replace the GNU-only grep/sed parsing of rust-toolchain.toml
    in the rust-checks job with a form BSD tools accept, and make sure scripts/check.sh
    runs under the macOS runner''s bash.'
- id: macos-ci-green
  title: Fix whatever the first real macOS CI run surfaces
  depends_on:
  - ci-toolchain-parse
  size: medium
  description: 'macos-ci-green: watch the first master CI run in which the macOS leg
    gets past toolchain install, and fix every macOS-only fmt, clippy, or test failure
    it reports under the parent epic''s platform-path rule.'
proposed_by: bbugyi200.athena.sase-157.land
parent_bead: sase-157
create_time: 2026-09-21 12:36:10
status: wip
bead_id: sase-157.10
---

- **PROMPT:** [prompts/202609/macos_ci_leg_green.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/macos_ci_leg_green.md)
- **PARENT:** [202609/macos_portability.md](https://github.com/sase-org/sase--plans/blob/main/202609/macos_portability.md)
- **BEAD:** [sase-157.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-157/sase-157.10.md)

# Plan: Make the required sase-core macOS CI leg actually run and pass

## Repository

All work happens in the **`sase-core`** repository, not the `sase` primary checkout.
Open it with the `/sase_repo` skill before reading or writing anything, and use only the
path that skill prints:

```bash
sase repo open sase-core -r "<phase-specific reason>"
```

`sase-core`'s `AGENTS.md` is binding: release-plz owns all versions, and `just check` /
`./scripts/check.sh` is the only verification entry point (never bare `cargo test` or
`cargo clippy`).

## Why this epic exists

This is remaining work of epic `sase-157` ("Make sase-core correct and green on macOS"),
found by its land agent. `sase-157`'s goal includes "a required macOS CI leg keeps it
that way", and its final phase (`sase-157.9`) was done-when "CI is green on both
platforms with the macOS leg required". That is not true:

- `sase-157.3` (commit `19274c0`) turned `rust-checks` in `.github/workflows/ci.yml`
  into an `ubuntu-latest` / `macos-latest` matrix. `sase-157.9` (commit `d99ba11`)
  removed the `continue-on-error` escape hatch so the macOS leg blocks.
- The macOS leg has **never got past toolchain install**. Its `Read pinned toolchain`
  step parses `rust-toolchain.toml` with
  `grep -E '^channel\s*=' ... | sed -E 's/.*=\s*"([^"]+)".*/\1/'`. BSD `sed` on the
  macOS runner does not understand `\s`, so the substitution never matches, and the
  whole line `channel = "stable"` becomes the toolchain name. `dtolnay/rust-toolchain`
  then runs `rustup toolchain install channel = "stable" ...` and fails with
  `error: invalid value 'channel' for '[TOOLCHAIN]...': invalid toolchain name: 'channel'`.
  See CI run `35623241380` (push of `d99ba11`), job
  `cargo fmt + clippy + test (macos-latest)`, step `Run dtolnay/rust-toolchain@stable`.
  The `components=` line uses the same `\s` construct.
- `sase-157.5` recorded exactly this as a `PROPOSED FOLLOW-UP`, but it was never fixed.
  Every macOS verification in `sase-157` therefore ran on the developer Mac (`mac` on
  the tailnet), not in CI.
- Because the leg now blocks, **sase-core `master` CI is red** since `d99ba11`, and the
  open release-plz PR (`chore: release v0.34.71`) is red for the same reason.
  `AGENTS.md` notes a red `master` fails every `Release-plz` run until fixed. Treat the
  first phase as urgent.

Do **not** fix this by restoring `continue-on-error` or dropping the macOS leg. A
blocking macOS leg is what the parent epic delivers.

## Verification environment

- `gh run list --workflow CI --limit 10`, `gh run view <id>`, and
  `gh run view <id> --log-failed` (run from the `sase-core` checkout) read CI status and
  logs. These are GitHub Actions data, not repo file contents, so they are allowed.
- The developer Mac (`ssh mac`, checkout at `~/projects/github/sase-org/sase-core`) is
  best-effort and was offline when this plan was written. Read the `tailnet` reference
  memory before using it. The parent plan (`plan:202609/macos_portability.md`,
  "Verification" section) documents the patch-ship-and-restore loop. Always leave that
  checkout clean at `master`. Use `zsh -lc` so `/opt/homebrew/bin/python3.12` is on
  `PATH`.
- Agents never commit (host-owned completion). A phase's changes reach `master` only
  when that phase finishes. That is why the phases are split: `macos-ci-green` observes
  the CI run that `ci-toolchain-parse`'s commit triggers.
- Every phase still runs `just check` on Linux before finishing.

## Phase `ci-toolchain-parse` — Make the CI toolchain-parse step portable to the macOS runner

**Files:** `.github/workflows/ci.yml`; `scripts/check.sh` only if the bash audit below
finds a problem.

1. Rewrite the `Read pinned toolchain` step so it produces the same outputs on GNU and
   BSD userlands. The simplest fix is POSIX bracket classes: replace every `\s` with
   `[[:space:]]` in both the `grep -E` and `sed -E` expressions, for both `channel` and
   `components`. Any equivalent portable approach is fine, for example `awk`, or
   dropping the parse in favour of the toolchain file. The step must not depend on
   `setup-python`, because it runs before that step. Keep the Linux leg's resolved
   toolchain byte-identical: today it resolves `stable`.
2. Make the step fail loudly when parsing fails. After computing `channel`, reject an
   empty value or one containing whitespace or `=`, with a clear `::error::` message. A
   future parse regression should then name itself instead of surfacing as an opaque
   rustup error.
3. Audit the rest of the macOS leg for other GNU-only or bash-4-only constructs before
   the first real run finds them one at a time. `scripts/check.sh` runs through
   `#!/usr/bin/env bash`. On a macOS runner that may be Apple's `/bin/bash` 3.2, not a
   Homebrew bash 5, so confirm it avoids bash-4+ features: associative arrays,
   `${var,,}`, `mapfile`, `"${empty_array[@]}"` under `set -u`, and so on. Only change
   `check.sh` if the audit finds a real incompatibility, and keep its no-argument
   behaviour byte-identical.
4. Verify the new parse expressions against a BSD `sed`/`grep` if one is available: on
   `mac` if it is online, otherwise say plainly that BSD verification is left to CI.
   Check that `actionlint` (if installed) and a YAML parse are clean, and that
   `just check` passes on Linux.

### Done when

The parse step produces `channel=stable` and the correct components on both GNU and BSD
tools, fails with a clear error on a bad parse, `check.sh` has been audited for bash 3.2
(with fixes only if needed), and `just check` passes on Linux.

## Phase `macos-ci-green` — Fix whatever the first real macOS CI run surfaces

**Files:** whatever the run implicates. Expect `.github/workflows/ci.yml`, cfg-gated
code under `crates/`, and tests.

This phase starts after `ci-toolchain-parse` has landed on `master`.

1. Find the CI run for the `master` commit that carries the `ci-toolchain-parse` change
   (`gh run list --workflow CI --branch master`). If it is still running, wait for it
   with the `/sase_monitor` skill (for example monitoring
   `gh run watch <id> --exit-status`). Never poll in a loop or promise to resume later.
   If the macOS leg still fails at toolchain install, fix the parse step first.
2. Read the macOS job's failures (`gh run view <id> --log-failed`). This is the first
   time the macOS leg has run fmt, clippy with `-D warnings`, and the full workspace
   test suite on a GitHub runner. Expect some of these:
   - **clippy on macOS**: `dead_code` / `unused_imports` warnings in code that is only
     used behind `#[cfg(target_os = "linux")]`. `sase-157.4` added non-Linux arms to
     `crates/sase_gateway/src/sudo_runner/platform.rs`, and the earlier Linux gates
     never compiled the macOS side with `-D warnings`. Fix these by cfg-gating the
     helper or import to the platform that uses it. Do not blanket
     `#[allow(dead_code)]`.
   - **tests that passed on the developer Mac but not on the runner**. The runner's
     `TMPDIR` is under `/var/folders/...` (symlinked via `/private/var`), it runs as a
     different user, and it may lack tools the Mac has (e.g. `setsid`). Fix each one
     under the parent epic's platform-path rule, which is now in sase-core's `AGENTS.md`
     ("Platform paths"): canonicalize both sides of a path comparison, or neither. For
     each site, decide whether production or the expectation is wrong, and record which
     in the commit body. Do not delete coverage, and do not `#[ignore]` a test to get
     green. A test that is genuinely Linux-only gets `#[cfg(target_os = "linux")]`, the
     way `linux_identity_backend_advertises_detached_execution` already is.
   - **known pre-existing parallel-load flakes**, already filed as task beads `sase-15d`
     (telemetry `concurrent_writers_preserve_every_delta`), `sase-15e` (sudo_runner
     `post_spawn_identity_failure_reaps_barred_worker`, empty `worker.pid`) and
     `sase-15f` (sudo_runner
     `cwd_removed_after_authentication_fails_before_dispatch_and_cleans_up`). If one of
     them is what turns the macOS leg red, fixing it here is in scope, because a
     blocking macOS leg that flakes is not done. Record that on the matching task bead
     with `sase bead note <task-id> "..."` instead of re-filing it. Otherwise leave them
     to their own beads.
3. Also check the release-plz PR's CI (`chore: release v0.34.71`) once `master` is
   fixed. It shares the same workflow, so no separate fix should be needed. Only report
   its state.
4. Verify locally with `just check` on Linux. If `mac` is online, also run the
   implicated crates there through `./scripts/check.sh test -p <crate>` (and
   `./scripts/check.sh clippy`), then restore that checkout clean.

If the run is fully green on both legs with no changes needed, this phase makes no code
change and records that outcome, with the run ID, in its close note.

### Done when

Every macOS-only failure from the first real macOS CI run is fixed (or confirmed absent,
with the run ID recorded), no coverage was deleted or ignored to get there, and
`just check` passes on Linux. A green `master` run on both legs, with the macOS leg
blocking, after this phase lands is the acceptance check for this epic's land agent.

## Out of scope

- Extending `wheel-smoke` or `release-scripts` to macOS.
- The other maintainability-audit findings (§3.3–§3.11) the parent plan lists as out of
  scope.
- Moving the macOS leg off the push path. The `ci.yml` cost comment already records the
  two-speed precedent. Only act on it if macOS minutes become an actual problem.
