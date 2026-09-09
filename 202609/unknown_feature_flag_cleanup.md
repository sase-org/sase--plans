---
tier: tale
title: Automatically clean unknown saved feature flags with a distinct notice
goal:
  Remove unregistered keys from valid machine-local feature-flag state during process
  startup, show an accurate and visually distinct cleanup notice in ACE, and eliminate
  repeated state warnings after successful cleanup.
size: medium
proposed_by: bbugyi200.athena.0hh.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.research.1q.cdx](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.research.1q.cdx/README.md)
  - [bbugyi200.athena.research.1q.cld](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.research.1q.cld/README.md)
- **COMMITS:**
  - [d67efe2](https://github.com/sase-org/sase--research/commit/d67efe2013f8902768584347e523e32f71c812aa)
    — docs(research): recommend a machine-as-attribute UX for remote dispatch
  - [f406806](https://github.com/sase-org/sase--research/commit/f4068068fb1414f336de74ce0a0a8a96c7064821)
    — docs(research): recommend unified remote agent UX

# Plan: Automatically clean unknown saved feature flags

## Outcome and scope

Starting `sase ace` with a valid saved entry such as `remote_dispatch: false` should
remove that entry from `$SASE_HOME/feature_flags.json` and show a clear cleanup toast
once the application is ready. Later starts should be quiet unless new unknown entries
appear or cleanup fails.

This is one medium tale: the storage transaction, binding, startup coordination,
presentation, and regression coverage form a bounded change that one follow-up
implementation agent can complete. Implement the Rust core support first and its Python
callers second. No independent phase agents are needed.

Automatic cleanup belongs at the existing process installation boundaries used by ACE,
AXE, and the agent runner. Keep the pure resolver and ordinary `current_flags()`/store
inspection calls free of cleanup writes. Consequently, `sase flag list`/`show` and
doctor can still diagnose a stale file before an installing process cleans it. No new
CLI command, option, configuration key, or feature flag is needed.

Only remove syntactically valid keys absent from the complete registry of this running
SASE version. Preserve every registered saved value, including explicit values equal to
the default and values shadowed by environment or CLI overrides. Do not use the set of
enabled or non-default flags as the registry.

This deliberately changes the existing downgrade policy: an unknown saved key could be
retired, misspelled, or belong to a newer release; startup will remove all three. A
later upgrade will use registry/config defaults until that choice is saved again.
Document this plainly. User-authored YAML, overlay/local config, and inherited
environment values retain their existing diagnostics and are not rewritten by this
feature.

## Evidence and implementation locations

- `src/sase/feature_flags/snapshot.py` reads saved state in `_saved_state_input`,
  combines it in `_build_snapshot`, caches it in `current_flags`, and emits the
  recurring warning in `_log_install`. `install_process_feature_flags` is idempotent but
  can receive a snapshot built by an earlier reader.
- `src/sase/feature_flags/resolver.py` creates `unknown_key` diagnostics for several
  sources. The cleanup target is specifically saved state, identified from saved keys
  and the registry, never parsed from diagnostic prose.
- `src/sase/feature_flags/state.py` is already a strict, thin Rust adapter. Existing
  get/set wire payloads have exact field validation and version 1.
- Open `sase-core` with `/sase_repo` and
  `sase repo open sase-core -r "Implement transactional unknown saved feature-flag cleanup"`.
  Use only its printed path. The relevant files relative to that repository are
  `crates/sase_core/src/feature_flag_state.rs`, `crates/sase_core/src/lib.rs`, and
  `crates/sase_core_py/src/lib.rs`. The store already owns bounded locking, file
  validation, atomic replacement, and tests for concurrent setters and write failure.
- `src/sase/main/ace_handler.py` disables local config and installs flags before
  creating `AceApp`. AXE and agent-runner boundaries are in
  `src/sase/main/axe_handler.py` and `src/sase/axe/run_agent_runner.py`.
- ACE startup scheduling lives in `src/sase/ace/tui/actions/_startup_loads.py` and
  `_startup_mount.py`. `_maybe_end_startup_stopwatch` identifies readiness of the
  initially visible surface. `AceApp.notify` in `src/sase/ace/tui/app.py` records toast
  history.
- `docs/configuration.md`, under Saved machine preferences, currently promises
  preservation of unknown keys across downgrades. Update that promise.

## 1. Add a transactional Rust reconciliation operation

Add an additive API such as `feature_flag_state_reconcile(sase_home, registered_keys)`.
Pass the authoritative registry keys from Python; Rust owns comparison, locking,
validation, mutation, and the structured result. Do not duplicate the registry in Rust
or implement a Python JSON read/modify/write fallback.

The operation must:

1. Validate the explicit registered-key input before touching state. An empty registry
   is valid and means every valid saved entry is unregistered; a missing or malformed
   registry input is an error, not an empty registry.
2. Acquire the same bounded exclusive lock used by get/set, reread the latest file while
   holding it, and calculate removals from that file. Never write back the earlier
   Python snapshot. This preserves concurrent changes to registered keys and serializes
   competing cleanup processes.
3. For a valid file, retain registered entries exactly and perform a single existing
   atomic replacement only when at least one key is removed. An all-unknown file becomes
   a valid version-1 file with an empty flags object.
4. Leave a missing state file absent and a clean existing file byte-for-byte unchanged,
   including its modification time. Existing lock sidecars may still be used; no new
   backup, tombstone, or cleanup-journal format is required.
5. Preserve malformed JSON, invalid UTF-8, oversized data, unsupported versions, invalid
   top-level schema, invalid keys, and non-boolean values. Return the existing
   diagnostic semantics; do not salvage and overwrite part of an unusable document.
6. Return a typed, versioned outcome with the resulting/latest readable flags, state
   path, sorted keys actually removed, and diagnostics sufficient to distinguish
   successful cleanup, a valid no-op, and failure/unusable state. Make that distinction
   explicit (for example, a status of `cleaned`, `unchanged`, `unusable`, or `failed`);
   an empty flags mapping alone cannot establish that a file was successfully cleaned.
   Prefer a new outcome type so existing get/set payloads and the on-disk version remain
   compatible. Unknown valid keys remain round-trippable through the registry-neutral
   get/set APIs.
7. Represent expected operational cleanup failures without claiming deletion. Preserve
   readable preferences on failed replacement; leave the destination intact and clean up
   temporary files. Lock/read failure must report its path and reason. Track the
   replacement commit point: a subsequent lock-release problem must not claim that a
   completed replacement was undone.

Expose/export this operation and outcome through the Rust public API and PyO3. Release
the GIL while waiting for the store lock and doing file I/O so invoking the binding from
a Python worker actually leaves ACE responsive. Keep missing bindings, incompatible wire
schemas, and invalid API inputs distinguishable from recoverable filesystem/lock
diagnostics.

## 2. Coordinate cleanup with process installation

Add a strict immutable Python outcome decoder and thin adapter in
`src/sase/feature_flags/state.py` (split focused helpers if file-size gates require it).
Preserve the hard Rust dependency and existing strict field/type checks. Do not turn a
stale wheel or malformed binding result into a successful or silently skipped cleanup.

Give `install_process_feature_flags` an internal keyword-only deferred-cleanup mode for
ACE; default installation runs reconciliation synchronously for AXE and the runner.
Reuse one coordinator for both execution modes:

- Capture the complete registry and the unknown saved keys observed at installation. A
  valid cached snapshot must not bypass this first-install check. Do not add state
  writes to imports, completion, the pure resolver, ordinary snapshot reads, or test
  override contexts.
- With no observed unknown saved keys, schedule no maintenance and print no cleanup
  notice. With candidates, reconcile once; a subsequent process startup will detect
  entries added after this inspection.
- For synchronous installation, settle cleanup before logging the installed snapshot.
  For ACE, retain a typed pending request/result in process memory until the startup
  worker delivers it. Cache invalidation must not silently discard the pending request
  or the eventual notice.
- Suppress only `source == "state"` / `code == "unknown_key"` installation warnings
  delegated to this cleanup attempt. Keep every other source and diagnostic visible.
  Never suppress unknown YAML/environment keys just because the same key was removed
  from state.
- After successful reconciliation, update saved-state/diagnostic metadata so the Flags
  pane does not retain obsolete state warnings. Preserve installed effective decisions,
  registry defaults, provenance, CLI/test pins, and inherited transport semantics. A
  concurrent registered preference change must not hot-switch a running process's
  effective flags.
- Read/modify process metadata under its existing short lock, but do not hold that
  Python lock during deferred Rust I/O. Publish the result without overwriting a newer
  snapshot or losing concurrent invalidation. Use a guarded pending/running/completed
  lifecycle so repeated install/startup callbacks cannot duplicate writes or notices.
- On operational failure, continue using the already resolved valid preferences and show
  an accurate cleanup-failed warning. If there was no readable state, retain the
  existing lower-layer resolution behavior. Do not retry every refresh/key press; retry
  on the next startup. A scheduling failure must also surface the deferred problem
  rather than hiding it forever.

For a race where this installation observed stale keys but another process removed them
first, use the valid transaction result to confirm that they are now absent. Distinguish
"already removed" from keys this transaction removed. This matters when ACE auto-starts
AXE before ACE's deferred cleanup executes. Do not infer successful cleanup from an
unreadable or unusable file.

## 3. Show a distinct, accurate notice

ACE calls installation with cleanup deferred. Once its initially visible surface is
ready and the startup stopwatch has ended, schedule one background worker for the
pending request. The scheduling callback must be synchronous and thin; disk I/O,
locking, and Rust calls run off the event loop and serial message pump. Marshal the
outcome back to the UI thread and use `AceApp.notify` so the notice is preserved in
normal toast history. Teardown cancels/guards delivery through the existing worker
lifecycle without making cleanup gate app exit.

Use an information toast with the title **Feature flags cleaned up**, a bold green
**AUTO-CLEANED** label, and a 15-second timeout. Example visible copy:

```text
Feature flags cleaned up
AUTO-CLEANED
Removed 1 unregistered saved flag: remote_dispatch
State: /example/sase-home/feature_flags.json
```

Use singular/plural counts, deterministic key ordering, and a compact list (at most five
keys, with bounded individual display lengths and an overflow count). Keep the complete
keys in the typed result. Make the meaning clear without color; use existing theme-aware
information-toast treatment plus the success label, not global toast CSS changes. Escape
paths and other dynamic text using the appropriate Rich/Textual facilities.

When another process won the cleanup race, use an information notice titled **Feature
flag state cleaned up**, with **ALREADY CLEAN** and copy such as "Previously
unregistered saved flag is already absent: remote_dispatch". It is shown only for
candidates observed by this launch, not on every clean startup.

For failure, use a warning toast titled **Feature flag cleanup failed**, with the state
path, actionable underlying reason, and "Will retry on next startup" when the attempt
did not commit. Do not report removed keys or say "left unchanged" if a replacement
committed before a later error. Retain existing corruption diagnostics; success is not a
warning diagnostic and should not make the doctor or Flags pane show a false unhealthy
state.

AXE/runner synchronous installation uses the same result/copy contract through a compact
Rich success panel or labeled block on **stderr**. Honor normal color/TTY detection and
provide readable plain text when redirected. Emit no cleanup prose on stdout, including
for `sase axe status --json`. Avoid a second visible success through warning-level
logging. Repeated installation in one process emits no duplicate notice; a clean later
process emits none.

## 4. Regression coverage and documentation

Use temporary `SASE_HOME` directories and test-owned registries throughout; do not clean
the user's live state file during implementation or verification.

Extend Rust store tests and PyO3 binding tests to cover:

- Mixed registered/unknown entries, both boolean values, exact preservation of
  registered choices, multiple removals with deterministic results, an empty registry,
  invalid registry input, and an all-unknown file.
- Missing/empty/clean state and repeated reconciliation with no rewrite, using bytes and
  modification time to verify idempotence.
- Every unusable-file category above, bounded lock timeout, failed atomic replacement,
  and the absence of false success or temporary-file litter.
- Competing cleanup operations and a concurrent setter of a registered key: exactly one
  transaction removes a given stale entry; no registered update is lost. Use
  barriers/controlled locks rather than timing-only race assertions.
- Exported binding shape, strict decoding, and actual execution through the new binding,
  including a Python-thread responsiveness check while its lock wait releases the GIL.

Extend `tests/feature_flags/test_state.py`, `test_snapshot.py`, and `test_boundaries.py`
or focused adjacent modules. Verify cached-before-install behavior, pure reads remaining
non-mutating, effective precedence/transport unchanged, metadata settling after cleanup,
failure followed by a successful new-process retry, idempotent install, and
source-specific warning handling. Keep raw get/set unknown-key preservation tests;
update only assertions whose installation behavior deliberately changes.

Add focused ACE startup/notice tests for post-readiness scheduling, one toast, no
UI-thread I/O, no Python cache lock held during a slow worker, concurrent snapshot
invalidation, race/already-clean copy, worker/scheduling failure, teardown, literal
markup-looking paths, and multiple/long keys. Exercise the first-start cleanup and
second-start silence journey with real temporary state. Add a targeted visual snapshot
for the cleanup toast and inspect its rendered output for readability. Verify non-TTY
stderr and JSON stdout separation for non-TUI installation. Reset any new process state
in the existing test reset helper to avoid cross-test contamination.

Update the Saved machine preferences section of `docs/configuration.md` with cleanup
timing, state-only scope, downgrade implications, successful/no-op messaging, and
failure preservation/retry. Leave the documented tolerance of unknown keys in portable
YAML intact. Update store/API docstrings that would otherwise imply installation always
preserves unknown keys.

## 5. Binding delivery and verification

Read each repository's current instructions before implementing. Build and exercise the
changed Rust binding against the Python caller using the supported local Rust-install
workflow and the checkout returned by `sase repo open`.

Extend `tools/smoke_sase_core_rs_feature_flag_state` and its tests to exercise
reconciliation; make the required-binding check include the new API. Before the Python
caller is released, its `sase-core-rs` dependency floor must refer to an actually
published version exposing that API. Use the existing release/ratchet workflow; do not
invent a future version, manually edit Rust release-plz-owned versions, or hide a stale
wheel with a fallback. Treat the Rust release and Python dependency-floor compatibility
as a delivery prerequisite for this tale, not an unrelated follow-up.

Run `just check` from the opened Rust repository; its script checks formatting, clippy,
and the entire workspace including PyO3. Core-only `cargo test -p sase_core` is
insufficient. Run the focused Python/ACE tests and inspect the new visual snapshot, then
run `just check` in the SASE repository. Follow the `lint_and_test.md` escalation rules
for `just check-full` and use `/sase_monitor` for long-running checks. Validate the
published minimum with the existing required-binding and feature-flag-state smoke checks
before release.

## Acceptance criteria

1. With valid saved `remote_dispatch: false`, ACE becomes responsive, then the entry
   disappears and a distinct, accurate toast appears once. A subsequent fresh start
   shows neither the old state warning nor a cleanup toast.
2. Registered values and concurrent registered updates survive exactly; cleanup neither
   changes effective precedence nor restarts ACE/AXE or launches agents.
3. Unknown config/environment entries remain diagnosable. Invalid whole state files
   remain intact. Failure is reported honestly and can recover next start.
4. ACE remains responsive during a slow cleanup lock/write; notices survive startup
   cache activity and handle another process cleaning first.
5. JSON stdout stays valid, normal clean starts do not rewrite state, required
   Rust/Python/visual checks pass, and supported published installs contain the required
   binding.

<!-- sase:referenced-by:start -->

## Referenced By

| Relation | Artifact | Why | Uses |
| --- | --- | --- | ---: |
| cited-by | [agent:bbugyi200.athena.0hh.f0--code][1] | prompt reference @plan:202609/unknown_feature_flag_cleanup.md | 1 |

[1]: https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0hh.f0.md

<!-- sase:referenced-by:end -->
