---
tier: epic
title: 'Task-bead backlog, epic 2: fix the five most important open task beads'
goal: 'The five most important workable task beads in the sase project are resolved
  on master and closed with recorded evidence: sase-1og (the check-full flake-baseline
  gate exits 2 before judging anything), sase-18t (ToolRun store "database is locked"
  under parallel load), sase-1a7 (the Files ctrl+j anchor-scroll node that keeps Full
  CI red), sase-12f (the monitor-capacity fakey e2e that exits 124 under load), and
  sase-1ot (owner-approved rotation and purge of 27 historical secret-scanner hits
  in public sase--agents history). The land agent then starts the next backlog epic,
  five beads at a time, until no workable task bead remains.

  '
decisions:
  dormant_flakes:
    ask: Should the lander close flakes with zero failures in the last 30 days as
      canceled?
    default: close
    choices:
      close: Lander re-checks every evidence source and closes still-quiet flakes;
        a later +1 reopens them
      keep: Dormant flakes stay open and compete for later epics like any other bead
    why: Single-sighting flakes like sase-1ax and sase-1f5 are near 30 quiet days
    answer: close
phases:
- id: flake-baseline
  title: Make the flake-baseline gate reach a truthful verdict again (sase-1og)
  depends_on: []
  size: medium
  description: 'flake-baseline: drop the duplicate fixed-at directive that makes tools/selection_health
    --fail-on-new-flake exit 2, add a regression test that parses the committed baseline,
    then reconcile the 64 exceedances the unmasked gate reports (fixed-at for verified
    fixes, baseline entries only under existing open beads, follow-ups for the rest),
    and close sase-1og.'
- id: toolrun-busy
  title: Stop the ToolRun store failing with "database is locked" under parallel load
    (sase-18t)
  depends_on:
  - flake-baseline
  size: large
  description: 'toolrun-busy: identify which sase-core ToolRun store transactions
    hold the SQLite write lock while launcher, settle and show paths read it, fix
    the contention at its source in sase-core with a principled busy policy, move
    the core pin, retire the tests/tool/ baseline entries with fixed-at, and close
    sase-18t.'
- id: capacity-e2e
  title: Make the weight-two land-agent-session capacity e2e load-robust (sase-12f)
  depends_on:
  - flake-baseline
  size: large
  description: 'capacity-e2e: find what consumes the fakey land agent''s 5 s budget
    under load in tests/fakey/test_monitor_capacity_e2e.py (delayed child bootstrap,
    capacity scan under the host-wide slot lock, or harness polling), fix the cause
    instead of the timeout, prove it with a loaded stress loop, and close sase-12f.'
- id: deck-anchor
  title: Fix the Files ctrl+j page-anchor scroll race (sase-1a7)
  depends_on:
  - flake-baseline
  size: large
  description: 'deck-anchor: reproduce the intermittent wait_for_anchor_scroll timeout
    in test_files_ctrl_j_scrolls_page_anchor_to_top (it fails about 1 in 4 even in
    isolation), fix the Files spread scroll/anchor race in production code, prove
    it with isolated and loaded stress loops, and close sase-1a7.'
- id: agents-secret-purge
  title: Rotate and purge the historical secret-scanner hits in public sase--agents
    history (sase-1ot)
  depends_on: []
  size: large
  description: 'agents-secret-purge: triage the 27 private-audit findings without
    printing values, hand the owner a rotation checklist, design and gate a reviewed
    history purge of sase-org/sase--agents that quiesces publication and resets every
    clone, re-audit the purged tip, and close sase-1ot.'
proposed_by: bbugyi200.athena.sase-1oh.land--1
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-27 09:20:57
status: done
bead_id: sase-1ou
---

- **PROMPT:** [prompts/202610/backlog_epic_2_top_five_task_beads.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/backlog_epic_2_top_five_task_beads.md)
- **BEAD:** [sase-1ou](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ou/README.md)

# Plan: Task-bead backlog, epic 2: the five most important open task beads

## Background

Epic `sase-1oh` (backlog epic 1) landed on 2026-10-27 and closed its five beads. Its
lander then re-triaged the open task beads (sase master `c687d8b6eb`):

- **28 more beads were closed** with evidence notes:
  - 27 dormant flakes, closed `canceled` under `dormant_flakes = close`. Each had zero
    gate-countable failures in the 1,071 full-lane selection-health records of the last
    30 days.
  - `sase-12s`, closed `done`: it is covered by epic `sase-1ok`'s stranded-bead wait
    alert.
- **About 375 task beads remain open.** The epic-1 verdict artifact
  `file:explicit:accb34387e533e1c0cc7aca8` still holds per-bead verdicts, importance,
  size and evidence. Read it with
  `sase artifact read file:explicit:accb34387e533e1c0cc7aca8 "<why>"`. It is a snapshot
  from 2026-10-26: treat every row as a lead and re-verify before acting.
- **Excluded as not workable:**
  - snoozed: `sase-po`, `sase-1m7`;
  - owned by active epics: `sase-1ol` (sase-1om), `sase-1bw` (sase-1nf), `sase-1f3` and
    `sase-1nc` (sase-1nf.14.1), `sase-1dj` (sase-1ih).

Ranking used the owner's order: red gates and release blockers, then data loss or
security, then host health, then recurring CI flakes, then everything else. Full CI is a
release prerequisite, so a flake that keeps Full CI red counts as a release blocker. The
five beads:

- **`sase-1og`: deterministic red gate.** `just check-full`'s last step exits 2 on every
  run. Verified at `c687d8b6eb`.
- **`sase-18t`: release-gate flake cluster** (importance 5). The ToolRun store's 250 ms
  `busy_timeout_ms` loses under parallel load. Nine of the 64 nodes the unmasked flake
  gate reports are `tests/tool/` nodes. Two are known siblings with the same "database
  is locked" signature:
  - `test_detach.py::test_watchdog_reports_ended_join_monitor`;
  - `test_settlement_retention.py::test_handoff_end_to_end_publishes_one_notification`.
- **`sase-1a7`: release-gate flake** (importance 5). Full CI py3.12 failed on this node
  on 10-23 (40000661173), 10-25 (40138926091), and twice on 10-26 at `49b1b9088b`.
- **`sase-12f`: release-gate flake** (importance 5). Full CI 40199727112 attempt 2
  (10-25, py3.13) failed only this node. 67 local full-lane failures through 10-18; +8.
- **`sase-1ot`: security, owner-approved.** 27 scanner hits remain in public git
  history. The owner approved rotation plus purge at gate
  `custom-57bc7a62-1331-4a3b-900d-a1b2d4612e5d` (scope `both`).

**Known red on master, owned elsewhere.** Master Gate and Full CI currently fail on
nodes that active epics already track as DISCOVERED ISSUEs. This epic must not fix them.
Phase workers treat them as pre-existing when `sase tool run check` reports them, and
cite the owning epic:

- `tests/test_run_agent_runner_slot_lineage.py`,
  `tests/test_run_agent_runner_slot_scan_capacity.py` and
  `tests/test_core_agent_scan_options.py` capacity nodes: `sase-1oj`;
- `tests/test_agent_load_cli.py` (hard-coded `athena` hostname) and
  `tests/completion/test_snapshot.py`: `sase-1om`;
- `tests/agents_sync/test_cli.py::test_sync_help_distinguishes_full_cached_and_refresh_modes`
  (`--retry-key`): `sase-1nv.5.1`.

## Global constraints for every phase

- **Own your bead.** Each phase names the task bead it owns. Read it first with
  `sase bead read <id> -r "<why>"`; its description, `task_type_fields`, notes and +1
  evidence hold the repro. Re-verify that the defect still reproduces at your base
  before changing anything. If other work already fixed it, close it `done`, citing the
  fixing commit and your reproduction.
- **Close on evidence.** Close the owned bead with
  `sase bead close <id> --note "<evidence>"` and the default `done` resolution. Close it
  only after the fix is verified. The note names your phase bead, your verification
  commands and their results.
- **No shortcuts.** Never do any of these just to get green:
  - weaken an assertion;
  - raise a timeout, cap or budget;
  - add a symvision pragma or suppression;
  - `xfail` or skip a node;
  - add a flake-baseline entry for a node with no open bead.

  A changed production policy (for example a store's busy handling) is a product fix
  only when lock-holder or timing evidence justifies it. Tests must exercise that
  production policy, not a test-only knob.

- **Flake proof.** A flake fix needs a before/after failure rate from the same stress
  loop under comparable load, recorded in the close note. Run the loop isolated and
  under parallel load (`-n` workers plus a concurrent load source). Use `/sase_monitor`
  for long loops; never end a turn to wait.
- **Flake baseline.** When your fix retires a node listed in
  `tests/reproducible_flake_baseline.txt`, replace its entry with a
  `# fixed-at: <UTC instant> <node id>` directive. Put a preceding comment naming your
  bead and fixing commit, as the file's header prescribes. Then run
  `.venv/bin/python tools/selection_health --fail-on-new-flake` and confirm your nodes
  no longer appear.
- **Rust-core boundary.** Shared backend behavior belongs in sase-core's `sase_core`
  crate. Open it with `sase repo open sase-core` and read its `AGENTS.md`. When sase
  must build a newer core, move `sase-core-revision.txt` past that commit with
  `just ratchet-core-revision` (see `docs/rust_backend.md`). Never edit sase-core
  versions or changelogs. Active epics (for example `sase-1oe`) may move the pin too:
  rebase before you ratchet, and re-ratchet if the pin moved under you.
- **Verification.** Run `sase tool run check` in every repo you change. Run targeted or
  repeated nodes directly with `.venv/bin/python -m pytest`. `just check-full` is
  explicit-only (decision `check-full-is-explicit`); do not run it.
- **Human-only and outward-facing actions.**
  - Never run `just install`, `just install-dev`, raw `sudo`, a force-push, a history
    rewrite, or any GitHub repository settings change. Propose them through `/sase_gate`
    (and `/sase_sudo` for privileged commands).
  - Never print, note, attach publicly or commit a secret value.
- **Memory.** Read the notes your work touches before changing code:
  - `tui` before TUI, pilot-test or visual-snapshot work;
  - `sase_beads` before closing beads;
  - `lint_and_test` before finishing.

  Do not edit `sase/memory/**`. If a note becomes inaccurate, append a
  `PROPOSED FOLLOW-UP:` note to your phase bead.

- **Beads.** Phase workers never create beads. Record discovered work as
  `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own phase bead.

---

## Phase `flake-baseline`: make the flake-baseline gate reach a truthful verdict (sase-1og)

**State at planning time** (`c687d8b6eb`, 2026-10-27).

- **The red step.** `.venv/bin/python tools/selection_health --fail-on-new-flake`, the
  last step of `just check-full`, exits 2:
  `tests/reproducible_flake_baseline.txt:408: duplicate fixed-at entry for tests/test_check_sase_core_rs_bindings_tool.py::test_dev_extension_exposes_every_collected_name`.
  The parser at `tools/selection_health:~112-115` raises on a duplicate.
- **The two directives.**
  - line 380: `2026-09-08T22:11:29Z`, from `a5e642c929`;
  - line 408: `2026-09-22T13:25:12Z`, from `e8d238688f`.
- **The verdict behind it.** The planner deleted line 380 in a temporary copy and passed
  it with `--flake-baseline`. The gate then exited **1**: 64 reproducible flakes exceed
  the baseline (records after 2026-08-15T17:22:27Z, at most 5 failures per run). It also
  listed 10 no-longer-collectable node IDs and some `# fixed-at:` entries that retire
  nothing. The 64 include:
  - `tests/tool/` siblings of `sase-18t`;
  - the zsh `sbd` node of `sase-13a`;
  - the registry node of `sase-1em`;
  - `sase-137`'s palette node;
  - deterministic breakages from past landings, such as the terminology and parser-help
    goldens;
  - the epic-owned capacity and completion-snapshot nodes listed in Background.
- **Context.** `check-full` has been masked at this step since 2026-09-22. Fixing only
  the duplicate turns exit 2 into exit 1, so this phase must also reconcile the
  exceedances.

**Steps.**

1. **Fix the duplicate.** Keep the later `2026-09-22T13:25:12Z` directive, which retires
   the evidence of both fixes. Give it a preceding comment naming both commits, and
   delete the other.
2. **Add a regression test** that loads the committed baseline through the tool's own
   parser, so a duplicate or malformed directive fails `just check` instead of only
   `check-full`. It must use the tool's parse path, not a copy of its rules.
3. **Reconcile every exceedance.** Run the gate and classify each node:
   - **(a) Deterministic breakage since fixed on master.** Verify that the node passes
     at your base and find the fixing commit. Add a `# fixed-at:` directive with a
     comment naming the commit.
   - **(b) Currently red because of an active epic.** This covers the Background list.
     Do not baseline these; list them, with their owning epic, in your close note.
   - **(c) Load flake with an existing open flake bead.** Search with
     `sase bead search <node>` and the verdict artifact. Add a baseline entry under
     `# <bead-id>`, following the header convention. The `tests/tool/` siblings go under
     `# sase-18t`; phase `toolrun-busy` retires them.
   - **(d) Load flake with no bead.** Write one `PROPOSED FOLLOW-UP` per root-cause
     cluster, giving the nodes and the gate-countable records. The land agent files the
     beads. Do not baseline these.
4. **Prune.**
   - Remove `# fixed-at:` entries the gate reports retire nothing.
   - Remove node entries the gate reports are no longer collectable.
   - Remove entries whose bead is closed `done`: `sase-sf`, `sase-xb`, `sase-12h` and
     `sase-x6` per the bead's description, plus any others you find.
   - Leave entries of the dormant flakes closed `canceled` on 2026-10-27 in place: they
     cost nothing while quiet, and a recurrence should arrive as a +1 rather than a red
     gate.
5. **Verify.**
   - The new regression test passes.
   - The gate no longer exits 2. Record its exit code; every remaining exceedance is
     class (b) or (d) and named in your close note.
   - `sase tool run check` passes, apart from the known epic-owned reds.
6. **Close `sase-1og`.**

## Phase `toolrun-busy`: stop the ToolRun store failing with "database is locked" (sase-18t)

**State at planning time.**

- **The store.** The ToolRun ledger lives in sase-core. `src/sase/core/tool_run.py` is
  the thin Python adapter over `sase_core_rs`. Every adapter call defaults to
  `busy_timeout_ms=250`; other callers repeat that default, for example
  `src/sase/tool/receipts.py:43`, `src/sase/ace/tui/tool_runs/snapshot.py:104`,
  `summaries.py:194` and `src/sase/history/chat_catalog_provenance/artifacts.py:107`.
- **Prior fixes.** `0a1bdf5279`, `e5c02dc8a4` and `4d8940d4b7` raised the timeout for
  specific helpers only, and siblings kept failing through 10-18/10-19, including a
  Master Gate shard.
- **The failure.** `RuntimeError: tool run store is busy: database is locked` under
  full-lane load; the nodes pass in isolation. Affected nodes include:
  - `tests/tool/test_lifecycle_controls.py::test_stop_live_inline_run_settles_stop_requested`
    (the bead's node);
  - `test_settlement_retention.py::test_handoff_end_to_end_publishes_one_notification`;
  - `test_detach.py::test_watchdog_reports_ended_join_monitor` and
    `::test_watchdog_stops_unjoined_run_when_starter_dies`;
  - `test_executor.py::test_nested_run_has_parent_and_no_duplicate_logs`;
  - `test_inline_escalation.py::test_fast_success_matches_inline`;
  - `test_demand_runs.py::test_foreground_run_records_context_usage_and_grant`;
  - `test_known_continuation.py::test_agent_run_stops_on_a_new_item`;
  - `test_wrapper_fidelity.py` group-kill nodes.

  Check each against your evidence before claiming it.

- **Possibly the same mechanism:** `sase-19t`
  (`test_handoff_end_to_end_settles_through_proc`).

**Steps.**

1. **Find the lock holder.** Reproduce the contention under parallel load: loop the
   `tests/tool/` nodes at `-n 8` with a concurrent load source. Instrument the sase-core
   store, or use SQLite tracing, to see which transaction holds the write lock and for
   how long. Check the journal mode, whether readers open write transactions, and how
   long settle, retention and triage writes run.
2. **Fix it in sase-core** at the source:
   - shorten or split long write transactions;
   - use `BEGIN IMMEDIATE` for writers and make sure reads never escalate;
   - centralize one justified busy policy (for example bounded retry with backoff on
     `SQLITE_BUSY` for idempotent operations) in the store, instead of 250 ms per call.

   Update the Python adapter and callers to use the core policy. Remove the scattered
   per-call overrides where the policy now covers them.

3. **Integrate.** Run `sase tool run check` in sase-core, ratchet sase's core pin
   (rebase first), then run `sase tool run check` in sase. Add sase-side tests only if
   the binding usage changes.
4. **Prove it.** Record before/after failure rates from the same loaded loop. Retire the
   `# sase-18t` baseline entries with `# fixed-at:`, and confirm the gate no longer
   lists them.
5. **Siblings.** If `sase-19t` (or another open bead) has the same root cause, append a
   `PROPOSED FOLLOW-UP` naming it and your evidence. The land agent closes it as a
   duplicate.
6. **Close `sase-18t`.**

## Phase `capacity-e2e`: make the weight-two land-agent-session capacity e2e load-robust (sase-12f)

**State at planning time.**

- **The node.**
  `tests/fakey/test_monitor_capacity_e2e.py::test_weight_two_land_agent_session_retains_one_claim_through_real_dispatch_and_delayed_child_bootstrap`
  (line ~45). The bead's `node_id` still has the pre-rename `..._land_family_...` name.
- **The failure.** The test monkeypatches the harness `_WAIT_TIMEOUT` and
  `_FAKEY_RELEASE_TIMEOUT` to 5.0 s (lines ~77-78). Under load, the fakey land agent
  exits 124, then passes solo in about 65-70 s.
- **Evidence.**
  - 67 local full-lane failures, 7 of them in gate-countable records, the last on 10-18;
  - Full CI 40199727112 attempt 2 (2026-10-25, py3.13) failed only this node.
- **Possibly relevant code.** The runner-slot path takes a capacity-only scan under the
  host-wide slot lock. `sase-1ho` measured about 11 s for it. Epic `sase-1oj` pruned
  that scan (`e7d9f7a479`, 2026-10-27). Check whether the failure rate changed after
  that commit before assuming a cause.

**Steps.**

1. **Reproduce and time it.** Stress the node under parallel load at your base, and
   record where the wall-clock goes: dispatch, the delayed child bootstrap, the slot
   scan, release, and harness polling.
2. **Fix the cause.**
   - If production latency dominates (for example scan work under the lock), fix that in
     the owning code (sase-core where shared).
   - If the harness waits on wall-clock where an event exists (a marker, a done file, a
     notification), make it event-driven.

   Do not raise `_WAIT_TIMEOUT` or `_FAKEY_RELEASE_TIMEOUT`.

3. **Prove it.**
   - Record before/after loaded stress rates.
   - Run the whole `tests/fakey/test_monitor_capacity_e2e.py` file, plus
     `sase tool run check`.
   - Retire the node's baseline entry with `# fixed-at:` if one exists.
4. **Close `sase-12f`.** Name the renamed node in the note.

## Phase `deck-anchor`: fix the Files ctrl+j page-anchor scroll race (sase-1a7)

**State at planning time.**

- **The node.**
  `tests/ace/tui/widgets/decks/test_deck_spread_pilot.py::test_files_ctrl_j_scrolls_page_anchor_to_top`.
  Its final wait is the shared `wait_for_anchor_scroll(...settled_on_live_target)` (5.0
  s) in `tests/ace/tui/widgets/decks/_deck_settle.py`.
- **History.**
  - Repairs: `7dd39ff95f` ("defer Files spread scroll"), and sase-1mk.3, whose note
    claimed a fix on 10-19.
  - It recurred after both: Full CI py3.12 on 10-23 (40000661173), 10-25 (40138926091),
    and twice on 10-26 at `49b1b9088b`.
  - sase-1b2.11 saw it fail about 1 in 4 **isolated** runs on a clean tree. The failure
    is a real race, not only load.

**Steps.**

1. **Read the `tui` memory note.**
2. **Reproduce in isolation** with a repeat loop. Capture the scroll offset, the anchor
   target and the layout or refresh events at failure time.
3. **Find the race.** Candidates:
   - the deferred scroll firing before the Files spread's layout settles;
   - a background refresh re-anchoring after ctrl+j;
   - the anchor target changing between key press and paint.
4. **Fix it in the production Files spread scroll/anchor code**, so ctrl+j converges on
   the page anchor deterministically. Keep the 5 s wait.
5. **Prove it.**
   - Record before/after rates for an isolated loop (at least 50 runs) and a loaded loop
     (the `tests/ace/tui/widgets/decks/` directory at `-n 6`, repeated).
   - Run the deck pilot file and `sase tool run check`.
   - Retire any baseline entry with `# fixed-at:`.
6. **Close `sase-1a7`.**

## Phase `agents-secret-purge`: rotate and purge the historical scanner hits (sase-1ot)

**State at planning time.**

- **The audit.** A private audit of the public `sase-org/sase--agents` sidecar ran at
  tip `5066d26a` with core scanner rules v1. Of 138,525 tracked files, it found 27 hits
  and skipped 1, with 0 errors. Two private attachments on bead `sase-1oh.5` hold the
  details (paths and rule ids only):
  - the per-finding report `audit-report.md` (sha256 `53a69059…66a6`);
  - rollout material `rollout-gate-material.md` (sha256 `e20f23a3…f978f`).
- **Already in place.**
  - New publications are scanned and withheld (sase-1oh.5.1).
  - Push protection is enabled on sase--agents.
  - The owner approved rotation plus purge (scope `both`) at gate
    `custom-57bc7a62-1331-4a3b-900d-a1b2d4612e5d`. That gate only recorded the request.
- **Adjacent risk.** `sase-1or` (open): an unattributed GH013 rejection re-pushes the
  same agents batch on every drain. A history rewrite must not trigger re-push storms
  from stale hidden clones.

**Steps.**

1. **Triage each finding** from the private report: known-value, credential-pattern or
   env-dump; and live credential, test/placeholder, or false positive. Work on a local
   checkout opened with `/sase_repo`. Never print, note or commit a value. Write the
   triage as a **private** (`-K`) attachment on your phase bead, using paths, rule ids
   and verdicts only.
2. **Rotation checklist.** For every live credential, name the service and the human who
   rotates it, as a private attachment. Rotation is human work. Raise it as its own
   `/sase_gate`, or as part of step 4's gate, so the owner can confirm rotation before
   the purge.
3. **Purge procedure.** Write a reviewed, reproducible procedure:
   - **The tool.** For example `git filter-repo` with a path or blob list derived from
     the report.
   - **Quiesce first.** Stop agents-sync publication on athena, apollo and mac before
     the force-push.
   - **The coordinated force-push.**
   - **Reset every hidden and primary sidecar clone** on all three hosts.
   - **Consistency afterwards.** Publication outbox and ledger consistency, so withheld
     or blocked items are not replayed. Check this against `sase-1or`'s re-push path.
   - **GitHub caveats.** Cached views, forks, and requesting a GitHub cache purge for
     sensitive data.

   Dry-run the rewrite on a scratch clone and compare tree counts.

4. **Gate it.** Propose execution through `/sase_gate` with the exact commands per host.
   Never rewrite history or force-push without that approval. The gate's follow-up:
   - runs the procedure;
   - re-runs the private audit on the purged tip, recording aggregates only;
   - confirms publication resumes cleanly.
5. **Close `sase-1ot`** after the re-audit shows no live-credential hits, citing the
   gate ids and aggregate counts. If the owner declines or defers the execution gate,
   record that on `sase-1ot`, leave it open, and finish the phase with a
   `PROPOSED FOLLOW-UP`.

---

## Landing (epic land agent)

Do not close the epic bead until steps 1–4 pass on the landed master tip.

1. **Status audit.** Run
   `sase bead read sase-1og sase-18t sase-1a7 sase-12f sase-1ot -r "<why>"`. Each must
   be closed `done` with a note citing verification evidence. The one exception is
   `sase-1ot` when the owner declined or deferred its execution gate: it stays open,
   with the owner's answer recorded.
2. **Re-verify on master** with quick checks:
   - **sase-1og:** the baseline regression test passes, and
     `tools/selection_health --fail-on-new-flake` does not exit 2. Every remaining
     exceedance is epic-owned or has a filed bead.
   - **sase-18t:** a short loaded loop of the `tests/tool/` nodes shows no "database is
     locked".
   - **sase-1a7** and **sase-12f:** a short isolated and loaded loop of each node
     passes. Scheduled Full CI runs after the fixes do not fail these nodes; wait for
     one with `/sase_monitor`. Other Full CI reds owned by active epics do not block
     this check.
   - **sase-1ot:** the gate ids and the aggregate re-audit result are recorded.
3. **If a bead is open or its check fails,** reopen it and its phase bead with
   `sase bead open`. Finish small gaps in this turn, or rerun
   `sase bead work <epic-id>`. Never close one of the five as `canceled` or
   `superseded`.
4. **Wrap up.** Run `sase tool run check`. Turn warranted `PROPOSED FOLLOW-UP:` notes
   into task beads through `/sase_new_task`, including the class-(d) flake clusters from
   `flake-baseline`. Close beads a phase proved to be duplicates (for example `sase-19t`
   if `toolrun-busy` showed the same root cause) as `superseded`, naming the keeper.
   Then close the epic bead.
5. **Start the next backlog epic** while workable task beads remain. "Workable"
   excludes:
   - snoozed beads;
   - beads owned by another active epic;
   - `in_progress` beads with a live holder;
   - flag beads whose removal condition is not yet met.

   Do this as the last action of the landing turn: `sase plan propose` ends the turn.
   1. **Re-triage.** List open task beads with `sase bead list -t task -n 0`. Use the
      verdict artifact `file:explicit:accb34387e533e1c0cc7aca8` as leads. Close every
      bead you can **verify** is a duplicate (resolution `superseded`, naming the
      keeper) or obsolete or already fixed (`canceled` or `done`), each with evidence.
      Add a pointer note to the keeper. Fan out read-only helpers if the set is large.
   2. > [!decision] dormant_flakes = close For each open flake bead, gather every
      > evidence source:
      >
      > - the selection-health full-run store (`~/.sase/test-selection/<project>`,
      >   30-day window, records with at most 5 failures), searched under both the
      >   current and any renamed node id;
      > - ToolRun check failures (`sase tool failures`);
      > - recent Full CI and Master Gate logs;
      > - the bead's +1 dates.
      >
      > Close each flake with zero failures across all of them as `canceled`, with a
      > "not reproducing in N runs over 30 days; +1 to reopen" note. Visual and
      > sase-core flakes are not in the full-run store; judge those by the other
      > sources.

      > [!decision] dormant_flakes = keep Leave dormant flakes open; they compete like
      > any other bead.

   3. **Rank and plan.** Rank the remaining beads by impact: red gates and release
      blockers, then data loss or security, then host health, then recurring CI flakes,
      then everything else. Write the next epic with `/sase_plan`. It owns the **five**
      most important beads in **at most five phases**, and one phase may own several
      related beads.
      - Leads from the 2026-10-27 triage, in rough order:
        - `sase-1os` (small, security: prompt migration writes legacy prompts into the
          sidecar unscanned);
        - `sase-1cb` (stitch create pushes auto-rebased commits without checking the
          combined tree);
        - `sase-1cs` with `sase-vr` (installs ignore uv.lock; CI-only Textual drift,
          which likely also explains `sase-1gg`'s `text-area--gutter` KeyError in Full
          CI on 10-26);
        - `sase-1nh` (verified live at `c687d8b6eb`);
        - `sase-1db` (the attachments-private store never verifies visibility);
        - `sase-163`;
        - `sase-1or`;
        - the `sase-x1` models-panel cluster (Full CI failures on 10-26);
        - `sase-19d`, `sase-1mu`, `sase-13a`, `sase-120`, `sase-1em`, `sase-o7`;
        - `sase-j0` (`in_progress` under a dead holder).
   4. **Decisions.** Keep them minimal. Use them to let the owner opt out of feature
      work, memory changes, or anything else whose value is questionable.
   5. **Pass the chain on.** Copy this landing section's chaining rules into the new
      plan's landing section, so every epic hands off to the next until no workable task
      bead remains. Do not carry the `dormant_flakes` decision forward unless dormant
      flakes remain.
   6. If no workable task bead remains, do not propose a plan. Report that the backlog
      is clear, listing the beads left only because they are snoozed, owned by another
      epic, or not-yet-due flags.
