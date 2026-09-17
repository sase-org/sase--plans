---
tier: tale
title: Unify Claude Fable usage identities across probes and passive events
goal:
  Show one accurate Fable usage window across passive updates, full refreshes, and
  existing cached state, eliminating the seven_day_overage_included/scope? duplicate.
size: medium
proposed_by: bbugyi200.athena.0m8
create_time: 2026-09-17 08:38:43
status: wip
---

# Claude Fable usage identity fix

## Outcome and scope

Implement this as one medium tale spanning SASE and its Rust core. The bug is bounded:
two collection paths assign different identities to the same provider allowance.
Canonicalize that identity in Rust, preserve ordering and independent allowances, and
verify the existing Python adapter and indicator end to end. No new CLI options,
configuration defaults, feature flags, transport, or refresh scheduling are needed.

Open `sase-core` with `/sase_repo` and use the returned checkout path. All Rust paths
below are relative to that checkout; other paths are relative to SASE. Read each
repository's instructions before implementing. Do not implement shared normalization or
cache reconciliation rules in the Textual renderer or in a Python fallback.

## Diagnosis and evidence

The supplied screenshot, `~/tmp/screenshots/20260917_082050.png`, shows an all-model
weekly allowance at 12% remaining, Fable at 19%, and a second
`seven_day_overage_included/scope?` entry at 19% with the same reset countdown.

The root cause is confirmed at each layer:

1. Claude Code's installed executable, version 2.1.274, explicitly labels
   `seven_day_overage_included` as the Fable limit. Its `unifiedWindows` schema
   describes an optional per-model weekly subscription bucket. An independent primary
   report in
   [the Claude Code issue tracker](https://github.com/anthropics/claude-code/issues/73770)
   also identifies this field with the Fable allowance. This is a known model-specific
   window, not evidence that every overage-related bucket is a Fable alias.
2. `src/sase/llm_provider/usage/_claude_support_windows.py::event_window` recognizes
   session and all-model weekly event keys, but converts this key into
   `window:seven-day-overage-included`, with unknown applicability and the raw vendor
   label. Its prose parser already converts `Current week (Fable)` into
   `weekly:claude-fable-5`, applicable to `claude-fable-5`.
3. `src/sase/llm_provider/usage/claude.py::_claude_rate_limit_event_observation` emits
   partial observations. Rust's `provider_usage/store.rs` merges them by key, so the two
   spellings coexist. A newer complete `/usage` inventory removes an older omitted
   alias; a subsequent passive event adds it again. Out-of-order observations are fenced
   by ordering tokens and tombstones.
4. `src/sase/default_config.yml` always shows the canonical Fable window, but applies
   the generic below-20%-remaining threshold to the unknown alias. This explains why the
   extra entry becomes visible at 19% rather than throughout the usage cycle.
5. `_usage_indicator_format.py` deliberately renders unknown applicability as the vendor
   label followed by `/scope?`. That fallback is useful for genuinely unknown windows
   and should remain.

Read-only inspection of the live usage cache found a probe Fable record at 81% used and
a newer passive alias at 82% used, with the same reset timestamp. Replaying the existing
projection and formatter without modifying state reproduced:

```text
12% 2d11h · fable 18% 2d11h · seven_day_overage_included/scope? 18% 2d11h
```

The percentages can differ between these records because the passive update is newer.
Never deduplicate arbitrary windows by matching percentages or reset times.

## Implementation

### 1. Canonicalize the known alias at the Rust observation boundary

Add a small provider-specific helper under `crates/sase_core/src/provider_usage/` and
integrate it with the existing observation validation/normalization path in `mod.rs`.
Use the existing `provider_usage_validate_observation` PyO3 contract, already called by
both Python collection paths and by the store writer; no new wire schema is required for
this identity correction.

For provider `claude`, normalize the exact legacy identity
`window:seven-day-overage-included` to `weekly:claude-fable-5`, a clear Fable label, and
`Models { model_ids: ["claude-fable-5"] }`, matching the existing prose parser. Use a
narrow explicit alias rule, not substring inference from arbitrary labels. Make it
idempotent. Preserve usage percentages, resets, durations, observation times, source,
and vendor state. Preserve the observation's completeness, account context, generation,
and ordering metadata.

Validate incoming data before applying compatibility normalization so malformed
measurements or metadata do not become valid accidentally. Keep the existing
duplicate-key validation contract; any alias/canonical collision within one observation
must have an explicit tested outcome rather than silently dropping the entire passive
event. Prefer a narrow collapse of this verified alias pair, selecting the newer
observation time and the canonical spelling on a tie; unrelated duplicate keys must
still fail validation.

Keep Python transport, fraction-to-percent conversion, and passive event queueing in
place. The Python helper may still assemble its legacy wire record before invoking Rust,
but the validated observation returned to its caller must be canonical. Document that
boundary where useful; do not duplicate the new alias map in Python.

### 2. Recover existing cache records with the same identity rule

Correcting new observations alone leaves the already-stored alias visible until a
successful full refresh. Reuse the Rust helper during decoding of a validated Claude
provider record in `provider_usage/store.rs`, before returning its public snapshot or
merging an incoming observation.

Rekey the stored-window map consistently with each window's key and normalize the
retained last-attempt observation. If alias and canonical windows coexist, retain the
winner by the store's existing `(ordering_token, received_at)` ordering, preferring the
canonical spelling on an exact tie. Never add, average, or choose a larger usage
percentage as a reconciliation shortcut. Preserve unrelated windows and providers.

Handle tombstones deliberately: an old alias tombstone can mean that a complete probe
removed the duplicate while retaining the real Fable window. Do not rename that
tombstone onto the canonical key and thereby suppress valid Fable data. Discard the
obsolete alias tombstone after canonicalization, while respecting a real canonical
tombstone when considering an alias candidate. Preserve generation fencing and the
complete-inventory ordering markers.

`load_provider_usage` must remain read-only. Normalize in memory on reads and let
ordinary successful store writes serialize the normalized records through the existing
atomic write path. No live cache deletion, hand editing, forced network refresh, or new
schema version is necessary. Old processes that still emit the alias should be handled
safely by the fixed store boundary.

Keep normalization bounded by the existing window-count limits and inside the
established backend load/write paths. Do not introduce disk access, subprocesses, or
network requests into indicator rendering or Textual callbacks.

### 3. Verify the binding and frontend integration

Add regression coverage to the Rust provider-usage tests and the PyO3 tests in
`crates/sase_core_py/src/lib.rs`, plus focused Python tests around
`tests/llm_provider/test_claude_usage.py` and the existing usage store/configuration and
indicator presentation suites. New focused test modules are fine if existing files are
at their size limits.

Use synthetic fixed-time payloads and isolated temporary stores. Tests must not read
credentials, invoke a real inference request, or mutate the user's usage cache.

Required cases:

- A passive event with `five_hour`, `seven_day`, and `seven_day_overage_included` yields
  session, weekly-all, and exactly one canonical Fable window. Fractional utilization is
  converted once, and reset/source metadata survives unchanged.
- Probe then passive, passive then probe, and interleaved complete/partial refreshes
  keep one Fable identity. The newest accepted value wins; older late arrivals and stale
  account generations cannot overwrite it or resurrect a deleted window.
- A raw legacy cache fixture containing alias-only and alias-plus-canonical records
  loads correctly, including either record being newer, exact ties, and the alias
  tombstone situation described above. Build that fixture directly so the fixed writer
  cannot normalize away the setup. Loading leaves file contents unchanged; a subsequent
  ordinary write and reload retains canonical state.
- The existing Rust-backed Python validation binding returns the canonical identity, and
  the full store-to-projection-to-renderer path produces a single `fable` label with the
  newest remaining percentage and countdown, without the raw key or `/scope?`. Test
  values below, at, and above 20% remaining, plus a canonical Fable `never` override, so
  the duplicate cannot bypass the configured window policy.
- Unknown future Claude buckets, the separate `overage` bucket, other providers,
  malformed numeric data, and unrelated duplicate keys retain their current behavior.
  Session and weekly-all allowances remain independent. Genuinely unknown selected
  windows continue to display `/scope?`.

Do not change the generic renderer to hide this field, rename every unknown window, or
suppress the unknown-scope suffix globally. No intended styling change means a focused
text-rendering integration regression is sufficient; do not update PNG goldens unless
implementation actually changes that surface.

## Verification and delivery

Build/install the modified core into SASE's workspace environment using the existing
`just rust-install` workflow with the checkout returned by `sase repo open`. Verify
tests use that wheel, rather than the currently installed 0.34.41 wheel observed during
diagnosis. Exercise the new Rust, PyO3, collector, store, and presentation regressions.

Run the Rust repository's `just check` (or `./scripts/check.sh`) so verification
includes the binding crate. `cargo test -p sase_core` alone is insufficient. Run SASE's
`just check` after its file changes. Follow `/sase_monitor` for long checks, running
formatting first as the verification memory requires. Broaden to `just check-full` only
when required by the repository's selection/broadening rules.

Keep source pinning and release integration concrete: SASE's CI builds the SHA in
`sase-core-revision.txt`, and released installations use the published dependency
window. Coordinate the source pin and published minimum through the existing core
release/ratchet workflow once the fixing commit/release exists. Do not invent a future
SHA/version, manually change Rust release versions, or claim an old published wheel
contains the fix. Report any pending release dependency explicitly.

Completion means both fresh events and pre-fix stored duplicates produce one accurate
Fable window, the ordering/configuration regressions pass, both repositories' required
checks pass, and the implementation includes the appropriate binding/source-release
integration. Submit changes through the host-owned finalization workflow.
