---
tier: tale
title: Fix Grok usage collection when zero usage is omitted after reset
goal:
  Grok's verified zero-usage billing responses produce a fresh included-allowance
  observation and clear collector failures through normal refresh, while ambiguous or
  invalid billing payloads remain errors.
size: medium
proposed_by: bbugyi200.athena.0je
create_time: 2026-09-11 10:08:36
status: wip
---

# Plan: Fix Grok usage collection after the weekly reset

## Scope and tier

This is a medium tale: one coding agent can implement the bounded Grok normalization
change across `sase-core` and `sase`, add regression coverage, and verify recovery. The
Rust binding and its Python caller are one coordinated change. Separate epic phases
would add handoffs without providing useful independent work.

Keep process launch, ACP transport, deadlines, and executable verification in the Python
collector. Put the affected billing normalization in Rust, following the project's
shared backend boundary. Preserve existing usage-store, refresh, routing, and
presentation contracts. This plan does not require new CLI options, configuration
settings, feature flags, or memory edits.

## Diagnosis and evidence

The failure was reproduced read-only on 2026-09-11 with the workspace collector and
installed `grok 1.0.25 (f7e67d6988e2) [stable]`. Two bounded ACP sessions sent only
`initialize` and `_x.ai/billing`; no model prompt or billing mutation was sent, and no
usage-cache refresh was submitted during diagnosis.

`sase usage list -p grok --json` reported:

- `diagnostic: grok_billing_usage_percent_missing` and
  `collection_reason: malformed_payload`.
- The retained weekly window had `used_percent: 100`, with reset at
  `2026-09-11T13:19:34.664647Z`.
- Last successful collection was `2026-09-10T17:53:54.969751Z`; the first recorded
  failure was `2026-09-11T13:27:38.937176Z`, about eight minutes after reset.
- Collector health was `degraded` with two consecutive failures. The retained window's
  freshness was `unknown`, `reset_passed` was true, and the provider's summary and known
  constraints were empty.

Both live billing responses contained a new weekly period and the following fields (only
the relevant subset is reproduced):

```json
{
  "config": {
    "currentPeriod": {
      "type": "USAGE_PERIOD_TYPE_WEEKLY",
      "start": "2026-09-11T13:19:34.664647+00:00",
      "end": "2026-09-18T13:19:34.664647+00:00"
    },
    "isUnifiedBillingUser": true,
    "billingPeriodStart": "2026-09-11T13:19:34.664647+00:00",
    "billingPeriodEnd": "2026-09-18T13:19:34.664647+00:00"
  },
  "subscription_tier": "SuperGrok Heavy"
}
```

`creditUsagePercent`, `used`, and `monthlyLimit` were absent. Additional fields were
on-demand and prepaid balances, which must not contribute to included allowance.

The causal chain is in `src/sase/llm_provider/usage/grok.py`:

1. `_collect_grok_billing` successfully receives and unwraps the config.
2. `_usage_percent` requires either an explicit `creditUsagePercent` or a valid legacy
   `used.val / monthlyLimit.val` pair. The live response supplies neither.
3. `_observation_from_config` returns `grok_billing_usage_percent_missing`, discarding
   the otherwise valid new period from this observation.
4. The Rust store deliberately preserves older windows after collection failures. The
   CLI therefore shows the last exhausted allowance, its passed reset, its old age, and
   the new collector failure. The displayed zero remaining is historical.

An in-memory replay of the live response reproduced this error. Adding only
`creditUsagePercent: 0.0` to an in-memory copy produced a validated complete weekly
observation with `used_percent: 0`, duration `604800`, and the new September 18 reset.
No implementation file was changed for that experiment.

Upstream corroboration was read from the official `xai-org/grok-build` checkout at
commit `37949780c144e37df692e3d669051a21fec24f20`:

- [Billing wire types](https://github.com/xai-org/grok-build/blob/37949780c144e37df692e3d669051a21fec24f20/crates/codegen/xai-grok-shell/src/extensions/billing.rs):
  `BillingConfig.credit_usage_percent` is optional and omitted during serialization when
  absent. `Cent.val` defaults to zero because proto3 JSON can represent a zero-valued
  Cent as `{}`. `BillingConfigResponse` uses snake-case `subscription_tier`; its nested
  config uses camelCase.
- [Grok's own balance conversion](https://github.com/xai-org/grok-build/blob/37949780c144e37df692e3d669051a21fec24f20/crates/codegen/xai-grok-pager/src/app/effects/helpers.rs):
  `credit_balance_from_config` prefers an explicit percentage, otherwise uses the legacy
  ratio when a positive limit exists, otherwise produces zero usage.
- [Official usage documentation](https://docs.x.ai/grok/faq#usage--limits) confirms the
  included allowance has a provider-reported weekly reset.

Together, the reset timing, repeated response shape, successful explicit-zero replay,
and upstream conversion identify an omitted-zero compatibility defect in SASE. The
backend's internal serialization was not inspected; the fix should rely on the verified
public wire shape and upstream consumer behavior.

Two closely related normalization gaps belong in the same fix:

- Legacy `used: {}` with a positive `monthlyLimit.val` fails today, while
  `used: {"val": 0}` succeeds. Upstream explicitly defines the former as zero.
- The collector only reads `subscriptionTier`, so the real response loses its
  `SuperGrok Heavy` plan label. Existing fixtures use the camel-case variant and
  therefore miss this mismatch.

## Implementation

### 1. Add the Rust billing normalization contract

Open the core repository with
`sase repo open sase-core -r "Implement Grok omitted-zero billing normalization"` and
use only the printed path. Read its `AGENTS.md`. If upstream context is needed, open
`gh:xai-org/grok-build` with `sase repo open`; do not fetch repository files through web
URLs.

Add a focused module under `crates/sase_core/src/provider_usage/`, exported through
`provider_usage/mod.rs`, for pure Grok billing-config normalization. Move the affected
percentage, period, completeness, and plan-label decisions from
`_observation_from_config`, `_billing_period`, `_usage_percent`, and their numeric,
timestamp, and string helpers into this module. Keep unrelated providers unchanged.

Expose it through `crates/sase_core_py/src/lib.rs`, for example as
`provider_usage_normalize_grok_billing`. Use an explicit typed/versioned request
containing the decoded billing payload, required probe identity/fencing fields, request
start time, and injected observation time. Return the existing validated
`ProviderUsageObservationWire`; retain observation/public/store schema version 1.
Ordinary malformed vendor data returns the existing structured error observation. An
invalid binding request remains a binding validation error. Do not add a Python fallback
for a missing Rust binding.

Implement these rules in order:

1. An explicitly present `creditUsagePercent` must be a finite, nonnegative JSON number.
   Zero is valid. Boolean, null, string, negative, and nonfinite values remain invalid
   and must not activate a fallback. Preserve the existing ability to represent over-100
   usage; do not copy Grok's display-only clamping.
2. If the percentage is absent and a valid positive legacy `monthlyLimit.val` exists,
   use `used.val / monthlyLimit.val * 100`. Treat an existing empty `used` object as
   zero, following upstream's Cent default. Preserve strict validation for explicitly
   malformed Cent values and nonpositive limits. Do not extend this into accepting a
   completely absent legacy `used` object in this change.
3. If percentage and both legacy amount fields are absent, recognize the verified
   unified-billing shape only when `isUnifiedBillingUser` is exactly `true` and
   `currentPeriod` has a supported weekly/monthly type plus finite parseable start and
   end with end later than start. Normalize this shape to zero used. The provider's
   period establishes the window; do not require a previously cached exhausted window or
   a reset that just passed. A later probe can still observe zero usage, and a first
   probe can land in a fresh period.
4. All other missing-percentage cases remain
   `malformed_payload / grok_billing_usage_percent_missing`. In particular, `{}`, a
   period alone, a missing/false/nonboolean unified marker, and an invalid period cannot
   assert full remaining capacity. Malformed or incomplete legacy fields cannot be
   bypassed by the unified-billing zero rule.

Preserve weekly/monthly/generic window keys and labels, account-wide applicability,
timestamp precision, modern-period precedence over legacy billing dates, legacy monthly
dates, and partial observations for an explicit valid percentage with no reset. Keep
`vendor_state: unknown`; collection does not imply routing permission.

Read nonempty `subscription_tier` from the top-level live response, with the existing
top-level/config `subscriptionTier` forms as compatibility fallbacks. Prefer the native
snake-case form if both top-level forms exist. Neither credentials nor
on-demand/prepaid/history fields belong in the normalized observation or diagnostics.

### 2. Wire the Python collector to the core

Replace the affected Python normalization with a thin adapter using
`require_rust_binding`. It supplies the required probe context and observation clock and
returns the core result. Delete superseded normalization helpers and imports so there is
one implementation of the changed semantics.

Keep the current ACP envelope handling (mapping, JSON-string result, supported nested
result), account eligibility/authentication classification, process cleanup, identity
checks, and error transport unchanged. Ineligible/API-auth responses must still be
classified before an omitted-zero response can become a subscription observation.

The normal refresh path must record the successful new observation and successful
refresh attempt, replacing the old period and clearing the failure streak. Do not
manually edit/delete the usage cache, reset failure counters, or synthesize capacity
from the current clock. The store already has the necessary recovery behavior.

Update the subscription-usage section of `docs/llms.md` with a short explanation of
Grok's supported omitted-zero shape and the normal refresh recovery command.

### 3. Add meaningful regression coverage

Add Rust unit tests for the normalizer and a PyO3 round-trip/validation test for the new
binding. Extend `tests/llm_provider/test_grok_usage_probe.py` and its strict fake ACP
fixture as needed. Use synthetic dates and amounts in checked-in fixtures; make the new
fixture match the observed mixed snake-case/camelCase wire shape.

Cover these independent behaviors:

- The verified unified weekly response with omitted percentage/amounts yields a complete
  zero-used observation, the correct new period, and `SuperGrok Heavy`. The equivalent
  monthly shape and explicit zero agree with it.
- Fractional nonzero and over-100 percentages retain existing semantics, and an explicit
  valid percentage takes precedence over conflicting legacy amounts.
- Existing legacy monthly ratios still work; `used: {}` and `used: {"val": 0}` both
  produce zero with a positive limit. Invalid, absent, or zero denominators and
  malformed explicit numerators remain errors.
- The omitted-zero guards reject absent/false/invalid unified markers, incomplete or
  reversed periods, unrecognized period types, and ambiguous empty configs. Keep the
  existing period-only missing-percentage regression failing as malformed.
- Explicit invalid percentages, including null, boolean, string, negative and nonfinite
  numbers, do not fall through to either zero or the legacy ratio. Exercise nonfinite
  input through the actual Python binding boundary as well.
- Native `subscription_tier` works, older `subscriptionTier` forms still work, and
  unused billing fields do not leak into observations.
- Existing missing-reset partial observations, API/free/enterprise accounts, login
  failures, missing ACP method, executable mismatch, timeout, and process cleanup
  behavior continue to pass. Cover supported ACP wrapper forms when adapting the
  normalizer.

Add a deterministic recovery integration test using a temporary usage store and the
existing observation/refresh-attempt APIs. Seed an exhausted old weekly window, record
two malformed probe failures after its reset, and then record the fixed new-period
observation and successful refresh attempt. Assert exactly one current weekly window,
fresh age, the new reset, 100% remaining, health `ok`, zero consecutive failures,
cleared `failing_since`, and advanced `last_success_at`. Include a still-malformed
attempt that preserves the old window, so the fix does not weaken failure retention.
Test the resulting CLI presentation for this projected snapshot; no new UI rendering
policy is needed.

## Verification and completion

1. Read `lint_and_test.md` through `sase memory read` before implementing/verifying.
   Rebuild the workspace extension with `just rust-install` after changing the core and
   binding. The Justfile recognizes the opened core checkout; use its supported
   `sase_core_dir` override if the printed repository path differs. Run `just install`
   if the workspace environment needs setup. Do not manually edit release versions or
   dependency pins to make the local build work.
2. Run the focused Grok probe and new normalization/recovery tests using the workspace
   virtualenv. Run `just check` in `sase`. In the opened `sase-core` root, run
   `just check` or `./scripts/check.sh`, which covers the workspace and PyO3 binding
   tests. A core-only `cargo test -p sase_core` is insufficient. Follow `/sase_monitor`
   for long verification; use `just check-full` through a monitor if repository
   selection rules require escalation.
3. After tests pass, run one bounded real Grok usage probe using the changed workspace
   code, followed by the normal Grok refresh and cached inspection:
   `sase usage refresh -p grok --json` and `sase usage list -p grok --json`. Ensure the
   CLI and refresh worker use the updated Python code and Rust extension; a globally
   installed older SASE binary does not validate this change. Inspect only needed
   billing fields and sanitized observations, never credentials.
4. If the account still has zero usage, expect the current weekly period, 100% left, a
   future reset, a fresh age, and collector health `ok`. If usage has accrued, expect
   its real nonzero percentage instead. The historical September dates are evidence, not
   hard-coded acceptance values. If a live probe is unavailable, report that limitation
   separately from passing deterministic regression tests.

The implementation is complete when the supported omitted-zero response succeeds through
the real collector/core path, normal refresh replaces the stale period and clears
failures, malformed responses remain errors, and both repositories' required checks
pass. Let host-owned completion handle repository publication in dependency order so the
Python caller is paired with a Rust build exposing the new binding.
