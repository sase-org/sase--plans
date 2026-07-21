---
tier: tale
title: Re-anchorable date bounds and inclusive until days
goal: Date filters retain stable semantic specifications, relative windows resolve
  against the current operation time, and day-granular until bounds include the full
  named day across commits and plans.
create_time: 2026-07-21 10:21:23
status: done
bead: sase-8h.1
parent: sase/repos/plans/202607/commits_filter_correctness.md
prompt: '[202607/prompts/reanchorable_date_bounds.md](prompts/reanchorable_date_bounds.md)'
---

# Plan: Re-anchorable date bounds and inclusive until days

## Context and scope

The commit query parser currently resolves every date token to an epoch immediately. That makes relative filters such as
`since:24h` unequal when the same canonical query is parsed again, pins long-lived TUI sessions to parse time, and
treats a date-only `until:` as midnight at the start of the named day. This phase replaces that parse-time epoch with a
stable bound specification, resolves it only where an epoch is consumed, and aligns the adjacent Plans filtering
behavior. It deliberately leaves collection truncation, widened git prefetch windows, and capped-status UI changes to
the later phases of epic `sase-8h`.

No Rust core or VCS wire-format changes are needed. The provider-neutral `CommitFilters` boundary remains an epoch pair.
Existing TUI workers, refresh scheduling, collection caching, and render paths remain structurally unchanged; date
resolution is cheap, in-memory work performed at their existing call sites.

## Date-bound model and resolution

Refactor `src/sase/vcs_log/dates.py` around a small frozen `TimeBound` value produced by `parse_time_bound`. The value
will retain normalized source semantics and structured payload for relative offsets, `today`/`yesterday`, absolute
calendar days, and minute-precise instants. It will expose resolution against an explicit reference time and boundary
direction:

- `since` resolves day-granular values to local start-of-day.
- `until` resolves day-granular values to the final epoch second before the next local midnight, so DST transitions are
  handled by constructing adjacent configured-timezone midnights rather than adding a fixed 24 hours.
- Relative and minute-precise forms retain exact-instant semantics for either direction.

Keep `VcsLogDateError` and the accepted grammar stable, and update `DATE_HELP` to explain that day-granular `until`
values are inclusive. Centralize configured-timezone normalization so production callers can capture one real reference
time per operation while tests inject deterministic aware or naive references.

## Stable commit query values

Change `CommitLogFilterValues` in `src/sase/vcs_log/filter_query.py` so `since` and `until` carry frozen bound specs
rather than epochs while `since_text` and `until_text` remain the canonical rendering inputs. Parsing the same query
text must produce equal, hash-identical values regardless of elapsed wall time, keeping canonical round trips and
dictionary cache keys stable.

During query validation, capture one shared reference time and resolve `since` as a lower boundary and `until` as an
upper boundary before comparing them. This makes `since:X until:X` a valid full-day window for day-granular forms
without changing existing error tokens, spans, or messages for genuinely inverted windows.

Add explicit reference-time inputs to `backend_filters` and `compile_commit_matcher`. Resolve both bounds once per call,
then continue producing the existing epoch-based `CommitFilters` and cheap predicate. Mechanically migrate the
commit-pane collection/filtering call sites and bound-presence checks without introducing new refresh paths, blocking
work, or cache shapes. Each collection or in-memory filtering operation should use one captured reference so both ends
of a window agree.

## CLI and Plans alignment

Update `src/sase/main/vcs_handler.py` to parse bound specs, resolve them once at command execution, apply start/end
boundary direction correctly, and perform the current empty-window validation on the resolved epochs. The downstream
collector and renderer continue receiving `CommitFilters`, preserving provider and JSON contracts.

Migrate `src/sase/plan_search/filter_query.py` to resolve the shared `TimeBound` API with one parse-time reference,
using inclusive end-of-day semantics for `until:` while retaining the plan filter value shape and all other matching
behavior. Update its in-memory tests accordingly. The Rust-backed `sase plan search` CLI already filters `YYYY-MM-DD`
values inclusively at the date level and supports additional month-granular grammar; preserve that behavior and add
CLI-level regression coverage proving a plan created during the named `--until` day remains included rather than
replacing its established parser or changing Rust.

## Verification

Extend focused tests to cover:

- stable parsing and equality of relative `TimeBound` values plus injected-time re-resolution;
- start/end resolution for ISO days, `today`, and `yesterday`, including spring-forward and fall-back
  configured-timezone boundaries;
- exact-instant behavior for relative and minute-precise bounds;
- same-day query validity, unchanged canonical rendering, stable equality/hashing across delayed reparses, and
  explicit-time backend/matcher behavior;
- VCS CLI propagation of end-of-day epochs and preservation of invalid-window diagnostics;
- Plans filter inclusive-day matching and `sase plan search --until` inclusion of an entry created later on that day;
- affected commit-pane cache/snapshot behavior and deterministic visual fixtures after the bound representation changes.

Run `just install` first as required for an ephemeral workspace, then the focused date/query/CLI/Plans/TUI tests.
Finally run `just check`, address every failure, and rerun focused regressions where any fix touches the date or cache
seams. Inspect the final diff and working tree before closing only `sase-8h.1`; leave parent epic `sase-8h` and sibling
beads open.
