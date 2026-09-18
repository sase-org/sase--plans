---
tier: tale
title: Stop non-actionable managed-scratch notifications
goal:
  Normal soft-target excursions stay quiet while real storage and hardware faults still
  notify.
size: small
proposed_by: bbugyi200.athena.0n6
create_time: 2026-09-18 15:09:43
status: wip
---

# Stop non-actionable managed-scratch notifications

## Context

The Athena `poseidon-cache-watch` timer has emitted repeated pairs of notifications
since it was installed: a warning that managed SASE scratch is above the 16 GiB soft
target with "no reaper progress," followed shortly afterward by a recovery. Recent
history contains roughly fifteen such warning/recovery cycles in four days. At the
latest warning the watcher recorded 96,431,791,819 bytes of scratch while the root
filesystem still had 172,230,717,440 bytes free, the watcher service was succeeding, and
a later reaper pass reduced the scratch tree normally.

The warning predicate in the linked `chezmoi` repository's
`home/bin/executable_poseidon-cache-watch` does not actually observe reaper execution,
eligible entries, removal failures, or live builds. It treats two hourly samples above
16 GiB whose total size did not decrease as proof that the reaper is stuck. That is an
invalid inference: 16 GiB is the threshold for SASE's age- and liveness-safe pressure
pass, not a hard size invariant; fresh or active build trees are intentionally retained
and can keep growing between samples. The same watcher already has a separate,
actionable root-filesystem free-space check, and SASE's own hourly managed-temp job owns
cleanup policy.

## Implementation

1. In the configured linked `chezmoi` repository, remove the hourly managed-scratch
   total sampler and its `scratch` warning/recovery transition from
   `home/bin/executable_poseidon-cache-watch`. Remove its call from `run_checks` and any
   watcher-only settings/helpers that become unused. Do not emit a final recovery while
   retiring the check: existing `delivered_scratch`, `level_scratch`, and sample fields
   in the runtime state file should simply become inert so deployment itself does not
   create another non-actionable notification.
2. Remove `POSEIDON_SCRATCH_SOFT_BYTES` from `home/lib/poseidon_cache.sh` if it has no
   remaining consumer. Keep the actual fault checks unchanged: low root free space,
   Poseidon mount/capacity, retired-target drift, sccache configuration, measurement
   failures for retained signals, and SMART health must continue to notify on their
   existing state transitions and hysteresis.
3. Replace the current test that expects a warning from two high scratch samples with a
   regression in `tests/bash/poseidon_cache_watch_test.sh` that runs the watcher
   repeatedly with safe root free space and large/changing fake scratch totals and
   proves that neither a soft-target warning nor a recovery is emitted. Add focused
   coverage for the retained root-space signal: crossing the low-free-space floor alerts
   once, repeated unhealthy checks stay quiet, the hysteresis band does not recover
   early, and recovery above the configured threshold is emitted only after a real
   alert.
4. Update `docs/poseidon_cargo_cache.md` to state that exceeding the managed-temp soft
   target is not independently alertable. Explain that SASE owns age-safe cleanup and
   that the watcher reports actual root free-space pressure instead, so operators know
   which notification remains actionable.

## Validation

1. Run the focused Bash watcher test file with `bashunit` to exercise the false-alert
   regression and retained root-pressure transitions.
2. Run the linked `chezmoi` repository's `just check` gate to cover formatting, shell
   lint, all Bash tests, and the repository's other configured checks.
3. Confirm the managed source no longer contains the removed soft-target notification
   text or a live `scratch` transition, while the low-root-space notification and its
   recovery hysteresis remain present.
4. After the host-owned commit is applied through the repository's required
   `chezmoi update -a --force` workflow, confirm the deployed watcher matches the
   managed source and a timer invocation exits successfully without emitting a scratch
   notification. Do not delete the old watcher state or dismiss existing notifications
   as part of this fix.

## Expected outcome

Normal build-driven excursions above the 16 GiB managed-temp soft target no longer
create warning/recovery chatter. The user still receives transition-deduplicated alerts
when a real storage or hardware condition requires attention, including low root free
space, and recoveries remain tied only to alerts for those real conditions.
