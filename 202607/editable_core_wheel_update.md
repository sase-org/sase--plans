---
tier: tale
goal: 'Comprehensive SASE updates upgrade a stale managed sase-core-rs wheel even
  when the host and all installed plugins are editable checkouts, with honest previews,
  results, and restart behavior in both the CLI and Admin Center.

  '
create_time: 2026-07-15 13:46:50
status: wip
prompt: 202607/prompts/editable_core_wheel_update.md
---

# Plan: Update managed SASE core inside editable installs

## Diagnosis

The failing install is neither wholly managed nor wholly editable. Runtime inventory shows editable `sase`,
`sase-github`, and `sase-telegram` packages, but `sase-core-rs` is a transitive managed wheel at `0.4.0`. The Admin
Center's independent latest-version enrichment correctly finds the compatible `0.4.1` wheel and renders it as available.

The comprehensive update planners do not represent that combination:

- The CLI's editable route selects editable receipt requirements and only considers non-editable _top-level receipt
  requirements_ managed work. Because `sase-core-rs` is a dependency of `sase`, not a top-level uv-tool requirement,
  `sase update --dry-run --json` currently emits three skipped, current editable packages, `command: []`, and
  `managed: null`.
- The Admin Center's full-update preview similarly filters runtime inventory to editable host/core/plugin records. When
  all editable roots are current, `dev_update_blocking_reason()` stops before confirmation and reports only the first
  skipped reason, producing the screenshot's “3 editable checkouts skipped; sase: already current” error even though
  managed core work remains.

The bug is therefore an update-plan modeling gap: a comprehensive update loses managed transitive core work as soon as
editable packages select the dev path. Selected-plugin updates must continue pinning core; only the comprehensive
`sase update` operation should re-resolve it.

## Implementation

1. Extend comprehensive update routing and planning to represent managed core reconciliation alongside editable checkout
   work.
   - Identify a wheel-installed runtime core as managed work even when it is absent from the uv-tool receipt's top-level
     requirements.
   - Preserve the receipt as the source of truth for the editable host and injected plugin set, and use an explicit uv
     operation that re-resolves the compatible core dependency without converting, dropping, or unexpectedly upgrading
     editable/plugin sources.
   - Keep selected-plugin update behavior unchanged: it must continue to pin core and unrelated packages.
   - Treat current or safely skipped editable roots as independent from managed core work. They should remain visible in
     the plan, but must not block a valid core-wheel update.

2. Make CLI execution, dry-run rendering, and JSON describe the combined plan honestly.
   - Run safe editable fast-forwards/reconciliation and the managed core pass in a deterministic order, aborting with
     actionable diagnostics on a real failure while never modifying unsafe git states.
   - Report the operation as mixed whenever both editable inventory and managed core reconciliation are present,
     including the managed command/package in dry-run output and the actual core transition in results.
   - Compute `changed`, counts, no-op output, journaling, and axe restart from actual outcomes so a core-only change
     restarts once and a true no-op does not restart.
   - Reuse the existing schema-2 combined `dev`/`managed` structure unless an implementation constraint proves a
     compatibility-breaking schema change is necessary.

3. Align the Admin Center `u` action with those comprehensive semantics.
   - Carry managed-core intent through the preview instead of treating a no-actionable-root dev plan as terminal.
   - Show a confirmation that distinguishes skipped/current editable checkouts from the pending core-wheel operation and
     retains the incoming-commit preview.
   - Execute both parts through the tracked-task path, aggregate their outcome, and produce one success/failure
     notification and one restart decision.
   - Preserve post-restart version receipts so a core-only update is represented as a dependency transition, while
     cancellation and genuine all-current states remain non-mutating and clear.

4. Add regression coverage at each contract boundary.
   - CLI routing/dry-run/live tests: an editable host/plugins receipt plus a stale wheel core yields a mixed plan,
     invokes the managed core operation, reports `0.4.0 -> 0.4.1`, and restarts only when changed.
   - Dev safety tests: dirty, diverged, detached, current, and fetch-failed editable roots remain untouched while
     eligible managed work can proceed; managed failure cannot be reported as success.
   - Admin Center tests: reproduce the screenshot state, assert `u` opens the combined confirmation rather than emitting
     the skipped-checkout error, then cover core-only success, true no-op, failure, cancellation, receipt, and restart
     behavior.
   - Add or extend the hermetic real-`uv` harness with an editable stand-in tool and stale transitive dependency. Prove
     the chosen command upgrades the dependency while preserving the editable primary and injected packages; keep this
     network-dependent check in the existing slow-test category.

5. Update the user-facing update documentation to state that comprehensive updates re-resolve a managed core wheel even
   in dev/editable mode, while selected-plugin updates leave core pinned.

## Validation

Run focused update-router, dev-update, CLI JSON/rendering, Admin Center action, and receipt tests while iterating. Run
the real-`uv` regression when the package index is available. Finally run `just install` and the required `just check`
for the complete repository validation, and inspect the resulting diff/status to ensure only the intended
implementation, tests, documentation, and proposed plan artifacts changed.

## Expected result

In the reported state, the Updates tab still shows the three editable checkouts as current, but `u` offers and performs
the `sase-core-rs 0.4.0 -> 0.4.1` managed update. Direct `sase update` behaves the same way. The editable source paths
and plugin set are preserved, unsafe checkouts are never changed, and ACE/axe restart exactly once only after an actual
package transition.
