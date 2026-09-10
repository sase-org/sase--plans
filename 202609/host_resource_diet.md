---
tier: epic
title: Host resource diet for parked runners and the notification store
goal: "Parked agent runners and the ACE TUI stop consuming CPU, RSS, and swap in
  proportion to on-disk history: runner-slot admission scans only live capacity state,
  and the notification store stays O(live) via compaction.

  "
phases:
  - id: capacity-scan-mode
    title: Capacity-only artifact scan in the Rust core
    depends_on: []
    size: medium
    description:
      "capacity-scan-mode: add an additive sase-core scan option that skips done
      artifact dirs before parsing and returns only running/waiting records, exposed
      through the scan-options wire and sase_core_rs with Rust tests and the
      revision-pin bump."
  - id: slot-poll-diet
    title: Make parked runners cheap
    depends_on:
      - capacity-scan-mode
    size: medium
    description:
      "slot-poll-diet: switch the runner-slot wait loop to the capacity-only scan, share
      one scan per poll window host-wide under the existing lock, and add jittered
      backoff with a slot-state change signal while preserving admission semantics."
  - id: notification-compaction
    title: Keep notifications.jsonl O(live)
    depends_on: []
    size: medium
    description:
      "notification-compaction: add crash-safe automatic compaction with a retention
      window to the Rust notification store, archiving old dismissed rows to a sibling
      JSONL file with tests and unchanged Python caller behavior."
  - id: verify-resource-diet
    title: Live verification and perf floors
    depends_on:
      - slot-poll-diet
      - notification-compaction
    size: small
    description:
      "verify-resource-diet: capture after-measurements on the live host, add perf-floor
      benches for the capacity scan and notification snapshot reads, and record residual
      hotspots on the epic bead."
proposed_by: bbugyi200.athena.0ih
create_time: 2026-09-10 11:44:14
status: wip
---

- **PROMPT:**
  [prompts/202609/host_resource_diet.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/host_resource_diet.md)

# Host Resource Diet: O(live) Runner-Slot Admission and Notification Store Compaction

## Problem

athena has been observed with load average ~31–46, 31+ GB of swap in use, and the
`sase ace` TUI pinned at 115–135% CPU. Direct measurement (2026-09-10) shows two SASE
code paths burning resources in proportion to _all history_ instead of _live state_:

**1. Runner-slot admission is O(entire artifact history), polled every 2 s, per parked
runner, under the host-wide lock.** Each runner parked in `wait_for_runner_slot()`
(`src/sase/axe/run_agent_wait_slots.py`) loops on `_RUNNER_SLOT_POLL_INTERVAL = 2`
seconds. Every iteration takes the exclusive host-wide `~/.sase/runner_slots.lock` and,
while holding it, calls `_scan_runner_slot_records()` →
`scan_agent_artifacts(sase_projects_dir())`. Measured on the live host:

- One scan visits 12,584 artifact dirs and parses ~30,000 marker JSON files: ~4.3 s in
  the Rust scanner plus ~1.1 s rehydrating 12,589 records into Python wire objects.
- A single scan balloons the Python process heap to ~930 MB peak RSS — which matches the
  observed ~500–900 MB RSS of every parked runner. With 28 runner processes alive and
  only 8 RUNNING, the 20 parked runners held ~16.6 GB RSS combined. That is the primary
  swap driver.
- The scan runs while holding the lock, so parked runners convoy: with K waiters the
  lock is held nearly continuously (K × ~5 s of work per 2 s interval), each parked
  runner burns ~0.5–1 core doing nothing, and genuine claim/release operations queue
  behind scans.

Only a handful of records matter for admission: records with a live `running` claim or a
`waiting` marker. Everything else (done history) is scanned, shipped to Python, and
discarded every 2 seconds.

**2. The ACE TUI re-parses a 43 MB notifications file that is 96% dead rows.** py-spy
sampling of a busy `sase ace` shows ~38% of samples inside `read_notification_snapshot`
(`src/sase/notifications/store.py`, Rust-backed).
`~/.sase/notifications/notifications.jsonl` is 42.9 MB with 10,844 rows of which only
438 are live — 10,406 are dismissed. One snapshot read costs 0.16–0.4 s and the file is
re-read whenever the notifications surface is dirty, which on a busy host is nearly
every tick. No compaction mechanism exists anywhere in the store (verified: no
compact/archive/prune/retention code in `src/sase/notifications/`).

Both problems share one root cause: hot paths whose cost grows with unbounded on-disk
history. Fixing them per the `rust_core_backend_boundary` rule means the scan and store
changes land in the linked `sase-core` repo (`crates/sase_core`) with wire/binding
updates, and the Python callers in this repo consume them.

## Non-Goals (candidate follow-up beads, not phases)

- The shared `CARGO_TARGET_DIR` on the spinning disk and concurrent `just check`
  cargo/pytest IO storms (infra/chezmoi config; see the `two-speed-verification`
  decision — bursty, not the steady-state burn).
- Re-architecting parked runners to not hold a full Python runtime while waiting.
- The dependency-wait loop in `src/sase/axe/run_agent_wait.py` (already cheap: an
  `os.path.exists` probe every 2 s with a 60 s fallback).

## Phases

### Phase 1: `capacity-scan-mode` — capacity-only artifact scan in the Rust core

size: medium depends on: (none)

In the linked `sase-core` repo, add a capacity-only mode to the agent artifact scanner
and expose it through the scan-options wire and the `sase_core_rs` binding:

- When the new option is set, the scanner returns only records relevant to runner-slot
  capacity: artifact dirs carrying a `running` marker or a `waiting` marker. Dirs with a
  done marker (finished runs) are skipped _before_ parsing their other marker files, so
  the per-dir cost for history is at most one existence check instead of ~2.4 JSON
  parses. Scan stats should still report dirs visited/parsed so the reduction is
  observable.
- Preserve existing semantics for all other scan modes; the new option must be purely
  additive to `AgentArtifactScanOptionsWire` (`src/sase/core/agent_scan_wire.py` on the
  Python side) and default off.
- Add Rust tests covering: mostly-done synthetic trees return O(live) records and parse
  O(live) marker files; running/waiting records keep every field the capacity snapshot
  consumes (`sase_core_rs.runner_capacity_snapshot` request fields, queue weights, wait
  priorities, timestamps, process identity for liveness probes).
- Follow this repo's established convention for landing a sase-core change that this
  repo consumes (revision pin `sase-core-revision.txt`, rebuild via
  `just rust-dev-install`, and any scan-wire floor like the pattern bead sase-wg used).

Acceptance: on a tree shaped like production (≥10k done artifact dirs, tens of
running/waiting), the capacity-mode scan parses marker files only for non-done dirs and
returns only running/waiting records; existing scan tests remain green in both repos.

### Phase 2: `slot-poll-diet` — make parked runners cheap

size: medium depends on: capacity-scan-mode

In this repo, rework the runner-slot wait loop in `src/sase/axe/run_agent_wait_slots.py`
so a parked runner is near-free:

- Switch `_scan_runner_slot_records()` to the phase-1 capacity-only scan mode. This
  alone removes the ~930 MB heap balloon and most of the per-poll CPU from every parked
  runner.
- Share scans across waiters: at most one capacity scan per poll window host-wide. Under
  the existing `runner_slots.lock`, persist the scan result (or a derived snapshot) with
  a freshness token; a waiter whose token is fresh reuses it instead of rescanning. Keep
  the check-and-claim atomic: a claim may only be made against state validated under the
  lock.
- Add jittered backoff to `wait_for_runner_slot()`: keep ~2 s responsiveness right after
  parking, back off toward a bounded maximum (~15–30 s) while the queue is unchanged,
  and reset to fast polling when slot state changes. Use a cheap change signal (for
  example a slot-state token file touched by claim/release and waiting marker mutations
  — `update_agent_artifact_index_for_marker_mutation` call sites show where marker
  mutations already funnel) so a freed slot wakes waiters quickly instead of waiting out
  the full backoff.
- Preserve admission semantics exactly: queue-weight validation, wait priorities and
  deference windows, serial-family claim reuse, fail-closed behavior when the limit is
  unavailable, and kill handling. The existing unit tests around
  `_try_claim_runner_slot` / `wait_for_runner_slot` must keep passing; extend them for
  snapshot reuse and backoff-reset behavior.

Acceptance: a runner parked for a slot on a production-shaped tree consumes <2% of a
core in steady state (versus ~50–100% today) and does not rehydrate full-history scan
records; slot claims still happen within a few seconds of capacity freeing.

### Phase 3: `notification-compaction` — keep notifications.jsonl O(live)

size: medium depends on: (none)

In the linked `sase-core` repo (the Rust notification store owns all reads/rewrites;
Python callers live in `src/sase/notifications/store.py`), add compaction with a
retention policy:

- Dismissed notifications older than a retention window (config-backed default on the
  order of 1–2 weeks) are moved out of `notifications.jsonl` into a sibling archive file
  (append-only, e.g. `notifications-archive.jsonl`) rather than deleted, so history
  remains greppable. Live and recently-dismissed rows stay, because the notification
  inbox UI can display dismissed entries (`include_dismissed=True`).
- Compaction runs automatically under the store's existing locking — for example during
  snooze-reconciling reads or rewrites when the dead-row count or file size crosses a
  threshold — so no operator action is needed. It must be crash-safe (atomic replace; a
  crash mid-compaction loses no live row).
- Decide per `sase/memory/sase_flags.md` whether the behavior change needs a feature
  flag; the note's guidance wins.
- Add tests: compaction preserves every live and in-window dismissed row byte-for-byte
  at the semantic level, snoozed rows are never archived while snoozed, snapshot counts
  before/after compaction agree, archive rows are valid JSONL.
- Python side: expose whatever thin call-through is needed and keep
  `read_notification_snapshot()` behavior identical for callers. Update the sase-core
  revision pin per convention.

Acceptance: after the first automatic compaction on a production-shaped store (~11k
rows, ~4% live), `notifications.jsonl` shrinks to O(live + recent dismissed) rows and
one snapshot read costs a small fraction of today's 0.16–0.4 s; no notification is lost
from the inbox UI, including dismissed entries within the retention window.

### Phase 4: `verify-resource-diet` — live verification and perf floors

size: small depends on: slot-poll-diet, notification-compaction

Prove the fixes on the live host and pin them with regression floors:

- Capture before/after evidence: `ps` CPU/RSS of parked `run_agent_runner` processes,
  py-spy samples of `sase ace` showing the `read_notification_snapshot` share, host swap
  and load. The "before" numbers are recorded in this plan's Problem section; the
  measurement commands are reproducible from it.
- Extend the perf bench/floor suite (`tests/perf/bench_agent_scan.py` and the
  perf-floors machinery that guards `scan_agent_artifacts.synthetic_6p_200pp`) with a
  capacity-mode scan floor on a mostly-done synthetic tree, and add a notification
  snapshot-read bench/floor on a mostly-dismissed synthetic store, so history-shaped
  regressions fail loudly.
- Record the after-measurements and any residual hotspots as notes on this epic's bead;
  genuinely new discovered work becomes proposed follow-ups per the epic-worker rules.

Acceptance: floors exist and pass; live measurements show parked runners and the TUI
notification path no longer among the host's top steady-state CPU/RSS consumers.
