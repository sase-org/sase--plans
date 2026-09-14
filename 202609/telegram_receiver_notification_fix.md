---
tier: tale
title: Fix Telegram receiver failure notifications blocking recovery
goal:
  Restore tg_inbound ticks and receiver retry after a launch failure by satisfying the
  notification store contract.
size: small
proposed_by: bbugyi200.athena.0kx
create_time: 2026-09-14 15:10:44
status: wip
---

# Fix Telegram receiver failure notifications blocking recovery

## Confirmed incident

The errors are still occurring on September 14, 2026. The requested digest,
`~/.sase/axe/error_digests/digest_20260914_150345.txt`, contains 100 failures of the
`telegram` lumberjack's `tg_inbound` chop, from 14:52:42 through 15:03:41 EDT. Every
entry ends with `ValueError: dedup_key requires plus_one_note`. The associated priority
notification is from `axe`, timestamp `2026-09-14T15:03:45.329439-04:00`, ID
`bb147db6-d235-44a6-9b06-23a1985bc54a`.

Fresh scheduled run records confirmed the same exception beyond the digest, including
`20260914T150718_561659`, which finished at 15:07:19.990026 EDT with exit code 1. Recent
runs were failing approximately every 6.5 seconds. Read current run records under
`~/.sase/axe/lumberjacks/telegram/chops/tg_inbound/` when implementing; individual run
logs rotate quickly.

The durable receiver history contained 59 rows, all in `error`, with no active receiver.
The newest was proc `a1kmfjg4qwm9`, finished at 13:07:10 EDT. Its termination reason is
`launch-failure`, with
`could not start command: [Errno 2] No such file or directory: 'sase_chop_tg_inbound'`.
Inspect it with `sase proc show a1kmfjg4qwm9 --format json` if still retained. An
executable-resolution fix already exists in sase-telegram commit
`8586f9159ef4577c795e8f3f386879f709e56d11`
(`fix(receiver): anchor telegram proc executables`). The notification exception now
prevents reaching the corrected launch path.

## Root cause and scope

Use `/sase_repo` to open `sase-telegram` before working there, and `sase-core` for the
Rust dependency used by its checks. Use the returned checkout paths. The implementation
belongs in these sase-telegram files:

- `src/sase_telegram/receiver.py`
- `tests/test_receiver.py`

`ensure_receiver_running()` calls `_notify_receiver_launch_failure()` before checking
its five-minute backoff and before submitting a replacement proc. The notification
helper supplies the dedup key `telegram-receiver-launch-failure` to
`upsert_notification()` but omits `plus_one_note`. The Rust store validates that field
even when creating the first notification. Consequently the helper raises on every tick,
creates no alert, and indefinitely blocks retry despite the old failure timestamp.

The shared contract is implemented in sase-core's
`crates/sase_core/src/notifications/store.rs::upsert_notification` and already covered
by
`crates/sase_core/tests/notification_store_parity.rs::notification_upsert_rejects_dedup_key_without_plus_one_note`.
The SASE Python adapter in `src/sase/notifications/store.py` correctly forwards the
supplied fields. Preserve this contract; fix the Telegram call site.

An isolated diagnostic using the actual Python adapter and installed Rust binding
reproduced the exception and observed zero retry submissions for a 301-second-old failed
proc. Temporarily supplying a valid `plus_one_note` allowed both subsequent ensure calls
to reach submission while retaining exactly one notification. No production state or
source code was changed.

## Implementation

1. Supply a non-empty, single-line `plus_one_note` in the receiver helper's
   `upsert_notification()` call. Derive it from the failed proc ID and existing
   normalized launch error message, so an upsert race records useful context. The
   notification timestamp already provides the default plus-one timestamp. Keep the
   existing sender, dedup key, diagnostic notes, log attachment, and snapshot precheck,
   including its treatment of dismissed notifications. This preserves one alert across
   ordinary repeated ticks and lets the Rust store resolve a race atomically.
2. Update the existing notification mocks to accept and validate the keyword argument.
   They currently accept only the notification object, which allowed the invalid request
   to pass tests.
3. Add regression coverage in `tests/test_receiver.py` using a temporary `SASE_HOME` and
   the real notification adapter/store/Rust binding. Mock proc snapshots, credentials,
   executable resolution, and proc submission to keep the tests deterministic and
   independent of Telegram or live processes. Cover these behaviors:
   - A recent launch failure creates the diagnostic notification and returns the failed
     proc during backoff, without submitting a new proc.
   - Repeated calls preserve one notification and its existing plus-one count; also
     cover an already-dismissed matching notification.
   - An expired launch failure with an initially empty notification store creates the
     alert and reaches submission using the resolved executable. This must fail against
     the original code and pass after the fix.
   - Two simulated stale empty snapshot reads both reach the real upsert: one row
     remains, with a valid plus-one for the second request and no validation exception.
     Sequential calls with forced stale reads are enough to exercise this caller
     contract without timing-sensitive concurrency.

## Verification and acceptance

From the opened sase-telegram checkout, run
`just test tests/test_receiver.py tests/test_executables.py`, then `just check`. Its
Justfile installs the local SASE adapter and builds the local Rust binding; ensure those
dependencies resolve to the current SASE workspace and the opened sase-core checkout.
Use its documented source-directory environment overrides if necessary. Use
`/sase_monitor` for commands requiring a long wait. No shared backend API change is
expected.

Acceptance is that an empty notification store can receive the first receiver
launch-failure alert, repeated ticks do not flood the inbox, recent failures retain the
five-minute backoff, and expired failures reach the existing single-owner proc
submission path. The tests must exercise Rust validation.

After the approved fix is landed and installed through the normal host-owned workflow,
inspect fresh scheduled `tg_inbound` run records and the receiver proc history. Confirm
consecutive successful ticks and one active receiver using the resolved executable.
Record this runtime observation separately from test results; until deployment it
remains an unverified recovery step. If a new receiver proc still fails, inspect its new
log to distinguish that failure from the historical bare-executable error. Preserve the
historical proc records and digest as diagnostic evidence.
