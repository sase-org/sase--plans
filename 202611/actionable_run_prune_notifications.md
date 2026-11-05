---
tier: epic
title: Actionable ACE-run pruning notifications with a restorable prune review
goal: "Pressing Enter on an ACE-run pruning notification opens a polished review that,
  once confirmed, moves exactly the reviewed and still-eligible runs into a restorable
  trash in the background. Protections are re-verified before every move and again
  before any permanent purge.

  "
decisions:
  auto_purge:
    ask:
      Should the hourly preview job permanently purge confirmed run batches after the
      trash grace period?
    default: true
    why:
      Your confirmation authorizes the prune, and a second manual step would never
      reclaim the space
    answer: true
  decision_record:
    ask:
      Add a decisions memory record saying ACE-run pruning is a user-confirmed,
      restorable quarantine?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: core_quarantine
    title: Rust run-trash contract in sase-core
    depends_on: []
    size: medium
    description:
      "core_quarantine: add a sase-core module that quarantines confirmed ACE-run dirs
      into a journaled, restorable trash by atomic rename, re-checking every protection
      first. It also lists, restores, and protection-verifies purges, with PyO3 bindings
      and tests. The unattended apply refusal stays."
  - id: trash_callers
    title: Python facade, review snapshots, and CLI apply/restore
    depends_on:
      - core_quarantine
    size: medium
    description:
      "trash_callers: ratchet the core pin, add the Python facade and the
      review-snapshot payload module, make `prune-runs --apply` quarantine
      fresh-intersect-reviewed runs, and post a result notification. `artifact trash`
      gains run batches, and the artifact index stays consistent."
  - id: preview_notification
    title: Hourly preview emits an actionable, beautiful notification
    depends_on:
      - trash_callers
    size: small
    description:
      'preview_notification: the chop purges expired batches, writes a review snapshot,
      and attaches the review payload. It redesigns the report with project display
      names and per-project rows, supersedes stale previews, and drops the
      "preview-only" copy.'
  - id: review_modal
    title: TUI prune review modal behind notification Enter
    depends_on:
      - trash_callers
      - preview_notification
    size: medium
    description:
      "review_modal: route Enter on payload-bearing report notifications to a new prune
      review modal with ready, blocked, missing, and running states. On confirm it
      submits the CLI apply as a durable proc and reports completion. Includes badges,
      footer, help, visual snapshots, and an end-to-end fixture test."
proposed_by: bbugyi200.athena.18e
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 06:20:11
status: wip
---

- **PROMPT:**
  [prompts/202611/actionable_run_prune_notifications.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/actionable_run_prune_notifications.md)

# Actionable ACE-run pruning notifications

## What the user gets

Today the "ACE run pruning preview is available" notification is a dead end. Enter opens
a read-only report. The report lists directory keys such as `gh_bobs-org__bob-cli`,
shows only the first 20 of 2,634 runs, and ends with "Deletion is currently
preview-only." After this epic:

1. The inbox row reads like an invitation, not a log line:
   `🧹 1.2 GiB of old ACE runs can be pruned [prune]`. The footer shows `Enter: prune`.
2. **Enter opens the Prune Review**, a focused modal built from the notification's own
   snapshot. It opens instantly, with no 53-second re-plan on the UI thread:

   ```text
   ╭─ 🧹 Prune old ACE runs ──────────────────────────── snapshot · 33m ago ─╮
   │                                                                         │
   │    2,634 runs      1.2 GiB      5 projects                              │
   │    Keeping the newest 2 months (since Sep 2026) · 20,800 protected stay │
   │                                                                         │
   │    PROJECT        RUNS      SIZE                         SPAN           │
   │    sase          2,326   1.1 GiB  ████████████████████   Jul 7 – Aug 31 │
   │    bob-cli         199  54.2 MiB  █                      Jul 7 – Aug 29 │
   │    proj             68  21.0 MiB  ▌                      Jul 9 – Aug 30 │
   │    home             38   9.4 MiB  ▏                      Jul 8 – Aug 27 │
   │    actstat           3   1.2 MiB  ▏                      Jul 7 – Jul 11 │
   │                                                                         │
   │    ↺  Runs move to trash and stay restorable for 14 days.               │
   │    ✓  Each run is re-checked against fresh protections before it moves.│
   │                                                                         │
   │              [ 🧹 Move 2,634 runs to trash ]     [ Cancel ]             │
   ╰──────────────── y move to trash · r full report · n/esc cancel ─────────╯
   ```

3. `y` confirms (Cancel holds the initial focus, matching `ConfirmDialog`'s danger
   convention, so a double Enter never moves anything). The modal closes, a toast says
   "🧹 Moving 2,634 ACE runs to trash — re-verifying protections first". The work runs
   as a **durable proc** that outlives the TUI and shows in the proc indicators.
4. On completion a toast reads
   `🧹 Moved 2,630 ACE runs to trash · 1.2 GiB · restorable until Nov 19 · 4 kept (now protected)`.
   The preview row is retired. A result notification appears in the same Artifact
   Retention tab and carries the exact restore command. If anything went wrong, the
   preview stays and the toast says why.
5. When protection sources are unreadable, the same Enter opens the review in a
   **blocked** state. A red banner names the unreadable sources and explains that
   nothing can be pruned until they recover. The confirm action is absent. `r` always
   opens the full report, so the old Enter behavior stays one key away.

## Why this reverses "preview-only", and why it is now safe

Epic `sase-11h` (`plan:202609/disk_safety_and_epic_retirement.md`) retired ACE-run
deletion. Its objection was concrete: deletion trusted a caller-provided protection
snapshot with no transaction across the artifact index, bead and plan state, and
continuation publication, so a reference created mid-pass could lose data. That
objection was about **irreversible deletion on a possibly stale snapshot**. This design
removes that failure mode instead of building the transaction:

- **Confirm means quarantine, never delete.** Runs move into the trash by atomic
  same-filesystem `rename`. A mistaken move is fully reversible.
- **Re-verification happens at every boundary.** The CLI re-plans against fresh
  protections. Rust re-classifies each item immediately before its move, using the same
  predicate as the preview plus terminal and active-marker checks. Purge re-collects
  protections and **restores** any quarantined run that became referenced, instead of
  deleting it. Protection references are path- and timestamp-based, so a reference to a
  moved run's original location is still detected.
- **Missing evidence still fails closed.** Protection evidence fields are required on
  the new wire contract. A non-empty `sources_unavailable` blocks quarantine and purge
  with zero effects.
- **Unattended paths keep refusing.** `apply_agent_artifact_run_retention(apply=true)`,
  `sase disk reap --apply`, and the disk-pressure chop keep today's stable refusal
  untouched. Only a human confirmation (TUI review or explicit CLI `--apply`) can start
  a quarantine.

The residual risk is narrow. A brand-new reader of a 2-month-old terminal run (for
example, a continuation started in the ~1-minute window between re-plan and move) may
find its run missing. That run is still in the trash and can be restored, and the purge
check would restore it automatically if a reference appears.

## Measured facts that shape the design

Measured on athena (dry run only, nothing mutated):

- `sase artifact prune-runs --json` takes **~53 s**: 23,434 candidates, 2,634 selected,
  20,800 protected, 1.26 GB reclaimable, 0 unavailable sources, 5 projects (sase 2,326,
  bob-cli 199, proj 68, home 38, actstat 3). The review **must not** re-plan
  synchronously. It renders the notification's snapshot, and the background apply
  re-plans.
- `~/.sase/projects` and `~/.sase` share one filesystem, so a trash root under `~/.sase`
  supports atomic `rename`. Rust still checks the device ID and refuses cross-device
  items rather than copying.
- `Notification.action_data` is `dict[str, str]`. The existing `report` is a JSON
  string, and the review payload follows the same convention.
- The chop's current report shows directory keys, which violates
  `decisions:project-labels-never-keys`. Every new surface uses display names.
- An already-running TUI keeps executing its imported code. Keeping
  `action="ViewReport"` and adding an optional payload means a stale TUI still opens the
  report. It never shows "Unsupported notification action".

## Flow overview

```text
hourly chop ──collect protections──▶ purge expired batches (auto_purge)
     │                                   (restore newly-protected items)
     └─▶ plan ─▶ write snapshot <fp>.json ─▶ upsert ViewReport notification
                                              + action_data.run_prune_review
TUI Enter ─▶ parse payload ─▶ Prune Review modal (snapshot only, no re-plan)
     y ─▶ durable proc: sase artifact prune-runs --apply --snapshot <fp> --notify --json
              └─▶ fresh plan ∩ snapshot ─▶ Rust quarantine (lock, re-check, rename, journal)
                    ─▶ deindex moved dirs ─▶ result notification, retire previews
                    ─▶ typed op result ─▶ TUI on_complete toast
```

## Global rules for every phase

- **Never mutate the real host.** Do not run `--apply`, confirm the modal, restore, or
  purge against the real `~/.sase`. Use disposable projects roots, trash roots, artifact
  indexes, and notification stores in tests. A live `sase screenshot` of the modal is
  fine. Pressing `y` on the live host is not.
- Shared safety and policy logic lives in sase-core (`rust_core_backend_boundary`).
  Python sequences effects, gathers host observations, and renders output.
- Open `sase-core` only through `/sase_repo` and use the printed path. Read its
  `AGENTS.md`. Run core verification with `sase tool run check`.
- Main verification is `just check` through `sase tool run`. Long commands go through
  `/sase_monitor`. `just check-full` is not required.
- Read `sase/memory/lint_and_test.md` before finishing any phase that changes this repo.
  Read `cli_rules.md` before touching CLI options, `docs_writing.md` before editing
  `docs/`, and `tui.md`, `tui_perf.md`, `tui_screenshot.md`, and `ace.md` before TUI
  work, all through `/sase_memory_read`.
- Format human sizes with the existing `_human_size` convention (KiB, MiB, GiB) and
  counts with thousands separators. Do not invent a third formatter; consolidate the
  duplicated `_human_size` helpers if you touch them.

## core_quarantine

Work in the `sase-core` checkout opened through `/sase_repo`.

**Shared classification.** Extract the per-candidate protection and safety predicate
inside `apply_agent_artifact_run_retention` into one private function. Preview and
quarantine must call the same code, so they cannot disagree. That predicate covers
protected reasons, referenced timestamp and dir, recent month, future timestamp, and the
candidate safety violation. Leave the `apply=true` early refusal and its tests exactly
as they are.

**New module.** Add `agent_artifact_run_trash`, either as a flat file or as a directory
module with a facade `mod.rs` and `tests.rs`, keeping files at or under 1,500 lines.
Give it its own `AGENT_ARTIFACT_RUN_TRASH_WIRE_SCHEMA_VERSION = 1` and a getter. Write
free functions over `*Wire` serde structs, with `thiserror` errors.

1. `quarantine_agent_artifact_runs(request)`:
   - **Required evidence:** `sources_unavailable`, `protected_dirs`,
     `protected_timestamps`, `recent_months`, `current_timestamp`, `items`,
     `trash_root`, `batch_id`, and `created_at` carry **no** `#[serde(default)]`. An
     omitted field is a deserialize error. A non-empty `sources_unavailable` returns
     `blocked_reason = "protection_unavailable"` with zero filesystem effects.
   - **Validation:** reuse the existing projects-root safety check. The trash root must
     be absolute, must not be a symlink, must not sit inside the projects root, and the
     projects root must not sit inside it. `batch_id` must be a safe slug. Hold an
     exclusive lock file in the trash root with a bounded wait (reuse `store_lock`). A
     busy lock returns `blocked_reason = "busy"`.
   - **Per item, in deterministic order:**
     1. Re-classify with the shared predicate.
     2. Run `fresh_active_marker_reason`.
     3. Check symlinks and the canonical root.
     4. Check that the run dir and the trash root share a device ID.
     5. Measure the tree size.
     6. Create `<trash_root>/<batch_id>/runs/<project>/<YYYYMM>/<DD>/`.
     7. `rename` the run dir to `<…>/<timestamp>`.

     Any failed check yields outcome `kept` with machine-readable reasons:
     `protected:<reason>`, `not_terminal`, `active_marker:<file>`, `cross_device`,
     `trash_conflict`, or `unsafe_path`. Quarantine never calls `remove_dir_all`.

   - **Journal:** `<batch>/journal.jsonl`, append-only. A header line is written and
     fsynced before the first rename. One line follows each moved item, holding the
     original path relative to the canonical projects root, the project, the timestamp,
     and the bytes. A footer line records totals and status. The trash layout mirrors
     the original relative path, so a crash between rename and journal append still
     leaves the run discoverable and restorable.
   - **Empty shards:** after the moves, run the existing empty-shard walk in apply mode,
     limited to the shard roots of projects that lost runs. Honor watched paths and the
     budget. `remove_dir` keeps failing closed.
   - **Result:** batch_id, status (`complete`, `partial`, or `blocked`), counts
     (requested, quarantined, kept, errors), bytes_quarantined, per-item outcomes,
     removed_empty_shards, and journal_path.

2. `list_agent_artifact_run_trash(trash_root)` returns batches newest first: id,
   created_at, status, run count, bytes, projects, and orphan items found by layout
   without a journal line. It also returns an unreadable-journal count and never fails
   the whole listing because of one bad batch.
3. `restore_agent_artifact_run_trash(trash_root, projects_root, batch_id, items?)` takes
   the lock and renames each item back, recreating parent shards. An original path that
   already exists yields `conflict` and the item stays in trash. It returns the restored
   dirs (Python re-indexes them) and removes a batch dir once it is empty.
4. `purge_agent_artifact_run_trash(trash_root, projects_root, before | purge_all, sources_unavailable, protected_dirs, protected_timestamps)`
   has required evidence fields, and a non-empty `sources_unavailable` blocks it with
   nothing deleted. For each eligible batch, an item whose original dir or timestamp is
   now protected is **restored** (`restored_protected`, or `kept` when the restore hits
   a conflict). Any other item is deleted with `remove_dir_all` only after asserting its
   canonical path is strictly inside the canonical trash root with no symlink
   components. Purge returns freed bytes and restored dirs, and removes emptied batches.

**Path comparison.** Original paths no longer exist after a move. Compare
`canonical(projects_root)/relative_path` against protected dirs normalized the same way.
Canonicalize both sides or neither, per sase-core's macOS `/tmp` rule.

**Bindings.** Bind all four functions plus the schema getter in the `sase_core_py`
domain that already binds `apply_agent_artifact_run_retention` (find it with `rg -l`).
Register each in `register_<domain>` and add round-trip tests. This is additive with no
breaking wire change, so use a `feat:` commit subject.

**Tests** (with disposable dirs):

- An omitted evidence field errors.
- Unavailable sources block with no filesystem change.
- Every `kept` reason above is reached, including a planted symlink ancestor and a
  destination conflict.
- A successful move preserves bytes and layout.
- A crash simulation (rename done, no journal line) is still listed and restored.
- A restore round trip is byte-identical, and restore reports conflicts.
- Purge blocks on unavailable sources, restores a newly protected item, and refuses a
  symlink planted inside the trash.
- A busy lock blocks.
- The existing `apply=true` refusal tests pass unchanged.

Run `sase tool run check` and put the finalized core commit SHA in the phase completion
note for `trash_callers`.

## trash_callers

**Pin and mirrors.** Move `sase-core-revision.txt` past the core commit with
`just ratchet-core-revision`, following `docs/rust_backend.md`. Exit 2 means a ratchet
happened, not a failure. Mirror the new schema constant in Python and
`tools/validate_sase_core_rs`, plus their tests. Follow the existing release-floor
procedure in that doc, and never hand-edit versions.

**Facade.** Add `src/sase/core/agent_artifact_run_trash.py` with typed frozen
dataclasses and these functions:

- `run_trash_root()` returns `sase_home() / "trash" / "ace-run"`, outside every scanned
  projects tree.
- `quarantine_ace_runs(plan, protections, *, reviewed=None)`.
- `list_ace_run_trash()`.
- `restore_ace_run_trash(batch_id)`.
- `purge_ace_run_trash(protections, *, purge_all=False)`. Its cutoff is now minus
  `artifacts.retention.trash_grace_days`.

After a quarantine, call `delete_agent_artifact_index_artifacts(moved_dirs)`. After any
restore, including restores during a purge, call
`upsert_agent_artifact_index_artifacts(restored_dirs)`. Report index failures in the
result. Never swallow them, and never undo the moves because of them. Leave
`apply_ace_run_retention` as the refusal that `disk_footprint_reap.py` keeps using.

**Review snapshot and payload.** Add `src/sase/core/agent_artifact_run_prune_review.py`
as the single owner of these pieces:

- `preview_fingerprint(plan)`, moved verbatim from the chop so existing dedup keys stay
  stable.
- Snapshot files at `~/.sase/artifact_retention/run_prune_snapshots/<fp>.json`. Each
  file is written atomically and holds schema_version, an aware `captured_at`,
  keep_recent_months, every selected item (project, timestamp, artifact_dir,
  size_bytes), the empty-shard count, and sources_unavailable. `load_snapshot(fp)`
  returns `None` when the file is missing or invalid. GC keeps the newest 8 snapshots
  and never deletes the one just written.
- `RunPruneReview` payload with schema_version 1. It holds:
  - fingerprint, captured_at, keep_recent_months, and the first kept month (`YYYY-MM`);
  - counts (candidates, selected, protected, empty_shards) and reclaimable_bytes;
  - per-project aggregates (project key, runs, bytes, oldest and newest timestamp),
    sorted by bytes descending;
  - the first 12 unavailable sources plus a total count.

  The payload is encoded as compact JSON under `action_data["run_prune_review"]`.
  `parse_run_prune_review(action_data)` returns `None` for absent, malformed, or
  unknown-schema payloads and never raises. It carries project **keys**; surfaces
  resolve display names when they render.

**CLI `sase artifact prune-runs`** (follow `cli_rules.md`: excellent help, sorted
options, a short alias for every public long option):

- `-a/--apply` now quarantines the selection into restorable trash. Rewrite the
  description and help: a dry run by default, restorable for `trash_grace_days`, with
  recent, referenced, incomplete, and non-closed-bead runs protected.
- New `-s/--snapshot FP` (requires `--apply`) restricts the apply to the intersection of
  the fresh plan and the snapshot, matched by canonical artifact dir. Reviewed items
  that are no longer eligible are reported as kept with their fresh reasons. Eligible
  but unreviewed items are untouched and reported as `not_reviewed`. A missing snapshot
  exits 2 with `blocked: snapshot_unavailable` and no mutation.
- New `-n/--notify`. After an apply, it upserts the result notification and dismisses
  the live preview rows (sender `axe`, dedup keys starting with `artifact_run_prune:`;
  reuse `mark_dismissed`).
  - The result row uses icon 🧹, color green (amber when partial), tags
    `artifact-retention, ace-run, trash`, `action="ViewReport"`, and dedup key
    `artifact_run_trash:<batch_id>`.
  - Its report has a headline, key-value pairs (runs, size, batch, restorable until,
    kept), by-project rows with display names, kept-by-reason bullets, and the restore
    command `sase artifact trash restore runs:<batch_id>`.
  - Notification write failures are logged to stderr and never change the exit code.
- Text output: a rich panel "Moved N run dirs (size) to trash · batch <id> · restorable
  until <date>", a kept table grouped by reason, and the restore hint. Use display names
  throughout.
- JSON: give the envelope its own schema constant (2), no longer aliased to the
  retention wire version. `execution` becomes the quarantine result, adding batch_id,
  restorable_until, kept_by_reason, and deindexed. Add a `snapshot` block (fingerprint,
  reviewed, eligible_now, not_reviewed).
- Exit codes: 0 for a completed batch (kept items are normal), 1 for blocked or errors,
  2 for usage errors or a missing snapshot.
- Add `ARTIFACT_PRUNE_RUNS = "artifact.prune-runs"` to `src/sase/ops/names.py`, plus an
  emitter in a new `src/sase/ops/commands/artifact.py` modeled on
  `emit_proc_kill_result`. The CLI emits a typed result (batch_id, quarantined, bytes,
  kept, kept_by_reason, restorable_until, blocked_reason) whenever a result path is
  configured.

**CLI `sase artifact trash`:**

- `list` adds an "ACE Run Trash" panel (batch, created, runs, size, projects by display
  name, restorable until, status) and an additive `run_batches` JSON key.
- `restore` accepts `runs:<batch_id>`, prints restored and conflict counts, and
  re-indexes.
- `purge` also purges run batches past grace after collecting fresh protections.
  Unavailable protections block only the run part, which exits 1 with a clear message.
  `--all` includes young batches and is still protection-verified.
- Update the help text.

**Also in this phase:**

- Regenerate the footprint catalog (`src/sase/footprint/catalog.jsonl`) for the new
  options with its existing generator.
- Update the CLI and configuration docs that describe `prune-runs`, `trash`, and
  `trash_grace_days`; find them with `rg -n "prune-runs|trash_grace" docs`.

> [!decision] decision_record When accepted, use `/sase_memory_write` to add a decisions
> strand (for example `run-prune-quarantine`). It should state that ACE-run pruning is a
> user-confirmed, restorable quarantine, that unattended paths still refuse, and that
> purge re-verifies protections. Name what it reverses from `sase-11h` and give a reopen
> condition, then run `sase memory init`. When declined, follow the skill's
> declined-change rule.

**Tests** (disposable roots and stores):

- Dry-run output is unchanged.
- Apply quarantines and leaves the index rows deindexed.
- The snapshot intersection keeps reviewed-but-now-protected runs and leaves unreviewed
  eligible runs untouched.
- A missing snapshot exits 2.
- Unavailable protections move nothing; the files and index rows remain.
- Restore re-indexes the runs and returns them byte-identical.
- Purge restores a run that gained a text or index reference.
- `--notify` writes the result row and retires the previews, and a store write failure
  keeps exit 0.
- `sase disk reap --apply` still moves no run.

## preview_notification

In `src/sase/scripts/sase_chop_artifact_run_prune.py`:

1. Collect protections once per run.

   > [!decision] auto_purge When accepted, call `purge_ace_run_trash(protections)`
   > first. Skip quietly when the trash lock is busy. Log the purged, restored, and
   > freed counts. A purge failure sets `check_error` but still previews. This runs
   > regardless of `artifacts.retention.enabled`, because the user's confirmation
   > authorized it. When declined, the chop never purges; document
   > `sase artifact trash purge` as the only purge path.

2. Plan, write the snapshot (and GC), then notify. Keep `action="ViewReport"` and
   `report`, and add `action_data["run_prune_review"]`. If the trash lock is held (an
   apply is in flight), skip the notification with reason `prune_in_progress`.
3. **Copy:**
   - Actionable notes: `1.2 GiB of old ACE runs can be pruned`,
     `2,634 runs · 5 projects · older than Sep 2026 · 20,800 protected`, and
     `Press Enter to review — runs move to a 14-day restorable trash.`
   - Blocked notes: `ACE run pruning is paused`,
     `3 protection sources are unreadable; nothing will be pruned until they recover.`,
     and `Press Enter for details.`
   - Icon 🧹 for the actionable variant. Keep `!` with red for the blocked variant.
   - Update the plus-one notes to match.
4. **Report redesign** (ChopReport):
   - Headline `2,634 old ACE runs · 1.2 GiB reclaimable`.
   - Key-value pairs for Runs, Reclaimable, Projects, Protected, and Keeping.
   - "By project" rows (Project display name, Runs, Size, Span) sorted by size.
   - Empty-shard rows when there are any, at most 10.
   - Muted lines: `Enter: review and move to trash (restorable 14 days)` and
     `CLI: sase artifact prune-runs --apply`.
   - Remove every "preview-only" sentence.
   - Resolve labels with `load_project_ref_display_snapshot()`.
5. **Supersede:** persist the last emitted dedup key beside the snapshots and pass
   `supersedes=` on create, so at most one live preview row exists.
6. Update the `artifact_run_prune` job description and the `trash_grace_days` comment in
   `src/sase/default_config.yml`, and the job text in `docs/axe.md`.

**Tests:** extend the retention chop output-contract tests (find them with
`rg -l artifact_run_prune tests`).

- The payload round-trips.
- The blocked variant renders correctly.
- Supersede leaves exactly one live row.
- The order is purge, then plan, then snapshot.
- A busy lock skips purging and notifying.
- Display names appear in the report.
- The `auto_purge` declined branch makes no purge call (when declined).

## review_modal

**Dispatch seam.** In `handle_view_report`
(`src/sase/ace/tui/actions/agents/_notification_handlers.py`), consult a tiny registry
module, `_notification_report_reviews.py`, that maps an `action_data` key to a review
opener. Register `run_prune_review` mapped to the prune review. A missing or malformed
payload falls through to `ReportModal`, so behavior degrades to today's. The Agents-tab
Enter path goes through `open_notification_action` and inherits this automatically.

**Row affordances.** For payload-bearing rows, show the badge `[prune]` instead of
`[report]` and the 🧹 icon in the notification row and the toast (`_toasts.py`). Show
the footer Enter label `prune`. Add it through the hint-fragment machinery in
`notification_modal_constants.py` so the tier-fit tests stay green at 120 columns.
Update the `?` help content wherever it describes notification Enter.

**Pure view model.** Add `src/sase/ace/tui/models/artifact_run_prune_review_view.py`,
with no Textual imports. It turns a `RunPruneReview`, the cached project display
snapshot, and `now` into:

- the title and freshness chip, plus a stale warning past 24 h;
- hero stats and the keep-window sentence;
- the top 8 project rows (label, runs, size, bar fraction, span such as
  `Jul 7 – Aug 31`), then a `+N more projects` line;
- the state: `READY`, `BLOCKED`, `SNAPSHOT_MISSING`, or `RUNNING`.

**Modal.** Add `src/sase/ace/tui/modals/artifact_run_prune_modal.py` plus styles that
follow the existing modal stylesheet conventions. Match the mock above, borrowing the
`ConfirmDialog` palette roles (warning and accent for READY, error for BLOCKED).

- **Keys:** `y` confirms (READY only). `n`, `escape`, and `q` cancel. `r` pushes
  `ReportModal` on top, and closing it returns to the review. Tab and arrows move
  between buttons, and Enter activates the focused button. Initial focus is Cancel.
- **Width:** the modal fits 80- and 120-column terminals. Names truncate with `…`, and
  numbers are right-aligned.
- **State checks:** run the snapshot-file existence check in a pump-free worker after
  mount (`tui_perf` rules 1 and 2). Confirm stays disabled until the check settles.
  Detect RUNNING from the in-memory proc projection for concurrency key
  `artifact-run-prune`.
- **Banners:** BLOCKED lists up to 6 sources plus `+N more`. SNAPSHOT_MISSING says "This
  preview is no longer available. The next hourly preview, or
  `sase artifact prune-runs`, will refresh it." RUNNING says "A prune is already
  running."
- **Consequence line:** verify what the Archive shows for a quarantined run (see
  `probe_retained_archive_content`), then state it accurately in one muted line, for
  example "Archived agents stay listed; their local run files move to trash."

**Confirm flow** (an app mixin method beside the other notification handlers):

- Submit through `_submit_durable_proc`:
  - argv:
    `sase_argv("artifact", "prune-runs", "--apply", "--snapshot", fp, "--notify", "--json")`
  - `operation=ARTIFACT_PRUNE_RUNS` and
    `request_fingerprint=durable_fingerprint(ARTIFACT_PRUNE_RUNS, fp)`
  - `concurrency_keys=("artifact-run-prune",)` and
    `duplicate_message="ACE run pruning is already running"`
  - a 🧹 display name and `notify_on_complete=False`
- Show the start toast immediately.
- `on_complete` parses the typed result:
  - success shows the completion toast described above;
  - blocked or failed shows an error toast naming the reason ("Preview kept; see the
    proc log").
- Then refresh the inbox count through the existing path. The CLI has already retired
  the preview and posted the result row.

**Tests:**

- **View model:** formatting, sorting, truncation, every state, and stale age.
- **Pilot:**
  - Enter on a payload row opens the review, Enter on a plain report opens
    `ReportModal`, and a malformed payload opens `ReportModal`.
  - `y` in READY submits exactly one durable proc with the snapshot argv
    (`_submit_durable_proc` mocked).
  - `y` in BLOCKED, SNAPSHOT_MISSING, or RUNNING submits nothing.
  - `r` opens the report and returns.
  - Cancel keys submit nothing.
  - The duplicate message appears, and both `on_complete` toasts render.
  - The footer and badge show `prune`.
- **Visual snapshots** (per `tui_screenshot.md`): READY and BLOCKED at 120 columns, plus
  one narrow width.
- **End-to-end** (disposable fixture projects root with old terminal runs and one
  referenced run):
  1. Run the chop function to produce the snapshot and notification.
  2. Dispatch Enter through the real dispatcher in a pilot app and press `y`.
  3. Execute the captured argv synchronously against the fixture.
  4. Assert that the eligible runs are in trash, the referenced run is untouched, the
     index rows are dropped, the result row exists, and the preview is retired.
  5. Run `sase artifact trash restore runs:<batch>` and assert byte-identical runs and
     restored index rows.
- Capture a live `sase screenshot` of the review for the completion note. Do not confirm
  it.

## Non-goals

- No unattended pruning. The chop never applies. `disk reap` and the pressure chop keep
  refusing run deletion.
- No immediate permanent deletion from the TUI. The only paths to permanent deletion are
  purge after grace (or `trash purge --all`), and both are protection-verified.
- No change to indexed-file retention (`sase artifact prune` and `reclaim`) or to its
  trash semantics.
- No new config knob. Run trash reuses `artifacts.retention.trash_grace_days`.
