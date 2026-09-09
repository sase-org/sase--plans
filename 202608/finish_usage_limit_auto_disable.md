---
tier: epic
title: Finish usage-limit auto-disable correctness and surfaces
goal:
  Usage-limit failures atomically establish one provider-disable window across
  concurrent agents, retry decisions stay attributed to the provider that actually
  failed after overrides and fallbacks, user pattern replacement is literal, and the
  promised provenance, configuration, documentation, and end-to-end acceptance coverage
  are present.
parent_bead: sase-n4
phases:
  - id: atomic-disable
    title: Make first usage-limit disable atomic in sase-core
    depends_on: []
    size: medium
    description:
      "atomic-disable: add a Rust-core and Python-binding operation that atomically
      writes a provider disable only when no active record exists, returns whether this
      caller won the window, preserves the existing unconditional manual replacement
      APIs, and proves first-writer behavior under real contention."
  - id: runtime-correctness
    title: Correct matching, provider attribution, and end-to-end behavior
    depends_on:
      - atomic-disable
    size: medium
    description:
      "runtime-correctness: consume the atomic store result in usage-limit enforcement,
      make replace_patterns literal even for an empty list, keep retry attribution
      pinned to the execution provider recorded for each attempt including fallback
      attempts, and add a full fakey invocation-to-disable-to-notification acceptance
      test that proves one attempt, unchanged error propagation, and no collateral
      provider disable."
  - id: surface-and-document
    title: Restore disable provenance and document usage-limit policy
    depends_on: []
    size: medium
    description:
      "surface-and-document: render automatic, manual, and unknown disable provenance in
      the current split Launch Control provider modules and top-bar tooltip, add the
      commented usage_limit configuration block without disturbing newer config fields,
      document detection, reset hints, machine-wide expiry, clearing, retry precedence,
      and extension/replacement semantics, and cover the surfaces including intentional
      visual snapshots."
proposed_by: bbugyi200.athena.sase-n4.land
bead_id: sase-n4.5
create_time: 2026-09-09 19:50:26
status: wip
---

- **PROMPT:**
  [prompts/202608/finish_usage_limit_auto_disable.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_usage_limit_auto_disable.md)
- **PARENT:** [202608/llm_usage_limit_auto_disable.md](llm_usage_limit_auto_disable.md)
- **BEAD:**
  [sase-n4.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-n4/sase-n4.5.md)

# Plan: Finish usage-limit auto-disable correctness and surfaces

## Why this child epic exists

The four original phase beads are closed, but landing verification found that the
combined tree does not satisfy the approved epic:

- Phase `sase-n4.4` has no commit, and its reported UI provenance, default-config block,
  and documentation are absent from the current tree. The relevant Models-panel and
  top-bar lines still blame to commits from before `sase-n4`.
- `replace_patterns: true` with `patterns: []` restores the built-in patterns instead of
  replacing them with the explicitly empty list. This contradicts the approved escape
  hatch and prevents a user from disabling a stale built-in detector cleanly.
- Retry detection scans every provider after a known execution provider fails to match.
  A concrete probe attributes a known Codex attempt containing Claude limit prose to
  Claude. The approved plan permits provider scanning only when the execution provider
  is unknown.
- The retry tracker captures its execution provider once before the loop. Commit
  `96b48d0a` landed after the enforcement phase and now records the authoritative
  `exec_llm_provider` per invocation, but the retry path does not consume it. A fallback
  attempt can therefore be evaluated against the original provider.
- The notification dedup test calls the handler three times sequentially. Production
  performs `get_active_provider_disable()` and an unconditional set as separate
  Rust-lock acquisitions. Two processes can both observe no record, both replace it, and
  both notify. The Rust store has no conditional set operation, so the required
  first-writer behavior is not atomic.
- The existing fakey test calls `handle_possible_usage_limit()` directly. It does not
  exercise the real provider subprocess, invocation error path, retry loop, notification
  store, or preservation of the original provider error, so the epic's combined
  acceptance criterion has not run end to end.

This plan contains only that remaining work. The optional notification action proposed
by `sase-n4.3` is tracked separately and is not a completion requirement.

## Current-tree integration constraints

Read the current implementations before editing; do not reconstruct the files as they
looked when `sase-n4` began.

- Provider-disable state is shared domain behavior. Its conditional first-writer
  operation belongs in the linked `sase-core` repository under
  `crates/sase_core/src/provider_disable.rs`, with the pyo3 binding beside the existing
  `provider_disable_set_relative` and `provider_disable_set_until` functions. Open the
  linked repository through `/sase_repo` before reading or changing it.
- Preserve the unconditional replace semantics used by ACE's manual provider manager.
  Auto-disable needs a new conditional operation; changing the existing setters would
  break manual duration changes.
- The core operation must prune expired/malformed records and decide absence plus write
  under one lock. Return both the active record and an unambiguous inserted/won flag so
  Python increments telemetry and sends a notification only for the winning write.
- Coordinate the pyo3 symbol, Python facade, `tools/validate_sase_core_rs`, and the
  published `sase-core-rs` dependency window. Do not land Python code that imports a
  binding absent from the supported wheel range.
- Commit `96b48d0a` added authoritative alias-launch provenance and writes
  `exec_llm_provider` into agent metadata from `_invoke.py`. Refresh retry attribution
  from the attempt that just failed instead of treating the loop's initial provider as
  permanent. Never scan other providers when that authoritative identity is known.
- Commit `23c953bc` added `llm_provider.model_alias_history_limit` to the same schema,
  default-config, and documentation areas. Commit `76c332bd` added generated feature
  flag schema content. Preserve both changes and their validation gates while inserting
  usage-limit documentation.
- Launch Control provider rendering has already been split into
  `models_panel_provider_rendering.py` and related focused modules. Put presentation
  logic in the current module boundary, not the pre-split compatibility facade.

## Phase `atomic-disable`: Make first usage-limit disable atomic in sase-core

Add a conditional Rust domain operation for a relative duration and an exact expiry, or
one well-typed operation that represents both modes. Under the existing provider-disable
file lock it must:

1. validate the provider, source, clock, and requested expiry/duration exactly as the
   existing setters do;
2. read and prune the current state at the supplied clock;
3. return the existing active record with `inserted=false` when the provider already has
   one, without changing its `created_at`, expiry, or source;
4. otherwise write the requested record atomically and return it with `inserted=true`.

Keep the current unconditional setters and their replacement tests unchanged for manual
ACE use. Add domain tests that release multiple contenders through a barrier and prove
exactly one inserted result, one stable stored record, no expiry extension, and no loss
of unrelated provider records. Cover expired-record replacement and lock/error paths.

Expose the outcome through `sase_core_py` with a stable JSON-shaped payload and binding
tests. Add a typed Python facade result or helper without weakening the strict
`TemporaryProviderDisable` wire validation. Extend the binding validator and smoke tests
so stale wheels fail loudly. Coordinate the core release and `pyproject.toml` version
window according to the repository's existing core-release workflow.

## Phase `runtime-correctness`: Correct matching and attempt attribution

Change `_merge_with_built_in()` so key presence plus `replace_patterns: true` makes the
configured list authoritative, including an empty list. Add a regression proving that
empty replacement disables the built-in match while ordinary additive and non-empty
replacement behavior remain unchanged.

Use the new conditional provider-disable facade from `usage_limit_disable.py`.
Telemetry, logging at the new-disable level, and
`notify_provider_usage_limit_disabled()` must run only when the operation reports
`inserted=true`; losing contenders return the detection without extending the active
window or notifying. Replace the sequential "concurrent-style" test with genuine
contention against the authoritative store and assert one winner, one metric increment,
and one durable notification.

Fix retry attribution as a per-attempt fact:

- When the provider that actually ran is known, detect only against that provider. Do
  not fall through to `find_usage_limit_detection_for_error()` after a known provider
  fails to match.
- Before classifying the failed attempt, consume the authoritative execution-provider
  provenance `_invoke.py` wrote for that attempt, including after a fallback switches
  models/providers. Keep the unknown-provider scan only for workflows that genuinely
  cannot identify an execution provider.
- Prove a Codex attempt containing quoted Claude limit prose still follows Codex retry
  policy, and prove a fallback provider's own usage limit is attributed to that fallback
  rather than the initial provider.

Extend the real fakey subprocess harness with a usage-limit failure scenario. Through
`run_execution_loop`, prove that the trigger:

- invokes fakey exactly once and consumes no retry sleep;
- disables only fakey with source `usage_limit`;
- writes exactly one rich notification and one telemetry event for the disable window;
- leaves every other provider untouched;
- records the retry-skip reason on the attempt snapshot; and
- raises the original provider failure with the fakey trigger still present rather than
  masking or replacing it with detection, storage, or notification behavior.

Retain the transient plain-429 retry regression and all real Claude, Codex, and Grok
corpus cases.

## Phase `surface-and-document`: Restore the missing phase

Centralize a small presentation helper for `TemporaryProviderDisable.source` and reuse
it on both required surfaces:

- render `source="ace"` as manual and `source="usage_limit"` as usage-limit/automatic in
  active provider rows and the selected-provider description in Launch Control;
- include the same distinction for every provider in the top-bar disable tooltip; and
- treat `source` as an open vocabulary, rendering a safe readable form of unknown
  non-empty values rather than crashing or silently calling them manual.

Keep the compact top-bar pill width stable unless an intentional visual decision says
otherwise. Add focused unit tests for manual, automatic, and unknown provenance plus
Models-panel and indicator PNG coverage. Inspect actual/expected/diff artifacts before
accepting any golden changes.

Insert the fully commented `llm_provider.usage_limit` block immediately after retry in
`src/sase/default_config.yml`, preserving the newer model-alias history and feature-flag
configuration. Explain global defaults, per-provider overrides, additive patterns,
literal empty/non-empty replacement, exclusions, reset hints, min/max clamping of
provider-reported hints, notification control, the 24-hour fallback, machine-wide
self-expiry, and early clearing through Launch Control.

Update `docs/configuration.md`, `docs/llms.md`, and the Launch Control/provider-disable
section of `docs/ace.md`. Document that usage-limit classification precedes retry only
for a positive provider-scoped match, that a plain transient 429 still retries, and that
fallback may proceed only to a different enabled provider. Link the sections rather than
duplicating an entire pattern corpus in every file.

## Verification and completion criteria

Each phase runs `just install`, its focused Rust/Python/unit/visual tests, and
`just check`. The child epic's land agent runs `just check-full` through `/sase_monitor`
because this work crosses the broadening Rust-binding, root config/schema, notification,
retry, and TUI sets.

The child epic is complete only when:

- true concurrent contenders create one stable disable window and exactly one
  notification/metric event;
- explicit empty pattern replacement, provider-scoped retry matching, and fallback
  provider refresh have focused regressions;
- the fakey subprocess acceptance test proves the complete behavior and unchanged
  original error;
- Launch Control and the top-bar tooltip distinguish manual, usage-limit, and unknown
  provenance;
- the shipped config and user docs fully explain the feature while retaining newer
  adjacent settings; and
- all focused checks and the monitored exhaustive gate pass, with unrelated failures
  routed through the phase-note follow-up process.
