---
tier: epic
title: Finish usage-limit landing integration
goal:
  The published Python dependency floor guarantees every usage-limit first-writer
  binding, and the Antigravity auto-disable acceptance test follows the current
  @xsmall-only shipped routing contract.
phases:
  - id: published-core-floor
    title: Require the published first-writer core release
    depends_on: []
    size: small
    description:
      "published-core-floor: integrate with the concurrently running sase-n8.8 floor
      phase, require sase-core-rs 0.27.15 or newer within the existing 0.28 ceiling,
      refresh lock metadata, and prove the declared minimum wheel exposes all required
      first-writer and intervening core capabilities."
  - id: agy-xsmall-integration
    title: Follow Antigravity's current xsmall-only routing
    depends_on: []
    size: small
    description:
      "agy-xsmall-integration: update the post-start Antigravity usage-limit acceptance
      coverage to assert that auto-disable removes the current @xsmall pool member,
      preserve the captured reset-hint behavior, and verify the @small pool no longer
      contains Antigravity."
proposed_by: bbugyi200.athena.sase-n4.5.land
parent_bead: sase-n4.5
bead_id: sase-n4.5.4
create_time: 2026-09-09 19:50:26
status: wip
---

- **PROMPT:**
  [prompts/202608/finish_usage_limit_landing_integration.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_usage_limit_landing_integration.md)
- **PARENT:**
  [202608/finish_usage_limit_auto_disable.md](https://github.com/sase-org/sase--plans/blob/main/202608/finish_usage_limit_auto_disable.md)
- **BEAD:**
  [sase-n4.5.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-n4/sase-n4.5.4.md)

# Plan: Finish usage-limit landing integration

## Verified residue

The three original `sase-n4.5` phases are implemented and closed. The landing audit
found only two current-tree integration gaps:

1. `sase-core` commit `dc87c8e` shipped the atomic provider-disable try-set bindings in
   release `0.27.15`, but `pyproject.toml` still accepts `sase-core-rs>=0.27.11`. A
   non-advisory `tools/probe_core_floor` run reports `stale_actionable`: the declared
   floor lacks six now-published capabilities, including
   `provider_disable_try_set_relative` and `provider_disable_try_set_until`. Concurrent
   active phase `sase-n8.8` is already assigned to raise the same shared dependency
   window for alias-history schema 22, so this work must integrate with its landed state
   instead of overwriting or duplicating it.
2. Post-start commit `9568dd47` added captured Antigravity quota prose and an acceptance
   test that checks disabling `agy` removes its `@small` pool member. Later commit
   `630f4ea7` moved Antigravity exclusively to `@xsmall` but did not update that new
   usage-limit test. The combined base branch therefore carries a stale assertion across
   two individually valid changes.

This plan contains only those remaining integration items. Closing `sase-n4.5`, its
post-close Symvision cleanup, follow-up-task recording, and setting the parent plan to
done remain the resumed parent land agent's responsibilities and are not phases here.

## Phase `published-core-floor`: require the published release

Start from the latest base branch and re-read `pyproject.toml`, `uv.lock`, the
`sase-core` release history, and phase bead `sase-n8.8` before editing.

1. If `sase-n8.8` has already landed a dependency floor of at least `0.27.15`, retain
   its exact compatible window and lock result; do not make a cosmetic duplicate edit.
   Verify that its chosen floor covers the two provider-disable first-writer bindings.
2. Otherwise raise the `sase-core-rs` lower bound to `0.27.15` while preserving the
   existing `<0.28.0` compatibility ceiling, and refresh only the relevant lockfile
   package metadata using the repository's normal dependency-lock workflow. Do not widen
   to an unreleased version or alter unrelated dependencies.
3. Run the non-advisory core-floor probe against the linked `sase-core` checkout and
   require an `ok` result for the published minimum. Run the version validator, binding
   validator, and their focused tests so a clean install from the declared minimum—not
   merely the linked development build—proves the required binding names, wire shape,
   and first-writer behavior.
4. Run `just install` before repository verification and then `just check`. If another
   active epic still causes a pre-existing gate failure, record exact evidence on its
   causal bead and preserve this phase's focused green results.

Acceptance: every supported published wheel contains the first-writer operations used
unconditionally by `usage_limit_disable.py`, `tools/probe_core_floor` no longer reports
the declared floor stale, and concurrent `sase-n8.8` work is preserved rather than
duplicated.

## Phase `agy-xsmall-integration`: align the acceptance test

Integrate base commits `9568dd47` and `630f4ea7` together rather than reverting either.

1. In `tests/test_llm_provider_usage_limit_disable.py`, replace the stale `@small`
   expectation with the current `@xsmall` contract. Use `XSMALL_MODEL_ALIAS_NAME`,
   rename the test to state the pool it actually guards, and keep the assertion that the
   captured Antigravity quota failure creates a `source="usage_limit"` disable and makes
   the exact `agy/gemini-3.7-flash-high` member unavailable.
2. Retain `630f4ea7`'s explicit assertion that `@small` has no Antigravity member. Do
   not add `agy` back to `@small`, change the shipped alias defaults, or weaken the
   first-writer/reset-hint behavior added by `sase-n4.5` and `9568dd47`.
3. Run the focused usage-limit defaults, enforcement, retry, alias-default, and fakey
   acceptance tests that cover the combined behavior. Confirm the captured
   `Resets in 4h14m50s` message still yields the clamped/grace-buffered reset duration
   and that the provider disable removes only `agy` from routing.
4. Run `just install` first and `just check` after the focused tests. Because the parent
   feature crosses the Rust binding, root dependency window, retry path, config, and TUI
   surfaces, the child epic's landing verification must run `just check-full` through
   `/sase_monitor` with a `--next` action.

Acceptance: the current base branch's `@xsmall`-only Antigravity routing and its
usage-limit auto-disable acceptance coverage agree, while the original provider error,
first-writer window, notification, retry attribution, and other providers remain
unchanged.
