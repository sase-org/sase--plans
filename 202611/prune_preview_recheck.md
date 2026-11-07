---
tier: tale
title: Make the ACE-run prune panel recover from stale blocked previews
goal:
  A recovered prune preview always reaches the inbox, and `y` in the prune panel never
  silently does nothing.
size: medium
decisions:
  blocked_y:
    ask:
      On a blocked or missing preview, should `y` start a durable re-check proc that
      republishes the preview?
    choices:
      recheck:
        y runs a re-check proc; adds `prune-runs --publish-preview` and one durable op
      explain: y only explains via toast; recovery waits for the next hourly preview
    default: recheck
    why:
      You expected a proc; transient gaps like project onboarding otherwise block
      pruning up to 1h
    answer: recheck
proposed_by: bbugyi200.athena.19i
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-07 08:00:53
status: wip
---

# Fix: `y` in the ACE-run prune panel silently does nothing

## Diagnosis (verified on athena, 2026-11-07)

The panel in the screenshot is the review for the **07:05** preview notification
(`c00266cd…`, dedup key `artifact_run_prune:9cf37865b2ec7195`). Its payload says two
protection sources were unreadable (`body:beads`, `body:plans`), so the modal's view
state is `BLOCKED`. In that state `ArtifactRunPruneModal.action_confirm`
(`src/sase/ace/tui/modals/artifact_run_prune_modal.py`) does `return` with no toast, and
the footer still advertises `y move to trash`. That explains the silent `y`. The deeper
problem is that **the inbox should not have been showing this blocked preview at all**:

1. **Why the preview was blocked.** The `body` project was being onboarded this morning.
   Its primary repo was registered by 06:55, but its `beads`/`plans` sidecars were not
   cloned until 07:11:57–07:11:59. The 07:05 hourly `artifact_run_prune` chop
   (`src/sase/scripts/sase_chop_artifact_run_prune.py`) correctly reported `body:beads`
   / `body:plans` as unavailable. That code is in `_collect_project_sidecar_refs` in
   `src/sase/core/agent_artifact_run_protection.py`. Re-running
   `collect_ace_run_retention_protections()` now returns no unavailable sources, so the
   protection scan is healthy today. Blocking during that window was the correct,
   conservative behavior.

2. **Why the recovery never reached the inbox (root cause).** At 07:43 the chop ran
   again. It produced a clean preview with fingerprint `ffb095216887bac3`, and
   `~/.sase/artifact_retention/run_prune_snapshots/ffb095216887bac3.json` has
   `sources_unavailable: []`. That fingerprint is **identical to the Nov 4 preview**,
   because no run has aged out since then. The dedup key is
   `artifact_run_prune:<fingerprint>`, so the key flipped A → B → A. The Rust store's
   `upsert_notification` (`sase-core/crates/sase_core/src/notifications/store.rs`)
   matches the newest row with that `(sender, dedup_key)` **including dismissed rows**.
   It also honors `supersedes` only on its create branch. As a result:
   - the 07:43 preview was appended as a `+1` ("Still available: 1.2 GiB…") to the Nov 4
     row `39e3c67c…`. That row had been dismissed at 07:05 when B superseded it, and it
     predates the review payload (no `run_prune_review`);
   - `supersedes=artifact_run_prune:9cf37865…` was ignored, so the blocked 07:05 row
     stayed live;
   - `last_preview_dedup_key` now points at `ffb09521…`. Every later hourly run will
     keep `+1`-ing the dismissed row, so **the inbox is stuck on the blocked panel until
     the run set changes**.

So the fix has three parts. Preview dedup must be scoped to an _episode_, so a
fingerprint that recurs after it was retired starts a fresh, live row. Stale preview
rows must always be retired, which also self-heals the stuck state on athena. And `y`
must never be a silent no-op.

The Rust store semantics (`+1` respects dismissal; `supersedes` applies only on create)
are deliberate and are documented in `upsert_notification`'s docstring in
`src/sase/notifications/store.py`. Keep them: dismissing a deduplicated row should keep
repeats quiet. Episode keys solve this entirely in the Python preview publisher, which
is where the preview policy already lives, so **no sase-core change is needed**.

## Part 1: Episode-scoped preview publishing (root-cause fix)

Create `src/sase/core/agent_artifact_run_prune_publish.py`. It becomes the single owner
of publishing a run-prune preview to the inbox. Move into it, from the chop:
`_notify_actionable_preview`, `_should_notify`, `_notification_notes`, `_plus_one_note`,
`_preview_report`, `_read_last_preview_key`, `_write_last_preview_key`,
`_format_timestamp`, `_month_label`, and the sender/tags/`_PREVIEW_COMMAND` constants.
If the module would exceed the repo's line cap (`just toobig`), split the report builder
into a sibling module.

Public API:

- `publish_run_prune_preview(plan) -> RunPrunePublishOutcome` returns a frozen
  dataclass: `fingerprint`, `dedup_key | None`, `notification_id | None`,
  `action ∈ {"created", "plus_oned", "retired", "none"}`, `selected`, `bytes`,
  `unavailable_sources`.
- `end_run_prune_preview_episode() -> int` retires every live preview row and clears the
  last-preview key. It returns the number of rows retired.
- `PREVIEW_DEDUP_PREFIX = "artifact_run_prune:"`. Reuse it from
  `src/sase/artifact_cli/prune_runs.py` instead of that file's private copy.

`publish_run_prune_preview` runs entirely under an exclusive `fcntl.flock`. The lock
file is `.publish.lock` in the snapshot directory (`_snapshot_dir()` in
`src/sase/core/agent_artifact_run_prune_review.py`). Reuse an existing sase file-lock
helper if one fits. The lock serializes the hourly chop and the TUI re-check (Part 3),
so their retire steps cannot dismiss each other's rows. Under the lock:

1. If the plan is not notify-worthy (`_should_notify` is false: nothing selected, no
   empty shards, no unavailable sources), retire all live preview rows, clear the key
   file, and return `action="retired"` (or `"none"` when nothing was live). Today a
   blocked row lingers forever once sources recover and nothing is reclaimable; this
   step fixes that.
2. Otherwise compute `fingerprint = preview_fingerprint(plan)`, write the snapshot, and
   build the review, all exactly as today.
3. **Choose the dedup key.** Episode keys have the form
   `artifact_run_prune:<fingerprint>:<episode>`, where `<episode>` is 8 hex characters
   from `uuid4()`. Reuse the previous key only if all of these hold:
   - the last-preview key file holds an episode-format key (three `:`-separated parts);
   - its fingerprint part equals `fingerprint`;
   - no live (non-dismissed) preview row exists with a different key. Preview rows are
     those with `sender == "axe"` and a `dedup_key` starting with
     `PREVIEW_DEDUP_PREFIX`. Load them with
     `load_notifications(include_dismissed=False)`.

   Otherwise mint a new episode key. Legacy plain keys such as athena's current
   `artifact_run_prune:ffb095216887bac3` are therefore never reused. That is what makes
   the next hourly run after deploy self-heal.

4. Call `upsert_notification(...)` exactly as today, with
   `supersedes=previous_key if previous_key != dedup_key else None`.
5. If the outcome is `"created"`, dismiss every other live preview row whose key differs
   from `dedup_key`. Use `mark_many_dismissed`. This covers stale rows that `supersedes`
   cannot reach, such as the blocked `9cf37865…` row.
6. Write `dedup_key` to the key file.

Same-episode repeats still `+1` the existing row. A row the user dismissed in the
current episode therefore stays quiet, which preserves today's behavior.

Wire it in:

- **Chop:** `sase_chop_artifact_run_prune._run` calls `publish_run_prune_preview(plan)`
  whenever the trash purge is not `busy`, replacing
  `if _should_notify(plan) and not busy: _notify_actionable_preview(plan)`. The busy
  path still skips everything. Logging, `emit_summary`, `_reason_for`, and the
  `check_error` status are unchanged.
- **Apply:** in `src/sase/artifact_cli/prune_runs.py`, `_notify_result` calls
  `end_run_prune_preview_episode()` instead of `_dismiss_live_previews()` after an
  unblocked apply. Delete `_dismiss_live_previews`. Ending the episode stops a later
  recurrence of the same fingerprint, for example after
  `sase artifact trash restore runs:<batch>`, from `+1`-ing a row the apply retired. A
  blocked apply still leaves previews untouched; keep the existing test for that.

## Part 2: `y` is never a silent no-op (modal)

In `src/sase/ace/tui/modals/artifact_run_prune_modal.py`, make `action_confirm` dispatch
on `self._view.state`:

| State              | `y` / primary button                                                                                                                                                                                |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `READY`            | Unchanged: submit the durable `artifact.prune-runs --apply --snapshot` proc.                                                                                                                        |
| `CHECKING`         | Warning toast: "Still checking this preview — try again in a moment."                                                                                                                               |
| `RUNNING`          | Warning toast: "ACE run pruning is already running." (today this is silent unless the projection flips after READY.)                                                                                |
| `BLOCKED`          | `recheck`: submit the re-check proc (Part 3). `explain`: warning toast "Pruning is paused: N protection sources were unreadable when this preview was captured; the hourly preview re-checks them." |
| `SNAPSHOT_MISSING` | `recheck`: submit the re-check proc. `explain`: warning toast "This preview is no longer available; the hourly preview will refresh it."                                                            |

Make the chrome match the action:

- **Primary button.** `READY` keeps "🧹 Move N runs to trash" with the `warning`
  variant. Under `recheck`, `BLOCKED` shows "↻ Re-check protections" and
  `SNAPSHOT_MISSING` shows "↻ Refresh preview"; both are enabled with the `primary`
  variant. Every other state keeps the move label, disabled. Cancel keeps initial focus.
- **Footer.** Make the footer static state-aware: give it an id it can be updated
  through in `_refresh_render`. Show `y move to trash · r full report · n/esc cancel`
  only in `READY`. Under `recheck`, `BLOCKED`/`SNAPSHOT_MISSING` show
  `y re-check now · r full report · n/esc cancel`. Otherwise show
  `r full report · n/esc cancel`.
- **Banners.** The `BLOCKED` banner gains a muted line: "Press y to re-check now; the
  hourly preview also re-checks." (`recheck`) or "The hourly preview re-checks
  automatically." (`explain`). Under `recheck`, `SNAPSHOT_MISSING` says "Press y to
  refresh it now, or wait for the next hourly preview."
- **`RUNNING` banner.** The re-check shares the prune concurrency key, so change it to
  "A prune or re-check is already running."

Keep the modal under the line cap. Put new submission and completion logic in a sibling
helper module (for example `src/sase/ace/tui/modals/_artifact_run_prune_recheck.py`)
rather than growing the modal.

## Part 3: Durable "re-check" proc

> [!decision] blocked_y = explain Skip this entire part. Do not add the CLI option,
> operation, emitter, or producer site. Part 2's `explain` rows apply instead.

**CLI.** Add `-P, --publish-preview` to `sase artifact prune-runs`
(`src/sase/main/parser_artifact.py`, kept alphabetical). Help text: "Re-check
protections and publish the result as the inbox prune preview, exactly like the hourly
housekeeping job (does not purge trash)." In `handle_prune_runs`:

- Reject `--publish-preview` with `--apply`, `--snapshot`, `--project`, `--limit`, or
  `--keep-recent-months` (exit 2, with a clear stderr message). A published preview must
  use the global hourly policy. `--projects-root`, `--index-path`, and `--json` stay
  allowed.
- Plan with the default policy, then call `publish_run_prune_preview(plan)`.
- Emit a typed result through a new `emit_artifact_prune_runs_preview_result` in
  `src/sase/ops/commands/artifact.py`, under the new operation
  `ARTIFACT_PRUNE_RUNS_PREVIEW = "artifact.prune-runs-preview"` in
  `src/sase/ops/names.py` (also add it to `__all__`). The payload carries `fingerprint`,
  `selected`, `bytes`, `unavailable_count`, `unavailable_sources` (at most 6), and
  `notification_action`. Success means the re-check and publish ran, even when the
  findings are still blocked.
- JSON output uses `mode: "publish_preview"` plus a `publish` block. Text output prints
  the normal dry-run table plus one line: "Published inbox preview" or "Retired stale
  previews (nothing to prune)". Exit 0 on success and 1 if publishing raised.

**TUI submission.** Under `recheck`, `BLOCKED`/`SNAPSHOT_MISSING` `y` submits:

- argv: `sase_argv("artifact", "prune-runs", "--publish-preview", "--json")`
- operation: `ARTIFACT_PRUNE_RUNS_PREVIEW`
- concurrency key: `artifact-run-prune` (shared with apply, so a re-check and an apply
  never overlap)
- label and display name: "Re-check ACE run prune"
- `duplicate_message`, `reload_on_complete=False`, `notify_on_complete=False`, and
  `on_complete` as for apply

Check `durable_fingerprint` and request dedup in `src/sase/ace/tui/durable_submit.py`.
If identical request fingerprints are deduplicated after completion, include a
per-submission nonce so a second re-check of the same review is accepted. The submission
shows the toast "↻ Re-checking ACE run prune protections…" and dismisses the modal,
mirroring apply.

Register the new producer site, `artifact.run_prune_recheck`, next to
`artifact.run_prune` in `src/sase/ace/tui/_proc_producer_sites_actions.py`.

**Completion toasts.** All variants also call `_refresh_notification_count`.

| Completion                      | Toast                                                                                                         |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Unavailable sources remain      | Error: "🧹 Still paused: K protection sources are unreadable (<first source>). See the proc log."             |
| Preview is clean and actionable | Information: "🧹 Protections re-checked — a fresh preview is ready: N runs · <size>. Open it from the inbox." |
| Nothing to prune                | Information: "🧹 Protections re-checked — nothing to prune."                                                  |
| Proc failure                    | Error: "🧹 Re-check failed: <reason>. See the proc log."                                                      |

Because of Part 1, a clean re-check creates the fresh row and retires the blocked one.
The user then opens the new review (`READY`) and `y` starts the real prune proc.

## Tests

Each regression test must fail on the current code.

- **Incident regression (real store).** Add to
  `tests/test_axe_chop_output_contract_retention_prune_real_store.py`. Publish plan A,
  then a blocked plan B (with `sources_unavailable`), then A again. Assert exactly one
  live preview row; it is a _new_ row whose `id` differs from A's first row;
  `parse_run_prune_review` yields A's fingerprint with no unavailable sources; and B's
  row is dismissed.
- **Same-episode repeat.** Keep the existing `+1` test; the same row gets one `+1`.
- **User dismissal respected.** Publish A, dismiss its row, publish A again. Assert no
  new row and that the dismissed row got the `+1`.
- **Legacy self-heal.** Seed the key file with the plain `artifact_run_prune:<fpA>`, a
  dismissed row with that key, and a live row `artifact_run_prune:<fpB>`. Publishing A
  creates a fresh episode row and dismisses the `fpB` row.
- **Nothing to prune.** This retires live preview rows and clears the key. A later
  recurrence of an old fingerprint creates a fresh row.
- **Lock.** Concurrent publishes are serialized. A light test is enough: the lock file
  is taken, or two threads leave exactly one live row.
- **Update existing tests.** In
  `tests/test_axe_chop_output_contract_retention_prune*.py`, move
  `script._notify_actionable_preview`, `_read_last_preview_key`, `write_snapshot`, and
  `upsert_notification` patches to the new module. Change exact `dedup_key ==`
  assertions to the episode form (prefix `artifact_run_prune:<fp>:`).
- **Apply.** In `tests/main/test_artifact_cli_prune_runs.py`, a successful apply ends
  the episode (rows dismissed, key cleared), and a blocked apply leaves previews live.
- **CLI (`recheck`).** `--publish-preview` rejects each incompatible flag with exit 2,
  publishes, emits the typed result, and prints the JSON `publish` block.
- **Modal** (`tests/ace/tui/test_artifact_run_prune_modal.py`). Replace
  `test_non_ready_states_never_submit` with state-specific tests:
  - no state except `READY` ever submits `ARTIFACT_PRUNE_RUNS`;
  - under `recheck`, `BLOCKED` and `SNAPSHOT_MISSING` submit exactly one
    `ARTIFACT_PRUNE_RUNS_PREVIEW` proc with the argv and concurrency key above;
  - `RUNNING` and `CHECKING` submit nothing and post a warning toast;
  - footer and button labels match each state;
  - each completion-toast variant is covered.
- **Producer inventory.** `tests/ace/tui/test_proc_producer_inventory.py` passes with
  the new site.

## Docs and generated files

Read the `docs_writing` reference memory before editing docs.

- `docs/reference/automation/scheduler-housekeeping.md`: update the dedup sentence. A
  repeat of the current preview `+1`s it; any change, including a return to an earlier
  fingerprint, starts a fresh row and retires older preview rows; a clean
  nothing-to-prune run retires them too. Update the review-panel paragraph for the new
  `y` behavior in blocked or missing states.
- `docs/reference/artifact-references.md` (Run directory retention): mention
  `--publish-preview` (`recheck` only).
- `docs/reference/cli/artifact.md` and `src/sase/footprint/catalog.jsonl` are generated.
  Regenerate them with `just fix`; do not hand-edit them.
- The PNG golden `artifact_run_prune_review_blocked_120x40` (and possibly the ready
  goldens, if the footer changes) will change. Read the `tui_screenshot` memory, then
  run a targeted `just fix-tui-screenshots -- <prune modal selectors>` through
  `/sase_monitor`. Inspect every golden change before finishing.

## Verification

- `just check` must pass. It runs inline or through the prepared-completion monitor
  described by `/sase_final`. Do not run `just check-full`.
- Manual check on athena once installed:
  1. Run `sase artifact prune-runs --publish-preview` (`recheck`), or wait for the next
     hourly `artifact_run_prune` run.
  2. Confirm the inbox shows exactly one fresh "1.2 GiB of old ACE runs can be pruned"
     preview and the 07:05 "paused" row is gone.
  3. Enter opens a `READY` panel, and `y` creates a "Prune ACE runs" proc.
