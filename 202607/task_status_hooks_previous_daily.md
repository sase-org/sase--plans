---
tier: tale
title: Rolling daily-link reconciliation for Obsidian task statuses
goal: "bob task-status-hooks uses today and the latest existing earlier daily note to
  recognize recently active task links, while stale in-progress tasks in area and
  project notes are reset safely and observably.

  "
create_time: 2026-09-09 19:53:25
status: wip
---

# Plan: Rolling daily-link reconciliation for Obsidian task statuses

## Context and behavioral contract

`bob task-status-hooks` currently treats the selected current daily note as both the
mutable Pomodoro ledger and the sole source for task-status reachability. It promotes
tasks reachable from live links beneath open Pomodoros, follows the existing transcluded
dependency graph, and clears unreachable Next (`[*]`) tasks, but it deliberately
preserves every In Progress (`[/]`) task. The new behavior should retain the current
daily note's validation and structural cleanup responsibilities while adding a read-only
rolling activity window for deciding whether area/project work is still legitimately in
progress.

The current daily note remains the path selected by `BOB_DAY_FILE` or the existing
local-date/`BOB_NOW` fallback. Starting strictly before that note's date, find the
newest canonical `<vault>/YYYY/YYYYMMDD.md` file that exists; choosing the maximum valid
earlier date rather than assuming yesterday exists must naturally handle multi-day gaps
and year boundaries. No earlier note is a valid, non-error result. A `BOB_DAY_FILE`
whose filename contains a valid daily date supplies the search anchor, preserving
deterministic fixtures and manual overrides; otherwise use the existing effective
current date. Ignore malformed daily filenames and candidates dated today or in the
future.

Today remains required to exist, contain `## Pomodoros`, and satisfy the
single-open-timed-Pomodoro guard. The selected historical note is optional and
read-only: never perform duplicate cleanup, link retirement, marker repair,
canceled-reference removal, or Pomodoro relocation in it. If the latest existing
historical daily note has no Pomodoros section, treat it as an empty historical link
source rather than mutating it, failing the whole sync, or silently falling through to a
still older file.

Build a rolling recent-activity identity set from non-retired block links in the
Pomodoros sections of the structurally normalized current note and the selected
historical note. Reuse the existing link grammar, fence/list-item rules, note resolver,
and canonical `(vault-relative note path, block ID)` task identity. Preserve each daily
note as the resolution context so same-note `[[#^block]]` links resolve against the note
in which they occur and identical raw link text from two dailies cannot overwrite the
wrong resolution. Struck historical links are retired and do not count; aliases, embeds,
and Pomodoro markers continue to normalize to the same task identity. Extend the roots
through the existing eligible transcluded-task dependency graph so an active dependency
chain remains coherent and cannot oscillate across repeated runs.

Keep current-note status promotion, current-note structural reconciliation, and
dependency/Blocked derivation otherwise unchanged: the historical note is evidence that
an already-In-Progress task is recent, not a reason to promote a Ready task to Next.
This separation avoids resurrecting yesterday's historical work while still providing
the requested two-note grace window.

## Status reconciliation and reporting

Reuse the shared frontmatter parser and exact area/project predicates from
`src/native/projects.rs`; area/project scope means YAML `type: [[area]]` or
`type: [[project]]`, including the already-supported quoted variants, rather than
directory names or tags. Record that classification on each scanned note so the
transition pass can apply the new policy without changing which files participate in
task resolution, dependency indexing, or Blocked reconciliation.

For a recognized Tasks task currently at `[/]` in an area or project note, reset the
checkbox to Ready (`[ ]`) when its canonical identity is absent from the rolling
recent-activity set. Tasks without a usable trailing block ID cannot have a
corresponding daily block link and therefore qualify for reset. Preserve `[/]` when the
task is directly represented by either daily note or remains reachable through an
eligible dependency chain. Do not broaden the rollback to ordinary notes, daily notes,
generated reference notes, terminal statuses, unknown/custom statuses, or other checkbox
states. Preserve the existing higher-priority Blocked derivation contract for tasks with
recognized open `[dependsOn:: ...]` targets, so the command still computes one stable
final state and remains idempotent instead of alternating between Ready, active, and
Blocked across runs.

Represent `[/] -> [ ]` separately from the existing `[*] -> [ ]` clear action in
`src/native/task_status_hooks.rs`. Add an explicit result collection and human/JSON
reporting for cleared in-progress tasks, include it in change and summary counts, and
retain path, line, block ID, description, dry-run, and dependency annotations
consistently with current status reports. Add an optional vault-relative historical
daily path (and enough reference-count context to make selection diagnosable) to the
result without changing the meaning of the existing `daily_file`, which continues to
identify the mutable current ledger. Update command help so environment-variable and
guard-rail language clearly distinguishes the current ledger from the optional
historical lookup.

## Tests and documentation

Add focused unit coverage in `src/native/task_status_hooks.rs` for historical daily
selection and transition policy: yesterday present, missing intervening days/weeks, year
rollover, no earlier daily, malformed/future filenames, `BOB_DAY_FILE` date anchoring,
empty historical Pomodoros, same-note link resolution, retired links, quoted/bare
area/project classification, and stable interaction with dependency and Blocked states.

Extend the CLI integration coverage in `tests/cli.rs` with a compact two-daily vault
that proves:

- a link in the latest existing earlier daily protects an area/project `[/]` task even
  when several calendar dates are absent;
- a current-day link and a dependency reachable from either rolling root are protected,
  while a link found only in an older-than-selected daily is not;
- unlinked `[/]` tasks in both area and project notes become `[ ]`, including a task
  without a block ID, while the same status in an ordinary note remains unchanged;
- the previous daily file is byte-for-byte unchanged, current structural cleanup still
  affects only today, and a historical note without a Pomodoros section is harmless;
- dry-run reports the new action without writing, JSON exposes the selected historical
  source and new change list, application preserves line endings, and a second run is a
  no-op;
- current-note missing/section/multiple-timed guards and Tasks status registry behavior
  remain atomic and unchanged.

Update `docs/task-status-hooks.md` as the authoritative contract and align the README
summary. Document the search semantics, the read-only nature and link eligibility of the
historical note, area/project frontmatter scoping, exact rollback and Blocked/dependency
precedence, reporting fields, and revised transition table. Remove the obsolete blanket
statements that In Progress is never lowered while retaining that guarantee for
out-of-scope notes and terminal/custom statuses.

## Verification

Run the module/unit tests and the targeted `task_status_hooks` CLI tests first, then the
repository's normal formatting, linting, and full test suite. Inspect the final diff for
accidental historical-note writes or unrelated CLI/schema changes, and exercise both
human and JSON dry-run output with deterministic `BOB_NOW`/`BOB_DAY_FILE` fixtures
before considering the implementation ready.
