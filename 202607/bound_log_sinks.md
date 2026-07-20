---
tier: tale
title: Bound and harden SASE log sinks
goal: 'Durable SASE logs remain bounded and parseable under concurrent writers, completed
  hook captures retain useful head and tail diagnostics without growing indefinitely,
  and atomic writers clean only their own stale temp siblings.

  '
create_time: 2026-07-20 16:58:40
status: wip
prompt: 202607/prompts/bound_log_sinks.md
---

# Plan: Bound and harden SASE log sinks

## Context

The production audit behind bead `sase-8g.10` found several independent durability gaps: `dev_update.jsonl`,
`runs.jsonl`, `events.jsonl`, and some TUI diagnostic sinks could grow indefinitely; one `runs.jsonl` record was torn by
concurrent writers; a completed hook capture reached 17.9 MB; and abandoned atomic-rewrite temp files remained in the
notifications and pending-actions directories.

The current tree has partial protections that should be preserved and consolidated rather than duplicated. TUI telemetry
has a local size check and single-generation rotation, toast history has queueing plus line-count compaction, and
`tui.log` already uses `RotatingFileHandler`. Those paths do not share one concurrency-safe append contract, and the
size check in the telemetry writer is outside its data-file lock. Notification rewrites are implemented in the linked
Rust core, while pending actions and notification-gate JSON use Python atomic writers, so stale-temp cleanup must be
added at the actual writer boundaries on both sides of the core boundary.

## Bounded, atomic durable-log writes

Introduce one internal Python log-writing primitive for the durable sinks in scope. It should serialize a complete
record before taking a dedicated sibling lock, rotate the current file to a single `.1` generation while holding that
lock when the configured size threshold would be crossed, and append each encoded line with one `O_APPEND` `os.write`. A
short/failed write is a failed record, never a sequence of writes that another process can interleave. Keep the existing
best-effort/non-raising behavior for diagnostic-only writers.

Route the named JSONL sinks through that contract: agent runs and events, the dev-update journal, TUI stalls/git
operations/launch timing/agent loads and related telemetry, and toast history. Preserve the toast writer thread and
recent-history API; coordinate any retained line-count compaction with the same sibling lock so it cannot race an append
or rotation. Keep `tui.log` on its existing rotating logging handler, but cover its bound in regression tests and align
the named sinks on explicit, testable defaults and overrides. Readers must continue to skip malformed historical lines
and consume the current generation normally.

Audit adjacent call sites that write these same canonical files so no alternate Python writer bypasses the shared append
path. Do not broaden this into retention changes for agent transcripts, axe lumberjack logs, or unrelated artifact-local
JSONL files, which have different lifecycle contracts.

## Bounded completed hook captures

Add a focused helper for completed hook-output files that preserves a fixed byte-budgeted head and tail (approximately
200 KiB and 300 KiB) separated by a clear elision marker that reports omitted content. Run completion-marker parsing,
retry counting, and any metahook matching that needs the full output before replacing the completed capture; keep the
`===HOOK_COMPLETE===` marker and lint/pytest summaries in the retained tail. Install the bounded result atomically so
TUI, fix-hook, and summarization readers never observe a partially rewritten capture. Running hook files remain
untouched until completion, and small captures remain byte-for-byte unchanged.

Avoid redundant full-file reads in the completion path where practical. The truncation helper must be best-effort:
failure to compact diagnostics cannot change the hook's pass/fail result.

## Stale atomic-write temp cleanup

At Python atomic-write boundaries used by pending actions and notification-gate durability, reap stale temp siblings
immediately before creating a new temp file. Match only the exact target's hidden temp prefix and `.tmp` suffix, use a
24-hour age threshold, ignore races and filesystem errors, and leave fresh or unrelated files untouched. Centralize the
matching/cleanup behavior in a small core utility so the two writers do not drift.

In `sase-core`, apply the equivalent target-scoped cleanup inside the notification store's locked atomic rewrite path;
this is where `.notifications.jsonl.<pid>.<timestamp>.tmp` files are created. Add direct Rust coverage for stale, fresh,
and unrelated siblings. Do not expose a new Python binding unless a frontend needs to invoke cleanup directly; the
behavior belongs in the writer helper and should occur opportunistically on the next rewrite.

## Regression coverage and validation

Add targeted tests for:

- rotation of at least one previously unrotated sink, preservation of the current record, and replacement of only the
  single `.1` generation;
- a two-process `runs.jsonl` append stress case that produces the exact expected record count with every line valid
  JSON, plus tolerant reading of a pre-existing malformed historical line;
- TUI/logging best-effort behavior and the existing toast-history and `tui.log` bounds;
- hook capture no-op below the cap and head/marker/tail preservation above it, including the completion marker and a
  tail summary;
- Python and Rust atomic-writer GC removing only old temp siblings for the same target while preserving fresh and
  unrelated files.

Run `just install` before repository checks so the local Rust binding and dependencies match the workspace. Run focused
Python tests during implementation, focused `cargo test` coverage for the notification store, then finish with
`just check` and `just rust-check`. Review both repository diffs and statuses, re-run any affected focused tests after
final cleanup, and close only bead `sase-8g.10` after all validation passes; leave parent epic `sase-8g` open and do not
create additional beads.
