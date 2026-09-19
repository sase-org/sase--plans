---
tier: tale
title: Refresh the Telegram receiver when its loaded runtime becomes stale
goal:
  Inbound Telegram commands and launches survive SASE updates without duplicate
  consumers or skipped updates.
size: medium
proposed_by: bbugyi200.apollo.0n
create_time: 2026-09-18 20:53:44
status: wip
---

# Refresh the Telegram receiver when its loaded runtime becomes stale

## Outcome

Restore inbound Telegram slash commands and agent launches on Athena immediately, and
make the persistent receiver replace its interpreter whenever the installed SASE,
sase-telegram, or `sase_core_rs` runtime changes. The replacement must preserve the
single `getUpdates` consumer and the durable Telegram offset, so an update received at
the code-change boundary is handled once by a fresh runtime rather than acknowledged and
lost by a mixed old/new process.

## Confirmed diagnosis

- Athena is the enabled Telegram host: `~/.sase/telegram_is_enabled` exists, the AXE
  `telegram` routine is healthy on its five-second cadence, and its `tg_inbound` and
  `tg_outbound` jobs are configured.
- The inbound receiver is running, but its proc began on 2026-09-14 and still invokes
  the legacy `sase_chop_tg_inbound --receiver` entry point. The currently installed
  plugin resolves the canonical receiver to `sase_job_tg_inbound --receiver`, proving
  that the persistent process predates in-place source and environment updates.
- Its log shows Telegram updates arriving. Callback queries continue to work, while
  `/list` fails on an old `runner_capacity_snapshot` calling convention and free-form
  agent launch fails because the cached `_CollectedDirectives` type lacks the newer
  `hold_occurrences` field. The receiver catches each handler exception and advances
  `update_offset.txt`, which explains the user-visible silent failure.
- A fresh interpreter in Athena's same uv-tool environment exposes
  `sase_core_rs.agent_hold_list` and the current `runner_capacity_snapshot(request)`
  signature. Therefore the current checkout/wheel state is healthy; the stale binding
  warning in the receiver log comes from its cached pre-update extension, not from a
  still-broken installation.
- Restarting ACE/AXE does not recycle this detached proc. Stable request fingerprinting
  intentionally replays the same live receiver, and the service-host migration keeps a
  compatibility branch while Athena's `service_host` flag is off. A one-time kill will
  repair the current incident, but without a code change the next in-place update can
  recreate it.

## Implementation

1. Add a receiver-runtime generation helper in `sase-telegram`.
   - Derive a deterministic fingerprint from the canonical receiver executable and the
     resolved installed code roots for `sase`, `sase_telegram`, and `sase_core_rs`.
     Include stable file identity facts such as relative path, size, and nanosecond
     modification time for regular runtime files; ignore `__pycache__`, `.pyc`, and
     other interpreter-written cache files so normal imports never trigger a refresh.
   - Treat a missing, replaced, added, or modified runtime file as a generation change.
     Make transient scan errors fail safe: stop polling, wait for the on-disk generation
     to settle across two observations, and then refresh instead of consuming an update
     with a possibly torn environment.
   - Keep the helper independent of Telegram credentials and persistent state, and add
     focused unit tests for unchanged trees, host/plugin/native-extension changes,
     executable replacement, ignored cache churn, and a temporarily unstable scan.

2. Re-exec the persistent receiver at safe update boundaries.
   - Capture the baseline generation when `--receiver` starts. Recheck before every long
     poll and immediately after `getUpdates` returns, before dispatching each update. If
     the generation changed, re-exec the canonical `sase_job_tg_inbound --receiver`
     entry point in the same environment. Re-exec keeps the existing supervised process
     slot and concurrency ownership, preventing a second Telegram consumer under both
     the legacy proc supervisor and future service host ownership.
   - If an update was fetched just before change detection, do not save its offset; the
     fresh receiver must fetch it again. Preserve the existing per-update offset rule
     for updates that completed normally.
   - If re-exec fails, log a bounded diagnostic and exit nonzero. The legacy five-second
     ensure tick will submit a replacement, while the declarative service proc's
     `restart: on-failure` policy will restart it after service-host rollout. Clean
     exits for Telegram disablement or credential loss remain unchanged and must not
     loop.
   - Add integration tests proving no-op polling on an unchanged generation, refresh
     before polling, refresh after a fetch without offset advancement, canonical argv,
     exec-failure behavior, and unchanged disabled/credential-loss semantics.

3. Document and verify the lifecycle contract.
   - Update `docs/inbound.md` and `docs/architecture.md` to explain that the receiver is
     long-lived but generation-aware, how it avoids mixed imports across editable or
     managed updates, and why offsets make the handoff lossless.
   - Run the receiver/inbound test modules while iterating, then run the repository's
     full `just check`. Review the diff and confirm only `sase-telegram` code, tests,
     and documentation needed for this fix changed.

4. Recover and revalidate Athena without broad process disruption.
   - Resolve the currently active proc by exact `origin=telegram-receiver`; do not use a
     stale hard-coded proc id or kill unrelated procs. Record its start time, argv, and
     last log lines, stop that one proc through `sase proc kill`, and let the existing
     five-second `tg_inbound` job re-arm it.
   - Confirm exactly one replacement receiver is running with a new start time and the
     canonical `sase_job_tg_inbound --receiver` argv. Verify its log has no
     mixed-runtime traceback, the fresh interpreter's binding/signature probe remains
     healthy, and a read-only slash command plus one disposable agent-launch message are
     processed. If interactive Telegram input is unavailable to the implementing agent,
     report the exact two user checks instead of fabricating end-to-end success.

## Acceptance criteria

- Athena has exactly one active Telegram receiver, and inbound slash commands and
  free-form agent launches no longer fail with the observed signature/type errors.
- Changing any monitored SASE/plugin/native runtime file causes the receiver to re-exec
  once after the new generation settles; cache-file churn does not.
- An update fetched across the refresh boundary is not skipped: its offset remains
  unadvanced until the fresh receiver dispatches it successfully or applies the existing
  bad-update policy.
- The solution works with both legacy durable-proc ownership and service-host
  `restart: on-failure`, without a duplicate `getUpdates` consumer.
- Existing callback, media, command-registration, disablement, credential-loss, and
  per-update offset tests remain green, and `just check` passes in `sase-telegram`.

## Non-goals

- Do not enable the `service_host` beta flag or edit Athena/Apollo machine overlays;
  live service enablement remains a separate rollout concern.
- Do not change the shared SASE or sase-core APIs: the fresh-runtime probes prove those
  installations are currently compatible, and this defect is receiver lifecycle
  ownership rather than backend behavior.
- Do not reset Telegram offsets, pending actions, command caches, or credentials.
