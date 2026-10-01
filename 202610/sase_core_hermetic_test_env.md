---
tier: tale
title: Fix sase-core's red master and make the cargo test gate hermetic
goal:
  sase-core master CI is green again, and scripts/check.sh test runs every cargo test in
  a hermetic, CI-identical git and SASE environment. A test that depends on ambient git
  identity or config now fails locally before landing, instead of only on the ubuntu CI
  runner, and a canary test keeps that guarantee from being silently removed.
size: small
proposed_by: bbugyi200.athena.0uo
create_time: 2026-10-01 00:18:07
status: wip
---

# Plan: Fix sase-core's red master and make the cargo test gate hermetic

## Problem

sase-core `master` CI has been red since `a031ee4`
(`feat(file-history): generic git file-history index in sase-core`). Seven consecutive
master pushes have failed CI, and so has every CI run of the `chore: release v0.36.2`
release-plz PR. Only the `cargo fmt + clippy + test (ubuntu-latest)` job fails, and it
always fails on one test:

```
file_history::tests::shallow_clone_marks_health_and_incomplete
panicked at crates/sase_core/src/file_history/tests.rs:38:5:
git ["commit", "-qm", "post"] failed: Author identity unknown
fatal: unable to auto-detect email address (got 'runner@runnervm8df0l.(none)')
```

### Root cause

`init_repo()` in `crates/sase_core/src/file_history/tests.rs` writes `user.name` and
`user.email` into the **source repo's local config**. The test then runs
`git clone --depth 1` into a new directory and commits (`git commit -qm post`) **in the
clone**. A clone does not inherit its source's local config, so the clone has no
identity, and git's behavior depends entirely on the host:

| Host                   | Global identity | Hostname auto-detect | Result |
| ---------------------- | --------------- | -------------------- | ------ |
| athena / dev machines  | yes             | yes (FQDN)           | pass   |
| GitHub `macos-latest`  | no              | yes                  | pass   |
| GitHub `ubuntu-latest` | no              | no (`(none)` domain) | FAIL   |

### Why the gate did not catch it

`scripts/check.sh` exists so that "local verification cannot silently drift from what CI
checks". It enforces parity on **tool selection**: the same cargo steps and the same
pyo3 interpreter. It does not enforce parity on the **environment the test processes
observe**. `cargo test` runs with the developer's ambient git config, git identity, and
environment variables, and none of those exist on the CI runners. No amount of care by
the landing agent could have caught this. `sase tool run check` passed honestly.

Detection already works. The `ci_watch` AXE job sweeps `sase-org/sase-core` every five
minutes and raises a SASE notification for red CI. What is missing is **prevention**:
the pre-landing gate must be able to fail in the same way CI does.

### Evidence already gathered during planning (on athena)

- Ran the full workspace suite through `./scripts/check.sh test --no-fail-fast` under
  exactly the environment recipe below: **5396 passed, 1 failed**. The one failure is
  `shallow_clone_marks_health_and_incomplete`, with
  `fatal: no email was given and auto-detection is disabled`. The recipe reproduces the
  CI failure locally and breaks nothing else.
- Removing the global config alone is **not enough** on athena. With
  `GIT_CONFIG_GLOBAL=/dev/null` and no `user.useConfigOnly`, git still guesses an email
  from the FQDN hostname (`athena.bbugyi.ddns.net`), and the clone-then-commit succeeds.
  Disabling identity guessing is the piece that makes every host behave like the Ubuntu
  runner.
- An earlier run also had one `sase_gateway` failure,
  `sudo_runner::tests::dispatch::resume_from_skips_prior_commands_and_probe_failure_uses_plain_sudo`.
  That test passes on 3 of 3 isolated reruns. It belongs to the known sudo_runner
  load-flake family (`sase-15h` and its siblings), has nothing to do with git, and is
  out of scope.

## Design

**Principle:** test processes run in a declared environment that is the same on every
host and on CI. Any ambient input known to differ between agent hosts and CI is
neutralized. This is enforced at the one choke point that both `just test` /
`just check` and the CI workflow already go through, `scripts/check.sh test`. A canary
test then proves the environment is in force, so it cannot be silently removed later.

Inputs neutralized for the `cargo test` process only:

1. **Global and system git config**: `GIT_CONFIG_GLOBAL=/dev/null` and
   `GIT_CONFIG_NOSYSTEM=1`. Tests stop seeing the developer's identity, hooks
   (`core.hooksPath`), signing, rename, color, and other config that CI does not have.
2. **Git identity guessing**: inject `user.useConfigOnly=true` through
   `GIT_CONFIG_COUNT=1`, `GIT_CONFIG_KEY_0=user.useConfigOnly` and
   `GIT_CONFIG_VALUE_0=true`. This needs no temp file. Git then refuses to invent an
   identity on any host, including FQDN hosts and the macOS runner. A repo-local
   `user.name`/`user.email` still works normally.
3. **Ambient git env**: unset every exported `GIT_*` variable before setting the vars
   above. That covers identity overrides such as `GIT_AUTHOR_*` and `GIT_COMMITTER_*`.
   It also covers repo-location leaks such as `GIT_DIR`, `GIT_WORK_TREE` and
   `GIT_INDEX_FILE`, which appear when a gate runs from inside a git hook. Also unset
   `EMAIL`, which git uses as an identity fallback.
4. **Ambient SASE env**: unset every exported `SASE_*` variable. A SASE agent run
   carries about 35 of them (`SASE_AGENT`, `SASE_AGENT_NAME`, `SASE_ARTIFACTS_DIR`, and
   others). sase-core production code reads many of these, but CI never has them. Tests
   that need one already set it themselves, as `tests/artifact_ref_commit_budget.rs`
   does. The 5396-pass evidence above already ran with `SASE_*` cleared.

These are deliberately left untouched: `PATH`, `HOME`, `CARGO_*`, `RUSTUP_*`,
`PYO3_PYTHON`, `LD_LIBRARY_PATH` / `DYLD_LIBRARY_PATH`, `TMPDIR`, `RUST_*`, and the
`UPDATE_*` golden/contract regeneration switches.

### Alternatives considered and rejected

- **Fix only the test.** This does not prevent recurrence. The next clone-then-commit,
  or any other reliance on ambient identity or config, lands green locally again and
  turns CI red.
- **Only `GIT_CONFIG_GLOBAL=/dev/null`.** Insufficient: git still auto-detects an
  identity on FQDN hosts (proven on athena above).
- **A shared cross-crate git test helper plus a lint that bans raw `Command::new("git")`
  in tests.** There are 10 ad-hoc `fn git(` helpers across `sase_core`, `sase_core_py`
  and `sase_xprompt_lsp`. Sharing one across crates needs a new workspace crate or a
  cargo feature, which brings release-plz and hakari overhead. A convention also only
  catches the call sites it lints, while the hermetic gate catches every identity or
  config dependence no matter which helper made the call. Revisit only if hermetic-gate
  failures from ad-hoc helpers become a recurring cost.
- **`env -i` plus an allowlist.** It is the most hermetic, but brittle: cargo, rustup,
  pyo3, wrappers and proxy vars all have to be enumerated. A class-level denylist
  (`GIT_*`, `SASE_*`, `EMAIL`) covers the inputs known to differ between agent hosts and
  CI.
- **Isolating `HOME`.** sase-core reads `HOME` in 16 places, so this is a plausible
  future class. But overriding `HOME` around `cargo test` disturbs rustup, cargo and
  pyenv resolution. Deferred: reopen if a CI failure is ever traced to `HOME` state.
- **A `.cargo/config.toml` target runner wrapper.** It would also rewrite the
  environment for `cargo run` of the gateway binaries, and the gate is check.sh anyway.
- **Branch protection or required PR checks on sase-core `master`.** This conflicts with
  host-owned completion pushing directly to master, and detection is already covered by
  `ci_watch`.

## Implementation

Work in the linked sase-core checkout. Open it with
`sase repo open sase-core -r "<reason>"`, use only the printed path, and read its
`AGENTS.md` before editing. Nothing changes in the sase repo. No binding or wire change
is involved, so `sase-core-revision.txt` needs no ratchet.

### Step 1: hermetic test environment in `scripts/check.sh`

- Add a helper (suggested name `run_hermetic_tests`) that runs its arguments in a
  subshell:
  - Unset every exported variable whose name starts with `GIT_` or `SASE_`, plus
    `EMAIL`. Enumerate exported names with `compgen -e`.
  - Export `GIT_CONFIG_GLOBAL=/dev/null`, `GIT_CONFIG_NOSYSTEM=1`, `GIT_CONFIG_COUNT=1`,
    `GIT_CONFIG_KEY_0=user.useConfigOnly` and `GIT_CONFIG_VALUE_0=true`.
  - `exec "$@"`.
- Keep the helper bash 3.2 compatible, because the macOS runner may use `/bin/bash`: no
  `mapfile`, no associative arrays, no `${var,,}`.
- In `cmd_test`, call `configure_pyo3_python` first, as today, so `PYO3_PYTHON` and the
  library path are in place. Then route **both** `cargo test` branches (default
  `--workspace` scope and explicit package selection) through the helper.
- Do not apply the helper to `fmt-check`, `fmt`, `features`, `check` or `clippy`. They
  do not execute test code, and cargo itself needs no git config here because
  `Cargo.lock` has no git dependencies.
- Add a comment block in the style of the existing `check.sh` comments. It should name
  the incident: `a031ee4` committed in a shallow clone with no identity, which passed on
  dev hosts and macOS but failed only on ubuntu-latest. It should also explain why
  `useConfigOnly` is required (FQDN auto-detect) and why `SASE_*` is cleared (agent-only
  vars that CI never has).
- Update the `usage()` text for `test`, for example: "cargo test --workspace [args...]
  under the hermetic test environment".

### Step 2: canary integration test

Create `crates/sase_core/tests/hermetic_test_env.rs`. Integration tests are
auto-discovered, and a separate test binary means no other test in the same process can
`set_var` around it. The canary asserts, with a clear failure message:

- None of `EMAIL`, `GIT_AUTHOR_NAME`, `GIT_AUTHOR_EMAIL`, `GIT_COMMITTER_NAME`,
  `GIT_COMMITTER_EMAIL` or `GIT_DIR` is set, and no `SASE_*` variable is set.
- In a fresh `tempfile::tempdir()` repo (`git init`, with no identity configured, and
  every git call made with `-C <tmp>` so the sase-core checkout's own `.git/config` can
  never supply an identity), `git commit --allow-empty -m canary` **fails**. Assert that
  the exit status is not success. Do not tie the assertion to git's exact stderr wording
  beyond an optional loose check.
- The failure message must say that cargo tests must run through `scripts/check.sh test`
  (`just test` / `just check`, which is also what CI runs). That script installs the
  hermetic environment. The message should point to the sase-core `AGENTS.md` "Build and
  verify" note.

If the hermetic block is ever removed from `check.sh`, CI fails this canary on both
operating systems. A bare `cargo test --workspace` on a dev host fails it with guidance,
which matches the existing "never run bare cargo" rule.

### Step 3: prove the gate reproduces CI, then fix the test

1. With steps 1 and 2 in place, run
   `just test -p sase_core shallow_clone_marks_health_and_incomplete`. Confirm it
   **fails** with the identity error, which shows the local gate now reproduces the CI
   failure.
2. In `crates/sase_core/src/file_history/tests.rs`, extract the identity and signing
   lines from `init_repo()` (`user.name`, `user.email`, `commit.gpgsign=false`) into a
   small helper such as `configure_identity(repo: &Path)`. Call it from `init_repo()`
   and on the shallow clone right after the clone succeeds, before the `post` commit.
   Add a one-line comment saying that a clone does not inherit its source's local
   config.
3. Rerun the targeted test and confirm it passes.

### Step 4: document the contract in sase-core `AGENTS.md`

Add a bullet under **Build and verify** in the file's existing terse style. It should
cover these points:

- `just test` and `just check` run tests in a hermetic environment: no global or system
  git config, git identity guessing disabled (`user.useConfigOnly`), and ambient
  `GIT_*`, `SASE_*` and `EMAIL` cleared. The environment is the same on every host and
  on CI.
- A test that commits must set `user.name`/`user.email` in the repo it commits in, and a
  clone does not inherit its source's local config.
- A test that needs a `SASE_*` variable sets it itself.
- The `hermetic_test_env` canary fails when tests run outside this environment.

## Verification

- Step 3.1: the targeted test fails before the fix, with the identity error, under
  `just test`.
- `just test -p sase_core file_history` passes after the fix.
- `just test -p sase_core --test hermetic_test_env` passes.
- Canary negative check, local only and not committed: temporarily make `cmd_test` skip
  the helper. Confirm the canary fails with its guidance message, then restore the
  helper.
- The full gate passes: `sase tool run check` in the sase-core checkout, with an
  explicit tool timeout of 10 minutes or more. If a sudo_runner test fails, rerun it
  alone first. The sudo_runner tests are a known load-flake family (see
  `sase bead list -T flake`), so do not weaken any assertion.
- After the host lands the commit, which happens outside the implementer's turn, CI on
  `ubuntu-latest` and `macos-latest` turns green, `ci_watch` posts its resolution +1,
  and the `chore: release v0.36.2` release-plz PR goes green on its next run.

## Notes for the implementer

- Do not edit any `version` field, path-dependency pin or `CHANGELOG.md`, because
  release-plz owns them. Use a Conventional Commit subject, for example
  `fix(file-history): give the shallow-clone fixture an identity and run tests hermetically`.
- Keep every touched file at or under 1,500 lines. Do not use `macro_rules!`.
- The sase repo itself has no changes in this plan.
