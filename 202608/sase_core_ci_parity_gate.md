---
tier: tale
title: Give sase-core a runnable CI-parity gate so red commits stop reaching master
goal:
  A sase-core agent can run every gate CI runs with one command that works in the
  standard SASE agent environment, CI runs that same entry point so the two cannot
  drift, and the repo's AGENTS.md makes running it before each commit mandatory.
size: small
proposed_by: bbugyi200.athena.yd
create_time: 2026-08-12 08:10:09
status: done
---

- **AGENTS:**
  - bbugyi200.athena.yd
- **COMMITS:**
  - [865da85](https://github.com/sase-org/sase-core/commit/865da857f400027393364d35569c6543d6e0890c)
    — feat: add CI-parity verification gate

# Give sase-core a Runnable CI-Parity Gate

## Incident Summary (already healed — read before assuming CI is red)

`actstat --repo sase-org/sase-core` showed three consecutive failing settled commits on
`master` (`c0f1ca4`, `a71794c`, `a509dcc`), across both the `CI` and `Release-plz`
workflows. Those specific failures are **already fixed**:

- `a7d5c9e` ("fix: restore green CI on sase-core") landed the code fixes and is green on
  both workflows.
- Release PR #112 (`chore: release v0.26.0`) was force-refreshed by release-plz onto the
  fixed master, went green, and merged. The `Release-plz` run that had been failing at
  `Merge release PR` -> `Wait for checks to pass` succeeded.

So this plan is **not** about re-fixing those failures. It is about the process hole
that let them land on `master` in the first place, which is still wide open and will
produce the same outcome again on the next sase-core change.

Do not start by "fixing CI". Start by confirming with
`actstat --repo sase-org/sase-core` that master is green, then implement the gate below.

## Root Cause

### RC1 (primary): the workspace-wide test command does not build in the agent environment

`crates/sase_core_py` pins `pyo3 = { version = "0.22", features = ["abi3-py312"] }`.
`pyo3-build-config` hard-errors when the interpreter it discovers is older than 3.12:

```
error: failed to run custom build command for `pyo3-build-config v0.22.6`
  error: cannot set a minimum Python version 3.12 higher than the interpreter
  version 3.11 (the minimum Python version is implied by the abi3-py312 feature)
```

In the standard SASE agent shell, `python3` resolves through a pyenv shim to
**3.11.13**, so `cargo test --workspace`, `cargo clippy --workspace --all-targets`, and
`cargo build --workspace` all fail to build locally — with an error that reads like a
broken environment rather than a code problem. `cargo test -p sase_core` works fine, so
the natural fallback silently drops the `sase_core_py` crate from local verification.

That is exactly the failure that landed: `a509dcc` bumped
`ARTIFACT_REF_CONTEXT_WIRE_SCHEMA_VERSION` from 1 to 2 in
`crates/sase_core/src/artifact_ref/wire.rs` but left three `sase_core_py` binding tests
asserting the old value, and CI caught it only after the push.

This is verified, not inferred. With a 3.12+ interpreter selected explicitly, the same
workspace builds and passes end to end on the current master:

```bash
PYO3_PYTHON=/usr/bin/python3.13 cargo clippy --workspace --all-targets -- -D warnings  # clean
PYO3_PYTHON=/usr/bin/python3.13 cargo test --workspace                                 # all green
```

### RC2: no repo-local verification entry point, and AGENTS.md never names the gates

sase-core has no `justfile`, no `Makefile`, and no `scripts/`. Its `AGENTS.md` is 8
lines covering release-plz version ownership only — it never tells an agent what to run
before committing. The three CI gates live exclusively inside
`.github/workflows/ci.yml`, so "what CI will check" is knowable only by reading the
workflow.

That is how `c0f1ca4` shipped rustfmt drift: a hand-edited `use` list in
`crates/sase_core/src/lib.rs` and a stray blank line in
`crates/sase_core/src/content_layout.rs` that `cargo fmt` would have rewritten in one
second.

### RC3 (amplifier): a red master stalls the release train on a 6-hour cron

`master` is unprotected and agents push to it directly, so CI is a post-hoc detector,
not a gate. `.github/workflows/release-plz.yml`'s `release-plz-merge` job runs
`gh pr checks "$PR" --watch --fail-fast` against the release PR, and the release PR is
cut from master — so one red master commit fails every subsequent `Release-plz` run
(push-triggered plus the `23 */6 * * *` cron) until someone fixes master. v0.26.0 sat
unreleased for roughly 16 hours for this reason, and `actstat` showed compounding red.

This behavior is correct signal and should not be softened; RC1/RC2 are what make it
fire. No change is planned here.

### RC4 (already mitigated, mechanism unidentified): fixture-construction flake

CI runs #1052 and #1055 failed in
`editor::completion::tests::commit_inventory_skips_sidecars_before_reporting_the_row_cap`
with `git commit failed: fatal: could not parse HEAD`, raised from the test's own
`commit_at` fixture helper — never from production code. `a7d5c9e` replaced the
per-commit subprocess loops in the two scale tests with a single `git fast-import`
stream, and CI has been green since.

The precise mechanism was never identified. An audit of the remaining `commit_at` call
sites (`crates/sase_core/src/editor/completion.rs`, `crates/sase_core_py/src/lib.rs`,
`crates/sase_core/tests/artifact_ref_commit_budget.rs`) confirms no other test builds
history in a loop — every remaining call site creates one to five commits. Step 5 adds
cheap insurance to the fixture helpers; it is deliberately small because the flake is
not currently reproducing.

## Goal

One command — `just check` — runs the same three gates CI runs, works in the standard
SASE agent environment without the caller knowing about PyO3 interpreter constraints,
and is the command CI itself invokes, so local and CI verification cannot drift apart.

## Non-Goals

- Do not add branch protection or a required-PR workflow to sase-core. SASE agents push
  to master directly by design; the fix is a working local gate, not a merge queue.
- Do not soften `release-plz-merge`'s check watching (see RC3).
- Do not touch the `wheel-smoke` job, `release-plz.yml`, `pr-title.yml`, or
  `release-plz.toml`.
- Do not edit any `sase/memory/*.md` note or generated instruction shim in the **sase**
  repo. See "Follow-Ups".

## Repo Access

All file changes in this plan are in the **sase-core** repo, which is a linked repo, not
your workspace checkout. Open it first and use only the path that command prints:

```bash
sase repo open sase-core -r "Add a CI-parity verification gate to sase-core"
```

All paths below are relative to that checkout root.

## Steps

### 1. Add `scripts/check.sh` — the single source of truth for the gates

Create an executable (`chmod +x`) bash script that resolves a usable interpreter and
dispatches subcommands. Requirements, not literal code:

- `#!/usr/bin/env bash` with `set -euo pipefail`; `cd` to the repo root derived from
  `${BASH_SOURCE[0]}` so it works from any directory.
- `resolve_pyo3_python`: return immediately if `PYO3_PYTHON` is already set and
  non-empty (respect an explicit caller override). Otherwise probe, in order,
  `python3.14`, `python3.13`, `python3.12`, then plain `python3`; for each one found via
  `command -v`, accept the first whose
  `-c 'import sys; raise SystemExit(0 if sys.version_info >= (3, 12) else 1)'` succeeds,
  and `export PYO3_PYTHON` to its resolved path. If none qualifies, fail with an
  explicit message naming `abi3-py312`, the crate (`crates/sase_core_py`), and the
  `PYO3_PYTHON` override — never a bare non-zero exit.
- Subcommands: `fmt-check` (`cargo fmt --all -- --check`), `fmt` (`cargo fmt --all`),
  `clippy` (`cargo clippy --workspace --all-targets -- -D warnings`), `test`
  (`cargo test --workspace`), and `all` (default: `fmt-check`, then `clippy`, then
  `test`, stopping at the first failure). Unknown subcommand -> usage text on stderr and
  exit 2.
- Only `clippy`, `test`, and `all` need `resolve_pyo3_python`; `fmt`/`fmt-check` must
  keep working with no suitable interpreter present, since rustfmt does not build the
  workspace.
- Comment the `resolve_pyo3_python` function with _why_ it exists: the pyenv-provided
  `python3` on this machine is 3.11, `abi3-py312` refuses to build against it, and the
  resulting build error is opaque enough that agents fall back to `-p sase_core` and
  skip the binding tests. Keep that comment — it is the whole point of the file.

### 2. Add a `justfile` that delegates to the script

Thin wrapper for muscle memory shared with the sase repo. `default` recipe runs `check`;
recipes `check` (`./scripts/check.sh all`), `fmt`, `clippy`, and `test` each delegate to
the matching subcommand. No logic in the justfile — the script stays the single source
of truth so CI (which has no `just`) and agents run identical commands.

### 3. Wire `.github/workflows/ci.yml` to the same script

In the `rust-checks` job, replace these three bare steps:

```yaml
- run: cargo fmt --all -- --check
- run: cargo clippy --workspace --all-targets -- -D warnings
- run: cargo test --workspace
```

with three named steps that call `./scripts/check.sh fmt-check`,
`./scripts/check.sh clippy`, and `./scripts/check.sh test` respectively. Keep them as
three separate steps and give each a `name:` that still reads like the command it runs —
`actstat` reports the failing step name, and collapsing them into one step would make
its output less useful.

This wiring is the load-bearing part of the plan: it is what guarantees the local gate
cannot silently drift from CI. Do not skip it in favor of documentation alone.

Note: `ubuntu-latest` ships Python 3.12+, so `resolve_pyo3_python` is a no-op on CI
today; its value there is failing loudly with a named cause if that ever changes.

### 4. Document the gate in sase-core's `AGENTS.md`

Append a short `## Verification` section to the existing 8-line file (this is
sase-core's own hand-maintained instruction file, not a sase-memory-generated shim —
approving this plan is the approval to edit it). It must state:

- Run `just check` (or `./scripts/check.sh`) from the repo root before every commit; it
  runs the same gates as CI.
- `crates/sase_core_py` builds PyO3 with `abi3-py312`, so the workspace only builds when
  a Python >= 3.12 interpreter is reachable; the script finds one and exports
  `PYO3_PYTHON`, and fails loudly when it cannot.
- Never verify with `cargo test -p sase_core` alone: it excludes the `sase_core_py`
  binding tests, which is how three stale schema-version fixtures reached master in
  `a509dcc`.
- `master` is unprotected and a red commit there also fails every `Release-plz` run
  until it is fixed, which is why the pre-commit gate matters more here than in a
  PR-gated repo.

Keep it tight — a handful of lines, matching the file's existing terse voice.

### 5. Harden the git fixture helpers (cheap insurance for RC4)

In each of the three `init_git_repo` helpers —
`crates/sase_core/src/editor/completion.rs`, `crates/sase_core_py/src/lib.rs`, and
`crates/sase_core/tests/artifact_ref_commit_budget.rs` — add `git config gc.auto 0` and
`git config maintenance.auto false` alongside the existing `user.name` / `user.email` /
`core.abbrev` config calls, with a one-line comment that this keeps background git
maintenance from racing fixture construction.

Be honest in the comment: this removes a known class of interference, not a confirmed
cause. If it makes a helper awkward, keeping the helpers uniform matters more than
squeezing it in — all three or none.

## Verification

Run all of this from the sase-core checkout:

1. `just check` passes end to end on unmodified master. (Baseline confirmed while
   authoring this plan: clippy clean, full workspace test suite green.)
2. `./scripts/check.sh fmt-check`, `clippy`, and `test` each pass when run individually.
3. Negative test — the gate actually catches things: introduce a stray blank line in any
   `.rs` file, confirm `just check` fails at the fmt stage, then `just fmt` and confirm
   it passes again. Leave no residue.
4. Negative test — interpreter resolution: run
   `PYO3_PYTHON=/nonexistent/python just test` and confirm the failure is the PyO3 build
   error for the explicitly overridden interpreter (proving the override is respected),
   then run `just test` unset and confirm it self-resolves and passes.
5. `bash -n scripts/check.sh` parses, and the script is executable in git
   (`git ls-files -s scripts/check.sh` shows mode `100755`).
6. After landing, confirm CI is green on the new head with
   `actstat --repo sase-org/sase-core`, and confirm the three renamed steps appear as
   distinct steps in the run.

## Risks

- **New `scripts/check.sh` dependency in CI**: if the script is not executable in git,
  every CI run breaks immediately. Verification step 5 covers this;
  `bash scripts/check.sh` in the workflow is an acceptable fallback if the mode bit
  proves troublesome.
- **`PYO3_PYTHON` churn**: switching interpreters between runs invalidates the pyo3
  build script cache and triggers a rebuild of the pyo3 crates. Expected, harmless, and
  the reason the script prefers a stable resolution order rather than "newest
  available".
- **Drift is deferred, not eliminated**: if a future CI job adds a gate outside
  `rust-checks` (feature-matrix, docs), it must be added to the script too, or the
  parity guarantee erodes. The AGENTS.md text should make the script the obvious place
  to add gates.

## Follow-Ups (not in this plan)

- The sase repo's `rust_core_backend_boundary` memory tells agents to update tests in
  `../sase-core` but never says how to verify them. Adding "run `just check` in
  sase-core" there would close the loop from the sase side. That is a `sase/memory/*.md`
  edit and requires the user's explicit permission in a live conversation, so it is
  deliberately excluded here — raise it with the user rather than doing it under this
  plan.
