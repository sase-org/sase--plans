---
tier: epic
title: Make sase-core correct and green on macOS
goal: 'The managed-tmp reap guard refuses broad roots on every platform and no test
  can perform a live reap, every sase-core workspace test passes on macOS, and a required
  macOS CI leg keeps it that way.

  '
phases:
- id: reap-guard
  title: Fix the managed-tmp reap guard and disarm its test
  depends_on: []
  size: small
  description: 'reap-guard: canonicalize the unsafe-root denylist so the guard fires
    on macOS, widen it to the platform temp aliases and $TMPDIR/$HOME, and rewrite
    the guard test so it can never apply a removal.'
- id: check-args
  title: Let the verification gate run a filtered suite
  depends_on:
  - reap-guard
  size: xsmall
  description: 'check-args: forward trailing arguments from scripts/check.sh and the
    justfile through to cargo test so a filtered or skipped run is possible without
    bypassing the documented gate.'
- id: macos-ci-advisory
  title: Add an advisory macOS CI leg
  depends_on:
  - check-args
  size: small
  description: 'macos-ci-advisory: turn rust-checks into an os matrix with a non-blocking
    macos-latest leg so every later phase gets real macOS signal without turning master
    red.'
- id: sudo-identity
  title: Gate the procfs process-identity token to Linux
  depends_on:
  - macos-ci-advisory
  size: medium
  description: 'sudo-identity: split process_identity_token so the procfs body is
    target_os = linux with an explicit unsupported-platform error elsewhere, add a
    config test seam so the detached handshake tests keep running off Linux, and make
    the macOS story for the sase_sudo_runner console script explicit.'
- id: sudo-started-path
  title: Fix the detached handoff started-path mismatch
  depends_on:
  - sudo-identity
  size: medium
  description: 'sudo-started-path: stop comparing a caller-supplied --started-path
    against one derived from a canonicalized --detach-dir, and correct the remaining
    sudo_runner cwd and sudo-order path expectations.'
- id: gateway-attachments
  title: Decide how attachment validation treats symlinked ancestors
  depends_on:
  - macos-ci-advisory
  size: medium
  description: 'gateway-attachments: adjudicate whether a platform symlink ancestor
    should make a notification attachment undownloadable, implement the decision in
    validate_attachment_path, and fix the five route tests.'
- id: core-path-expectations
  title: Reconcile canonicalized paths across sase_core and the bindings
  depends_on:
  - macos-ci-advisory
  size: medium
  description: 'core-path-expectations: fix the five remaining sase_core and sase_core_py
    failures caused by comparing canonicalized paths against caller-supplied ones,
    deciding per site whether production or the expectation is wrong.'
- id: lsp-uri
  title: Make LSP definition URIs agree with their expectations
  depends_on:
  - macos-ci-advisory
  size: small
  description: 'lsp-uri: resolve the canonical-versus-supplied path disagreement behind
    the two sase_xprompt_lsp definition failures.'
- id: macos-ci-required
  title: Make the macOS leg required and document the loop
  depends_on:
  - sudo-started-path
  - gateway-attachments
  - core-path-expectations
  - lsp-uri
  size: small
  description: 'macos-ci-required: drop the advisory escape hatch so the macOS leg
    blocks, and document the macOS verification loop and the platform-path rule for
    future contributors.'
proposed_by: bbugyi200.athena.0oj
create_time: 2026-09-21 06:25:40
status: done
bead_id: sase-157
---

- **PROMPT:** [prompts/202609/macos_portability.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/macos_portability.md)
- **BEAD:** [sase-157](https://github.com/sase-org/sase--beads/blob/main/pages/sase-157/README.md)

# Plan: Make sase-core correct and green on macOS

## Repository

All work in this epic happens in the **`sase-core`** repository, not in the `sase`
primary checkout. Open it with the `/sase_repo` skill before reading or writing
anything, and use only the path that skill prints:

```bash
sase repo open sase-core -r "<phase-specific reason>"
```

`sase-core`'s `AGENTS.md` is binding: release-plz owns all versions (never hand-edit
`[workspace.package].version`, crate `[package].version`, or path-dependency pins), and
`just check` / `./scripts/check.sh` is the only verification entry point — never
`cargo test -p sase_core` alone, because that skips the `sase_core_py` binding tests.

## Why this epic exists

The consolidated maintainability audit
(`research:202609/sase_core_maintainability_audit/sase_core_maintainability_audit.md`)
raised two findings this epic addresses:

- **§3.1 CRITICAL** — the `/tmp` reap guard is inert on macOS, and a unit test performs
  a live, destructive reap.
- **§3.2 HIGH** — CI is Linux-only, so tests that fail on the platform the project is
  actually developed on are invisible.

### The audit's numbers are stale; these are the measured ones

The audit measured `9a5c568` and reported **22** failures (19 `sudo_runner`, 2
`sase_xprompt_lsp`, 1 `managed_tmp`). A full workspace run on macOS 26.5 / Apple Silicon
at `03036af`
(`cargo test --workspace --no-fail-fast -- --skip broad_cleanup_roots_are_rejected`,
`PYO3_PYTHON` pointed at a 3.12 interpreter) measures **32** failures, distributed
differently:

| Target                                  | Failing | Notes                                                                      |
| --------------------------------------- | ------- | -------------------------------------------------------------------------- |
| `sase_core --lib`                       | 4       | plus `broad_cleanup_roots_are_rejected`, skipped because it is destructive |
| `sase_core_py --lib`                    | 1       | audit reported none here                                                   |
| `sase_gateway --lib`                    | 25      | 20 `sudo_runner` + **5 `routes`** the audit never saw                      |
| `sase_xprompt_lsp --lib`                | 1       | audit said 2, and misattributed the cause                                  |
| `sase_xprompt_lsp --test jsonrpc_stdio` | 1       | audit reported none here                                                   |

Two audit details are wrong at `03036af` and should not be carried into implementation.
The LSP failures are **not** `Option::unwrap()` on `None` at `server.rs:5774` / `:5861`;
there is one `--lib` failure (`definition_uses_definition_path_outside_workspace_root`,
`server.rs:5981`) and one integration failure (`jsonrpc_stdio.rs:479`), both
path-canonicalization disagreements. And `sudo_runner` is not a single procfs cluster —
it splits into two distinct causes, below.

### Root cause A — canonical versus caller-supplied paths (22 of 32)

On macOS `/tmp` resolves to `/private/tmp` and `/var` to `/private/var`. Code that
canonicalizes one side of a comparison and not the other therefore behaves differently
on macOS than on Linux. This single asymmetry explains every failure except the procfs
cluster, and it is the same defect §3.1 describes — the audit just did not notice how
far it reaches.

Note that the temp root varies with how the shell was started: a login shell on the
verification Mac exports `TMPDIR=/tmp/bb`, while a non-login shell gets
`/var/folders/...`. Both have a symlinked ancestor, so both reproduce.

### Root cause B — procfs behind `#[cfg(unix)]` (10 of 32)

`crates/sase_gateway/src/sudo_runner.rs:1633` gates `process_identity_token` with
`#[cfg(unix)]` but the body reads `/proc/sys/kernel/random/boot_id` and
`/proc/{pid}/stat`. macOS is `unix` and has no `/proc`, so the call fails with
`failed to read Linux boot id: No such file or directory (os error 2)`.

Calibrate this honestly before rewriting anything. Production already degrades
correctly: `platform_process_identity_available()` (`:420`) is
`#[cfg(target_os = "linux")]` and returns `false` elsewhere, so
`detached_execution_supported()` is false on macOS and detached execution reports
"detached sudo execution is unsupported on this platform" rather than misbehaving. The
failures come from tests that set `config.detached_execution = Some(true)`, forcing the
path on a platform with no identity backend. So this is a **gating and diagnosability**
defect, not a live production break — do not scope it as "implement macOS detached
sudo".

## Verification

### macOS is reachable; use it as the fast loop

A macOS machine is on the tailnet as SSH alias `mac` (see the `tailnet` reference
memory). It is **best-effort — offline unless powered on with the lid open**, so expect
connection timeouts and fall back to the CI leg when it is unreachable. It has a clean
`sase-core` checkout at `~/projects/github/sase-org/sase-core` and Rust 1.95.0.

Two things about that host are required knowledge:

- Its non-login `PATH` only has Python 3.9, which makes `pyo3-build-config` hard-error
  on the `abi3-py312` feature. A login shell (`zsh -lc`) exposes
  `/opt/homebrew/bin/python3.12`. `scripts/check.sh` resolves this itself; a raw `cargo`
  invocation needs `PYO3_PYTHON` exported.
- **Do not run the workspace suite there until `reap-guard` has landed or is applied to
  the tree**, or use `--skip broad_cleanup_roots_are_rejected`. With `TMPDIR=/tmp/bb`,
  an unguarded run reaps up to 2,000 entries under `/private/tmp` — including that
  machine's own temp root.

To verify uncommitted work on `mac` without creating commits (agents never commit — see
the `host-owned-completion` decision), ship a patch and restore afterwards:

```bash
git -C "$SASE_CORE" diff HEAD > /tmp/macos-verify.diff
scp /tmp/macos-verify.diff mac:/tmp/
ssh mac 'zsh -lc "cd ~/projects/github/sase-org/sase-core \
  && git fetch origin && git checkout -- . && git checkout master \
  && git apply /tmp/macos-verify.diff \
  && ./scripts/check.sh test"'
# Always restore that checkout when finished:
ssh mac 'zsh -lc "cd ~/projects/github/sase-org/sase-core && git checkout -- ."'
```

That checkout was clean at `master` when this plan was written. Leave it clean.

### Every phase still runs the documented gate

Run `just check` (equivalently `./scripts/check.sh`) in the `sase-core` checkout before
finishing any phase. macOS verification is **in addition to**, never instead of, the
Linux gate — several phases change shared path logic and could regress Linux.

## Shared rule this epic establishes

Every phase that touches a path comparison follows one rule, and states in its work
which branch it took:

> **Canonicalize both sides of a path comparison, or neither.** When production must
> canonicalize (for a security or identity check), the value it compares against —
> denylist entry, expected argument, allowed root, asserted expectation — must be
> canonicalized the same way. When production must preserve a caller-supplied path
> (because it is echoed back to the caller), it must not silently compare it against a
> canonicalized one.

A corollary that matters for the security-shaped sites: **a symlinked _ancestor_ is not
by itself evidence of an attack** on macOS, where `/tmp` and `/var` are platform
aliases. A check that rejects on "any ancestor is a symlink" is not equivalent to one
that resolves the path and confirms it stays inside an allowed root. Phases
`gateway-attachments` and `core-path-expectations` must decide this deliberately rather
than making the test match whatever the code currently does.

---

## Phase `reap-guard` — Fix the managed-tmp reap guard and disarm its test

**File:** `crates/sase_core/src/managed_tmp.rs`

### The defect

`validate_reap_root` (`:899`) canonicalizes the requested root and then compares it
against a denylist that is not canonicalized:

```rust
let resolved = root.canonicalize().unwrap_or_else(|_| root.to_path_buf());
...
for unsafe_root in [Path::new("/"), Path::new("/tmp"), Path::new("/var/tmp")] {
    if resolved == unsafe_root { return Err(ManagedTmpReapError::UnsafeRoot(...)); }
}
```

On macOS `resolved` for `/tmp` is `/private/tmp`, which matches no entry, so no denylist
entry can ever fire. The secondary `resolved == cwd || cwd.starts_with(&resolved)` guard
does not save it either, because a developer's cwd is not under `/private/tmp`. The
module doc promises the function "refuses broad roots"; on macOS it refuses nothing.

`reap_managed_tmpdir` is exported to Python with a caller-supplied `root`, and
`validate_reap_root` is its only defense — so this is an unguarded public API, not just
a test hazard. Realized harm today is bounded: all three production callers in the
`sase` repo pass `managed_tmpdir_root()`, which resolves to `~/.sase/tmp` (or
`$SASE_TMPDIR`), a dedicated directory.

### The work

1. Canonicalize each denylist entry before comparing, so the comparison is symmetric on
   every platform. Entries that do not exist must be skipped, not cause a failure.
2. Widen the denylist beyond the three current entries to cover the macOS aliases and
   the obvious catastrophic roots: `/private/tmp`, `/private/var/tmp`, the value of
   `TMPDIR`, and the user's home directory. Canonicalizing `/tmp` already covers
   `/private/tmp` on macOS, but list them explicitly so the guard does not depend on
   canonicalization succeeding.
3. Keep the existing `cwd` guard and its behavior unchanged.
4. Confirm the widening does not break the `sase` repo's callers.
   `managed_tmpdir_root()` defaults to `~/.sase/tmp`, which is neither `$TMPDIR` nor
   `$HOME`, so the default path is unaffected. A `$SASE_TMPDIR` pointed at `$TMPDIR` or
   `$HOME` would now be refused — that is the intended new behavior; note it in the
   commit body.

### Disarm the test

`broad_cleanup_roots_are_rejected` (`:1811`) passes `root = "/tmp"` through the shared
`request()` helper, which sets `apply: true`, `max_removals: 2000`, and
`default_horizon_seconds: 3 days`. On Linux the guard fires and nothing happens. On
macOS the guard is skipped and the test **executes a real reap**. A researcher
reproduced this twice; it deleted a live tool-session scratch directory
mid-investigation.

Rewrite it so a future guard regression cannot destroy anything:

- Build the request with `apply: false` so the call is a dry run regardless of whether
  the guard fires.
- Make it table-driven over every denied root, including `/`, `/tmp`, `/var/tmp`,
  `/private/tmp`, `/private/var/tmp`, `$TMPDIR`, and `$HOME`, asserting
  `ManagedTmpReapError::UnsafeRoot` for each.
- Audit the rest of the module's tests for any other case that reaches `request()` with
  a root outside a `tempdir()`. At the time of writing this is the only one, but the
  `apply: true` default makes the helper a standing hazard — consider defaulting it to
  `apply: false` and opting in per test.

### Done when

`just check` passes; the guard test asserts rejection for every denied root with
`apply: false`; and running the `sase_core` tests on `mac` **without** any `--skip`
leaves `/private/tmp` untouched.

---

## Phase `check-args` — Let the verification gate run a filtered suite

**Files:** `scripts/check.sh`, `justfile`

`cmd_test()` runs a bare `cargo test --workspace` and forwards nothing. That makes it
impossible to run a filtered or skipped suite through the documented gate — which is
exactly the fallback `AGENTS.md` warns about ("Never verify with
`cargo test -p sase_core` alone"), because the only way to filter today is to bypass
`check.sh` and lose its `PYO3_PYTHON` resolution.

Forward trailing arguments from `check.sh`'s `test` (and `clippy`) subcommands into the
underlying `cargo` invocation, and forward `just test`'s arguments into `check.sh`. Keep
the no-argument behavior byte-identical to what CI runs today.

This phase is deliberately tiny and lands early because the later phases need a cheap
way to re-run one crate on `mac` without a full workspace run.

### Done when

`./scripts/check.sh test -p sase_gateway` and `./scripts/check.sh test -- --skip foo`
both work, `./scripts/check.sh test` is unchanged, and `just check` passes.

---

## Phase `macos-ci-advisory` — Add an advisory macOS CI leg

**File:** `.github/workflows/ci.yml`

All three jobs are `runs-on: ubuntu-latest` while `crates/sase_core_py/pyproject.toml`
advertises macOS support on PyPI. That is the gap §3.2 names.

Convert `rust-checks` to a matrix over `ubuntu-latest` and `macos-latest`. Make the
macOS leg **non-blocking for now** —
`continue-on-error: ${{ matrix.os == 'macos-latest' }}` — so this phase can land while
31 failures remain, and every subsequent phase gets real macOS CI signal instead of
depending on a laptop that is usually asleep. The `macos-ci-required` phase removes the
escape hatch.

Also required here:

- Give the matrix legs distinct `Swatinem/rust-cache` `shared-key`s so the two platforms
  do not fight over one cache.
- Ensure the macOS runner satisfies `abi3-py312`. GitHub's `macos-latest` image ships a
  recent Python 3, but pin it with `actions/setup-python` rather than trusting the
  image, so a runner image update cannot produce the opaque `pyo3-build-config` error.
- Leave `release-scripts` and `wheel-smoke` on `ubuntu-latest`. Extending `wheel-smoke`
  to macOS is a reasonable follow-up but is not in this epic's scope.

macOS runner minutes bill at a higher rate than Linux. If that becomes a concern, the
`ci-two-speed-split` decision's pattern — an exhaustive matrix off the push path — is
the precedent to follow; note it in the workflow as a comment rather than implementing
it now.

### Done when

A push runs both matrix legs, the Linux leg is green and required, the macOS leg runs
and reports its failures without failing the workflow.

---

## Phase `sudo-identity` — Gate the procfs process-identity token to Linux

**File:** `crates/sase_gateway/src/sudo_runner.rs`

### The defect

`process_identity_token` (`:1633`) is `#[cfg(unix)]` but its body is procfs-only. Ten
tests fail with `failed to read Linux boot id: No such file or directory (os error 2)`,
most of them at the `:3186` test-helper call site. Read the calibration in "Root cause
B" above before starting: production already refuses detached execution off Linux via
`platform_process_identity_available()`, so the user-visible defect is a misleading
error message and a `#[cfg]` that claims more portability than the body delivers.

### The work

1. Split the function: keep the procfs body under `#[cfg(target_os = "linux")]`, and add
   a non-Linux `unix` arm that returns a `RunnerError` naming the real reason —
   something like "process identity tokens require Linux procfs; detached sudo execution
   is unavailable on this platform" — instead of a confusing `/proc` read error. Mirror
   the shape `sase_core/src/host_liveness.rs:325` already uses for the same problem.
2. Audit the rest of the crate for the same mis-gate: the audit measured **131
   `#[cfg(unix)]` against 6 `#[cfg(target_os = "linux")]`**. Grep for `/proc/` inside
   `cfg(unix)` blocks. The audit found only `sudo_runner.rs` genuinely mis-gated
   (`fleet_family.rs` and `contract.rs` mention `/proc` as data, not filesystem reads);
   confirm that still holds and say so.
3. Keep the ten tests meaningful rather than deleting coverage. Prefer a config test
   seam: `SudoRunnerConfig` already carries `process_identity_error` for forcing
   failures, so add the symmetric injection — an override that supplies a synthetic
   identity token — and have the platform-agnostic handshake tests use it. Only
   `#[cfg(target_os = "linux")]`-gate assertions that are genuinely about Linux identity
   semantics, the way `linux_identity_backend_advertises_detached_execution` (`:3351`)
   already is.
4. Decide and record the macOS story for the `sase_sudo_runner` console script. It is
   installed by the macOS wheel and `wheel-smoke` only asserts `--help` runs. Either
   document that detached execution is Linux-only and make `--capabilities` say so
   legibly, or stop installing the script on macOS. Silently shipping a console script
   whose main capability is unavailable is the outcome to avoid. Prefer the documenting
   option: `detached_execution_supported()` already returns an empty capability list on
   macOS, so the behavior is right and only the packaging claim needs fixing.

Do **not** implement a macOS identity backend (`sysctl` `kinfo_proc` / `kern.boottime`)
in this phase. If it is still wanted after this lands, it is a separate task — it adds a
platform-specific unsafe FFI surface to a privileged-execution path and deserves its own
review.

### Done when

The ten procfs failures pass on macOS, no coverage was deleted to get there,
`just check` passes on Linux, and the macOS packaging claim matches the behavior.

---

## Phase `sudo-started-path` — Fix the detached handoff started-path mismatch

**File:** `crates/sase_gateway/src/sudo_runner.rs`

Sequenced after `sudo-identity` because both phases edit the same file; there is no
logical dependency.

### The defect

`validate_handoff_paths` (`:662`) canonicalizes `--detach-dir` into `dir` and derives
`HandoffPaths` from it. The internal-root-executor path then compares the
**caller-supplied** `--started-path` against that derived, canonicalized value:

```rust
if started_path != &paths.started_path {
    return Err(cli_error(SudoRunnerExitStatus::InvalidInput, ...));
}
```

The parent builds `--started-path` by joining the _non-canonical_ detach dir, so the two
never match whenever the detach dir has a symlinked ancestor. That is every macOS temp
dir, and any Linux host with a symlinked `TMPDIR`. It surfaces as `InvalidInput` (13)
where `RunnerError` (14) was expected in
`internal_worker_does_not_execute_before_valid_witness` (`:4197`),
`post_spawn_identity_failure_reaps_barred_worker` (`:4291`), and
`post_spawn_publish_failure_reaps_barred_worker` (`:4332`), and as bare `RunnerError` in
the `detached_*` and `*_launcher_*` tests.

This one is a **real production bug**, not a test expectation problem: detached sudo
execution cannot complete its handshake on any host whose handoff directory has a
symlinked ancestor.

### The work

1. Canonicalize the caller-supplied `--started-path` the same way `--detach-dir` is
   canonicalized before comparing — or, equivalently and more simply, compare against
   the non-canonical join. Pick one and apply it consistently; the shared rule above
   requires both sides to agree. Note that `--started-path` may not exist yet, so plain
   `canonicalize()` will fail; canonicalize the parent and rejoin the file name.
2. Preserve the security intent. The check exists so an internal worker cannot be
   pointed at a started-path outside its handoff directory. Whatever form the fix takes,
   add a test that a started-path outside the detach dir is still rejected — on both
   platforms.
3. Fix the two remaining path expectations in this file, which are test-side:
   `successful_run_uses_expected_sudo_order_and_cleared_environment` (`:3486`,
   `/private/tmp` vs `/tmp`) and
   `command_runs_in_distinct_cwd_with_spaces_and_parent_cwd_is_unchanged` (`:3574`).
   Before changing an assertion, confirm the manifest `cwd` round-trip really is
   supposed to canonicalize — if a caller's `cwd` is echoed back canonicalized, decide
   whether that is the contract and say so.
4. Diagnose `cwd_removed_after_authentication_fails_before_dispatch_and_cleans_up`
   (`:3899`, `unwrap_err()` on an `Ok` value) on its own terms. It is in the same
   cluster but its failure shape differs, so it may be a distinct cause.

### Done when

All 20 `sudo_runner` failures pass on macOS, the out-of-handoff-dir rejection is covered
by a test that runs on both platforms, and `just check` passes on Linux.

---

## Phase `gateway-attachments` — Decide how attachment validation treats symlinked ancestors

**File:** `crates/sase_gateway/src/routes.rs`

### The defect

`validate_attachment_path` (`:3836`) rejects a path when
`contains_symlink_component(path)` (`:3866`) is true — and that helper walks **every**
ancestor, so a platform alias like `/var` or `/tmp` trips it. On macOS every file under
the system temp directory is therefore reported `downloadable: false`, which is what
fails `notification_detail_returns_notes_action_and_attachments` (`:9384`) and makes the
other four route tests unwrap a `None` token (`:9414`, `:9453`, `:9489`, `:9528`).

### The decision to make

The contract snapshot at `contract.rs:410` says `downloadable` is "false for missing,
oversized, symlinked, traversal, directory, or unknown-risk files". Decide explicitly
which of these two is the intended contract, and write the decision into the code as a
comment:

- **(a)** Any symlinked ancestor is disqualifying, macOS aliases included. Then macOS
  behavior is correct as-is and the five tests must build their fixtures somewhere
  without a symlinked ancestor — but be honest that this makes every macOS temp-dir
  attachment permanently undownloadable.
- **(b)** The real intent is "the resolved path must not escape an allowed root, and a
  symlink must not be used to redirect the final target". Then
  `contains_symlink_component` is too blunt, and the check should canonicalize and
  verify containment instead.

Weigh it against realized impact rather than tidiness: notification attachments in
practice live under `~/.sase/...`, which has no symlinked ancestor on macOS, so today's
production exposure is small either way. Option (b) is the more defensible contract, but
it relaxes a security check — if you take it, keep the final-component symlink rejection
and the `ParentDir` traversal rejection intact, and add a test that a symlink pointing
_out_ of the allowed root is still refused.

Whichever branch you take, update `contract.rs:410`'s wording to match. That snapshot is
hand-maintained with no drift test (audit §3.6c), so it will not be corrected for you.

### Done when

The five `routes` failures pass on macOS, the chosen contract is stated in a code
comment and reflected in `contract.rs`, escape-via-symlink is still covered by a test,
and `just check` passes on Linux.

---

## Phase `core-path-expectations` — Reconcile canonicalized paths across sase_core and the bindings

**Files:** `crates/sase_core/src/agent_artifact_run_retention.rs`,
`crates/sase_core/src/bead/cli.rs`, `crates/sase_core/src/git_object_sharing.rs`,
`crates/sase_core_py/src/artifact_refs/`

Five failures, same root cause A, but each needs its own adjudication — decide per site
whether production or the expectation is wrong, and say which in the commit body.

1. **`agent_artifact_run_retention::rejects_symlinked_ancestor_before_canonicalizing_candidate`**
   (`:902`) — reasons are `["invalid_run_path"]` where `["symlink_ancestor"]` is
   expected. The whole temp root has a symlinked ancestor on macOS, so the candidate is
   rejected by an earlier branch than the one under test. The test's _intent_ — that a
   symlinked project alias is detected and the run protected — is still valid; the
   fixture just cannot distinguish the two branches on macOS. Decide whether
   `invalid_run_path` is even correct here, because if it fires for every candidate
   under a symlinked root, macOS artifact-run retention silently protects everything and
   reclaims nothing.
2. **`bead::cli::create_and_remove_are_handled_with_mutation_summaries`** (`:3377`) and
   **`bead::cli::create_plan_path_is_relative_to_store_workspace_from_nested_cwd`**
   (`:3802`) — the stored `design` is the absolute `/private/tmp/.../sdd/plan.md`
   instead of the relative `sdd/plan.md`. Plan-path relativization against the store
   workspace fails when the two sides differ by canonicalization. This is a
   **production** bug: a bead created on macOS records an absolute, machine-specific
   design path in a store that is meant to be portable. Fix the relativization, not the
   assertion.
3. **`git_object_sharing::relative_alternates_resolve_from_object_database`** (`:656`) —
   `plan.status` is `"unexpected"` where `"expected"` is wanted. The test resolves
   `../../../primary/.git/objects` relative to the object database; canonicalization of
   the temp root breaks the comparison against the expected primary path.
4. **`artifact_refs::tests::refs::artifact_ref_bindings_round_trip_json_shapes`**
   (`crates/sase_core_py/src/artifact_refs/tests/refs.rs:157`) — a round-trip returns
   `/private/tmp/.../files/notes.md` for an input of `/tmp/.../files/notes.md`. Decide
   whether the binding is contracted to echo the caller's path or to return a resolved
   one. A round-trip that does not round-trip is the wrong default; prefer preserving
   the caller's path unless there is a stated reason to resolve.

Where the fix is a shared one, prefer adding a small test helper that yields a
canonicalized temp directory over sprinkling `canonicalize()` through individual tests.

### Done when

All five pass on macOS, each site's production-versus-expectation call is recorded, and
`just check` passes on Linux.

---

## Phase `lsp-uri` — Make LSP definition URIs agree with their expectations

**Files:** `crates/sase_xprompt_lsp/src/server.rs`,
`crates/sase_xprompt_lsp/tests/jsonrpc_stdio.rs`

Two failures, one cause:

- `server::tests::definition_uses_definition_path_outside_workspace_root`
  (`server.rs:5981`) — the returned URI path is
  `/private/var/folders/.../outside-workspace.md`; the expectation builds
  `/var/folders/.../outside-workspace.md`.
- `stdio_jsonrpc_initialize_and_completion` (`jsonrpc_stdio.rs:479`) — `saw_definition`
  is false for the same reason: the server's `result["uri"]` and the test's
  `Uri::from_file_path(&definition_path)` disagree.

Ignore the audit's claim that these are `Option::unwrap()` on `None` — that was true at
`9a5c568` and is not the failure at `03036af`.

Decide where the resolution happens. `tower_lsp_server`'s `UriExt::from_file_path`
canonicalizes only relative paths, so an absolute path passes through verbatim; the
canonicalization is happening in the server's own definition resolution. An LSP client
matches URIs by string, so a server that returns a resolved URI for a path the client
supplied unresolved can break go-to-definition on real editors. Prefer making the server
consistent — resolve on both sides or neither — over patching the two assertions.

### Done when

Both failures pass on macOS, the client-visible URI form is stated in a comment near the
definition handler, and `just check` passes on Linux.

---

## Phase `macos-ci-required` — Make the macOS leg required and document the loop

**Files:** `.github/workflows/ci.yml`, `AGENTS.md`, `README.md`

1. Remove the `continue-on-error` escape hatch added in `macos-ci-advisory` so the macOS
   leg blocks. Before doing so, confirm a full green macOS run on the current `master` —
   `AGENTS.md` notes that `master` is unprotected and a red commit there fails every
   `Release-plz` run until fixed, so this flip must not be speculative.
2. Add a short "Platform paths" section to `AGENTS.md` stating the shared rule this epic
   establishes (canonicalize both sides of a path comparison, or neither; a symlinked
   ancestor is not by itself an attack indicator), and note that macOS resolves `/tmp`
   to `/private/tmp` and `/var` to `/private/var`.
3. Document the macOS verification loop in `AGENTS.md`: CI covers it now, and
   `./scripts/check.sh` handles `PYO3_PYTHON` resolution on macOS just as it does on
   Linux.
4. Fix only the `README.md` "Build & test" instructions, which tell the reader to run
   raw `cargo test --workspace`, `cargo clippy --workspace --all-targets`, and
   `cargo test -p sase_gateway push_subscription` — precisely the habit `check.sh`'s own
   header and `AGENTS.md` forbid, and the one that let three stale schema-version
   fixtures reach master in `a509dcc`. Point it at `just check` / `./scripts/check.sh`.

   The rest of audit §3.7's documentation debt — the stale `sase_100` references, the
   missing `sase_xprompt_lsp` entry in the Layout block, the obsolete phase status, the
   `SASE_CORE_BACKEND` claims that contradict the `rust-core-required` decision, and the
   `PYPI_README.md` text rendered on pypi.org — is **out of scope**. Leave it for a
   separate task; do not let this phase grow into a docs rewrite.

### Done when

CI is green on both platforms with the macOS leg required, `AGENTS.md` carries the
platform-path rule and the macOS loop, and `README.md` no longer teaches raw `cargo`
verification.

---

## Out of scope

Audit findings §3.3 through §3.11 — the `cargo check` inner loop, the hub-file split,
the 2,672-symbol root prelude, the drifted manifests and missing `.pyi` stubs, the
remaining documentation contradictions, the false MSRV, the duplicate HTTP stack, the
missing lint ceiling, and the panic-shaped FFI/LSP error handling — are all real and all
separate. This epic does one thing: make the guard correct and the suite honest on
macOS.
