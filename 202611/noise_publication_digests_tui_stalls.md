---
tier: epic
title: Publication de-flap, truthful error digests, and TUI stall reduction
goal:
  agent_publication reports each (project, signature) cause once as a standing alert
  with a true attempt count and backoff. Its drain does bounded, reusable work with a
  cleanup that completes and leaves no scratch behind. Unrestorable prompt archives
  reach an audited terminal state instead of staying quarantined forever. The scheduler
  error ring and hourly digest group errors by signature, with durable true counts, a
  truthful coverage interval, and pruned digest files. Policy-initiated TUI garbage
  collections and sampled main-thread I/O no longer block the event loop for seconds,
  and tui_stalls.jsonl stays bounded, untorn, and usable as the acceptance metric.
parent_bead: sase-1tz.2
decisions:
  stall_history:
    ask:
      Should the rotated tui_stalls.jsonl keep at least 48 hours of rows instead of the
      sibling logs' 2 MiB cap?
    choices:
      hours:
        Size-rotate at a cap that holds 48 h of athena rows, and slim per-pause rows
      sibling:
        Rotate at the siblings' 2 MiB cap; soaks over a few hours must copy the log out
    default: hours
    why:
      tui_stalls.jsonl is the acceptance metric, and 2 MiB holds only ~4 h of athena
      rows
    answer: hours
phases:
  - id: error_groups
    title: Group the scheduler error ring by signature with durable interval counts
    description:
      "error_groups: a sase-core normalizer turns error text into a stable signature.
      append_error keeps a durable per-interval counter keyed by (routine, job,
      signature) next to a signature-fair sample ring. Failed or corrupt writes are
      counted and logged once instead of dropped."
    size: medium
    depends_on: []
  - id: digest_render
    title: Render one digest line per cause with true totals, coverage, and pruning
    description:
      "digest_render: the hourly digest snapshots and resets the interval counter
      atomically. It renders one line per cause with count, first/last time, and a
      sample under a true-total and coverage header, with no loss after downtime or a
      crash. Digest files are pruned on a 30-day horizon."
    size: medium
    depends_on:
      - error_groups
  - id: pub_standing
    title: Turn repeated publication failures into standing alerts with backoff
    description:
      "pub_standing: a sase-core decider keys publication failures by (project,
      signature) and reports only opened, changed, escalated, and cleared transitions.
      It backs off project-scoped failures. A standing store keeps true attempt counts
      and first/last-seen times, and one notification per alert fires only on
      transitions."
    size: medium
    depends_on:
      - error_groups
  - id: pub_cleanup
    title:
      Give drain cleanup a budget it can finish and sweep stranded publisher scratch
    description:
      "pub_cleanup: the drain reserves a cleanup window inside its timeout and records a
      verifiable cleanup result. Stale .sase-v2-stage/.sase-v2-backup scratch is swept
      under the publisher lock. Workspace retention stops treating agents-sync scratch
      in the machine-wide agents sidecar as a run's stranded work."
    size: medium
    depends_on: []
  - id: pub_inventory
    title:
      Bound inventory_build with reuse, an incremental history cache, and scoped queries
    description:
      "pub_inventory: build the hood inventory once per project per tick instead of once
      per batch. Cache parsed primary-history trailers by HEAD and parse only new
      commits. Scope index and dismissed-record queries to the requested hoods, and give
      inventory_build its own deadline slice."
    size: medium
    depends_on:
      - pub_standing
      - pub_cleanup
  - id: prompt_terminal
    title: Keep deferred prompt sources alive and add an audited terminal state
    description:
      "prompt_terminal: find why deferred prompt sources vanish and close each gap so
      durable obligations keep their bytes. Add a sase-core terminal outcome for
      genuinely unreconstructable archives. It stops retries, releases pool protection
      and any tied workspace hold, and leaves an audit record. It never counts as
      success."
    size: medium
    depends_on:
      - pub_cleanup
  - id: pub_doctor
    title: Make the publication doctor checks report standing and terminal state
    description:
      "pub_doctor: doctor's agent_publication_outbox check groups issues by (project,
      signature) and reports audited terminal rows as settled, so the count stays flat
      across ticks. Apply sase-1rb remedy (a): stale-only protection blocks are not a
      WARN."
    size: small
    depends_on:
      - pub_standing
      - prompt_terminal
  - id: stall_telemetry
    title:
      Rotate tui_stalls.jsonl, tolerate torn lines, and write fault headers only on
      faults
    description:
      "stall_telemetry: tui_stalls.jsonl size-rotates through the shared bounded-append
      helper and drops its non-atomic in-place prune. Every reader tolerates torn lines,
      and long stalls get bounded stack re-samples. tui_faults.log gets a header only
      when a fault actually happens."
    size: medium
    depends_on: []
  - id: gc_policy
    title: Stop policy-initiated garbage collections from blocking the loop for seconds
    description:
      "gc_policy: idle, backstop, and RSS-backstop collections stay bounded. The policy
      re-freezes survivors, defers full collections while the process is swapping, and
      cost-gates the backstops, while keeping the memory backstops. Before/after pause
      stats are recorded by trigger."
    size: medium
    depends_on:
      - stall_telemetry
  - id: heap_retainers
    title: Find and fix the top cyclic-garbage producers and heap retainers
    description:
      "heap_retainers: census the object types freed by idle full collections (up to 7M
      objects each) and the types retained across them. Fix up to the top three measured
      producers or retainers, and record allocated blocks, RSS, and collected counts
      before and after."
    size: medium
    depends_on:
      - gc_policy
  - id: ui_io_offload
    title: Move sampled synchronous filesystem work off the TUI main thread
    description:
      "ui_io_offload: move the launchable-MRU tick's stat and config-token peek, the
      hijack roster's per-row realpath, and project-record listing behind pump-free
      workers or freshness-token caches. This is the separate fix for apollo's low-GC
      blocking-I/O stalls."
    size: medium
    depends_on: []
  - id: ui_cpu_caches
    title: Cache compiled query profiles and cut per-second clan runtime work
    description:
      "ui_cpu_caches: memoize compiled built-in pane query profiles and stop
      re-serializing every clan member on each one-second runtime tick. Confirm the
      removed agent_share_fingerprint has no live caller."
    size: small
    depends_on: []
  - id: tui_acceptance
    title: Measure the combined TUI result and settle the TUI task beads
    description:
      "tui_acceptance: run the TUI benches and a comparable soak, then record
      before/after max GC pause, max stall, and peak RSS on sase-1tz.2. Close sase-1nc,
      sase-1gf, or sase-lx only on demonstrated resolution."
    size: small
    depends_on:
      - stall_telemetry
      - gc_policy
      - heap_retainers
      - ui_io_offload
      - ui_cpu_caches
proposed_by: bbugyi200.athena.sase-1tz.2
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-19 11:13:24
status: wip
---

- **PROMPT:**
  [prompts/202611/noise_publication_digests_tui_stalls.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/noise_publication_digests_tui_stalls.md)
- **PARENT:**
  [202611/notification_log_audit_fixes.md](https://github.com/sase-org/sase--plans/blob/main/202611/notification_log_audit_fixes.md)

# Plan: Publication de-flap, truthful error digests, and TUI stall reduction

This child epic implements phase `noise` (**sase-1tz.2**) of parent epic **sase-1tz**,
`plan:202611/notification_log_audit_fixes.md`. It covers items 3, 4, and 5: audit
findings F3 (agent_publication's chronic error state), F4 (the error digest hides storms
and loses counts), and F5 (TUI memory growth, GC pauses, and main-thread stalls).

## Read first (every phase)

- **Audit.** The source of truth is
  `research:202611/apollo_athena_sase_notification_and_log_audit/apollo_athena_sase_notification_and_log_audit.md`.
  Read it with `sase artifact read <ref> "<reason>"`. If the reference resolves as
  missing, open the research sidecar first with
  `sase repo open sase--research -r "<why>"`. The `…__cdx.md`, `…__cld.md`, `…__grk.md`,
  `…__mus.md`, and `…__gem.md` swarm reports in the same directory hold the raw F3–F5
  counts, log excerpts, and stall stacks.
- **Final parent decisions.** The parent epic's DECISIONS are final; do not re-ask them.
  None selects behavior in this epic. Phases that touch workspace retention
  (`pub_cleanup`, `prompt_terminal`) must keep the landed `held_heartbeat = yes`
  behavior intact: at most one hourly "still held" +1, and only on undismissed rows.
- **Rust core boundary.** Shared backend and domain behavior lives in `sase_core`. That
  covers error-signature normalization and grouping, publication failure accounting and
  standing-alert decisions, deferred-prompt classification, artifact-index queries, and
  the `perf_logs_query` reader. Open sase-core with
  `sase repo open sase-core -r "<why>"` and work only in the printed path. A phase that
  adds or changes a binding sase calls must move `sase-core-revision.txt` past that
  sase-core commit; see `docs/rust_backend.md` and `docs/contribute/rust-backend.md`.
  File I/O, locking, notifications, and Textual code stay in Python.
- **Verification.** `just check` is the agent verification recipe. Read the
  `lint_and_test.md` reference memory before finishing. Run `just check-full` only when
  explicitly instructed. Every behavior change gets focused tests, including the
  regression scenarios named in its phase.
- **Live hosts are read-only.** Global install is human-only, so live acceptance happens
  after the user installs. Phases may take read-only measurements on apollo and athena;
  the `tailnet.md` reference memory covers SSH access. Each measuring phase records the
  exact before-numbers and the command to repeat after install as a note on its own
  phase bead, plus a one-line summary note on **sase-1tz.2**. Never dismiss
  notifications, repair state, or delete files on either host. Propose any one-time
  cleanup (for example removing stranded publisher scratch) through `/sase_gate`.
- **Discovered work.** Phase workers are epic phase workers. Record follow-up work as
  `PROPOSED FOLLOW-UP:` notes on your own phase bead, never as new task beads.
- **Docs and memory.** User-visible behavior changes update the matching page under
  `docs/`. Read the `docs_writing.md` reference memory first. A survey of `tui.md`,
  `tui_perf.md`, and the always-loaded notes found nothing this epic makes false. If a
  phase finds such a note, it records a `PROPOSED FOLLOW-UP:` and does not edit memory
  itself. TUI phases must read `tui.md` and `tui_perf.md` before changing TUI code. Any
  phase that touches `tools/` reads the `tools.md` reference memory first.
- **Already landed: build on it, don't rebuild it.**
  - `b0de76422d` silenced the `gh_bbugyi200__nut` no-target storm (a missing target with
    nothing queued is silent). `pub_standing` keeps that test green.
  - `d152f200b2` removed `agent_share_fingerprint` from the UI loop. `ui_cpu_caches`
    only confirms this.
  - Child epic sase-1tz.1.1 landed two pieces this epic relies on. `store_guard`:
    identical +1s coalesce, and a +1 to a dismissed row is a no-op. `held_settle`:
    retention re-settles once per evidence fingerprint, with change-only +1s and the
    hourly heartbeat. Standing alerts and hold release build on both.
  - `06ae058b5f` (`sase_chop_managed_tmp_reap.py` standing skips) is the in-repo
    precedent for a standing state that logs only on change and fails open.

## Baseline measured on athena (2026-11-19, read-only)

- **Error ring.** `~/.sase/axe/recent_errors.json` holds 100 rows covering Nov 14 19:54
  to Nov 19 10:18. Of those, 44 are `publication/agent_publication`, 22
  `sdd-sidecar/workspace_sdd_clone_recovery`, and 20 `housekeeping/artifact_retention`.
  Every publication row's `error` is the same summary line
  (`agent_publication: projects=10 … errors=1`). The real cause is visible only inside
  `subprocess_diagnostic.output_excerpt`.
- **Digests.** `~/.sase/axe/error_digests/` holds 727 files going back to July. November
  digests contain about 5,956 `agent_publication` entries out of roughly 6,040.
- **Prompt archives.** Each unrestorable deferred prompt archive appears in the drain
  ledger exactly three times and then stops, because `DEFAULT_PUBLICATION_MAX_ATTEMPTS`
  is 3. The row then stays quarantined forever. It is never terminal, it is counted by
  doctor, it protects pool files, and `--retry-quarantined` can revive it. Examples:
  `bbugyi200.athena.1e6@50c90e3f2f54`, `…sase-1t3.4@ac19f907ae53`, `…1dm@dd56fe20753d`,
  `…toobig-a1.test_artifacts_beads_filtering.0@4305e739f006`.
- **Drain ledger.** `~/.sase/logs/agent_publication_drains.jsonl` has 526 rows.
  `stage_durations.inventory_build` has p50 16.7 s and max 103.8 s.
- **Workspace holds.** 28 "Workspace held: stranded work" notices name
  `sdd:agents prompt archive`. Their recovery diffs (for example
  `…/ace-run/202611/19/20261119091523/diagnostics/worktree_recovery/sdd_agents_prompt_archive-*.diff`)
  contain only `.sase-v2-backup-kyrs0ld1/…` files. Two scratch dirs from Nov 3 still sit
  at the root of the machine-wide sidecar,
  `~/.sase/projects/gh_sase-org__sase/repos/agents/`: `.sase-v2-backup-kyrs0ld1` and
  `.sase-v2-stage-6nhjyn0l`.
- **TUI telemetry files.** `~/.sase/logs/tui_stalls.jsonl` is 36 MB, plus a 2.1 MB `.1`.
  That is 44.8k rows over about 78 h, roughly 0.47 MB/h, and 37.8k of the rows are
  `tui_gc_pause`. `tui_faults.log` is 17 KB of session headers only.
- **GC pauses.** Gen-2 pauses by trigger, all on `MainThread` and all from explicit
  policy `gc.collect()` calls:

  | Trigger        |     n |    p50 |    p95 |    max |
  | -------------- | ----: | -----: | -----: | -----: |
  | `idle`         | 2,067 | 1.91 s | 3.26 s | 73.0 s |
  | `rss_backstop` |    80 | 2.07 s | 13.7 s | 31.8 s |
  | `backstop`     |    17 | 17.3 s |  324 s |  324 s |

  The 324 s backstop took 1.59M major faults. Single idle collections freed up to
  5.8–7.2M objects. The worst recovered stalls (331 s, 74 s, 62.5 s) each have a GC
  overlap of at least 98 %.

- **RSS.** Normal TUI instances peak at 3–6 GB RSS. From 06:00 to 07:30 on Nov 16,
  instances reached 15–35 GB (105–155M allocated blocks). `gc_policy` checks whether
  that runaway is already fixed.
- **Python.** Both hosts run CPython's classic three-generation collector: apollo on
  3.12.3, athena on 3.14.7 with `gc.get_threshold() == (2000, 10, 10)`. Do not plan
  around an incremental collector.

## Coordination with open work

- **sase-1r8** (closed) built the single drain loop, session-matched deferred archives,
  rules-versioned protection, and park-once notices. Extend those paths; do not fork
  them.
- **sase-1rb** (READY, stale protection blocks keep doctor WARN). `pub_doctor` applies
  remedy (a) inside the reviewer's `revalidation = automatic` decision. Close sase-1rb
  only if the landed change resolves its WARN, and say so in the close note. If the
  withheld `chat.md` bytes are still missing, record remedy (b) as a
  `PROPOSED FOLLOW-UP:`.
- **sase-1nc** (GC policy), **sase-1gf** (bench stall records), and **sase-lx**
  (selected-tribe p95). `gc_policy` and `heap_retainers` add evidence notes to sase-1nc.
  Only `tui_acceptance` closes any of the three, and only on demonstrated resolution.
- **sase-1tw** (SkillUseEvent snapshots) and **sase-1tx** (NotificationPlusOne growth)
  are open heap-retainer tasks. `heap_retainers` absorbs either one only if it ranks
  among the measured top retainers, and closes it only on demonstrated resolution.
- **Parent phase `defects` (sase-1tz.3)** owns the `prompt_prose` artifact-link wire
  fix. Until it lands, `pub_standing` collapses the recurring "referenced-by cites event
  … unknown variant `prompt_prose`" failure into one standing alert; it does not fix it.
- **Parent phase `hygiene` (sase-1tz.4)** only verifies the digest pruning that
  `digest_render` owns.

## Phase `error_groups`: signature-grouped ring with durable counts (item 4.1, 4.3)

**Today.** `append_error` (`src/sase/axe/_state_scheduler.py:177-203`) appends to
`~/.sase/axe/recent_errors.json` and keeps only the last 100 rows. It has three callers:
`Lumberjack._handle_error` (`src/sase/axe/lumberjack.py:494-519`),
`orchestrator._surface_crash_loop` (`src/sase/axe/orchestrator.py:233`), and
`_append_recovery_error` (`src/sase/sdd/_store_integration.py:256`), which already sends
a `failure_signature`. Failures are swallowed in three places:

- `except OSError: return` in `append_error`;
- `atomic_write_json` (`src/sase/axe/state.py:72-73`) swallows write errors;
- `read_json` returns `None` for a corrupt ring, which silently resets it to `[]`.

**Required changes.**

1. **Rust normalizer and grouping.** Add pure functions in sase-core, exposed through
   `sase_core_rs` and a thin facade such as `src/sase/core/axe_error_facade.py`:
   - A signature normalizer that masks volatile tokens and then lowercases, collapses
     whitespace, and truncates. Tokens to mask:
     - absolute paths;
     - hex SHAs and `agent@sha` forms;
     - ISO timestamps and durations;
     - pids;
     - run, request, and UUID ids;
     - bare integers.

     Golden tests use real audit strings. For example, the deferred-prompt messages for
     different agents must share one signature, while a missing-target error and a lock
     timeout must not. The managed-tmp `_skip_reason_class` masker is prior art.

   - A grouping function: entries in, groups out, keyed by
     `(lumberjack, job, signature)`, each with a count, first and last timestamps, and
     one bounded sample.
   - An entry's explicit `failure_signature` wins. Otherwise the signature comes from
     its `error` text.

   Move the pin.

2. **Durable interval counter.** Under the existing lock, `append_error` first updates a
   schema-versioned counter file, for example `~/.sase/axe/error_counts.json`. It holds
   `interval_start`, `total`, `write_failures`, and per-group
   `count`/`first_ts`/`last_ts`/`sample`. Only after that does it update the sample
   ring, so a failed ring write never loses the count.
3. **Signature-fair sample ring.** `recent_errors.json` stays bounded at 100 but caps
   samples per signature (for example 5). A loud signature can then no longer evict rare
   ones. Keep the entry fields, and add `failure_signature` when it is computed.
4. **No silent drops.** This path uses a strict write that reports failure, not the
   swallowing `atomic_write_json`.
   - Count each failed write, keep the count in process, and fold it into the counter's
     `write_failures` on the next successful write.
   - Log a failure once per process per failure kind.
   - Move a corrupt ring or counter aside (`*.corrupt-<ts>`) and log that once; never
     reset it silently.
   - The test-write guard (`best_effort_test_state_write_allowed`) keeps its current
     behavior.
5. Existing patch seams (`sase.axe.state.append_error`) and the `read_errors` contract
   stay compatible.

**Tests.**

- More than 100 errors between drains give exact counter totals.
- 1,000 entries of one signature plus three rare distinct ones leave the rare ones in
  both the ring and the counter.
- The concurrent-append spawn test (`tests/test_axe_state.py:132`) still yields exact
  totals.
- A forced write failure is counted and logged once.
- A corrupt ring is preserved aside.
- Rust golden tests cover the normalizer.

**Measurement.** Record each host's ring composition, the publication share, and the
ring's time span. After-install command:
`python3 -c "import json,collections;e=json.load(open('$HOME/.sase/axe/recent_errors.json'));print(len(e),collections.Counter(x['job'] for x in e).most_common(5))"`
plus `cat ~/.sase/axe/error_counts.json | head -c 2000`.

## Phase `digest_render`: one line per cause, true totals, pruning (items 4.2, 4.4)

**Today.** `src/sase/scripts/sase_chop_error_digest.py:18-40` sends every ring row newer
than `max(now−1h, last_digest_ts)`, comparing ISO strings. As a result, anything between
`last_ts` and `now−1h` after downtime is lost. `notify_axe_error_digest`
(`src/sase/notifications/senders.py:117-215`) writes
`~/.sase/axe/error_digests/digest_%Y%m%d_%H%M%S.txt` verbatim, one block per row, with
the note "N error(s) in the last hour" and no `dedup_key`. `last_error_digest_ts` is
written with a non-atomic `write_text`. Digest files are never pruned.

**Required changes.**

1. **Atomic interval rotation.** Under the counter lock:
   - Snapshot the counter interval and start a new interval at "now".
   - Persist the snapshot as a pending digest record with an interval id.
   - Notify.
   - Mark the record sent.

   If the job crashes between steps, the next run re-sends the pending record exactly
   once, deduplicated by an interval-id `dedup_key`. Nothing is lost and nothing is
   double-counted. The coverage interval is the true `[interval_start, now]`, so a
   delayed digest says how long it covers. Remove the `max(now−1h, …)` cutoff.

2. **Grouped rendering.** The header states the true total and causes, the samples kept,
   and the coverage, for example
   `312 errors from 7 causes (100 samples kept), 14:02–15:02`. It uses `≥` whenever
   `write_failures > 0` or a corrupt file was set aside, and states that count. Each
   cause gets one line with its count, routine/job, first/last time, and a truncated
   sample. The file then shows one detail block per cause, containing the existing
   traceback/subprocess-output rendering for its sample. The notification note becomes
   "312 errors from 7 causes since 14:02". `action`, `action_data`, and
   `error_report_path` keep their current shape, so ViewErrorReport and mobile
   attachments keep working.
3. **Pruning.** The digest chop prunes digest files older than 30 days. It always keeps
   the newest 50 and runs at most once per day, using a marker file. It is best-effort
   and logged. Reuse the `prune_old_bead_sync_logs` pattern
   (`src/sase/bead/_sync_logs.py:65-139`). Missing files already degrade gracefully in
   ViewErrorReport (`src/sase/ace/tui/actions/agents/_notification_handlers.py:65-101`)
   and in mobile attachments; keep it that way.
4. Make the last-digest state write atomic.

**Tests.**

- More than 100 errors between digests report the true total.
- A digest delayed five hours covers the whole interval with no loss.
- A crash after the snapshot and before notify re-sends once without double counting.
- Write failures surface as `≥` with the count.
- One loud signature plus rare ones gives exactly one line per cause.
- Pruning removes files older than 30 days but keeps the floor of 50.

**Docs.** Update `docs/reference/automation/scheduler-housekeeping.md`,
`docs/reference/automation/scheduler.md` (state tree, "Last 100 errors"),
`docs/reference/config/scheduler-behavior.md`,
`docs/reference/tui/notifications-modal.md`,
`docs/reference/automation/notification-cli.md`, and
`docs/reference/automation/notification-types.md`. If the job description changes,
regenerate the footprint catalog row with `tools/sync_footprint_catalog`.

**Measurement.** Record digest file counts and sizes on both hosts: athena has 727
files, and the audit counted 436 on apollo and 726 on athena. After-install command:
`ls ~/.sase/axe/error_digests | wc -l; tail -n 30 "$(ls -t ~/.sase/axe/error_digests/* | head -1)"`.

## Phase `pub_standing`: standing alerts and backoff (items 3.1, 3.4 data)

**Today.** `run_publication_job` (`src/sase/agents_sync/publication_job.py:100-260`)
does `summary.errors += 1` for each of these cases:

- listing projects fails;
- a selection outcome has an error, which happens once per project on every tick;
- target resolution raises;
- queued rows sit behind a missing target;
- `_drain_project` raises or returns `result.error`;
- the heartbeat write fails.

`result.error` itself folds in lock-busy skips, drain timeouts, charged prompt and
refusal item errors, artifact-link drain failures, and referenced-by diagnostics.
`sase_chop_agent_publication.py:38-39` turns any error into `check_error` with exit
code 0. The lumberjack then appends the bare summary line to the ring. `_safe_ring`
re-triggers the doorbell whenever work remains or a project errored, so a failing
project re-runs on the next 15 s tick.

**Required changes.**

1. **Signature.** Each would-be error becomes an observation with these fields:
   - project key;
   - failure kind: selection error, target unavailable with queued rows, drain error,
     drain timeout, lock busy, charged item error, prompt unrestorable, referenced-by
     error, or heartbeat write;
   - stage and failure scope;
   - Rust `classify_publication_error` class;
   - normalized detail from `error_groups`' normalizer.
2. **Rust decider.** Add a pure sase-core decider next to
   `decide_publication_failure_accounting`. It takes the previous standing alert, this
   tick's observation (or its absence after a clean pass), and `now`. It returns the new
   alert state, the transition to report (`opened`, `changed`, `escalated` once after 24
   h, `cleared`, or none), and `next_attempt_at`.
   - Project-scoped kinds get capped exponential backoff, with constants in Rust: for
     example 1 min doubling to a 30 min cap.
   - Row-scoped kinds get no new backoff, because attempts, quarantine, and parking
     already govern them.

   Add the binding and facade, and move the pin.

3. **Standing store.** Keep `~/.sase/agent_publication/standing_alerts.json`,
   schema-versioned, using `file_lock` plus an atomic write. Key it by (project,
   signature) and store the true attempt count, `first_seen`, `last_seen`,
   `last_reported`, the state, and `next_attempt_at`. A corrupt or unreadable store
   fails open to today's counting and is logged once.
4. **Job semantics.**
   - `summary.errors` counts only reported transitions. The chop's `reason` names them,
     for example `gh_sase-org__sase: opened prompt_unrestorable (attempt 1)`, so the
     ring and digest get a meaningful signature.
   - A tick whose only outcomes are unchanged standing alerts returns `ok` and adds
     `standing=<n>` to the summary line.
   - While a project-scoped alert is backing off, skip that project's drain and
     `_safe_ring` until it is due. Queued hints stay queued, and the trigger's
     `max_quiet` backstop still runs the job.
   - Add the standing alerts to the publisher heartbeat's `project_stats`, so doctor and
     the CLI can read live counts.
5. **Notifications.** Keep one notification per alert, sent with `upsert_notification`
   using sender `agent-publication` and dedup key
   `agent-publication-standing:<project>:<signature-hash>`.
   - `opened` and `changed` notify.
   - `escalated` adds one +1.
   - `cleared` updates the row without creating a new unread row.
   - Per-tick attempt counts live in the standing store, not in the notification store,
     so unchanged ticks never rewrite the notification store.

**Tests.**

- 20 ticks of the same failure produce one ring entry, one notification, and an
  `attempts=20` record with first/last times. Cover each failure kind: parked/withheld,
  missing target with queued rows, drain timeout, per-project drain error, and
  unrestorable prompt.
- A signature change reports once more.
- A clean pass clears the alert without a new unread row.
- Exit code 0 no longer counts as an error every tick.
- Backoff skips both the drain and the doorbell until the alert is due.
- A corrupt store fails open.
- The b0de76422d no-target-with-nothing-queued case stays silent.

**Docs.** Update `docs/reference/agents/hood-sync.md` and
`docs/reference/automation/notification-types.md`.

**Measurement.** Record the publication share of each host's ring, and publication
digest entries over the last 7 days, before the change. After-install command: the ring
snippet from `error_groups` plus `cat ~/.sase/agent_publication/standing_alerts.json`.

## Phase `pub_cleanup`: cleanup that completes, and no stranded scratch (items 3.2, 3.3 holds)

**Today.**

- **Timeout and cleanup.** The drain's SIGALRM timeout is
  `DEFAULT_PUBLICATION_DRAIN_TIMEOUT_SECONDS = 120.0`
  (`src/sase/agents_sync/_commit_publication_drain.py:40-109`). When it fires, the
  exception path in `_commit_publication_locked.py:151-208` gets a Rust-owned 1 s
  cleanup budget (`default_publication_cleanup_timeout`, `PublicationCleanupBudget`).
  That budget routinely fails, and the cleanup failure is attached only as an exception
  note.
- **Scratch.** `v2_io._apply_changed_payload` (`src/sase/agents_sync/v2_io.py:170-210`)
  creates `.sase-v2-stage-*` and `.sase-v2-backup-*` at the sidecar root. An alarm
  mid-apply leaves them behind. `clean_publication_worktree`
  (`git_sync_transaction.py:81`) only cleans the payload and prompt-archive families, so
  the Nov 3 pair has survived every pass since.
- **Workspace holds.** The agents sidecar is machine-wide, host-owned state, as
  documented in `dirty_agents_prompt_archive_repo`
  (`src/sase/llm_provider/commit_finalizer_state/_dirty_repos.py:117-180`). Workspace
  retention's `_collect_dirt` (`src/sase/turns/_workspace_retention/settle.py:174-249`)
  nevertheless counts that scratch as the run's stranded work, which holds the
  workspace.

**Required changes.**

1. **A cleanup window inside the timeout.** The drain alarm fires early enough to leave
   a cleanup reserve inside the overall timeout. Size the reserve from ledger data, not
   from the fixed 1 s; if the Rust default changes, change it in sase-core and move the
   pin.
   - The ledger records `cleanup_result` (`completed`, `deferred`, or `failed`) and its
     duration.
   - A deferred cleanup is completed by the next pass's `payload_clean`, and that
     completion is recorded, so every cleanup has a verifiable outcome.
   - Cleanup trouble stays diagnostic. It never becomes a second error.
2. **Sweep publisher scratch.** `payload_clean` removes root-level `.sase-v2-stage-*`
   and `.sase-v2-backup-*` directories under the publisher lock. Apply only runs while
   that lock is held, so scratch found at lock acquisition belongs to an interrupted
   apply. Before relying on that, verify that tracked payload files are restored from
   git by the existing family cleanup, so a backup is never the only copy. Log each
   removal.
3. **Retention classification.** `_collect_dirt` treats agents-sync publication scratch
   in the machine-wide agents sidecar the way `dirty_agents_prompt_archive_repo` does:
   it is not session-owned. A run's own Q&A prompt snapshot is still claimed.
4. **Explain the baseline miss.** Find out why the run baseline did not absorb scratch
   that predates the run. One hypothesis: the baseline and the settle probe disagree on
   untracked-directory versus file-level listing. Another: the baseline omits the
   `sdd:agents prompt archive` repo. Fix the mismatch at its root and add a regression
   test.
5. **Existing holds.** Workspaces already held only for that scratch must release within
   one `held_settle` re-probe interval after the sweep. Verify this in a test. Do not
   repair live state by hand; propose any one-time sweep through `/sase_gate`.

**Tests.**

- An interrupted apply leaves scratch that the next pass removes.
- A settle whose only dirt is agents-sidecar scratch releases the workspace.
- Q&A snapshot edits are still claimed.
- The ledger records `cleanup_result` for completed, deferred, and failed cleanups.
- A timeout leaves enough reserve for cleanup to complete in a fixture.

**Docs.** Update `docs/reference/agents/workspaces.md` and
`docs/reference/agents/hood-sync.md`.

**Measurement.** Record, on both hosts:

- the `.sase-v2-*` directories under every project's `repos/agents`;
- the count of held notices naming `sdd:agents prompt archive` (athena: 28 all-time);
- ledger `cleanup` outcomes.

After-install command:
`find ~/.sase/projects -maxdepth 3 -name '.sase-v2-*'; sase notify list -a -j -l 10000 -q 'agents prompt archive' | python3 -c 'import json,sys;print(len(json.load(sys.stdin)))'`.

## Phase `pub_inventory`: bounded inventory work (item 3.2)

**Today.** `inventory_build` (`_commit_publication_transaction_engine.py:99-114`) calls
`build_project_hood_inventory` (`src/sase/agents_sync/inventory.py:83-224`) for every
batch of 25 rows. Each call does a full O(N) pass:

- `historical_associations` (`inventory_history.py:36-131`) runs `git log` over the
  whole primary history and parses every trailer before filtering by hood;
- `indexed_records` (`inventory_sources.py:65-114`) queries the artifact index with full
  history and no limits;
- `dismissed_records` loads every summary and reads matching bundles.

**Required changes.**

1. **One build per tick.** Build the inventory once per project per tick and reuse it
   across batches while its inputs are unchanged: primary HEAD and index generation. A
   test proves the reused inventory equals a fresh build.
2. **Incremental history cache.** Persist a schema-versioned trailer cache per project,
   keyed by primary HEAD. A new HEAD that descends from the cached one parses only
   `git log <cached>..HEAD`. A non-descendant HEAD (history rewrite), a corrupt cache,
   or a schema mismatch falls back to a full rebuild. Keep the cache bounded.
3. **Scoped queries.** `indexed_records` and `dismissed_records` fetch only the
   requested hoods and revisions. If the artifact-index facade cannot scope today, add
   the scoped query in sase-core and move the pin.
4. **Deadline slice.** `inventory_build` gets its own deadline slice. Running out
   produces a resumable deferral that is not charged; reporting goes through
   `pub_standing`'s alert path. It must not consume the whole 120 s drain.

**Tests.**

- A large synthetic history parses only new commits after the first build. Count the
  parsed commits and git calls.
- A rewritten history triggers a rebuild.
- Scoped queries return the same per-hood records as the unscoped ones.
- A multi-batch drain builds the inventory once.

**Measurement.** Record `stage_durations.inventory_build` p50 and max from each host's
drain ledger (athena: p50 16.7 s, max 103.8 s over 526 drains). After-install command:
`python3 -c "import json;r=[json.loads(l) for l in open('$HOME/.sase/logs/agent_publication_drains.jsonl') if l.strip()];d=sorted((x.get('stage_durations') or {}).get('inventory_build',0) for x in r);print(len(d),d[len(d)//2],d[-1])"`.

## Phase `prompt_terminal`: source lifecycle and an audited terminal state (item 3.3)

**Today.** Here is how a deferred prompt row moves through its lifecycle:

1. `restore_deferred_prompt_archives`
   (`src/sase/agents_sync/prompt_archive/deferred.py:170-330`) classifies the row
   through Rust `classify_deferred_prompt_obligation`.
2. Unavailable sources stay pending for the 30-minute grace period.
3. After grace, the row is charged. `git_sync.py:193-233` passes `terminal_reason=None`
   for prompt failures.
4. After three charges the row is quarantined.
5. Nothing ever makes it terminal. Pool copies under `<workspace>/.sase/artifacts`
   (`core/prompt_artifact_staging_pool.py:36-82, 210-231`) stay protected forever, and
   `sase agent sync --retry-quarantined` can revive the row.

**Required changes.**

1. **Audit why sources vanish.** For the athena examples in the baseline, establish
   read-only why `raw_prompt.md`/`raw_xprompt.md` and any pool copy were missing when
   the restore ran. Candidate causes:
   - the run never had a prompt file (for example `toobig-*` test runs);
   - artifact retention removed it;
   - pool GC removed it;
   - the row names the wrong artifacts dir.

   Fix each gap found, so a durable obligation keeps its source bytes until it completes
   or reaches a terminal state, and add a regression test per gap. For example, artifact
   retention and pool GC must protect sources named by non-terminal outbox rows. Never
   mark an obligation successful to clear a queue.

2. **Rust terminal outcome.** Extend the deferred-prompt classification in sase-core
   with an explicit terminal `unrecoverable` outcome. It applies only when all of these
   hold:
   - the row is past the grace period;
   - no local source exists in any known location;
   - the sidecar has no archived copy;
   - the row has exhausted its attempts or is quarantined.

   Add the binding and move the pin.

3. **Python transition.** Set `terminal=True` and
   `terminal_reason="prompt_archive_unrecoverable: <evidence>"`. Append an audit record,
   for example to `~/.sase/agent_publication/terminal_obligations.jsonl`, with:
   - project, agent, revision, run id, and artifacts dir;
   - the probes performed;
   - timestamps.

   The transition stops retries, releases that revision's staging-pool protection, and
   is excluded from `--retry-quarantined` revival. Existing quarantined rows that meet
   the criteria transition on the next pass. A source that reappears before the terminal
   decision restores normally.

4. **Tied holds.** Confirm whether any workspace hold is tied to an unrestorable
   obligation, as opposed to `pub_cleanup`'s scratch cause. If one is, the terminal
   transition releases it through the retention settle path, and a test covers it.

**Tests.**

- A young unavailable row stays pending.
- A row past grace with attempts exhausted and no source becomes terminal: it writes an
  audit record, is never charged or reported again, and is never acknowledged as
  published.
- A row whose source reappears restores normally.
- `--retry-quarantined` skips terminal rows.
- Pool protection is released.

**Docs.** Update `docs/reference/agents/hood-sync.md` and `docs/reference/cli/agent.md`
(`--retry-quarantined`).

**Measurement.** Record on both hosts the quarantined prompt-archive rows and the 7-day
digest count of the "missing local source" reason. After-install command:
`wc -l ~/.sase/agent_publication/terminal_obligations.jsonl` plus the doctor command in
`pub_doctor`.

## Phase `pub_doctor`: doctor reflects standing and terminal state (item 3.4, sase-1rb)

**Today.**

- **Outbox check.** `doctor/checks_agent_publication_outbox.py:54-371`, with
  `problem_for_item` (`doctor/_agent_publication.py:148-201`), counts one issue for
  every row that is retired, quarantined, parked, or stalled (at least 3 attempts, or 24
  h old). That count is the `attention_count`. The audit measured 1,546 issues on apollo
  and 5,202 on athena.
- **Protection check.** `checks_agent_publication_protection.py` stays WARN on 53 stale
  blocks that nothing will revisit.

**Required changes.**

1. **Terminal rows.** Rows that are terminal as `prompt_archive_unrecoverable` are
   reported as settled and audited. They are counted and point at the audit record, but
   they are not issues.
2. **Group the rest.** Group quarantined, parked, and stalled rows by (project,
   signature), with one issue per group carrying its count and first/last time. Surface
   `pub_standing`'s live alerts as the check's issue list. The count stays flat across
   ticks while the state is unchanged.
3. **sase-1rb remedy (a).** When protection state is stale-only (0 enforced blocks),
   report it as non-WARN, with the stale count as information. Close sase-1rb if this
   resolves its WARN. Record remedy (b) as a `PROPOSED FOLLOW-UP:` if the withheld bytes
   are still unpublished.

**Tests.**

- N simulated ticks over unchanged state keep the same issue count.
- Terminal rows are not issues.
- Grouping is correct.
- Stale-only protection state is not a WARN.

**Docs.** Update `docs/reference/agents/hood-sync.md`, which documents the
`agent_publication_outbox` check, and any other page that lists these checks.

**Measurement.** Record `sase doctor -C state.agent_publication_outbox -j` and
`-C state.agent_publication_protection -j` on both hosts, with the same commands to run
after install.

## Phase `stall_telemetry`: bounded, untorn stall logs and honest fault logs (item 5.5)

**Today.**

- **Writer.** `log_tui_stall` (`src/sase/logs/tui_telemetry.py:112-147`) appends with
  `max_bytes=0`. Commit `c580b689c1` (sase-1t3.2) replaced size rotation with an hourly
  48-hour prune.
- **The prune.** It runs on whichever thread writes next. It parses the whole file while
  holding the GIL, keeps unparsable lines forever, and rewrites the file in place with a
  non-atomic `write_bytes`, which tears lines when the process is killed mid-rewrite.
- **Rotation helper.** The sibling logs use `src/sase/logs/_bounded.py`
  (`append_jsonl_record` and `_rotate_if_needed_locked`).
- **Readers that do not tolerate torn lines:**
  `tests/ace/tui/_bench_tui_jk_helpers.py:273-278` and
  `tests/ace/tui/test_residual_freeze_soak.py:120-126`. Rust `perf_logs_query`
  (`src/sase/stats/perf_query.py:95-110`) is unverified.
- **Fault log.** `_enable_tui_faulthandler` (`src/sase/main/ace_handler.py:33-51`)
  writes a session header at every start, even though faulthandler only writes on fatal
  signals.

**Required changes.**

> [!decision] stall_history = hours Route `log_tui_stall` through the shared
> bounded-append helper (current file plus `.1`) with a cap measured to hold at least 48
> h of athena rows. At today's ~0.47 MB/h that is ~24 MiB across both segments. Slim the
> rows: drop sub-threshold young-generation `tui_gc_pause` rows (37.8k of 44.8k rows)
> and roll them into the 5-minute heartbeat's counters instead. Keep the minimum detail
> the freeze report and the benches use. Then derive the smallest cap that still holds
> 48 h, and state it in code and docs. Fix the "at least 24h" docstring.

> [!decision] stall_history = sibling Route `log_tui_stall` through the shared helper at
> the siblings' 2 MiB cap with one `.1` segment. Document in the perf runbook that a
> multi-hour soak must copy the log out as it runs.

1. **Rotation.** Rotate `tui_stalls.jsonl` per the decision above. Remove the in-place
   prune: rotation replaces it, and no writer thread parses the whole file again. Update
   `tests/ace/tui/util/test_tui_telemetry_retention.py`.
2. **Torn-line tolerance.** Make every reader skip torn or invalid lines and count them.
   That includes the two test helpers above. Verify Rust `perf_logs_query`; if it is
   strict, fix it in sase-core and move the pin. `tools/tui_freeze_report` already skips
   them, and it reads `.1` too.
3. **Stack re-samples.** During an ongoing stall the watchdog re-samples the main-thread
   stack at a bounded cadence (for example every 10 s, at most 6 samples, each
   depth-capped). Recovered rows for 100-second stalls then carry more than the single
   1.5 s or 5 s sample. See `_stall_watchdog_monitor.py` and
   `_stall_watchdog_capture.py`.
4. **Fault spool.** faulthandler writes to a per-process spool file with no header, for
   example `~/.sase/logs/tui_faults.d/<pid>-<start>.log`. A clean exit disables it and
   deletes the spool if empty. At startup, every non-empty spool whose process is dead
   is appended to `tui_faults.log` under a header (pid, version, start time) and then
   deleted. Update `tests/main/test_ace_handler.py:28`.

**Docs.** Update `docs/contribute/perf-runbook.md` (retention and fault log).

**Measurement.** Record both hosts' `tui_stalls.jsonl*` sizes, torn-line counts, and
`tui_faults.log` sizes. Athena: 36 MB plus 2.1 MB, 0 torn, 17 KB. Apollo per the audit:
18 MB with 3 torn lines. After-install commands:
`ls -la ~/.sase/logs/tui_stalls.jsonl* ~/.sase/logs/tui_faults.log`, and a short Python
loop over `tui_stalls.jsonl` that counts lines raising `json.JSONDecodeError`.

## Phase `gc_policy`: no multi-second policy collections (item 5.2)

**Today.** `install_gc_policy` (`src/sase/ace/tui/util/_gc_policy_lifecycle.py:56-127`)
arms a 1 s `policy.tick_safe` interval.

- On the classic collector it raises `threshold2` to 10,000.
- It does one startup `collect()` plus `freeze()`, and never re-freezes.
- It runs a full `gc.collect()` synchronously on the main thread for three triggers:
  - `idle`: 3 s without input, after at least 100 gen-1 collections, behind a duty
    floor;
  - `backstop`: 5 minutes;
  - `rss_backstop`: 500 MB of RSS growth.

The baseline above shows that these policy collections cause every multi-second GC pause
on athena. The `backstop` trigger is the worst because it runs while the process is
swapping.

**Required changes.** No single policy-initiated collection may block the event loop for
seconds. Keep the memory backstops, as sase-1nc requires. Do not merely raise
thresholds, disable GC, or move collection to a worker thread; the perf runbook and
sase-1nc both reject those. Choose the combination by measurement from these levers:

1. **Re-freeze after each full collection.** Later collections then traverse only
   objects allocated since the last freeze. Reclaim garbage in the frozen set with a
   rare unfreeze-and-collect that runs only after a long idle period (minutes without
   input).
2. **Defer while swapping.** Never run a full collection while the process is swapping,
   meaning VmSwap or the recent major-fault rate is above a threshold. Defer it and emit
   a memory-pressure telemetry row.
3. **Cost-gate the backstops.** Predict the pause from the last full pause and the heap
   growth since. Skip any collection whose predicted pause exceeds the budget unless the
   user has been idle long enough.

Also check whether the Nov 16 15–35 GB RSS runaway (`capacity_scan_tui_runaway`, plan
`202611/capacity_scan_tui_runaway.md`) is already fixed. If it is not, record a
`PROPOSED FOLLOW-UP:`.

**Tests.** Unit-test each gate with the existing fake-collector policy test patterns:

- re-freeze cadence;
- the swap deferral;
- the cost gate;
- the backstop still firing when it is cheap or the user is idle long.

**Docs.** Update the GC section of `docs/contribute/perf-runbook.md`. Add an evidence
note on sase-1nc.

**Measurement.** Record gen-2 pause count, p50, p95, and max per trigger, plus peak RSS
per app instance, on both hosts before the change; the athena table above is the
baseline. Target on a non-swapping host: no policy-initiated pause over 2 s. Record the
after-install command; the Python snippet that produced the baseline (group
`tui_gc_pause` rows with `generation == 2` by `trigger`) is fine.

## Phase `heap_retainers`: cyclic garbage producers and retainers (item 5.1)

Single idle collections on athena free 5.8–7.2M objects. That is a large stream of
cyclic garbage that reference counting never frees, plus a multi-GB retained heap.

1. **Census.** In a representative soak, collect two censuses. The first ranks the types
   freed by full collections; sample `gc.garbage` under `DEBUG_SAVEALL` in a controlled
   run, or use an equivalent `gc.callbacks` census. The second ranks the types retained
   across collections, using the existing `SASE_TUI_GC_CENSUS=1` tooling
   (`_gc_policy_census.py`). Compare against sase-1t3's census
   (`~/.sase/perf/sase-1t3.9/…`).
2. **Fix the top three.** Fix up to three of the top measured cycle producers or
   retainers, for example by breaking cycles with weak references or explicit clears, or
   by bounding caches. sase-1tw and sase-1tx are candidates only if they rank. Remaining
   items become `PROPOSED FOLLOW-UP:` notes.

**Tests.** Add a focused regression test per fix, for example: the object no longer
participates in a cycle, or the cache stays bounded.

**Measurement.** Record allocated blocks, objects collected per idle collection, and RSS
from comparable soaks before and after. Add an evidence note on sase-1nc.

## Phase `ui_io_offload`: synchronous filesystem work off the main thread (items 5.3, 5.4)

This is the separate fix for apollo's low-GC blocking-I/O stalls. Apollo's worst stall
(116.8 s, 0.4 s GC overlap) was sampled in `_launchable_mru_tick` → `Path.stat`. Follow
`tui_perf.md` rules 1, 2, 8, and 10.

1. **`_launchable_mru_tick`** (`src/sase/ace/tui/actions/_launchable_mru.py:289-329`,
   every 2 s). On each tick it calls `peek_current_project_change_token`, whose 0.5 s
   cache is always expired at that cadence, and stats `~/.sase/vcs_macro_mru.json` on
   the main thread. It also calls `current_config_token()`, which computes synchronously
   when cold. Move the peek into the existing pump-free task (`spawn_pump_free_task`)
   while preserving its coalescing guards, or switch to a watcher.
2. **`_canonical_path`**
   (`src/sase/ace/tui/actions/agents/_hijack_roster_snapshot.py:157-164, 234-238`).
   `_index_by_artifacts_dir` calls it, which resolves every roster row with a
   per-component lstat. That happens on the UI thread inside `_schedule_hijack_decision`
   (`_hijack_decision_cache.py:194-196`), on footer refresh and leader mode. Move the
   indexing into the existing single-flight hijack worker, or cache resolved paths.
3. **`list_project_records` / `_project_record_has_spec`.** `load_project_alias_map`
   (`src/sase/project_aliases.py:239-256`) lists and parses every ProjectSpec and runs
   `is_file()` per record on every call. TUI callers include the bead filter index
   (`bug_links._normalize_external_project` via `widgets/artifacts/beads_filtering.py`),
   `resolve_current_project`, and the command-line context. Add a cache validated by the
   freshness token in `src/sase/_project_display_cache.py`, and give TUI paths the
   cached snapshot with background refresh. Confirm the exact hot caller from apollo's
   sampled stack frames (read-only).

**Tests.** Each tick and handler does no filesystem I/O on the main thread; patch
`os.stat`/`Path.resolve` to fail when called on the main thread during the action. The
cache invalidates correctly when a ProjectSpec changes. Run the existing j/k benches.

**Measurement.** From apollo's stall rows (read-only), record the count and max of
recovered stalls with `gc_overlap_s < 1` and their top frames. Record the after-install
command.

## Phase `ui_cpu_caches`: per-second CPU hotspots (item 5.3)

1. **`compile_query_profile`** (`src/sase/ace/query_profile/compiler.py:94-130`) runs
   validation, sorting, `json.dumps`, and sha256 on every call.
   `compiled_profile_for_builtin_pane` (`query_profile/pane_registry.py:29-45`)
   recompiles on every info-panel tick and filter keystroke. Memoize compiled built-in
   profiles, keyed by everything that affects the output.
2. **`aggregate_clan_runtime`** (`src/sase/core/agent_runtime_facade.py:14-29`) calls
   `asdict()` on every member, every second, for active clan rows. That runs through
   `_on_countdown_tick` → `_patch_agent_runtime_rows` → `compute_row_runtime`. Clans
   younger than an hour miss the display-bucket LRUs. Cache member wires by member
   identity and version, so that a tick only recomputes the suffix.
3. **`agent_share_fingerprint`.** It was removed by `d152f200b2`. Confirm there is no
   remaining caller, and record whether the audit's stall samples that cite it predate
   the commit (compare `sase_version` and timestamps).

**Tests.** Cache hits are correct and invalidate on input change. Runtime suffixes match
the uncached computation. Record bench p95 numbers before and after.

## Phase `tui_acceptance`: combined measurement and task-bead settlement (item 5 acceptance)

1. **Benches.** Run `pytest -s -m slow tests/ace/tui/bench_tui_jk.py`, which includes
   the sase-1gf nodes `test_bench_clan_jk_at_each_panel_fold_level` and
   `test_bench_agents_jk_with_and_without_the_link_rail`, and the sase-lx node
   `test_bench_selected_tribe_jk_at_each_fold_level`. Also run
   `pytest -s -m slow tests/perf/bench_tui_trace.py`, plus a comparable soak. Summarize
   with `tools/tui_freeze_report`.
2. **Record the result.** Note on **sase-1tz.2** the before/after max GC pause, max
   stall, and peak RSS, with the live after-install command for both hosts.
3. **Settle the task beads.** Close sase-1nc, sase-1gf, or sase-lx only on demonstrated
   resolution, citing the numbers. Otherwise add an evidence note to each and leave it
   open.

## Landing (child epic land agent)

1. Confirm every phase bead is closed and its notes are accounted for.
2. Confirm that each measuring phase recorded its before-numbers and after-install
   command on **sase-1tz.2**. Confirm the coordination notes exist on sase-1nc, and
   where applicable on sase-1rb, sase-1gf, sase-lx, sase-1tw, and sase-1tx.
3. Run `just check` on the combined tree. Confirm the `sase-core-revision.txt` pin
   includes every sase-core commit from `error_groups`, `pub_standing`,
   `prompt_terminal`, and any sase-core change made by `pub_cleanup`, `pub_inventory`,
   or `stall_telemetry`.
4. Triage the phases' `PROPOSED FOLLOW-UP:` notes.
5. Close **sase-1tz.2** with a note summarizing the verified result.

Never close the parent epic **sase-1tz** or any other ancestor.
