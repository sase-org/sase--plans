---
tier: tale
title: Restore sase-core CI after plugin input-type registry changes
goal:
  Fix the confirmed Rust formatting failures and pass the full required sase-core
  verification gate.
size: small
proposed_by: bbugyi200.athena.0x7
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0x7](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.0x7.md)
- **COMMITS:**
  - [e08f0c2](https://github.com/sase-org/sase-core/commit/e08f0c2b8b3102231bae5115756d3782580a2e53)
    — fix(ci): format plugin input type registry changes

# Restore sase-core CI after the plugin input-type registry change

## Objective and scope

Fix the current GitHub Actions failures in `sase-org/sase-core` and verify the complete
required local gate. This is a small, single-agent implementation: the confirmed code
defect is rustfmt drift in seven files. Implementation belongs in the linked `sase-core`
repository. This plan was prepared without changing tracked files in either repository.

Start from the assigned SASE workspace, open the repository with
`sase repo open sase-core -r 'Implement the approved sase-core CI formatting repair'`,
and use only the printed path. Read that checkout's `AGENTS.md` before editing. All
paths below are relative to that checkout. Recheck its status and HEAD so concurrent
work is preserved and already-fixed problems are not reintroduced.

## Diagnosis and evidence

Investigation on 2026-10-06 used `actstat --repo sase-org/sase-core -n 5 --format json`,
GitHub job logs and check annotations, and the repository's own read-only formatting
check.

- The failing HEAD is `af5df614a70a02995a1dc0b9c05337b719145289`,
  `feat(core): load plugin input_type registries with resolution, catalog, and LSP wiring`.
  Its parent `c57b662c7e95794520fdb025acd4cb76c90ebc90` passed CI.
- [CI run 37363781183](https://github.com/sase-org/sase-core/actions/runs/37363781183)
  failed on both Ubuntu and macOS. The failed step in both jobs was
  `cargo fmt --all -- --check`, executed as `./scripts/check.sh fmt-check`.
  [Ubuntu job](https://github.com/sase-org/sase-core/actions/runs/37363781183/job/111944171338)
  and
  [macOS job](https://github.com/sase-org/sase-core/actions/runs/37363781183/job/111944171319)
  both exit 1 after rustfmt reports the same seven files listed below.
- Running `./scripts/check.sh fmt-check` on the clean linked checkout also exits 1 with
  those formatting diffs. They are line wrapping and layout changes in the code added or
  modified by `af5df614`. `rustfmt.toml` sets `max_width = 80`. `rust-toolchain.toml`
  selects `stable`; local Rust was 1.98.1 and CI was 1.99.0. Both reject the checked-in
  formatting, so changing the toolchain pin is not required to address the observed
  defect.
- The dependency-feature, clippy, and Rust test steps were skipped after formatting
  failed. Their status on this commit is therefore unknown. The wheel build/import smoke
  job passed.
- The separate
  [release workflow scripts job](https://github.com/sase-org/sase-core/actions/runs/37363781183/job/111944171077)
  was cancelled without executing any steps. Its GitHub check annotation says: "The job
  was not acquired by Runner of type hosted even after multiple attempts". This is a
  hosted-runner allocation failure, not evidence of a script defect.
- The Release-plz push run and subsequent scheduled runs on this same SHA succeeded. No
  release/version recovery is indicated.

## Implementation

1. Refresh `actstat --repo sase-org/sase-core -n 3 --format json` and inspect current
   HEAD/status. If the repository has advanced, reproduce the current formatting result
   before applying the diagnosed repair.
2. Run `just fmt` from the opened checkout. Review the resulting diff. On the diagnosed
   SHA, rustfmt should modify only these seven files:
   - `crates/sase_core/src/editor/frontmatter.rs`
   - `crates/sase_core/src/macro_catalog/loader.rs`
   - `crates/sase_core/src/macro_input_types/registry.rs`
   - `crates/sase_core/src/macro_input_types/tests.rs`
   - `crates/sase_core_py/src/editor_completion/mod.rs`
   - `crates/sase_core_py/src/macro_input_types/mod.rs`
   - `crates/sase_macro_lsp/src/server/completion_items.rs`
3. Keep the repair mechanical: preserve registry loading, diagnostics, catalog
   resolution, Python bindings, and LSP behavior. Inspect any additional file changes
   before including them. Preserve unrelated existing changes. No workflow redesign,
   toolchain migration, version/changelog edit, Python consumer change, or
   `sase-core-revision.txt` update is needed for formatting.
4. Validate with the complete local gate described below. If a previously skipped stage
   exposes a reproducible defect introduced by the same commit, diagnose and repair that
   concrete failure, retaining assertions and adding focused regression coverage only if
   behavior changes. Re-run the complete gate after any further repair. Do not suppress
   checks to manufacture a pass.

## Verification

- Run `./scripts/check.sh fmt-check` after formatting; require exit 0.
- Run `git diff --check` and review `git diff --stat` plus the full diff to confirm the
  formatting repair has no behavioral changes.
- From the opened sase-core checkout, run `sase tool run check`, which is the required
  guarded `just check`. It covers formatting, dependency features, clippy with warnings
  denied, all workspace tests including Python binding tests, and release-workflow
  script tests. Do not substitute targeted tests for this gate or invoke bare `cargo`.
- Use the `/sase_monitor` instructions for long verification. Start the named tool
  inline as required by that skill; if it escalates or refuses on duration, follow its
  printed monitor command with explicit continuation to inspect the result, repair
  failures, and finish. Join an existing run instead of starting it again, and allow at
  least ten minutes for a full gate. Do not end a turn with an unmonitored verification
  command still running.
- Formatting alone needs no new tests. The full gate supplies the evidence for
  downstream stages that the failed CI run never reached, and includes the script tests
  GitHub could not schedule.

## Completion criteria and delivery

The repair is ready when the intended changes are reviewed, `fmt-check` and
`git diff --check` pass, and the full sase-core `check` gate exits successfully on the
final tree. Record the verification result and explain the independent hosted-runner
cancellation in the user-facing report.

Use `/sase_final` to declare the opened repository's changes for host-owned completion,
with a Conventional Commit message such as
`fix(ci): format plugin input type registry changes`. Do not manually create commits,
branches, releases, or pushes. After host publication, the new CI run must pass both OS
jobs, the release-workflow script job, and the wheel smoke job before reporting GitHub
Actions as green. Local success does not establish hosted success. If a hosted job is
again cancelled for runner allocation, inspect its annotations and retry the affected
workflow/job rather than changing application code without evidence. Use a SASE monitor
for any CI wait.
