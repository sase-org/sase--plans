---
tier: tale
size: medium
title: Finish the publication de-flap and error-digest gaps, then land epic sase-1tz.2.1
goal:
  The epic's remaining plan-required behaviors work. Quarantined prompt archives reach
  the audited terminal state. Standing alerts no longer re-report on volatile detail.
  Doctor shows live standing alerts. Error accounting drops nothing. The GC swap gate no
  longer defers policy sweeps for hours. Epic sase-1tz.2.1 and its parent phase
  sase-1tz.2 are then closed.
proposed_by: bbugyi200.athena.sase-1tz.2.1.land
proposed_with: claude/opus
bead: sase-1tz.2.1
status: done
---

- **PARENT:**
  [202611/noise_publication_digests_tui_stalls.md](https://github.com/sase-org/sase--plans/blob/main/202611/noise_publication_digests_tui_stalls.md)
- **BEAD:**
  [sase-1tz.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tz/sase-1tz.2.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1tz.2.1.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1tz.2.1.land.md)
- **COMMITS:**
  - [5c8ccd6](https://github.com/sase-org/sase-core/commit/5c8ccd6b879db3edb7444dec1dd71bc99f8c4eca)
    — fix(agent-publication): ignore volatile detail in standing alert decisions

# Plan: finish epic sase-1tz.2.1 and land it

## Context

Epic **sase-1tz.2.1** is `plan:202611/noise_publication_digests_tui_stalls.md`, which
`sase bead read sase-1tz.2.1` shows as its PLAN. All 13 of its phases are closed. Its
land agent verified the phases against current source and found the gaps listed below.
Each gap is caused by this epic and required by its plan, so it is still epic work.

The land agent already fixed three CI failures on `master` in its own commit:

- the `OutboxGroup` symvision finding (now `_OutboxGroup`);
- a reviewed artifact-dir audit exemption for
  `publication_scratch.py:sweep_publication_scratch`;
- a doctor group detail that now keeps row reasons and the newest error, which fixed
  `test_repeated_format_publication_failure_retires_and_doctor_says_drop`.

The land agent also triaged every phase `PROPOSED FOLLOW-UP:` and recorded the outcomes
in an epic note. Do not re-triage them.

Read the epic plan's "Read first" section before starting. These parts apply here:

- **Rust core boundary.** Open sase-core with `sase repo open sase-core -r "<why>"` and
  work only in the printed path.
- **Verification.** Run `sase tool run check` here, and the same command inside the
  sase-core checkout for Rust changes. Read the `lint_and_test.md` reference memory
  first.
- **Live hosts are read-only.** Never repair live state.
- **TUI code.** Read the `tui.md` and `tui_perf.md` reference memories before changing
  TUI code.
- **Docs.** Read the `docs_writing.md` reference memory before editing `docs/`.

Several touched files are near or over the `toobig` line budget. Split new code into a
focused module rather than growing an oversized file.

## Work items

### 1. Quarantined prompt archives reach the terminal state (phase `prompt_terminal`)

**Gap.** The plan requires: "Existing quarantined rows that meet the criteria transition
on the next pass." They never do. Every automatic pass lists the outbox with
`include_quarantined=False`:

- `src/sase/agents_sync/_commit_publication_outbox.py::active_agent_publications`;
- `src/sase/agents_sync/git_sync.py` (around line 140).

So `restore_deferred_prompt_archives`
(`src/sase/agents_sync/prompt_archive/deferred.py`) never sees a quarantined row. The
`request.quarantined` branch of `_attempts_exhausted` is reached only by unit tests that
call the classifier directly. Athena has 369 such rows and apollo 263 (sase-1tz.2.1.6
note #1). They stay quarantined forever, keep their pool files protected, and show in
doctor.

**Change.**

- Add a bounded per-pass terminal sweep of quarantined, non-terminal prompt-archive
  rows. Cap it at a small number of rows per project per pass, so the backlog drains
  over a few ticks inside the drain deadline.
- Classify each row through the same Rust `classify_deferred_prompt_obligation` path,
  including the existing source probes (`artifacts_dir` and the run `source_label`).
- When the classifier returns the terminal unrecoverable outcome, take the existing
  path:
  - set `terminal=True` with the deterministic
    `prompt_archive_unrecoverable: <evidence>` reason;
  - write the `terminal_obligations.jsonl` audit row through
    `record_terminal_prompt_transition`;
  - release pool protection.
- The sweep must never do any of the following to a row:
  - publish or acknowledge it;
  - charge it an attempt;
  - count it as a job error;
  - unquarantine it.

  A quarantined row whose source still exists stays quarantined; only
  `--retry-quarantined` revives it.

- Reuse the existing two-pass retirement (record the deterministic message, then retire
  when it matches), or retire in one step if that is cleaner for already-quarantined
  rows. Either way, keep `--retry-quarantined` excluding terminal rows.

**Tests.** Add tests that run through the automatic pass entry point, not just the
classifier:

- A pre-existing quarantined row (`attempts=3`, `quarantined=True`, an old
  missing-local-source `last_error`, no source anywhere, past grace) becomes terminal
  with exactly one audit row, and is never charged or reported as an error.
- The same row with its source present stays quarantined.
- The per-pass cap holds.

Update the "Unrecoverable prompt archives" section of
`docs/reference/agents/hood-sync.md` if the transition timing it describes changes.

### 2. Standing alerts stop re-reporting on volatile detail (phase `pub_standing`)

**Gap A: Rust decider.** In sase-core
`crates/sase_core/src/agent_publication_recovery/standing.rs`, the `(Some, Some)` arm
sets `detail_changed` from `observation.detail != previous.detail`. Python passes the
raw detail text and keys the alert on the normalized signature. The raw text can contain
a pid, an age, `agent@sha`, or a path; for example, `lock_holder_diagnostic` in
`src/sase/agents_sync/_git_sync_ops_lock.py` includes `pid` and `age`. So one unchanged
cause reports `changed` and adds a notification +1 on every attempt. For
`charged_item_error`, which has no backoff, that is every 15 s tick.

- Make `changed` depend only on the alert key (project and signature), `failure_kind`,
  and `error_class`. A detail-only difference is not a transition.
- On a `none` transition, store the newest observation's detail, so the alert shows a
  current sample without reporting.
- Add Rust unit tests:
  - a detail that differs only in volatile tokens yields `none` and stores the new
    detail;
  - a kind or class change still yields `changed`.
- Run `sase tool run check` in the sase-core checkout.
- No Python change is needed for this part. Do not add a sase-side test that depends on
  the new core behavior: sase CI builds the core at `sase-core-revision.txt` until the
  pin is ratcheted past the sase-core commit. Per `docs/rust_backend.md`, that happens
  through `just ratchet-core-revision` or the scheduled Core Pin Ratchet workflow. The
  closeout below does not wait for it.

**Gap B: errors that still count on every tick.** These paths still do
`summary.errors += 1` on every tick, all in
`src/sase/agents_sync/_publication_job_run.py`:

- "could not list enabled projects";
- "could not resolve publication targets";
- the leftover `selection_errors` loop after the project loop. That loop fires for
  projects deferred by the tick budget.

Route the two machine-wide failures through `apply_project_observations` against the
existing `"*publisher*"` pseudo-project, the same way the heartbeat-write alert works.
Use `STANDING_KIND_SELECTION_ERROR`, or `STANDING_KIND_TARGET_UNAVAILABLE` if that fits
the target-resolution case better. Each then reports only its transitions, and a clean
pass clears it.

A budget-deferred project's selection error must not count on every tick. Leave it for
that project's next processed pass, which records it through the standing path. Keep
diagnostics informative.

**Gap C: doorbell during backoff.** The budget-deferred branch calls
`_safe_ring(deferred_key)` without checking backoff. Ring only when
`is_due(standing_alerts, deferred_key, standing_now)`, as the main path does.

**Tests.** Add job tests in `tests/agents_sync/test_publication_standing.py`:

- 20 ticks of a failing project listing produce one reported transition and a cleared
  alert after recovery;
- a budget-deferred project with a selection error is not counted on every tick;
- a budget-deferred project in backoff is not rung.

### 3. Doctor shows live standing alerts as issues (phase `pub_doctor`)

**Gap.** The plan says: "Surface `pub_standing`'s live alerts as the check's issue
list." `_read_standing_alerts` in `src/sase/doctor/checks_agent_publication_outbox.py`
only adds them to `data` and attaches an alert to a matching row group. A project that
is failing with `target_unavailable`, `drain_timeout`, `lock_busy` or `heartbeat_write`
but has no stuck rows reports OK.

**Change.**

- Each live alert that matches no `(project, signature)` row group becomes one issue:
  - it counts toward `attention_count` and makes the check WARN;
  - its detail line shows the kind, attempts, first and last seen, and whether it is
    backing off until `next_attempt_at`.
- Alerts that match a row group stay folded into that group, as today.
- The count must stay flat across ticks while the state is unchanged.
- Keep the read strictly read-only and fail-open.

**Tests.** In `tests/doctor/test_checks_agent_publication.py`:

- an alert with no rows gives WARN with one issue;
- the count is unchanged on a second read with bumped attempts;
- a matching alert does not double-count.

Update the `agent_publication_outbox` check description in
`docs/reference/agents/hood-sync.md`.

### 4. Error accounting drops nothing (phases `error_groups` and `digest_render`)

All in `src/sase/axe/`:

1. **Lock failure.** `_state_scheduler_errors.py`, at the end of `append_error`, still
   has `except OSError: return`. If the lock file cannot be opened, the error vanishes
   silently. Count it as a pending write failure through the shared in-process buffer
   helpers in `_state_scheduler_shared.py`, and log it once per kind, as the plan's "No
   silent drops" item requires.
2. **Ring reset on rotation.** In `_state_scheduler_digests.py::rotate_error_interval`,
   a failed ring-reset write calls `bump_write_failures()` and then immediately
   `clear_pending_buffers()`, which erases that failure. Clear the buffers before the
   ring write, or carry the failure into the new interval's `write_failures`, so the
   next digest shows `≥` with the count.
3. **Python fallbacks.** `error_digest.py` keeps Python fallbacks that the
   `rust-core-required` decision forbids:
   - the grouping loop in `_group_entries` that runs after `group_errors(entries)`
     fails;
   - the `except` branch of `_signature_of` that falls back to case-folded text;
   - the `_wire_schema_version` default.

   Call the Rust facade directly and let a missing binding raise, as other required
   bindings do.

4. **Dead legacy code.** Remove what the old digest design left behind, after confirming
   nothing in `src/` reads it:
   - `read_last_error_digest_ts` and `write_last_error_digest_ts` (re-exported from
     `state.py`), plus the `last_error_digest_ts` file write in
     `src/sase/scripts/sase_chop_error_digest.py`;
   - the `errors=` path of `notify_axe_error_digest`
     (`src/sase/notifications/senders.py`) and `digest_record_from_errors`, if only
     tests use them.

   Update the tests that used them. Update the scheduler state-tree docs in
   `docs/reference/automation/scheduler.md` if they list that file.

**Tests.**

- A lock-open failure is counted and logged once.
- A ring-reset failure during rotation appears in the next digest's `write_failures`.
- The existing digest tests still pass, using the Rust facade only.

### 5. GC swap gate no longer defers policy sweeps for hours (phase `gc_policy`)

**Gap.** Recorded in sase-1tz.2.1.13 note #3. `_memory_pressure` in
`src/sase/ace/tui/util/_gc_policy_policy.py` treats any VmSwap level above
`SWAP_BYTES_PRESSURE_THRESHOLD` (16 MiB) as "swapping".

On athena instance 01f377a7, VmSwap stayed between 40 and 938 MiB from 18:24 to 06:11
with 0.0 major faults/s. That emitted 698 `tui_gc_memory_pressure` deferrals and ran no
policy sweep. CPython's automatic gen-2 then paused 12.0, 11.9 and 18.9 s, which caused
the 20.2 s max stall.

Separately, `rss_backstop` defers as `cost_unknown` until 15 minutes of idle whenever no
fresh cost sample exists: 25 rows on athena and 22 on apollo.

**Change.** Keep the plan's constraints: no worker-thread collection, no disabled GC,
and no merely raised thresholds.

- **What counts as swapping.** Treat the process as swapping only on paging activity:
  - the major-fault rate is above its threshold, or
  - VmSwap grew by more than a threshold since the previous fresh sample.

  A stable residual VmSwap is not pressure. Keep the `tui_gc_memory_pressure` row for
  real deferrals.

- **Bounded deferral.** Pressure may defer `backstop` and `rss_backstop` for at most a
  bounded window, for example 10 minutes. After that, a backstop may run once the user
  has been idle long enough, so the memory backstops sase-1nc requires still fire.
- **Cost estimate.** Seed or refresh the cost estimate from any observed full
  collection, including automatic gen-2 pauses the GC telemetry recorder already
  measures. Then `cost_unknown` does not hold `rss_backstop` until long idle once a
  pause has been observed.

**Tests.** Use the existing fake-collector patterns
(`tests/ace/tui/util/_gc_policy_test_support.py`, `test_gc_policy_policy.py`):

- residual swap with zero faults does not defer;
- swap growth or a high fault rate defers;
- deferral is bounded and a backstop eventually runs;
- an automatic gen-2 observation refreshes the cost estimate.

Update the GC section of `docs/contribute/perf-runbook.md` and add an evidence note on
sase-1nc that names the new gate.

### 6. Multi-batch inventory reuse test (phase `pub_inventory`)

The plan requires a test that "a multi-batch drain builds the inventory once". The
existing `tests/agents_sync/test_inventory_reuse.py` checks only object identity with a
fake builder.

Add a drain-level test: more than 25 queued rows in one project pass, with a counting
`build_project_hood_inventory`. It must assert one build across all batches, and a
rebuild when the primary HEAD changes between batches.

## Verification

- Run `just fmt`, then `sase tool run check` in sase. It must pass, or fail only on
  items triaged KNOWN with independent witnesses.
- Run `sase tool run check` in the sase-core checkout for item 2A.
- Do not run `just check-full`.

## Final step: land epic sase-1tz.2.1 and close its parent phase

Do these in the same turn as the code. None of them waits for this work's own commit,
push, CI, or the core pin ratchet.

1. Run `sase bead epic-symbols sase-1tz.2.1`. For every listed `--epic-symbol` entry,
   resolve the symbol: wire it up, privatize it, add a non-test pragma, or delete it per
   the `symvision.md` reference memory. It is empty today; keep it that way.
2. Close the epic: `sase bead close sase-1tz.2.1 --note "<verification>"`.
   - The note summarizes:
     - the land agent's verification: all 13 phases closed and verified against source;
       core pin `80fcc862` contains every epic core commit; follow-ups triaged in the
       epic note; three CI fixes landed;
     - items 1–6 above, with test evidence and the `sase tool run check` run id;
     - that the sase-core standing-decider commit reaches sase through the core pin
       ratchet.
   - Never use `--force` to make the close succeed. If the close is refused for leftover
     `--epic-symbol` entries, clean them up and close again.
3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   `sase bead read sase-1tz.2.1` shows
   (`plan:202611/noise_publication_digests_tui_stalls.md`, currently `status: wip`).
5. The epic's `parent_bead` is phase **sase-1tz.2** of epic **sase-1tz**. Verify that
   this child plan completed the phase's work:
   - publication de-flap with standing alerts and backoff;
   - a bounded drain with completing cleanup;
   - a terminal state for unrestorable prompt archives;
   - signature-grouped error ring and digests with durable counts and pruning;
   - TUI GC and I/O stall reduction with `tui_stalls.jsonl` as the metric.

   Also confirm that the before-numbers and after-install commands are on sase-1tz.2
   (notes #1–#8).

   Then close only that phase:
   `sase bead close sase-1tz.2 --note "<what you verified>"`.
   - The note states that live after-install acceptance is still pending human install.
     The after-install commands are in sase-1tz.2 note #8.
   - Never close epic **sase-1tz**; its own land agent owns it.
