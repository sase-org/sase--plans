---
tier: tale
title: ACE pilot harness cost reduction
goal: 'ACE pilot tests retain their existing assertions and production-startup coverage
  while ordinary AcePage contexts avoid unrelated background services, shut down promptly,
  and meet the epic''s sub-second entry/exit performance targets without flaky sleeps.

  '
create_time: 2026-07-20 11:06:08
status: done
prompt: 202607/prompts/ace_pilot_harness_cost.md
---

# Plan: ACE pilot harness cost reduction

## Context and boundaries

`AcePage` currently patches the ChangeSpec reads but otherwise constructs a complete `AceApp`, starts all post-mount
work, and waits for the mount-state loader. Every context therefore schedules agent/AXE refreshes, prompt-catalog
warming, update checks, a stall watchdog, and filesystem watchers even when a test only needs a mounted widget. Teardown
then follows the production lifecycle, including two watcher stops whose one-second join timeouts line up with the
reported 3.2–4.5 second config-pane teardown durations. The visual harness already demonstrates that startup data
sources can be replaced with deterministic fixtures, but the general pilot harness has no scoped fast-start contract.

This tale will change test infrastructure and, only where profiling proves a production shutdown defect, the shared
cancellation primitive. It will not delete or skip tests, move tests out of the fast lane, weaken assertions, lower
coverage, or alter Rust-core behavior. Tests that intentionally exercise startup loaders, watchers, update checks, or
shutdown persistence must retain an explicit path to the real behavior.

## Measure and isolate the cost

Establish a same-host baseline before implementation for an empty `AcePage` enter/exit and for
`tests/ace/tui/test_config_pane_widget.py`. Split elapsed time across app construction, `run_test().__aenter__`, the
`_mount_state_loads_done` wait, test body, and `run_test().__aexit__`. Use temporary diagnostic instrumentation or
high-resolution timing around the harness rather than permanent sleeps, then repeat after each change so the
optimization addresses measured blockers rather than merely relocating work.

In particular, distinguish time spent cancelling Textual workers from time spent stopping artifact/prompt-source
watchers and draining toast persistence. If a production cleanup operation reaches its timeout because it lacks a
reliable wakeup or cancellation signal, fix that primitive and cover the real lifecycle. If the operation is valid but
irrelevant to ordinary pilot tests, keep production behavior intact and prevent the test harness from starting it by
default.

## Give AcePage a scoped fast-start contract

Extend `src/sase/ace/testing/__init__.py` so ordinary `AcePage` contexts install and remove a well-defined set of
deterministic startup overrides for nonessential background services. Keep the core Textual mount path, widget
composition, ChangeSpec application, and any state required by existing assertions real. The overrides should avoid host
filesystem scans, watchers, prompt-catalog warming, and automatic update work that the page did not request, while
completing any startup flags/callbacks that the mounted UI expects.

Provide an explicit per-page opt-out (or equivalently scoped startup policy) for tests that cover those loaders or
lifecycle services. Make override cleanup exception-safe when app construction or mount fails, and preserve
caller-supplied patches/fixtures such as the visual startup data so the harness does not silently replace intentional
test state. Consolidate existing deterministic startup helpers only where doing so reduces duplicate patching without
coupling production code to pytest.

## Make shutdown cancellation prompt and structural

Use the profile to remove any remaining teardown timeout. Prefer explicit cancellation and wakeup over shorter timeouts:
stop timers and pump-free tasks, wake watcher threads before joining them, and ensure background work cannot call back
into an unmounted app. Keep production cleanup idempotent because both controlled quit and Textual unmount may invoke
it. Do not wait on work that the fast harness never started, and do not introduce fire-and-forget cleanup that leaks
threads, file descriptors, workers, or callbacks into a later test.

Add focused tests around the selected boundary. Harness tests should prove default contexts suppress only the named
nonessential services, the real-startup opt-out invokes them, overrides are restored after success and failure, and no
tasks/watchers remain owned by a closed page. If the watcher or another shared lifecycle primitive changes, add a
deterministic blocked-wait test showing that stop wakes and joins promptly without sleep-based synchronization.

## Validation and performance acceptance

Keep all existing config-pane assertions unchanged and repeatedly run its complete test file with pytest phase durations
enabled. Add or adapt a small `AcePage` regression test for the fast-start contract, favoring structural assertions over
a tight wall-clock check so normal concurrent host load cannot create flakes. Record separate repeated measurements
showing that an empty enter/exit is well under one second and representative config-pane teardowns are below roughly 0.5
seconds.

Then run the focused harness, lifecycle/watcher, startup, config-pane, and representative Agents/visual pilot tests,
including the explicit real-startup cases, more than once to catch leaked global patches and teardown races. Finish with
`just install` followed by `just check`, inspect the slow-duration table for the pilot bucket, and verify the selected
test count, markers, snapshots, and assertions have not been reduced. Only after these checks pass should `sase-86.2` be
closed; its parent epic remains open.
