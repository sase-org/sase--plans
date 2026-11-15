---
tier: epic
title: Stop recurring hourly job failures from dangling artifacts and in-flight prompt
  archives
goal: 'The hourly artifact_retention job reclaims dangling index rows instead of failing
  forever on a missing stored file, and agent_publication no longer emits axe digest
  errors for deferred prompt archives that are only seconds-to-minutes old and resolve
  on their own.

  '
phases:
- id: retention
  title: Artifact retention tolerates dangling stored files
  depends_on: []
  size: medium
  description: 'retention: let the trash store accept a missing payload on opt-in,
    pass the flag from the retention pass, and keep purge running so one bad row never
    wedges the hourly job.'
- id: pubnoise
  title: Grace window for unresolved deferred prompt archives
  depends_on: []
  size: medium
  description: 'pubnoise: treat a young unresolved prompt archive as a deferral (no
    error, no attempt charge) and only escalate to digest errors past a grace window.'
decisions:
  missing_payload_policy:
    ask: Should a dangling index row still get a restorable trash entry when its stored
      file is gone?
    choices:
      trash: Store a payload-less trash entry; the row metadata stays restorable and
        auditable
      drop: Remove the row from the live index directly with no trash entry
    default: trash
    why: payload-less entries reuse existing (None, None) restore semantics and keep
      the audit trail
    answer: trash
  grace_minutes:
    ask: How long may a deferred prompt archive stay unresolved before it counts as
      a digest error?
    choices:
      ten: Escalate after 10 minutes; faster alerting, may still emit noise for slow
        agent turns
      thirty: Escalate after 30 minutes; covers every transient case observed in the
        digests
      sixty: Escalate after 60 minutes; quietest, slowest to surface a genuinely stuck
        archive
    default: thirty
    why: the worst observed self-resolving case today cleared within about 25 minutes
    answer: thirty
  grace_skips_charge:
    ask: Should the grace window also skip charging publication attempts toward quarantine?
    default: true
    why: three 45-second ticks already reach the quarantine threshold while an agent
      is mid-turn
    answer: true
proposed_by: bbugyi200.athena.1cm
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-15 12:29:15
status: wip
bead_id: sase-1t0
---

- **PROMPT:** [prompts/202611/hourly_job_error_hardening.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/hourly_job_error_hardening.md)
- **BEAD:** [sase-1t0](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t0/README.md)

# Plan: Stop recurring hourly job failures from dangling artifacts and in-flight prompt archives

## Background

The axe error digests surface two recurring failure classes (see
`~/.sase/axe/error_digests/digest_20261115_121107.txt` for a representative sample):

1. `housekeeping/artifact_retention` fails every hour with
   `retention failed: failed to inspect stored artifact: No such file or directory (os error 2)`.
   Root cause (confirmed on athena): exactly one artifact index row,
   `default:b4d6d6231b64e4124fdd513b`, points at
   `~/.sase/artifacts/agents/gh_sase-org__sase/20260816091518/agents_metadata_search_committed_120x40-46c825183d6f.png`,
   which no longer exists on disk. The retention planner selects the row (it is due),
   `trash_artifact_files` (`src/sase/core/artifact_file_trash.py`) forwards
   `stored_path` to the Rust `artifact_file_trash_store` binding, and
   `store_in_reserved_entry` in the linked sase-core repo
   (`crates/sase_core/src/artifact_file/trash.rs`, the
   `ensure_regular_file(source, "stored artifact")` call) hard-fails on the missing
   file. The exception aborts the entire pass in `_run_locked_pass`
   (`src/sase/core/artifact_file_retention_pass.py`), so nothing else is trashed, the
   purge step never runs, and the same row wedges the job on every subsequent pass.
   `artifact_file_doctor.py` only _reports_ missing stored paths; nothing repairs them.

2. `publication/agent_publication` emits
   `could not restore deferred prompt archive for <agent>@<revision>: prompt archive is unresolved and a missing local source does not prove there was no prompt obligation`
   two to three ticks in a row, then the request publishes fine. Thirteen digests since
   2026-10-30 contain this pattern, every instance self-resolved, and both project
   outboxes are empty today. The `unavailable` classification (sase-core
   `crates/sase_core/src/agent_publication_recovery/complete.rs`) is deliberately
   conservative and must keep blocking acknowledgment; the problem is only that each
   blocked tick is _reported_ as a job error (`check_error`), and that each tick charges
   a publication attempt against `DEFAULT_PUBLICATION_MAX_ATTEMPTS = 3`
   (`src/sase/agents_sync/publication_outbox_operations.py`), so an agent still mid-turn
   can burn its whole quarantine budget in about two minutes of 45-second ticks.

A third digest error (the corrupted `plans` SDD sidecar in the bob-cli project) was a
one-time local git repair and has already been fixed by hand; it needs no code change.

## Phase retention: Artifact retention tolerates dangling stored files

All shared trash semantics live in the Rust core per the backend boundary; the Python
side only passes intent through. Work spans both repos — open the core with
`sase repo open sase-core` and follow its `AGENTS.md` recipe for exposing a core change
to Python.

### sase-core changes (`crates/sase_core/src/artifact_file/trash.rs`)

> [!decision] missing_payload_policy = trash

- Add an opt-in request field `missing_payload_ok: bool` (serde `#[serde(default)]`, so
  the addition is non-breaking and needs no
  `ARTIFACT_FILE_LIFECYCLE_WIRE_SCHEMA_VERSION` bump) to `ArtifactFileTrashRequestWire`.
- In `store_in_reserved_entry`, when `missing_payload_ok` is set and the source path is
  missing (ENOENT from `ensure_regular_file` _or_ from the later `move_file`, so the
  deletion race is covered), store the entry with `stored_filename: None` and
  `stored_path: None` instead of failing. The full record (including its original
  `path`) is already preserved in `record`, and restore's existing
  `(None, None) => None` arm reinserts the index row without touching the filesystem.
- Add `payload_missing: bool` (serde default `false`) to `ArtifactFileTrashEntryWire` so
  callers can count reclaimed-without-payload rows. Any other error, and a missing
  source without the flag, keeps failing exactly as today.
- Tests in the module's test file: flag unset + missing source still fails; flag set +
  missing source produces a payload-less entry that restores to an index row with no
  payload restore; flag set + present source behaves identically to today.
- Binding: the request dict is deserialized into the wire struct in
  `crates/sase_core_py` (find the domain with `rg -l artifact_file_trash_store`), so
  confirm the new field passes through and extend the round-trip test. Run
  `sase tool run check` in the core repo.

> [!decision] missing_payload_policy = drop Instead of a payload-less trash entry, skip
> the core change entirely: in the Python retention pass, detect rows whose `path` no
> longer exists and remove them from the live index under the existing exclusive index
> lock, counting them separately. The TOCTOU race (file deleted between check and trash
> call) then still fails a pass once, and recovers on the next hourly run.

### sase changes

- `src/sase/core/artifact_file_trash.py`: `trash_artifact_files` gains
  `missing_payload_ok: bool = False`, forwards it on the wire dict, carries the new
  `payload_missing` entry field into `TrashEntry`, and `TrashResult` exposes a
  `payload_missing_rows` count. A payload-missing entry contributes 0 to
  `bytes_reclaimed`.
- `src/sase/core/artifact_file_retention_pass.py` (`_run_locked_pass`): call
  `trash_artifact_files(..., missing_payload_ok=True)`; include the payload-missing
  count in the printed summary and the stamp payload. Additionally, run
  `purge_trashed_artifact_files` even when the trash step raises, so a pass that fails
  on trashing still purges expired trash before stamping `failed`.
- CI pin: sase code may only reference the new wire field once `sase-core-revision.txt`
  has moved past the core commit; ratchet it as `docs/rust_backend.md` describes
  (`just ratchet-core-revision`) inside this phase.
- Tests: a retention-pass test with an index row whose file is absent must complete the
  pass, trash the dangling row payload-less, still run purge, and stamp `enforced`.
  After this lands, the live wedge on athena self-heals on the next hourly run with no
  manual index surgery.

## Phase pubnoise: Grace window for unresolved deferred prompt archives

Python-only; no sase-core change. The `unavailable` classification and its
acknowledgment-blocking stay exactly as they are — only error accounting and attempt
charging change, and only for _young_ requests.

- Define the grace cutoff next to the restore logic in
  `src/sase/agents_sync/prompt_archive/deferred.py` (module constant, seconds).

> [!decision] grace_minutes = ten Set the constant to 600 seconds.

> [!decision] grace_minutes = thirty Set the constant to 1800 seconds.

> [!decision] grace_minutes = sixty Set the constant to 3600 seconds.

- In `restore_deferred_prompt_archives`, a failure is _pending_ (within grace) when all
  of: the classifier decision has the unavailable status (reason
  `prompt archive is unresolved and a missing local source does not prove there was no prompt obligation`),
  `restore_error is None`, and `now - request.created_at < grace`. A `created_at` of
  `0.0` (legacy rows) is always past grace. Return the pending/error distinction to
  callers — e.g. make the failures mapping carry a small value object (message +
  `pending: bool`) instead of a bare string, updating both call sites.
- `src/sase/agents_sync/git_sync.py` (the drain block around the `prompt_failures`
  handling and `_integrate_export_pass`):
  - A pending failure still blocks acknowledgment and keeps the request queued
    (unchanged invariant).
  - A pending failure is excluded from `ExportCounts.diagnostics` / the outcome error so
    `_drain_project` (`src/sase/agents_sync/publication_job.py`) does not set
    `result.error`, the tick reports `errors=0`, the chop result stays `ok`, and no axe
    digest entry is produced. Log it at info level as a deferral instead.
  - Past-grace failures keep today's behavior verbatim.

> [!decision] grace_skips_charge For a pending failure, call `update_agent_publications`
> with `increment_attempts=False` and no quarantine threshold (or skip the update
> entirely), so in-flight agents cannot be quarantined by the grace window; if the
> toggle is declined, keep charging attempts during grace exactly as today.

- `src/sase/agents_sync/_commit_publication_transaction_steps.py` mirrors the same
  pending handling where it consumes `restore_deferred_prompt_archives`.
- Tests: unit-test the pending predicate (unavailable + young vs old vs `created_at=0.0`
  vs explicit restore errors), and a drain-level test asserting a young unavailable
  request yields no error diagnostic, no attempt charge (per the toggle), and remains
  queued, while an old one still errors and charges.

## Verification

Each phase runs this repo's `just check` via `sase tool run check` (and the same in
sase-core for the retention phase) before finishing.
