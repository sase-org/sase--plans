---
tier: tale
title: Finish the platform-units and Services-tab phases
goal:
  Resolve the remaining service-phase linter ownership, verify both landed
  implementations, and close sase-11y.5 and sase-11y.7 without disturbing the parent
  epic.
size: medium
proposed_by: bbugyi200.athena.0mu
create_time: 2026-09-18 09:38:46
status: wip
---

# Finish and close the platform-units and Services-tab phases

## Outcome

Finish the already-landed work for phase beads `sase-11y.5` and `sase-11y.7`, remove
their temporary Symvision ownership, verify both phases against their approved plans,
and close exactly those two phase beads. Preserve the parent epic's core invariants: one
owner per supervised child, the `service_host` feature flag's legacy Off branch,
Rust-backed service config/state/status semantics, and an `axe` internal tab id until
the sunset phase.

This is a single `medium` tale because both implementation stitches are already on
`master`; one agent can perform the bounded acceptance audit, repair any seam defects,
resolve the linter debt, run the required checks, and close the two phases without
splitting ownership across agents.

## Current evidence

- `sase-11y.5` landed as `3fb42fa11e` (`feat(service): add platform unit integration`).
  Its current epic-symbol audit is empty.
- `sase-11y.7` landed as `c2befdbb3e` (`feat(tui): add services tab controls`). It
  consumed three of its original service status types but still owns 17 Justfile
  exemptions.
- Running Symvision without any epic-symbol exemptions reports exactly those 17 public
  definitions as unused: `CapturedServiceEnvironment`, `NativeInspection`,
  `NativeServiceDefinition`, `ServiceEnablement`, `ServiceEnvironmentError`,
  `ServiceFieldProvenance`, `ServicePlatformApplyResult`, `build_native_definition`,
  `clear_service_enablement`, `compose_service_config`, `inspect_native_service`,
  `read_service_environment`, `readiness_warnings`, `resolve_service_enablement`,
  `service_dir`, `service_platform_supported`, and `service_state_path`.
- The current tree re-keyed platform-unit symbols from `sase-11y.5` to `sase-11y.7`
  after the Services-tab stitch had already landed. That transfer did not create
  consumers and must not be repeated as generic bookkeeping.

## Implementation

1. **Re-establish the acceptance baseline.** Read the parent service-host plan, the two
   phase plans, the consolidated research report, and the current bead state via the
   audited SASE commands. Confirm that the worktree starts clean and re-run
   `sase bead epic-symbols` for both phases so later edits are based on current state.
   Read the TUI performance and repository lint/test memories before changing code.

2. **Audit the landed platform-unit phase and repair only demonstrated gaps.** Check the
   implementation and focused tests for the phase's required behavior: pure,
   dependency-injected systemd/launchd planning; stable non-workspace executable
   resolution; safe alternate-home suffixing; 0600 redacted environment capture and
   early host loading; legacy-owner, linger, disabled-login-item, CLI-readiness, and
   executable-drift diagnostics; idempotent init/uninstall; installed-manager versus
   detached-fallback lifecycle routing; machine-scoped onboarding once per invocation;
   and doctor integration. Keep manager inspection read-only and prevent tests from
   touching real units. If the audit finds a real mismatch, make the smallest in-scope
   correction with a regression test rather than broadening the epic.

3. **Audit the landed Services-tab phase and repair only demonstrated gaps.** Exercise
   both `service_host` flag states and confirm: `Services` display text with canonical
   internal id `axe`; `services`/legacy CLI aliases; off-thread, generation-aware
   snapshot refresh; configured running/stopped/disabled/unavailable proc rows with the
   routine/job tree under Scheduler; shared service action semantics; `x`, `r`, `!e`,
   and `!x` behavior; Scheduler-only quit semantics; loud `SVC` health; service/monitor
   gear exclusion; and one-time `ace.procs.default_query` seeding that preserves an
   explicitly cleared query. Keep synchronous I/O, config work, and subprocesses out of
   render, navigation, timer, and message-pump paths. Fix only reproduced omissions or
   regressions and add targeted tests for them.

4. **Resolve all 17 public-symbol findings by real ownership.** For each definition,
   search non-test consumers and apply the Symvision hierarchy instead of adding
   artificial imports:
   - In `service/env.py`, decide the internal ownership of `CapturedServiceEnvironment`,
     `ServiceEnvironmentError`, and `read_service_environment`.
   - In `service/platform.py`, do the same for `NativeInspection`,
     `NativeServiceDefinition`, `ServicePlatformApplyResult`, `build_native_definition`,
     `inspect_native_service`, `readiness_warnings`, and `service_platform_supported`.
   - In `service/status.py`, resolve `ServiceEnablement` and
     `resolve_service_enablement`; in `service/config.py`, resolve
     `ServiceFieldProvenance` and `compose_service_config`; in `service/state.py`,
     resolve `clear_service_enablement`; and in `service/paths.py`, resolve
     `service_dir` and `service_state_path`.

   Make an implementation detail private when it is used only within its defining
   module, and update annotations, exports, and focused tests consistently. Delete a
   genuinely dead definition and tests that exist only to preserve it. Keep a public
   name only when it gains an authentic non-test consumer, or when a verified
   non-Python/external consumer requires a precise Symvision pragma. Re-key an epic
   symbol only when an approved, still-open _concrete later phase_ explicitly names the
   future consumer; record that semantic reason and never use the parent epic as an
   undifferentiated holding pen. The result must contain no exemption owned by
   `sase-11y.5` or `sase-11y.7`.

5. **Clean the Justfile and preserve behavior-focused coverage.** Remove every resolved
   `sase-11y.7(...)` line (and any newly discovered stale `sase-11y.5(...)` line) from
   `_lint-symvision`. Adjust tests to assert observable service behavior through public
   entrypoints where practical; direct tests of private pure helpers are acceptable when
   they protect platform rendering/parsing boundaries. Do not weaken tests merely to
   silence the linter, and do not patch or vendor Symvision.

6. **Verify the two phases as one integrated tree.** Run `just install` first, then:
   - the focused service environment/platform, init/parser/handler, onboarding, doctor,
     service actions/status/config/state, Services tree/render/action, footer, proc
     gear/query/default-state, refresh race, tab-alias, and feature-flag tests;
   - `just test-visual`, inspecting actual/expected/diff artifacts before accepting any
     intentional Services-label/tree/footer changes;
   - the established `SASE_TUI_PERF=1`/slow navigation benchmark under changing service
     snapshots, recording Services-tab p95 below 16 ms and checking that render and
     navigation perform no synchronous I/O;
   - `just fix`, the exact no-exemption Symvision invocation (or equivalently
     `just _lint-symvision` after the Justfile cleanup), and `just check`.

   If `just check` selects the broadening lane, escalates, or reports unusual coverage,
   run `just check-full` through `/sase_monitor` as required. Re-run the two
   `sase bead epic-symbols` audits after all formatters and fixes; both must be empty.

7. **Record proof and close only the requested phases.** If genuinely out-of-scope work
   is discovered, append a `PROPOSED FOLLOW-UP:` note to the phase that exposed it; do
   not create a task bead from this epic-phase completion. Once every check above has
   passed, close `sase-11y.5` and `sase-11y.7` with separate notes naming their focused
   suites, visual/performance evidence where applicable, `just check` (and
   `just check-full` if run), and the empty epic-symbol audits. Do not close `sase-11y`,
   change other phase statuses, perform live athena/apollo rollout, or take ownership of
   the oneshot/sunset phases.

## Acceptance criteria

- The platform-unit and Services-tab behavior described by the approved phase plans is
  present, covered, and unchanged outside any demonstrated seam fixes.
- Symvision passes without an exemption owned by either requested phase and without fake
  production imports.
- Focused tests, visual inspection, the Services navigation performance gate, and the
  repository-required checks pass on the final tree.
- `sase bead epic-symbols sase-11y.5` and `sase bead epic-symbols sase-11y.7` both
  report no entries immediately before closure.
- Both `sase-11y.5` and `sase-11y.7` are closed with specific verification notes;
  `sase-11y` and all other phases remain untouched.
