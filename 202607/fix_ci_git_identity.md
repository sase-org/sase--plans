---
create_time: 2026-07-10 09:34:35
status: done
prompt: 202607/prompts/fix_ci_git_identity.md
tier: tale
---
# Fix CI Git Identity Failures

## Context

The failing CI run reports two Python test failures in `tests/sdd_store/test_materialize.py`. Both fail when
provider-owned SDD materialization tries to create a real Git commit in a newly cloned staging or numbered-workspace
checkout. Repository-local Git configuration is not copied by `git clone`, and GitHub Actions does not provide the
developer identity that allows these tests to pass locally. This leaves the tests dependent on ambient host
configuration even though the production contract documents that users must configure Git identity.

The bead-backend failure in the same historical run came from Rust 1.97 clippy diagnostics in sase-core. The current
sase CI run has already passed its Rust bead checks against the newer sase-core revision, so no sase-side bead-backend
change is needed.

## Plan

1. Make the SDD-store Git integration tests hermetic by supplying a scoped test author and committer identity for
   commits made by production code in cloned temporary repositories. Keep the setup local to the SDD-store test package
   so tests that intentionally exercise missing Git identity remain unaffected.
2. Cover both author and committer identity, and prevent ambient developer signing configuration from affecting
   temporary test commits, while preserving the existing production behavior and documented requirement for configured
   Git identity.
3. Re-run the two failing materialization tests with Git configured to forbid identity auto-detection, proving the tests
   no longer depend on the host. Then run the complete SDD-store test package and the repository-wide `just check`
   validation required for sase changes.
4. Re-check GitHub Actions status after local validation to distinguish the resolved sase-core bead-backend failure from
   the remaining Python test issue and report the evidence and final changed scope.
