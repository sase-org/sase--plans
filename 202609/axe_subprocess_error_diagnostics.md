---
tier: tale
title: Preserve useful diagnostics for failed AXE subprocesses
goal:
  AXE error digests retain bounded subprocess failure details and run identity after
  per-run logs are pruned.
size: medium
proposed_by: bbugyi200.athena.00h
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.00h](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.00h.md)
- **COMMITS:**
  - [c3c5e8a](https://github.com/sase-org/sase-core/commit/c3c5e8ac2bf3912465394379486ff21564c4f90b)
    — feat(axe): normalize chop subprocess diagnostics

# Preserve useful diagnostics for failed AXE subprocesses

## Outcome

When a scheduled chop fails, its hourly error digest must show the captured subprocess
error and identify the run. For the reported Telegram incident, the reader should see
`telegram.error.TimedOut: Timed out` and the relevant captured traceback instead of
having to search the lumberjack log behind an uninformative `exit code 1` and
`<no python traceback: subprocess error>`.

This is one bounded implementation across `sase-core` and `sase`. The Telegram plugin
was inspected to establish the cause; its polling behavior needs no change for this
incident.

## Verified incident and current status

The screenshot matches notification `326ee706-b880-4629-8341-e758d5ac695a`, sender
`axe`, timestamp `2026-09-06T21:31:16.089501-04:00`. Its attached report is
`~/.sase/axe/error_digests/digest_20260906_213116.txt`.

- The two `telegram/tg_inbound` failures occurred on September 6, 2026 at
  21:11:42.996558 and 21:12:21.936491 Eastern. Both exited with code 1.
- `~/.sase/axe/logs/lumberjack-telegram.log` contains the actual tracebacks:
  `get_updates(offset=offset, timeout=0)` failed through `httpcore.ReadTimeout` /
  `httpx.ReadTimeout` to `telegram.error.TimedOut: Timed out`. Each invocation exhausted
  the existing three retries, with 2/4/6-second backoff. The first invocation also
  logged `NetworkError` on intermediate attempts. This establishes a transient polling
  transport failure, without establishing whether Telegram or the network caused it.
- Polling recovered at 21:12:28. There were 223 successful poll summaries after the
  final failure through 21:37:09, and all ten retained run records then showed success
  with exit code 0. `recent_errors.json` contained no later Telegram failure. This
  status is an observation window, not a guarantee against recurrence.
- Source inspection confirms retries are already implemented in
  `sase-telegram:src/sase_telegram/telegram_client.py:_with_retry`. Inbound polling
  happens before advancing the saved update offset. No manual replay, offset mutation,
  restart, or live test message is required.
- An earlier 20:28:15 `exit code -7` belongs to a different digest and is not evidence
  of the same timeout cause. Do not fold it into this diagnosis.

Searches across all bead statuses for `tg_inbound`, `no python traceback`,
`error digest`, and subprocess diagnostics found no existing task for this reporting
defect. Existing tasks `sase-wd` (macOS child startup bad file descriptors) and
`sase-ve` (development installs missing Telegram scripts) have different causes and
scope.

## Root cause and implementation locations

At inspected `sase` revision `09c93253d`, `src/sase/axe/chop_runner_script_result.py`
unconditionally assigns `NO_PYTHON_TRACEBACK` to nonzero subprocess exits even though
combined stdout and stderr were streamed to a per-run log.
`lumberjack.py:_outcome_to_result` copies the output into the aggregate log, but
`_handle_error` persists only timestamp, lumberjack, job, error, and the placeholder
traceback. The digest renderer in
`src/sase/notifications/senders.py:notify_axe_error_digest` therefore cannot show the
subprocess evidence.

`src/sase/axe/_state_chops.py` retains only ten terminal runs per chop. At this polling
cadence the failed run logs are pruned long before the hourly digest. Looking up the
output only when rendering the digest cannot solve the defect.

## Implementation

1. **Define the shared diagnostic transformation in Rust.** Open the core repo using
   `sase repo open gh:sase-org/sase-core` with an audit reason, and follow its
   `AGENTS.md`. Add a small typed diagnostic request/result in
   `crates/sase_core/src/axe_chop/` and expose it through
   `crates/sase_core_py/src/lib.rs`, its binding tests, and the thin
   `src/sase/core/axe_chop_facade.py` adapter. Python supplies subprocess metadata and a
   bounded log sample; Rust owns normalization, truncation, and the shared diagnostic
   representation. Use the required binding without a Python fallback. Do not manually
   edit Rust release versions.

2. **Capture output at failure time.** Before finalizing a failed subprocess run, read a
   bounded tail of its combined output. Limit the filesystem read (for example, 64 KiB),
   then retain at most 200 lines and 16 KiB of normalized output, favoring the tail so
   the terminal exception remains visible. Handle malformed UTF-8, giant single lines,
   empty output, and missing/unreadable logs. Record whether output is absent,
   unavailable, or truncated. Strip terminal control sequences and redact Telegram bot
   credentials in captured URLs/token text before retaining the excerpt; bound the final
   serialized excerpt as well. An output-read failure must preserve the original
   subprocess failure.

3. **Carry and persist diagnostics.** Add an optional diagnostic payload through
   `ChopRunOutcome` (`chop_runner_types.py`), `ChopRunEntry` (`_state_chops.py`),
   `_ChopResult`, and `_handle_error` (`lumberjack.py`). Include the run ID, exit code,
   source log path, bounded output excerpt, and availability/truncation information.
   Persist the excerpt in the error record written by
   `_state_scheduler.py:append_error`, so it survives run pruning. Populate the same
   payload for script timeouts where captured output exists; keep signal exits
   recognizable by their exit code. Optional fields must default safely when reading
   historical state. Preserve real exceptions captured inside Python `except` blocks and
   the existing failure/timeout lifecycle and metrics.

4. **Render the evidence in the digest.** Extend `notify_axe_error_digest` with run
   identity and a clearly labeled subprocess output section, including
   truncation/unavailability markers. Do not label all subprocess output as a Python
   traceback; the excerpt itself can contain a real child traceback. Avoid displaying
   the misleading no-Python-traceback placeholder when captured child diagnostics are
   available. Continue rendering old records and genuine host tracebacks. A retained
   source-log path is contextual; the embedded excerpt must remain sufficient after that
   path has been pruned. Describe this behavior in the existing AXE/notification
   documentation.

## Regression coverage and acceptance

- Run a fixture subprocess that writes a chained Python traceback ending in
  `telegram.error.TimedOut: Timed out` and exits 1. Verify the runner outcome, stored
  run, scheduled error record, and digest preserve that terminal cause. Use fake output;
  tests must not call Telegram or process live updates.
- Render that digest after enough successful fixture runs to trigger normal pruning. The
  source failed-run log should be gone and the digest should still contain its
  diagnostic excerpt and run identity.
- Cover non-Python stderr, silent nonzero exit, timeout with partial output, negative
  exit code, unavailable log, control sequences, invalid UTF-8, and a giant line. Assert
  byte/line bounds and visible truncation. Include a synthetic bot-token URL and ensure
  the retained output and digest redact it.
- Keep real host tracebacks, legacy error records, successful runs, and current digest
  time-window/deduplication behavior working. Never introduce `NoneType: None` by
  capturing a traceback outside an active exception handler.
- Extend the existing runner tests in `tests/test_axe_chop_runner_script.py`, traceback
  tests in `tests/test_lumberjack_traceback.py`, state tests, and notification
  sender/digest tests. Add Rust transformation and PyO3 round-trip tests so field
  propagation is exercised across the backend boundary.

## Verification and scope

Before implementation, reread `lint_and_test.md` through `sase memory read`. Use the
opened core checkout for the workspace's binding build, initialize the isolated
environment with `just install` as needed, then run `just check` in `sase`. Run the core
repository's `just check` / `scripts/check.sh` gate, including the binding tests;
`cargo test -p sase_core` alone is insufficient. Use `/sase_monitor` for long
verification, and for any required `just check-full` landing gate. No UI snapshot
changes are expected.

Keep the existing Telegram retry count, polling cadence, exit status, offset semantics,
and error notifications. This plan fixes loss of diagnostic evidence; it does not
attempt to eliminate network timeouts, suppress failures, change retention policy,
repair historical reports, or restart the running services. Completion should report the
diagnostic fix and its tests separately from any fresh read-only observation of live
Telegram health.

<!-- sase:referenced-by:start -->

## Referenced By

| Relation | Artifact                              | Why                                                               | Uses |
| -------- | ------------------------------------- | ----------------------------------------------------------------- | ---: |
| cited-by | [agent:bbugyi200.athena.00h--code][1] | prompt reference @plan:202609/axe_subprocess_error_diagnostics.md |    1 |

[1]: https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.00h.md

<!-- sase:referenced-by:end -->
