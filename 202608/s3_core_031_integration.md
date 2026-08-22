---
tier: tale
title: Integrate the post-phase sase-core 0.31.0 release
goal:
  The newest published schema-4 core is supported and its release provenance is
  truthful.
size: medium
proposed_by: bbugyi200.athena.sase-s3.land
bead: sase-s3
create_time: 2026-08-22 16:14:22
status: wip
---

- **PARENT:** [202608/0ak_failure_recovery.md](0ak_failure_recovery.md)
- **BEAD:**
  [sase-s3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-s3/README.md)

# Integrate the post-phase sase-core 0.31.0 release

## Goal

Finish the one remaining `sase-s3` integration item: make the published `sase-core-rs`
dependency window include the newest schema-4 release and correct the current
`sase-core` changelog so it does not claim that the same runtime feature landed
independently in both 0.30.0 and 0.31.0.

## Confirmed drift

Phase `sase-s3.1` landed the schema-4 monitor-cleanup implementation in `sase-core`
commit `c7447f0` and published it as 0.30.0. After that phase closed, a second
`sase-s3.1` stitch (`7e1d09b`) added only a duplicate changelog entry and the normal
release workflow published 0.31.0. The diff from v0.30.0 to v0.31.0 contains only
Cargo/package versions and changelog metadata; the runtime source is identical. PyPI
does contain complete 0.31.0 wheels and an sdist.

The later main-repository phase commit `959d55926` nevertheless declared
`sase-core-rs>=0.30.0,<0.31.0`, excluding the newest published schema-4 release. This is
post-phase drift caused by the epic and must be reconciled before the parent epic lands.

## Implementation

1. Open the linked `sase-core` repository through `/sase_repo`. Update only the current
   changelog to describe 0.31.0 truthfully as a metadata/version follow-up whose runtime
   code is unchanged from 0.30.0. Preserve both published tags and shared history; do
   not rewrite, delete, or republish either release.
2. In the main SASE repository, use `tools/ratchet_core_window` to advance the declared
   and locked `sase-core-rs` window to the newest complete PyPI release. The expected
   result is `>=0.31.0,<0.32.0`; refresh `uv.lock` through the tool rather than editing
   lock metadata by hand. Inspect the generated diff and reject any unrelated package
   churn.
3. Confirm the installed binding reports agent-cleanup wire schema 4 and that the Rust
   cleanup planner remains the selected path for direct and cascaded live-monitor
   cleanup. No Python/TUI behavior change is intended.

## Verification

- In `sase-core`, run the repository-required install/check workflow and focused
  agent-cleanup planner, wire, PyO3, and Python parity tests needed to prove that the
  changelog-only correction did not disturb the released contract.
- In SASE, run `just install`, verify `tools/ratchet_core_window --check` reports no
  pending ratchet, run `tools/probe_core_floor --json`, and run the focused cleanup
  facade/parity, monitor-stop persistence, named-kill, TUI live-monitor-kill, and
  owner-cleanup tests used by `sase-s3.2`.
- Run `just check`. If scoped verification escalates or the packaging change is treated
  as a broadening trigger, run the required exhaustive check through `/sase_monitor` and
  diagnose any failure against the current tree.

## Non-goals

- Do not rewrite published git history or replace the 0.30.0/0.31.0 release artifacts.
- Do not change monitor-cleanup semantics, weaken schema checks, or remove the Python
  compatibility fallback.
- Do not close `sase-s3`, retire its Symvision entries, or update the parent epic plan's
  status in this tale; those remain the resumed parent land agent's closeout duties.
