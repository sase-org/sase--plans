---
tier: epic
title: Fix the 13 ranked defects from the apollo/athena notification and log audit
goal: 'Disk pressure names the owners it could not measure, and recovery runs even
  when attribution is incomplete. Held workspaces stop rewriting the notification
  store. agent_publication and the error digest report each cause once, with true
  counts. Measured TUI stalls shrink. The six targeted defects (prompt_prose wire,
  update races, Telegram transport, Codex bundle cap, triage request-id retry, gateway
  WAL checkpoint) are fixed. Notifications, logs, and state files follow their stated
  dedup and retention rules on both hosts.

  '
decisions:
  codex_bundle:
    ask: How should oversize provider instruction bundles get back under Codex's 32
      KiB cap?
    choices:
      trim: Shrink always-inlined content until every bundle fits; the cap stays 32
        KiB
      raise: Raise the cap only after a delivery test proves Codex accepts larger
        bundles
    default: trim
    why: Keeps the documented explicit channel without betting on undocumented Codex
      limits
    answer: trim
  blocked_waiters:
    ask: What should happen to waiters whose root blocker has terminally failed?
    choices:
      offer: The blocker notice gets a one-key expire action; waiters stay until a
        human acts
      auto: Waiters expire automatically once their root blocker is terminally failed
    default: offer
    why: Expiring a waiter discards user-launched work, so keep a human in the loop
    answer: offer
  held_heartbeat:
    ask: Keep an hourly "still held" +1 on undismissed Workspace-held notices?
    default: true
    why: Shows the hold is still live at one write per hour; dismissed rows never
      tick
    answer: true
phases:
- id: outage
  title: Honest disk attribution and the held-workspace re-settlement loop (items
    1-2)
  depends_on: []
  size: xlarge
  description: 'outage: stop reporting unmeasured disk as 0 B/complete, measure workspace
    roots first, run reaping independently of inventory completeness, and make pressure
    notices name unmeasured owners and escalate missed recovery targets. Settle held
    workspaces once per evidence fingerprint, +1 only when evidence changes, and add
    a sase-core store guard that coalesces +1s and never rewrites the store for dismissed
    rows.'
- id: noise
  title: Publication de-flap, truthful error digests, and TUI stall reduction (items
    3-5)
  depends_on:
  - outage
  size: xlarge
  description: 'noise: give agent_publication per-(project, signature) backoff with
    standing alerts, a bounded incremental drain, and a terminal state for unrestorable
    prompt archives. Aggregate the error ring and digests by signature with durable
    true counts and pruning. Cut measured TUI GC pauses and main-thread I/O stalls,
    with tui_stalls.jsonl as the acceptance metric.'
- id: defects
  title: Targeted high-confidence defect fixes (items 6-10)
  depends_on:
  - noise
  size: xlarge
  description: 'defects: add the prompt_prose artifact-link wire variant with a binding
    contract test, close the remaining runtime-update races, fix Telegram fallback,
    retry, and scope semantics, keep provider instruction bundles under their caps
    with a size gate, and advance bead_task_triage''s generation on request_id_conflict.'
- id: hygiene
  title: Gateway checkpoint fix, notification hygiene, and retention sweep (items
    11-13)
  depends_on:
  - defects
  size: xlarge
  description: 'hygiene: handle SQLite''s busy -1 checkpoint result and serve snapshots
    stale-while-revalidate. Fix the remaining notification dedup, amplification, orphan,
    blob-size, and no-op-write issues. Add age-based reapers that apply the stated
    log and state retention horizons.'
proposed_by: bbugyi200.athena.1eb
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-19 07:09:34
status: done
bead_id: sase-1tz
---

- **PROMPT:** [prompts/202611/notification_log_audit_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/notification_log_audit_fixes.md)
- **BEAD:** [sase-1tz](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tz/README.md)

# Plan: Fix the ranked apollo/athena notification and log audit defects

## Source and ground rules (every phase)

The source of truth is the consolidated audit
`research:202611/apollo_athena_sase_notification_and_log_audit/apollo_athena_sase_notification_and_log_audit.md`.
Read it with `sase artifact read <ref> "<reason>"`. The five per-researcher swarm
reports in the same directory (`…__cdx.md`, `…__cld.md`, `…__grk.md`, `…__mus.md`,
`…__gem.md`) hold the raw evidence: counts, log excerpts, and reproduction probes. Each
phase below handles exactly the ranked items assigned to it. The audit's finding ids
(F1–F13) are cited next to each item.

Rules that apply to every phase:

- **Size and planning.** Each phase is `xlarge`. The phase worker writes its own child
  epic plan from its section. That child plan may embed its own Plan Decisions,
  including memory decisions for any memory note its work makes false. It must not
  re-ask this epic's DECISIONS, which are final once approved.
- **Rust core boundary.** Shared backend behavior goes in `sase_core`. That includes the
  notification store, artifact-link wire, agent-scan index, and any disk-retention
  policy that already lives there. Open sase-core with `/sase_repo`
  (`sase repo open sase-core`). Any new or changed binding that sase calls must also
  move sase's `sase-core-revision.txt` pin past the sase-core commit (see
  `docs/rust_backend.md`).
- **Other repos.** Use `/sase_repo` for sase-telegram (phase `defects`) and for the
  chezmoi-managed SASE config if a host-scoping change lives there. Never reach those
  repos by any other path.
- **Already fixed: verify, don't reimplement.** The audit's "Already fixed" table lists
  `b0de76422d` (nut publication storm), `637a70a0f1` (SDD probe timeouts), `dd56fe2075`,
  `c6282f75b2`, sase-github `5f85ceb`, and the September telegram token storms. Phases
  confirm these stayed fixed where they overlap. They do not rebuild them.
- **Coordinate with open work.** Each phase lists the open beads and epics that overlap
  it. Read them with `sase bead read <id> -r "<why>"` before designing. Build on what
  has already landed, and record overlap with a bead note or link instead of forking a
  parallel design. Close an existing task bead only when this phase's landed change
  demonstrably resolves it, and say why in the close note.
- **Verification.** Run `just check` as the agent verification recipe. Read the
  `lint_and_test.md` reference memory before finishing. `just check-full` runs only when
  explicitly instructed. Every behavior change gets focused tests, including the
  regression scenarios named in each section.
- **Live hosts.** Global install is human-only, so live-host acceptance happens after
  the user installs. Phase workers may take read-only measurements on apollo and athena
  (see the `tailnet.md` reference memory for SSH access). Each phase records the exact
  before-measurement and the command to repeat after install in a bead note. Do not
  dismiss notifications, repair state, or delete files on either host as part of
  verification. Propose any such one-time command through `/sase_gate`.
- **Discovered work.** Phase workers are epic phase workers. Record discovered follow-up
  as `PROPOSED FOLLOW-UP:` notes on their own phase bead, not as new task beads.
- **Docs and memory.** User-visible behavior changes update the matching page under
  `docs/`; read `docs_writing.md` first. Memory edits go through `/sase_memory_write`.
  Read `sase_flags.md` before changing user-reaching behavior whose old branch must stay
  reachable.

## Phase `outage`: honest disk attribution and the held-workspace loop (items 1-2)

### Item 1: make disk attribution honest and decouple recovery from it (F2)

Evidence: apollo ran out of disk on Nov 16 (1.8 MiB free at the worst sample). Scheduler
routines crash-looped on `Errno 28`, and recovery reclaimed ~9.7 KiB against a 9.6 GiB
target (`reason=recovery_blocked`). All 119 `disk_pressure` notices blamed 150–193 GiB
of "unattributed" space. On apollo, 38 workspace rows read `0 B` while `du` finds ~54
GiB of workspaces. On athena, 43/43 workspace rows read `0 B` with ~567 GiB
unattributed. Apollo is at 87 % used again.

Root cause: `DiskFootprintMeter.size()` (`src/sase/core/disk_footprint_utils.py`, the
`budget.exhausted` branch) returns `0` with only a diagnostic once the shared
`InventoryScanBudget` (5 s / 50k nodes) runs out. Every later row then reads
`0 B, coverage: complete`, which looks the same as a truly empty owner.

Required changes:

1. An exhausted budget produces an explicit _unknown/unmeasured_ result, never `0`. Rows
   with an unmeasured owner report `coverage: partial`. Totals and "unattributed" math
   treat unknown as unknown, for example "≥X GiB attributed; N owners unmeasured". The
   unmeasured space must not inflate "unattributed".
2. Workspace and owner roots are measured first, or get their own budget, so large trees
   (artifact runs, cargo targets, uv tools) cannot starve them. A `du -skx` per
   workspace root is cheap.
3. The managed-temp reaper and pressure recovery run regardless of inventory
   completeness. Blocked attribution must never block reaping. Safety rules stay intact:
   no blind deletion, and protected artifacts are never treated as disposable.
4. The disk-pressure notification names the unmeasured owners. When recovery misses its
   target, a distinct escalation states the target, the amount reclaimed, and what
   blocked recovery. Give `disk_pressure` notifications a stable dedup key here, because
   this phase owns that notification. Phase `hygiene` then only handles
   `workspace-rescue`.
5. Athena logs ~195 recent "could not fully inspect" warnings that keep preserving one
   agent-tmp dir whose liveness evidence is unreadable. Keep refusing to reap it, but
   report the condition as one standing state instead of a warning every pass.
6. The audit identifies this as a regression of closed phase bead sase-1bf.4 ("cover
   every registered root and every workspace checkout before the stray walk"). Confirm
   or refute that, and record the finding as a note on sase-1bf.4. Do not reopen it;
   this epic owns the fix.

Coordination: epics sase-zw and sase-zw.8 ("Finish disk retention safety and integrated
footprint acceptance") are IN_PROGRESS and move disk-retention policy into Rust core.
Read both first. Put the fix wherever the inventory and reaper logic live once their
landed work is accounted for, and note the overlap on sase-zw.8.

Tests: a tiny budget yields unknown/partial rows, not `0 B/complete`. Workspace roots
are measured before the stray walk even with an exhausted budget. The reaper runs while
the inventory is incomplete. The pressure notice text names unmeasured owners. A missed
recovery target produces the escalation.

Acceptance: `sase disk list` on a host with large trees never shows an unmeasured row as
`0 B, coverage: complete`. Workspace row sizes track `du` within a stated tolerance.

### Item 2: stop the held-workspace re-settlement loop (F1)

Evidence: this loop is live on athena. Dismissed "Workspace held: stranded work" rows
still get +1s every ~9–30 s at the 500-+1 cap. The 7.4 MB notification store is
rewritten 16 times per minute (~170 GB/day), and ~230k redundant +1s were counted across
both hosts. Each pass also re-probes `git status` and re-snapshots worktree diffs (5.6
MB per pass for one apollo workspace). This happens inside `stale_running_cleanup`,
which already times out on apollo.

Root cause: `hooks/stale_running_cleanup` → `release_monitor_claim` → "retained with
other dirt" → `notify_retained_workspace`
(`src/sase/turns/workspace_retention_surface.py`). That function upserts on the fixed
dedup key `workspace-retention:<project>:<artifacts_ts>:<workspace_num>` with a
`plus_one_note` on every pass. In sase-core, every +1 takes the exclusive store lock,
reads all rows, and rewrites the whole file, even for dismissed rows.

Required changes (three independent layers):

1. **Settle once per evidence fingerprint.** Persist a fingerprint of the retention
   evidence with the retention record. On later passes, skip the git re-probe and diff
   re-snapshot when the evidence is unchanged. Change detection must be cheap and sound,
   for example git index/HEAD mtimes or a bounded re-probe interval. A workspace that
   becomes clean, or whose evidence changes, must still be released or re-settled within
   one re-probe interval.
2. **+1 only on change.** `notify_retained_workspace` adds a +1 only when the retention
   reason or evidence fingerprint changes.

   > [!decision] held_heartbeat Also emit at most one "still held" +1 per hour, and only
   > on undismissed rows.

   > [!decision] held_heartbeat = no No heartbeat. The row's original timestamp stands
   > until its evidence changes.

3. **sase-core store guard.** The notification store coalesces identical +1s within a
   window. It never rewrites the store for a +1 on a dismissed row unless the caller
   explicitly opts in. Update the binding and the pin. Phase `hygiene` relies on this
   guard for the `wait_checks` amplification fix.
4. One apollo workspace was held for 14 days by a probe error that named a path in a
   _different_ workspace. Check whether closed phase sase-1rg.3 ("Scope monitor
   retention evidence to the monitor's own workspace") covers starter-inherited repo
   lists. Fix the gap if it does not.

Tests: N consecutive cleanup passes over an unchanged held workspace cause zero store
rewrites and zero git probes after the first (heartbeat cadence aside). Evidence changes
are still detected. A +1 on a dismissed row is a no-op unless the caller opts in.
Identical +1s within the window coalesce.

Acceptance: record `stale_running_cleanup` runtime and the store-generation rate before
and after. The athena target is store generations no longer advancing on every cleanup
pass, down from ~16/min.

## Phase `noise`: publication, error digests, and TUI stalls (items 3-5)

### Item 3: de-flap agent_publication and make its obligations finish (F3)

Evidence: agent_publication fills 62/100 (apollo) and 44/100 (athena) of the
recent-error rings and ~94–96 % of November digest entries. `sase doctor` has
accumulated 1,546 / 5,202 `agent_publication_outbox` issues.

Prior work to read first: closed epic sase-1r8 ("Stop agent publication false blocks,
parked-row floods, and drain starvation", `plan:202611/agent_publication_errors.md`),
open task sase-1rb (stale publication-protection blocks never revisited), and commit
`b0de76422d`. Verify the `gh_bbugyi200__nut` storm stays silent.

Required changes:

1. **Backoff and standing alerts.** Parked, withheld, missing-target, timed-out, and
   per-project failure outcomes that repeat with the same (project, normalized
   signature) stop counting as fresh hard errors every tick. This applies even when the
   exit code is 0. Each such outcome becomes one standing alert with a true attempt
   count and first/last-seen times. The alert notifies on state transitions only, not
   every ~15–18 s.
2. **Bounded incremental drain.** `inventory_build` currently uses 96–115 s of the 120 s
   drain budget. Cleanup then fails its hard 1 s budget after the timeout. Lock waits
   were under 3 ms, so this is a work-bounding design problem, not contention. Make the
   drain do resumable or incremental work per tick, and give cleanup a budget it can
   actually complete, with a verifiable result.
3. **Deferred prompt archives that cannot restore (athena).** Audit the deferred-prompt
   source lifecycle so a durable obligation keeps its source bytes through completion
   and cleanup. Add one explicit terminal state for genuinely unreconstructable archives
   ("a missing local source does not prove there was no prompt obligation"). The
   terminal state stops retries, releases the associated workspace hold ("Workspace held
   … sdd:agents prompt archive"), and leaves an auditable record. Never mark an
   obligation successful just to clear a queue.
4. Doctor's `agent_publication_outbox` issue count reflects these standing and terminal
   states instead of growing every tick. Absorb sase-1rb if it falls out naturally.

### Item 4: aggregate the error ring and digests by signature with true counts (F4)

Root cause: `append_error` (`src/sase/axe/_state_scheduler.py`) keeps only the last 100
errors and silently drops them on `OSError`. The hourly digest
(`src/sase/scripts/sase_chop_error_digest.py`, sent via
`src/sase/notifications/senders.py`) selects entries newer than
`max(now−1h, last_digest_ts)` and dumps them verbatim. Digest files are never pruned.
Observed results: 93 apollo digests capped at exactly 100 entries, distinct failures
evicted by one loud signature, and lost events.

Required changes:

1. Aggregate by `(routine, job, normalized signature)` with a count, first/last
   timestamps, and one sample. Keep a durable full-interval counter separate from the
   bounded sample ring.
2. The digest header states the true total and coverage, for example "≥312 errors (100
   sampled)" plus a truthful coverage interval. Each cause gets one line.
3. A failed write never drops silently: count it, and log it once.
4. Prune digest files on an age horizon. This phase owns digest pruning; phase `hygiene`
   only verifies it.

Tests: more than 100 errors between drains, a delayed digest, a write failure, and one
loud signature alongside rare distinct ones (the rare ones must survive in the digest).

### Item 5: attack the measured TUI stalls (F5)

Evidence: athena logged GC pauses of 73 s (current log) and 324 s (rotated log). They
ran at 2.5–6.2 GB RSS with ~0.5 GB swapped and 100k+ major page faults over ~16.5M
blocks. Apollo's worst stall (116.8 s) had only 0.4 s of GC overlap. Its sampled stack
sits in synchronous filesystem work, so a GC-only fix will not cover both hosts.

Prior work to read first: closed epic sase-1t3 ("Recover TUI speed … shrink the heap")
and open tasks sase-1nc (GC policy still permits 32 s collections), sase-1gf, and
sase-lx (bench_tui_jk stall and budget failures). Read the `tui.md` reference memory
before changing the TUI.

Required changes:

1. Investigate heap retention behind athena's multi-GB RSS, and fix the top retainers.
2. Cap gen-2 collections, or move them to a safer schedule (idle-time collection, freeze
   after startup, or threshold policy), so no single collection blocks the event loop
   for seconds.
3. Move the sampled synchronous hotspots off the main thread, behind mtime/TTL caches:
   `_launchable_mru_tick` → `Path.stat`, `aggregate_clan_runtime`,
   `list_project_records`/`_project_record_has_spec`, `_canonical_path`,
   `agent_share_fingerprint`, and `compile_query_profile`.
4. Fix apollo's low-GC blocking-I/O stalls as a separate change.
5. Rotate `tui_stalls.jsonl` like its sibling JSONL logs (~2 MB; apollo's is 18 MB with
   3 torn lines), and make readers tolerate torn lines. Write `tui_faults.log` session
   headers (`src/sase/main/ace_handler.py`) only when a fault actually occurs.

Acceptance: `tui_stalls.jsonl` and the existing TUI benches are the metric. Record
before/after max GC pause, max stall, and peak RSS from a comparable soak. Close
sase-1nc, sase-1gf, or sase-lx only on demonstrated resolution.

## Phase `defects`: targeted defect fixes (items 6-10)

### Item 6: fix the prompt_prose wire contract (F6)

Host code emits artifact-link origin `prompt_prose` (`PROMPT_PROSE_ORIGIN` in
`src/sase/agents_sync/referenced_by_planning.py`). The origin is also used in
`src/sase/sdd/_artifact_link_store_support.py`, `_artifact_link_projection.py`,
`_artifact_link_event_canonical.py`, `_artifact_link_outbox_drain.py`, and
`src/sase/bead/cli_detail_links.py`. sase-core's `ArtifactLinkOriginWire`
(`crates/sase_core/src/artifact_link/wire.rs`) has no such variant, so athena's
citations for `plan:202611/research_commit_gap.md` fail every pass with "unknown variant
`prompt_prose`".

Required changes: add the variant in sase-core with the provenance semantics sase
already relies on (automatic, commit-scoped, rendered as "prose citation"), and move the
pin. Add a contract test that round-trips every host-produced origin through the real
binding (`sase_core_rs.artifact_link_validate_row`). Enumerate the origins from the
Python constants so a future origin fails the test. Do not relabel it as `prompt_ref`;
that would erase recorded provenance. Check the related launch-failure drift message
("artifact-reference wire is stale: expected 3, got 4") and record whether a
version-skew guard already covers it. After install, confirm the stuck event family
drains once (a read-only check).

### Item 7: close the remaining update races (F7)

Evidence: SIGBUS (`exit code -7`) in `sidecar_auto_sync`, `agent_auto_restart`, and
`pending_checks_poll` within seconds of updates. Also
`ModuleNotFoundError: sase_core_rs`, missing job scripts (`sase_job_tg_outbound`,
`sase_job_agent_auto_restart`), and a `BeforeSpawnCallback` import failure 36 s after an
update that passed its binding checks.

Required changes:

1. Build a small update-overlap reproducer before choosing between narrower guards and
   versioned runtime instances. It runs in a temp venv or fixture, never against the
   live global install.
2. Extend the existing guards (`ea7916d7a7` classifies scripts that vanish mid-swap as
   retryable; `1ccd5c0579` queues dev updates behind in-flight epic launches). A running
   job, launch, or finalizer that hits "script not found" or "binding missing" while the
   code-swap lock is held defers and retries instead of failing.
3. The dev-update "Install prebuilt Rust dev artifacts" step treats a cache miss
   (`hit:false, reason:commit-mismatch`) as a miss: exit 0 and record `hit:false`. It
   currently fails 18/37 apollo runs. Read the `tools.md` reference memory if the step
   lives under `tools/`.

Investigate SIGBUS separately only if the reproducer triggers it.

### Item 8: fix Telegram transport semantics and scope (F8)

All transport changes are in the linked sase-telegram repo
(`src/sase_telegram/telegram_client.py`).

1. `_send_single_message` falls back to plain text only on an entity-parse `BadRequest`.
   Check for it before its `NetworkError` parent. Timeouts and network errors propagate
   to the retry wrapper. A timeout does not prove non-delivery, so the fallback must not
   create duplicate, unformatted messages.
2. `@_with_retry` retries per chunk instead of re-sending chunks that were already
   delivered. Today it can make up to 8 HTTP sends per failing chunk.
3. Repeated `TelegramCredentialError`, `TimedOut`, and Bad Gateway failures collapse
   into rate-limited degraded-state logs, with one notice per state transition and no
   full tracebacks for transient errors. The Nov 14–15 gpg outage produced 408 identical
   tracebacks.
4. Scope the `tg_outbound` lumberjack to the sending machine, the same way
   `telegram_receiver` is scoped. Apollo spawns it ~12.7×/min but has sent nothing since
   Sep 5. If the scoping lives in the chezmoi-managed SASE config, open that repo with
   `/sase_repo`.

Tests in sase-telegram cover each trigger: a parse `BadRequest` falls back, a `TimedOut`
retries without falling back, a multi-chunk failure retries only the failed chunk, and
credential failures are rate-limited.

### Item 9: keep provider instruction bundles under their caps (F10)

Evidence: `src/sase/llm_provider/codex.py` declares the explicit root channel with
`max_bytes: 32 * 1024`. The compiled bob-cli bundle measures 36,186 bytes (sase's is
22,105). Every Codex launch since Nov 17 falls back to legacy delivery (~60–75/day
across both hosts) because always-inlined memory sections grew past the cap.

> [!decision] codex_bundle = trim Bring every oversize bundle under 32 KiB. Prefer
> renderer-level compaction that helps every project, such as denser always-inlined
> listings. Otherwise move content from core to reference memory in the offending
> project. All memory edits go through `/sase_memory_write`. bob-cli is a different SASE
> project; open it with `/sase_repo`.

> [!decision] codex_bundle = raise Write a delivery test that proves Codex honors a
> larger explicit bundle. Raise `max_bytes` to the verified limit with headroom. Trim
> only if a bundle still exceeds it.

Either way, add a render-size gate. A `sase doctor` check renders each enabled project's
root bundle for every provider that declares `max_bytes` and fails when a bundle exceeds
it. A `just check`-reachable test does the same for sase's own bundles, so growth is
caught at authoring time. Update `docs/instruction_bundles.md`.

### Item 10: advance the generation on request_id_conflict in bead_task_triage (F9)

Root cause: in `src/sase/scripts/sase_chop_bead_task_triage.py`, a gate-create failure
logs and `continue`s without advancing `generation`. The deterministic
`_request_id(project, bead, generation, kind)` therefore repeats forever against a gate
journal that rejects a reused id whose spec fingerprint changed. This caused 1,517
identical failures on athena in a week, and the affected bead never got its triage gate.

Required change: on `request_id_conflict`, advance the generation, or adopt and cancel
the journaled request, and then recreate. Other failure kinds keep the deterministic
retry.

Test: a bead whose body changes between a failed create and the retried create gets a
gate on the next pass.

## Phase `hygiene`: gateway, notification hygiene, and retention (items 11-13)

### Item 11: gateway WAL `-1` handling and stale-while-revalidate snapshots (F11)

Root cause: `PRAGMA wal_checkpoint(TRUNCATE)` legally returns `(-1,-1,-1)` when busy.
sase-core `crates/sase_core/src/agent_scan/index/checkpoint.rs` feeds those values into
`u64::try_from(...)`, so every busy checkpoint errors with "out of range integral type
conversion attempted" and never checkpoints. Separately, apollo's gateway logged
presentation snapshot builds of 158 s and ~30 min, which starved waiters into
`504 Gateway Timeout`.

Required changes: treat `-1` as busy/skipped (`Option<u64>` or a clamp, with
`checkpoint_busy` reported), and move the pin. Use passive checkpoints on read paths.
Serve cached snapshots stale-while-revalidate so a slow rebuild never blocks waiters.

Coordination: open epic sase-1s5 targets snapshot _build time_. Stale-while-revalidate
complements it rather than duplicating it, so read sase-1s5 and record the relationship
there.

### Item 12: notification hygiene bundle (F12)

1. **remote-attention dedup.** `_dedup_key` in `src/sase/dispatch/attention_inbox.py`
   includes `request_id:revision`, so re-emissions and edits create new unread rows.
   Athena has 1,615 such rows all-time, including duplicate pairs with identical
   revisions. Key by stable gate/bead identity instead. A new revision updates the one
   row in place; an identical re-emission is a no-op. Preserve closed sase-168.1's
   guarantee that a local dismissal sticks for its revision.
2. **wait_checks amplification.** One failed blocker fans out to every downstream waiter
   and +1s each to the 500 cap. Athena shows 4,590 +1s across its last 50 rows and 412
   "can never self-resolve" rows. Key notices by root blocker, freeze +1s once the
   blocker is terminal, and rely on phase `outage`'s store guard.

   > [!decision] blocked_waiters = offer The root-blocker notice offers a one-key action
   > that expires the terminally blocked waiters.

   > [!decision] blocked_waiters = auto Terminally blocked waiters expire automatically,
   > with one summary notice naming what expired.

3. **Orphaned `interaction_requests/`.** Apollo has ~2,500 pending dirs and athena
   ~5,030, mostly `task_triage`, whose notification was dismissed or whose bead was
   resolved. Add a reconciliation pass that settles them. Include the owed triage-volume
   fix documented by the `quiet-task-beads` rule
   (`docs/reference/automation/notifications.md`). Build on closed sase-ir's
   `_reconcile` fix in `sase_chop_bead_task_triage.py`.
4. **Oversized inline blobs.** `action_data` and `notes` embed 170–218 KB prompts and
   error dumps, and `action_data` is over half the active store. Store payloads over 8
   KB as sidecar files, and prune the 314/522 broken `files` references. The store
   format lives in sase-core.
5. **Success toasts, missing dedup keys, and nudge floors.** Make `file-hooks` successes
   silent (210 of 212 apollo rows are ✅). Add a stable dedup key to `workspace-rescue`;
   phase `outage` already handles `disk_pressure`. Add a size floor to ACE-prune
   priority nudges, which currently fire for 873 KiB on a host with 283 GiB free. Read
   the `ace.md` reference memory before touching `src/sase/ace/`.
6. **agent_state_refresh re-logging.** About 1,000 redundant warning lines per day per
   host re-examine unchanged terminal agents, up to 246 times per agent. Persist
   verdicts with an evidence fingerprint. Coordinate with open epic sase-1s5, which owns
   agent_state_refresh's dismissed-bundle lookup.
7. **No-op claim-ledger writes.** `src/sase/running_field/_hold.py` records "already
   pinned (no-op)" in `workspace_claims.jsonl` as an error thousands of times per night.
   Skip the write when nothing changed.

### Item 13: retention and clutter sweep (F13)

Add age-based reapers, preferably through the managed reaper infrastructure from sase-zw
and phase `outage`, for each class:

- athena's `tools/logs`: 489 MiB older than 30 days, against a stated 14-day
  post-settlement horizon. Apply the horizon.
- Error-digest files: verify phase `noise`'s pruning covers them.
- The 222 MiB `agent_artifact_index.sqlite…bak` from Sep 26, and similar stale index
  backups.
- Orphaned `.agent_name_registry.json.*.tmp` files (108 MB across hosts). Also fix the
  writer that leaks them.
- About 20,300 stale `.log.lock` files in `~/.sase/procs/logs/`. Also fix the leak
  source.
- `tui.log` age rotation. It has none, which is why long-resolved patterns still
  dominate all-time counts.

Verify `artifact_run_prune` is unblocked ("2 protection sources are unreadable", 19 +1s,
partly addressed by `b0de76422d`); fix it if not. The reapers perform the one-time
cleanup on their first run after install. Any extra manual cleanup on apollo or athena
goes through `/sase_gate`.

Out of scope, per the audit: capping bob-cli workspace counts, and changing
`sase.service` kill policy.
