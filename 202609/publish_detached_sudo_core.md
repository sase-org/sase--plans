---
tier: epic
title: Publish and consume the portable detached sudo core
goal:
  Make every pinned CI build and published SASE install use a portable sase-core release
  that contains the detached sudo runner, ownership, settlement, and remote handoff
  contracts completed by sase-12w.6.
parent_bead: sase-12w.6
phases:
  - id: core-portability
    title: Restore portable sudo-runner release builds
    depends_on: []
    description:
      "core-portability: fix the epic-introduced macOS initgroups type mismatch without
      weakening account-switch validation, prove the runner on Linux and with an
      Apple-target compile or release-equivalent check, and let the normal sase-core
      release workflow publish the repaired contracts."
    size: medium
  - id: consumer-ratchet
    title: Ratchet SASE onto the published sudo contracts
    depends_on:
      - core-portability
    description:
      "consumer-ratchet: after a complete non-yanked sase-core-rs release containing all
      sase-12w.6 Rust commits is published, advance the CI source pin, dependency floor,
      and lockfile to it and prove exact-floor binding plus focused sudo behavior."
    size: medium
proposed_by: bbugyi200.athena.sase-12w.6.land
create_time: 2026-09-18 19:34:57
status: wip
---

- **PROMPT:**
  [prompts/202609/publish_detached_sudo_core.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/publish_detached_sudo_core.md)
- **PARENT:**
  [202609/sudo_detached_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/sudo_detached_landing_repairs.md)

# Publish and consume the portable detached sudo core

## Why this remains part of `sase-12w.6`

The three implementation phases landed their source changes, but the consumer contract
is not integrated on a clean install:

- `sase-core` commits `9bf272e832`, `9e1ab3fa30`, and `09f543be41` implement runner
  ownership/live output, settlement/liveness policy, and remote handoff metadata. The
  current release commit is `a98913aaed` (`v0.34.57`).
- The SASE commits `7179ec2e23` and `12a37df037` call the new Rust bindings and persist
  the new attempt shape.
- SASE still pins CI to pre-epic core revision `8d5341a4d5` and declares/locks
  `sase-core-rs>=0.34.48,<0.35.0`. That revision and package do not expose
  `sudo_classify_attempt_liveness` or `sudo_authorize_settlement` and do not contain the
  runner and remote-attempt fixes.
- PyPI still exposes 0.34.48 as the newest package. The `v0.34.57` release workflow
  failed while building the macOS universal wheel: in
  `crates/sase_gateway/src/sudo_runner.rs`, the epic's account-switch path passes a
  `gid_t`/`u32` to macOS `libc::initgroups`, whose base-group parameter is `c_int`.
  Consequently no installable core release containing the epic exists yet, and the
  normal core-window ratchet correctly cannot update the lock.

Do not close `sase-12w.6`, run its final Symvision cleanup, or mark its linked plan done
inside this child plan. The parent landing agent owns those close-out steps after this
plan lands.

## Phase `core-portability`: restore portable sudo-runner release builds

Work in the linked `sase-core` repository opened through `sase repo open sase-core`.

1. Repair the `configure_process_group_and_account` / `initgroups` call so each Unix
   target passes the ABI type that its libc declares. Any conversion from `gid_t` to a
   narrower signed base-group type must be checked and must return a useful runner error
   rather than truncate. Preserve the existing order and failure handling for
   `initgroups`, `setgid`, and `setuid`.
2. Add focused coverage for the conversion/error boundary where it can run without
   privilege. Keep target-specific code small and explicit; do not disable the account
   switch or special-case the release workflow.
3. Run formatting, focused `sase_gateway` tests/clippy, and the repository's required
   `just check`. Also compile-check an Apple target (or use an equivalent release-wheel
   build) so the exact ABI mismatch is proven fixed instead of relying only on Linux
   tests.
4. Do not edit release versions manually. After the source commit lands, observe the
   normal release workflow and identify the first complete, non-yanked `sase-core-rs`
   release that contains this fix and all three epic commits. Record that version and
   release commit for the dependent phase.

Acceptance: the Rust runner retains its Linux behavior, compiles for the macOS wheel
target, and the normal release pipeline publishes a complete core package containing the
full `sase-12w.6` Rust contract.

## Phase `consumer-ratchet`: ratchet SASE onto the published sudo contracts

Work in the primary SASE repository. Use the published version and release commit from
the preceding phase; do not assume 0.34.57 if the portability fix requires a later
release.

1. Advance `sase-core-revision.txt` to a release commit that contains the three epic
   commits and the portability fix. Use the existing core-revision ratchet workflow
   where possible, and verify ancestry explicitly if core HEAD has advanced with other
   work.
2. Raise the `pyproject.toml` `sase-core-rs` minimum to the first complete published
   release containing those changes and refresh only the corresponding `uv.lock`
   requirement/package records. The current 0.34.48 release is incomplete, so the normal
   ratchet tool may refuse before selecting a newer floor; if so, make the same narrowly
   scoped floor edit and lock refresh its validation enforces, then run the
   ratchet/check again. Do not leave SASE accepting a core that lacks a binding it
   calls.
3. Prove the exact declared minimum exposes every statically required binding using the
   existing core-floor/binding smoke tooling, not merely the locally built linked core.
   Verify the pinned source revision contains the new liveness, settlement,
   remote-handoff, live-output, and startup-ownership contracts.
4. Run focused sudo core/execution/detach/SSH/acceptance tests and the repository's
   required `just check`. Distinguish any independently pre-existing full-suite issue
   from this integration, but do not waive a floor, pin, binding, packaging, or sudo
   failure.

Acceptance: clean CI and published installations resolve a complete core package with
all bindings and runner behavior required by SASE's landed sudo code, and both the
source pin and published dependency window enforce that contract.
