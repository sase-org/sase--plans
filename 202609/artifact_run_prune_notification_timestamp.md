---
tier: tale
title: Fix AXE artifact-run pruning preview notification timestamps
size: small
goal:
  Fix recurring AXE artifact_run_prune failures by supplying timezone-aware notification
  timestamps and verifying the chop against the real Rust notification store.
proposed_by: bbugyi200.athena.0le
create_time: 2026-09-15 13:04:32
status: wip
---

# Fix artifact-run pruning preview notification timestamps

## Diagnosis

The user reports this AXE error on athena, mac, and apollo. The supplied
`~/.sase/axe/error_digests/digest_20260915_125800.txt` records the housekeeping
`artifact_run_prune` run at `2026-09-15T12:24:23.087978-04:00`. It successfully previews
100 run directories and 47,590,524 reclaimable bytes, then raises:

```text
ValueError: plus-one timestamp must be a timezone-aware RFC-3339 timestamp
```

The confirmed failure path is:

1. `src/sase/scripts/sase_chop_artifact_run_prune.py` calls
   `_notify_actionable_preview()` whenever runs or empty shards are reclaimable, or a
   protection source is unavailable.
2. The notifier sets both `Notification.timestamp` and `plus_one_timestamp` to
   `local_now().isoformat()`.
3. `src/sase/core/time.py::local_now()` intentionally returns a **naive** datetime in
   the configured timezone, stripping its timezone offset for model arithmetic.
4. `src/sase/notifications/store.py::upsert_notification()` and
   `src/sase/core/notification_store_facade.py` pass the string through to Rust.
5. In the `sase-core` repository,
   `crates/sase_core/src/notifications/store.rs::upsert_notification()` constructs
   `PreparedPlusOne` before opening the store or choosing create versus update. Its
   `parse_aware_utc()` requires RFC-3339 with an offset. Consequently the notification
   fails even when no matching notification exists yet.

An investigation using the workspace interpreter and installed `sase-core-rs` version
`0.34.31` reproduced the exact exception without writing a notification. The current
producer emitted `2026-09-15T13:01:35.543645`; the established aware clock pattern
emitted `2026-09-15T13:01:35.543653-04:00`.

Commit `53c6c51c3` (`fix(retention): preserve protected artifact runs`, September
15, 2026) introduced the notification path and its naive timestamp. The previous chop
only logged the preview. The actionable-preview tests in
`tests/test_axe_chop_output_contract_retention.py` mock `upsert_notification()` and
never validate the timestamp or cross the Rust boundary, explaining the gap.

The current callers in `sase_chop_wait_checks.py`, `main/notify_handler.py`, and
`notifications/cli_plus_one.py` already use `datetime.now(get_timezone()).isoformat()`.
The shared defective producer explains the reported recurrence across machines; remote
installations were not inspected.

## Scope and design

This is a focused repair for one coding agent, so use a **small tale**. Correct the
Python producer to satisfy the existing Rust contract; shared store validation continues
to belong to Rust. No new backend behavior or Rust release is required.

Use `datetime.now(get_timezone()).isoformat()` when creating the preview notification,
consistent with the other producers. Generate it once and use that same value for the
notification and its plus-one timestamp. Keep `local_now()` for
`AceRunRetentionPolicy.now`, whose configured local wall-clock convention is
intentional. Add a short comment at the notification clock if useful to explain the
distinction.

Preserve retention selection and protection rules, the preview-only operation,
notification fingerprints and reports, repeated-preview deduplication, and the existing
`ok`, `no_op`, and `check_error` result semantics. Do not relax Rust's timestamp
validation, reinterpret naive timestamps in the store, change `local_now()` globally,
suppress notification errors, or add a feature flag. This fix requires no
notification-store cleanup or artifact deletion.

## Implementation

1. Add regression coverage before changing the producer. Reuse the chop context and plan
   fixtures in `tests/test_axe_chop_output_contract_retention.py` where practical. Put
   integration cases in a separate focused test module if needed to meet repository
   file-size limits.
2. Run the chop through `run_builtin_chop()` using a synthetic retention plan and an
   isolated temporary SASE home. Stub retention discovery/planning only; keep the actual
   Python notification store, facade, and installed Rust binding. Ensure notification
   JSONL, pending-action registration, and result files all use test paths. Never
   exercise the user's live inbox or artifact lifecycle.
3. Cover these behaviors:
   - A reclaimable preview completes with `status=ok`, produces its summary and report
     notification, and stores a parseable timestamp with an explicit offset in the
     configured timezone.
   - Running the identical preview again keeps one notification with the same original
     ID and creation timestamp, appends one plus-one with an aware timestamp and the
     expected note, and completes successfully.
   - A preview containing unavailable protection sources creates its warning
     notification and returns `status=check_error` with `reason=protection_unavailable`,
     without a timestamp exception. A repeat also deduplicates correctly.
   - Exercise an empty-shards-only actionable preview and retain the existing
     nothing-reclaimable/no-notification test.
4. Use the existing `tz_divergence` fixture for the regression path: configured
   `America/New_York` with host `UTC` proves the clock honors configuration. Strengthen
   the existing captured-payload tests to assert that the notification and plus-one
   timestamps are identical and timezone-aware. For deterministic offset assertions, use
   fixed winter/summer instants or compute the expected offset from the configured zone
   at the captured instant; do not hardcode the current season's `-04:00` offset.
5. Confirm the new integration test fails on the original code with the supplied
   ValueError. Then import `datetime` and `get_timezone` in the chop and change only the
   notification clock to the aware pattern above. Rerun the tests.

## Validation and acceptance

- Run focused tests via the workspace environment, for example
  `uv run pytest -q tests/test_axe_chop_output_contract_retention.py`, including the new
  integration module if one was added. Record the initial failure and successful run
  after the fix.
- Read `lint_and_test.md` through `sase memory read` and run `just check` after
  implementation. Use the documented monitor workflow if a check needs a long run.
  Inspect test selection and follow the note's escalation rules if required.
- Review the final diff for an aware notification timestamp, intact naive policy time,
  real Rust-backed regression coverage, and no artifact deletion behavior.
- The successful result is an actionable preview that reaches its intended summary
  status and notification creation/update on Linux and macOS regardless of host
  timezone. Test coverage must fail if the naive producer is restored.

## Rollout and operational confirmation

The correction ships with the Python SASE package and works with the already installed
Rust dependency. After the approved fix is landed and distributed by the normal
host-owned workflow to athena, mac, and apollo, inspect each machine's next housekeeping
`artifact_run_prune` run and corresponding preview notification. Confirm the timestamp
exception stops and repeated identical previews add a plus-one. If a protection source
is unavailable, the documented `check_error` remains an actionable result and must be
distinguished from this exception. Record which machines were actually verified; this
plan does not assume remote deployment or verification has already happened.

Do not run `sase artifact prune-runs --apply`, clear error history, or reset the
notification store as a workaround. Historical digests remain useful evidence.
