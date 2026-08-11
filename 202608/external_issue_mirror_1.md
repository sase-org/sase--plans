---
tier: tale
title: Mirror external tracker issues into task beads
goal: "Every external issue in each enabled project is mirrored idempotently into an
  open unsized task bead by a bounded, resumable AXE reconciliation flow, with a shared
  manual command, durable recovery state, detached-auth diagnostics, and deterministic
  cross-machine duplicate collapse.

  "
size: medium
proposed_by: bbugyi200.athena.sase-jd.4
bead: sase-jd.4
create_time: 2026-08-11 06:13:32
status: wip
---

- **PARENT:**
  [202608/external_artifact_ingestion.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_artifact_ingestion.md)
- **BEAD:**
  [sase-jd.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jd/sase-jd.4.md)

# Plan: Mirror external tracker issues into task beads

## Scope and invariants

Implement phase `issue_mirror` from the external-artifact-ingestion design. The mirror
is project-scoped by the stable ProjectSpec key and provider-neutral: it uses the
existing issue-listing capability and `IssueWire`, while tracker-specific commands
remain inside provider plugins. It creates task beads with status `open`, no size, no
tier, a normalized `external_ref`, the matching `bug:` reference, the remote body, and
an explicit provenance line. Upstream closure, reopening, deletion, or transfer never
changes bead lifecycle state; it produces one attributed append-only note and durable
drift state instead.

The local identity index is rebuilt while the owning bead-store write lock is held. A
covering `external_ref` or normalized `bug:` reference suppresses creation, and the
existing partial-unique database invariant remains the final local concurrency backstop.
The mirror batches all mutations for one pass into a single bead-sidecar commit and does
not advance reconciliation state until the batch has committed and published
successfully.

## Reconciliation engine and durable state

- Add a focused external-issue mirror module with typed request/result/state records,
  project resolution through enabled lifecycle records, capability probing, issue
  validation, deterministic ordering, label exclusion, identity indexing, mutation
  planning, note generation, and exact structured counters.
- Persist one atomic JSON state document per stable project key beneath the AXE checks
  lane. Track the backfill cursor, incremental `(updated_at, provider_id)` high-water
  mark, overlap window, last completed full repair time, provider identities and
  observed states, exponential retry/backoff data, and the last detached failure
  classification. Ignore malformed state safely and preserve the previous checkpoint on
  any unprocessed malformed record or failed local mutation.
- Treat the provider inventory as deterministic logical pages. Resume the first
  exhaustive backfill and later daily full-repair sweeps from durable cursors; bound
  each pass by page count, creation count, lock wait, and wall-clock work budget.
  Incremental passes re-read an overlap window and deduplicate by stable provider ID. A
  full repair detects records that disappeared from the complete inventory and appends
  one stale-link note without deleting or closing beads.
- On auth, rate-limit, outage, contention, publication, or record-shape failure, return
  a degraded result with an actionable reason, retain the last safe watermark, and leave
  a persistent exponential retry lease. Dry runs may read existing state but must not
  write beads, notes, cursors, backoff, or state files.

## CLI, configuration, AXE, and doctor surfaces

- Add `sase bead sync-external` to the sorted parser, handler facade, and command
  dispatcher. Support `-p/--project`, `-n/--dry-run`, and `-f/--full`; resolve omitted
  projects from the current checkout, render exact planned creations and a compact pass
  summary, and return a failing status for degraded/manual runs. Keep the command on the
  Python slow path because it orchestrates providers, project stores, publication, and
  mirror state rather than one Rust bead-store operation.
- Add validated `external_mirror.exclude_labels` configuration with an empty default and
  document it in the generated configuration reference. Keep pass safety budgets as
  conservative implementation constants unless an existing configuration convention
  requires exposing them.
- Register `sase_chop_external_issue_mirror` in package entry points and add the
  `external_issue_mirror` builtin to the checks lumberjack with a ten-minute cadence,
  two-minute timeout, and `for_each: {source: projects, vcs: [git, gh]}`. Read the
  stable project key from the target payload, decline unsupported issue providers
  cleanly, share the exact reconciliation path with the manual CLI, and emit counters
  for issues seen, creations, notes, conflicts, exclusions, malformed records, pages,
  and checkpoint advancement.
- Add an AXE doctor check that reports whether every configured mirror target has recent
  detached tracker-auth evidence. It should use the mirror's persisted detached
  failure/success evidence rather than assuming an interactive provider call proves the
  daemon environment, distinguish unsupported/no-run/auth-failed states, and give
  project-display-name guidance without leaking ProjectSpec keys into user-facing text.

## Cross-machine semantic duplicate collapse

Update the canonical Rust bead event reducer so independently created task-bead streams
carrying the same non-empty `external_ref` collapse deterministically at
integration/read time instead of making the whole store unreadable. Preserve the strict
local mutation and JSONL-import uniqueness checks, and limit collapse to the
event-reduction seam where concurrent hosted histories first meet. Choose a stable
winning bead ID, keep one materialized issue, and retain append-only event streams so no
source history is destroyed. Add Rust parity tests proving that event reduction
collapses the simultaneous-import shape in either stream order while direct
create/update/import conflicts still fail atomically.

## Tests and verification

- Add unit tests for project resolution, capability gating, normalized identity
  matching, unsized open creation payloads, exact dry-run output, exclusion labels,
  cursor overlap and tie-breaking, resumable backfill, daily repair, one-time
  state/stale notes, malformed records, state corruption, creation conflicts,
  lock/publication failures, checkpoint atomicity, persistent exponential backoff, and
  structured chop summaries.
- Add parser/dispatch/help tests for sorted commands and short aliases; config
  schema/default tests; for-each expansion and entry-point discovery tests; and doctor
  checks for success, no evidence, unsupported providers, and detached authentication
  failure.
- In `sase-core`, run formatting, focused bead-event tests, and the full Rust workspace
  check (`cargo fmt --all -- --check`, Clippy with warnings denied, and
  `cargo test --workspace`). Rebuild the local `sase_core_rs` extension through the
  shell repo, then run focused Python tests.
- In the SASE shell repo, run `just install`, `just check`, exercise
  `sase axe chop list --verbose` to verify per-project instance identities, run a
  dry-run mirror against test/fake providers without advancing state, and finish with
  `just check-full` because the change touches configuration, CLI dispatch, AXE
  scheduling, doctor registration, bead mutation orchestration, and the Rust core
  dependency.

Do not edit memory or generated agent-instruction files. Do not close the parent epic.
Record any out-of-scope discovery on `sase-jd.4` as a `PROPOSED FOLLOW-UP:` note instead
of creating a new bead.
