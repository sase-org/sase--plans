---
tier: tale
title:
  "Finish sase-core P0 (sase-165): full-crate extension freshness, ratchet behavior
  test, guide accuracy"
goal:
  An uncommitted edit anywhere under the linked sase-core crates makes `just check`
  rebuild the dev extension, the core-pin-ratchet step script is proved by executing it
  against stubs rather than by matching strings, and the sase-core guide names the
  features gate that `just check` really runs.
size: small
proposed_by: bbugyi200.athena.sase-165.land
bead: sase-165
create_time: 2026-09-22 12:44:26
status: wip
---

- **PARENT:**
  [202609/sase_core_p0_agent_maintainability.md](https://github.com/sase-org/sase--plans/blob/main/202609/sase_core_p0_agent_maintainability.md)
- **BEAD:** sase-165

# Finish sase-core P0 (sase-165): full-crate extension freshness, ratchet behavior test, guide accuracy

## Context

The land agent for epic `sase-165` (sase-core P0: fast loop, instruction delivery,
cross-repo truth) verified all seven phases. It found three gaps that the epic caused
and that must close before the epic can land. Everything else is verified. The host-side
landing steps (skill deploy, chezmoi apply, pin-ratchet run) are done or left to the
user. Read the epic's plan with `sase bead read sase-165 -r "<why>"`; its latest
land-progress note has the full verification record.

Repos: **sase** is the primary checkout. **sase-core** is a linked repo: open it with
`sase repo open sase-core -r "<why>"`, work only in the printed path, and read its
`AGENTS.md` before editing.

## Gap 1: the dev-extension freshness identity ignores most of the core (sase)

Phase `sase-165.5` added `tools/_sase_core_source_identity.py`. Its identity is
`git rev-parse HEAD` plus a digest of uncommitted and untracked changes under
`INPUT_PATHS = ("Cargo.toml", "Cargo.lock", "crates/sase_core_py")`. That list was
copied from the wheel cache key. The wheel cache only accepts clean checkouts and keys
on the commit, so the narrow list is harmless there. For the freshness stamp it is not:

- An uncommitted edit under `crates/sase_core/` does not change the identity. This was
  reproduced by appending a comment to `crates/sase_core/src/host_liveness.rs` in the
  linked checkout. `crates/sase_core/` is where most core edits happen.
- As a result, `just check` in sase still runs against a stale `sase_core_rs` after a
  behavior-only core edit, which is the exact failure the epic set out to remove (plan
  decision 5, KPI "Dev extension after a Rust edit → rebuilt by `_setup`").
- The same blind spot covers `crates/sase_workspace_hack` (added by sase-165.1), which
  `sase_core_py` depends on. It also covers `crates/sase_gateway` and
  `crates/sase_xprompt_lsp`, whose binaries `rust-install` builds and ships.

Fix:

1. Widen the shared list to the whole Rust build input:
   `INPUT_PATHS = ("Cargo.toml", "Cargo.lock", "rust-toolchain.toml", "crates")`.
   - Keep it as the one shared constant that `tools/sase_core_wheel_cache` imports.
   - The wheel cache's `_crate_input_digest` guard (some path must start with
     `crates/sase_core_py/`) still holds.
   - The cache key changes once, so each host takes one extra release build. Say so in
     the commit message.
   - Update the module docstring and the `INPUT_PATHS` comment so they describe the
     whole-crate scope and no longer describe the list as the wheel cache's.
2. Keep the identity cheap: git plumbing only, `target/` never hashed (it is gitignored,
   so `git status` never lists it). On the real linked checkout, confirm that
   `compute_identity` stays well under a second. The land agent measured the widened
   `git status`/`git diff` pair at about 20 ms.
3. Tests in `tests/test_validate_test_environment_tool.py`:
   - Extend `_git_core_repo` with a committed `crates/sase_core/src/lib.rs`.
   - Add `test_dirty_edit_under_sase_core_crate_sets_core_source_stale_bit`. It must
     fail before the fix and pass after it.
   - Add a test that an untracked file in a new file under `crates/sase_core/` also sets
     the bit.
   - Keep the existing five freshness tests passing.
   - Run `tests/test_sase_core_wheel_cache_tool.py` too.
4. In `docs/rust_backend.md`, change "HEAD plus local edits under the Rust crate inputs"
   to name the real scope: HEAD plus uncommitted edits under `crates/`, `Cargo.toml`,
   `Cargo.lock` and `rust-toolchain.toml`.

## Gap 2: the core-pin-ratchet fix has no behavior test (sase)

Step 3 of phase `pin-ratchet-bot` required a behavior test. Phase `sase-165.4` landed
`test_core_pin_ratchet_apply_tolerates_exit_two` in
`tests/test_github_actions_ci_master_gate.py`, but that test only checks line order and
substrings. The fix itself works: a manual run opened bot PR #302. A later edit could
still reintroduce an abort that the string test misses.

Add a behavior test in the same file that executes the real step script:

1. Extract the `Propose a core pin bump` step's `run` text with the file's existing
   workflow loader helpers. Replace `${{ github.repository }}` with a fixed
   `owner/repo`.
2. Run it with `bash` in a temp directory that holds a `sase-core-revision.txt`, with a
   stub directory first on `PATH`:
   - `python3`: with `--check`, exit `$CHECK_RC`. Otherwise write a 40-hex SHA into
     `sase-core-revision.txt` and exit `$APPLY_RC`.
   - `git`: log its argv to a file and exit 0.
   - `gh`: log its argv. For `api`, exit `$BRANCH_EXISTS_RC`. For anything else, exit 0.
   - Leave the real `tr` in place, or stub it if the host `tr` is unsuitable. Keep the
     test hermetic and quick.
3. Cover these cases in one parametrized test or a few small ones:
   - A pending bump (check 2, apply 2) exits 0, pushes branch
     `core-pin-ratchet-<first 12 hex>`, and calls `gh pr create` with it.
   - Check exit 0 exits 0 and makes no git or `gh pr create` call.
   - Check exit 3 exits 3 and makes no apply, push or PR call.
   - Apply exit 3 exits 3 with no push or PR.
   - A branch that already exists (`gh api` exits 0) exits 0 with no push.
4. Keep the existing string-assertion tests passing. Skip the test cleanly only if
   `bash` is missing.

## Gap 3: guide and docs accuracy after the fast-loop phase (sase-core + sase)

The epic's KPI for the sase-core guide is "verified claims". Phase `sase-165.1` added a
`features` gate to `scripts/check.sh all` and to CI, but the phase `sase-165.2` text
predates that gate:

1. sase-core `AGENTS.md` ("Build and verify"): the `just check` bullet says it runs
   "fmt-check, clippy with `-D warnings`, every test, and the script tests". Add the
   unified-features gate (`./scripts/check.sh features`) in its real order: fmt-check,
   features, clippy, tests, script-test.
   - Add one clause saying that a features failure is fixed by
     `cargo hakari generate && cargo hakari manage-deps`, which is the message the gate
     prints. The target is still about 97 lines, under the 150-line cap.
2. sase-core `README.md`: the `just check` comment line lists the same steps; add
   `features`.
3. sase `docs/rust_backend.md` (the paragraph beginning "A measured feature-unified
   `cargo build --release -p sase_core_py -p sase_xprompt_lsp --features sase_core_py/extension-module`"):
   add a short parenthetical saying that sase-core has since removed that crate feature
   (1d129cd), and that wheel builds pass `pyo3/extension-module` through maturin's
   `features`. The historical reasoning stays; only the command must not be copied.
4. The sase-core commit is docs-only. Use a Conventional Commit subject such as
   `docs(core): name the features gate in the just check step list`. Do not touch any
   `version` field or `CHANGELOG.md`.

## Verification

- sase: `just check` (per the `lint_and_test` memory; prefer `sase tool run check`).
  First make sure the new freshness tests fail without the `INPUT_PATHS` change.
- sase-core: `just check`, with a tool timeout of at least 10 minutes.
- Pre-existing `just check` failures that this work does not touch are already filed:
  `sase-15z`, `sase-14r`, a `sase-158` DISCOVERED ISSUE, and a `sase-142.5` DISCOVERED
  ISSUE. Do not re-file them.

## Out of scope

- Closing `sase-165`, the Symvision pass, and marking the epic plan done. The `sase-165`
  land agent resumes after this plan lands.
- Host actions left to the user:
  - `sase service init --yes` plus a service restart from an interactive shell.
  - `sase update`, then re-measuring a real agent's first sase-core check.
- The pre-existing test failures listed above.
