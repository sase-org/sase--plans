---
tier: epic
title: Make machine capacity legible and correct
goal: "The header capacity chip always matches what runner admission sees on this
  machine, and every unit (or zero) can be traced to a named agent and a stated reason:
  on the chip, on every Running-bucket row, in a keyboard-reachable inspector, and from
  `sase agent load`. The athena screenshot case (6 running, 5.0 occupied) reads `5/8`,
  not `3/8`.

  "
decisions:
  mark_style:
    ask: How should the per-row occupancy mark be drawn?
    choices:
      glyph:
        One verified glyph plus the number (2-3 cells); the phase picks and proves the
        glyph
      text: Plain cap1 / cap0 / cap- / cap? tokens; 4-5 cells, no font or collision risk
    default: glyph
    why: Agent rows are already dense; a glyph plus a number keeps the mark short
    answer: glyph
  publish_snapshot_file:
    ask:
      Also write each computed capacity report to a published snapshot file for
      non-Python readers?
    default: false
    why: The token-keyed scan cache already aligns readers; another file adds SSD churn
    answer: false
  capacity_decision_record:
    ask:
      Add a decisions strand saying capacity displays read admission's snapshot, never a
      display roster?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: roster-fixes
    title: Stop the TUI roster from dropping monitors and mislabelling holders
    depends_on: []
    size: medium
    description:
      "roster-fixes: give monitor rows their supervisor PID from agent_meta (then the
      running marker) on every loader path, index holders by agent row instead of
      workflow-step rows (closing task bead sase-1ol), and carry runner_claim_owner_key
      through the roster projection, with loader-level regression tests."
  - id: core-accounting
    title: Per-record capacity accounting in the Rust snapshot
    depends_on: []
    size: medium
    description:
      "core-accounting: in sase-core, add an accounting list (state, charged weight,
      weight source, claim, reason code) covering every request record, a top-level
      validity flag, and a monitor-inherited weight input; bump the schema version,
      update the binding, and adapt sase's record builders and legacy fallback."
  - id: machine-report
    title: Shared authoritative machine capacity report
    depends_on:
      - core-accounting
    size: medium
    description:
      "machine-report: one read-only Python adapter that builds the machine's capacity
      report from admission's shared scan cache, a fresh liveness probe, active holds,
      and the Rust snapshot, plus limit provenance, machine label, age, and a bounded
      rescan policy."
  - id: load-cli
    title: The sase agent load command and shared capacity view model
    depends_on:
      - machine-report
    size: medium
    description:
      "load-cli: a TUI-free view model (holders, live rows holding nothing grouped by
      reason, queued rows with real blockers, diagnostics) rendered by a new `sase agent
      load [-j]` command, plus a Reference section listing what holds capacity."
  - id: list-holder-parity
    title: Retire the Python holder predicate behind sase agent list
    depends_on:
      - machine-report
    size: small
    description:
      'list-holder-parity: source `holds_runner_slot`, `runner_slot_holders`, and
      `runner_slots_in_use` in `sase agent list -j` from Rust accounting, delete
      `runner_slot_holder_dirs` and the `status == "RUNNING"` fallback, and add
      per-entry capacity fields.'
  - id: tui-report-feed
    title: Feed the TUI from the authoritative report
    depends_on:
      - roster-fixes
      - machine-report
    size: medium
    description:
      "tui-report-feed: compute the machine report off the event loop on token change
      plus a 10 s poll, drive the chip values and row queue context from it, keep the
      roster projection only as a pre-first-report estimate and drift witness, and log
      capacity_gauge/capacity_drift telemetry."
  - id: cap-chip
    title: Machine-labelled cap chip with provenance, staleness, fault, and overage
    depends_on:
      - tui-report-feed
    size: medium
    description:
      "cap-chip: rename the gauge to a machine-labelled `cap` chip that keeps its label
      in compact density, marks overrides, staleness, invalid accounting, and over-limit
      load, teaches running-versus-cap in the tooltip and `?` help, and fixes the
      misleading docs example."
  - id: occupancy-marks
    title: Occupancy marks on Running rows and a Capacity detail line
    depends_on:
      - tui-report-feed
    size: medium
    description:
      "occupancy-marks: attach accounting entries to Agent rows, draw a charged-now mark
      on every Running-bucket agent node and clan container (including collapsed clans),
      and add a never-suppressed `Capacity:` line to agent and clan detail cards."
  - id: capacity-inspector
    title: Keyboard-first capacity inspector
    depends_on:
      - load-cli
      - cap-chip
    size: medium
    description:
      "capacity-inspector: a modal rendering the shared view model, opened from a chip
      click, leader `,l`, and the command palette, with jump-to-agent rows, complete or
      explicitly truncated lists, real queue blockers, and a copy-snapshot action."
proposed_by: bbugyi200.athena.research.5e.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-26 20:19:41
status: wip
---

- **PROMPT:**
  [prompts/202610/machine_capacity_transparency.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/machine_capacity_transparency.md)

# Plan: Make machine capacity legible and correct

## Why this epic exists

On athena at 17:27:51 the Agents header read `6 running` on the left and `load: 3/8` on
the right. The research report
`research:202610/machine_load_accounting_transparency/machine_load_accounting_transparency.md`
(read it with `sase artifact read` before starting any phase; if the read reports the
artifact missing, run `sase repo open research -r "<why>"` first) reconstructed the real
state:

- three workers at 1.0;
- two verify monitors (`TESTING`) at 1.0;
- one `-w 0` cutover monitor (`WAITING CUTOVER`) at an explicit 0.

That is 6 running and 5.0 occupied. The gauge dropped the two verify monitors.

The report's findings, which this epic implements in full:

- **C1 (P0).** The TUI's exact-delta loader builds live monitor rows with `pid=None`.
  Monitors keep their supervisor PID only in `agent_meta.json` and the project-spec
  RUNNING claim, and `workflow_state.json` has `pid: null`. Rust treats a monitor with
  no PID as not started, so its capacity silently vanishes. Workers survive the same
  path because their `workflow_state.json` carries the PID.
- **C2.** The gauge re-runs the admission formula over the TUI's lossy, windowed,
  eventually-consistent roster instead of admission's own scan. It has no authoritative
  source.
- **C3.** The roster projection drops `runner_claim_owner_key`.
- **C4.** Invalid weights fail admission closed but show as free capacity.
- **C5.** Every tooltip holder is labelled `checkout`/`diff` (also filed as task bead
  `sase-1ol`).
- **C6.** There are three implementations of "who holds capacity".
- **C7.** The Rust snapshot cannot say why a live record holds nothing.
- **C8.** Over-limit load reads as plain "at capacity".

The UX gaps also need closing:

- `running` and `load` look like the same quantity.
- Collapsed clans hide every occupancy signal.
- The tooltip is hover-only, truncated, and mislabelled.
- Limit provenance and scope are invisible.
- Compact density drops the label.

### Principles every phase follows

- **Keep the formula.** Keep weighted claims, max-within-claim, verify monitors at the
  starter's weight, and `-w 0` holding 0. This epic fixes what feeds the formula and
  what it reports, not the policy.
- **One snapshot for every display.** The chip, row marks, inspector, `sase agent load`,
  and `sase agent list -j` all read the snapshot admission uses. It is computed by the
  same Rust function over admission's shared, token-keyed scan cache
  (`runner_slots.scan.json`), with a fresh liveness probe. Acceptance: the TUI chip,
  `sase agent load -j`, and `_try_claim_runner_slot` agree on `occupied_capacity` and on
  the claim set, regardless of viewport, query, fold, tribe, project, machine tab, or
  roster completeness.
- **The Rust core classifies; Python renders.** Rust in sase-core owns every decision
  about whether a record holds capacity, and why or why not (see the
  `rust_core_backend_boundary` memory). Python may do the side effects admission already
  does in Python: reading the scan cache, probing PIDs, reading config. Python must
  never re-derive occupancy. Python adapters stay thin.
- **Display reads are read-only and never block admission.** They never take
  `runner_slots.lock`, never prune holds, never write waiting markers, and never bump
  `runner_slots.token`.
- **An unknown contribution never looks like spare capacity.** When accounting is
  invalid, unavailable (an older core), or stale, the display says so.

### Non-goals

- No change to the capacity formula, admission policy, or weights.
- No re-bucketing of `WAITING CUTOVER`/`TESTING` statuses out of the Running group. The
  research rejects this as a load fix: the occupancy marks explain zero-holding rows.
- No remote-machine capacity. The chip always describes the local machine and says so.
  Remote capacity is never inferred.
- Not the `sase-1oi.1` list-freshness gap. That bug is in the Agents-list
  watcher/delta/sanity path and belongs to the `sase-124` epic.
- No tracing of past over-limit incidents. The display only shows and explains overage.

### Interaction with in-flight work

The `sase-1oj` epic ("Cut athena's SSD writes") prunes the `capacity_only` scan:

- `sase-1oj.1` is done.
- `sase-1oj.4` ("Adopt the pruned capacity scan in sase") is in progress. It also
  ratchets `sase-core-revision.txt`.

The pruning shrinks `runner_slots.scan.json` from about 104 MB to under about 1 MB. That
is what makes reading the cache from the TUI affordable.

- `machine-report` and `tui-report-feed` must confirm that `sase-1oj.4` has landed
  before they rely on that cache size. If it has not, note it on the phase bead.
- Do not add steady-state SSD writes. That epic's goal applies here too.
- `core-accounting` also moves `sase-core-revision.txt`. Whichever lands second rebases
  onto the other's pin.

## Phase roster-fixes: stop the TUI roster from dropping monitors and mislabelling holders

This phase alone makes the screenshot case read `5/8`. It keeps the roster projection
correct, which still matters after `tui-report-feed`: the roster becomes the
pre-first-report estimate and the drift witness, and monitor rows need a real PID for
kill/clock/liveness display too.

### F1: monitor PID projection (C1)

The best single hook is `load_workflow_agents_from_snapshot` in
`src/sase/ace/tui/models/_loaders/_workflow_snapshot_loaders.py`, right after
`enrich_agent_from_meta_wire`. At that point `agent.is_monitor` is known and the
`record` carries both `record.agent_meta.pid` and `record.running.pid`.

Today `pid = wf_state.pid` is set near line 118. That is null for monitors because:

- monitor members are created with `stamp_creating_process=False`
  (`src/sase/turns/member.py`);
- the supervisor PID is written only to `agent_meta.json` and the RUNNING claim
  (`src/sase/monitor/start_launch.py`).

The fix:

- For real monitor-member rows whose `pid` is `None`, take `agent_meta.pid`, falling
  back to `running.pid`. This matches the order admission uses in
  `_record_liveness_probe` (`src/sase/axe/run_agent_wait_slots.py`) and in
  `src/sase/agent/_running_listing_common.py`.
- Mark the row `pid_liveness_unverified=True`. Do **not** mark it live. The existing
  probes then decide liveness:
  - `_filter_dead_pids` (`_agent_loader_normalization.py`);
  - `mark_live_artifact_delta_runners` on the delta path;
  - `_capacity_record_is_live` (`_agent_runner_slot_capacity.py`).

  Dead or reused PIDs and pending or terminal turns must stay excluded.

- Verify that the full, index-backed, bounded/viewport, and exact-delta
  (`load_artifact_delta_agents` → `_load_agents_from_artifact_snapshot_sources`) paths
  all reach this hook. Apply the same projection anywhere one does not.
- Re-run the dedup suites. `dedup_running_vs_workflow` (`_dedup.py`) matches
  `ace-monitor` RUNNING claims to workflow rows, and those rows now arrive with a PID.
  There must be no duplicate rows and no lost merged fields.

### F3: holder labels (C5, closes `sase-1ol`)

`_apply_runner_capacity_snapshot` in `src/sase/ace/tui/models/agent_runner_slots.py`
builds `display_agent_by_artifact_dir` and `source_agent_by_artifact_dir` with last-wins
dict comprehensions. Workflow step rows (`setup`, `checkout`, `diff`, …) share their
agent's `artifacts_dir` and overwrite it.

- Skip `is_workflow_child` rows when indexing (or prefer agent rows) in **both** maps.
  The display map also routes QUEUED status and queue positions, so the same bug can
  misattribute those.
- Add a regression test in which step rows share the directory.
- When the phase is done, close the bead:
  `sase bead close sase-1ol --note "<what you verified>"`.

### F6: claim owner key (C3)

- Add `runner_claim_owner_key` to the TUI `Agent` model.
- Populate it from `AgentMetaWire.runner_claim_owner_key` during meta enrichment.
- Forward it from `capacity_record_from_agent` (`_agent_runner_slot_capacity.py`) so the
  roster projection groups claims like admission does.

### Tests

Build real monitor artifact directories: `agent_meta.json` with a PID,
`workflow_state.json` with `pid: null`, and a RUNNING claim where relevant. Push them
through the actual loaders, including `load_artifact_delta_agents`. Do not inject `pid`
directly; that is why the existing `tests/test_capacity_snapshot_parity.py` fixture
could not catch C1. Cover:

- **Monitor PID, weight 1:** a live meta PID with a null workflow PID counts 1.0 on
  every loader path.
- **Monitor PID, explicit `-w 0`:** the row is present and holds a zero-weight claim
  instead of vanishing.
- **Exclusions:** a dead supervisor, a reused PID (process-identity mismatch), a pending
  gate turn, and a terminal monitor are each excluded, with no ghost claim.
- **Screenshot shape:** 3 workers, 2 verify monitors, and 1 zero-weight monitor, loaded
  through the delta path and spliced into a full roster, give 5.0.
- **Claim grouping:** two parallel branches with different owner keys in one displayed
  session give two charges, and overlapping serial records give one charge.
- **Holder labels:** step rows sharing an artifact dir leave the holder labelled with
  the agent's name.

## Phase core-accounting: per-record capacity accounting in the Rust snapshot

Work in the linked sase-core repo: run `sase repo open sase-core` and read its
`AGENTS.md`. The module is `crates/sase_core/src/runner_capacity/`:

- `wire.rs`
- `snapshot.rs`
- `claims.rs`
- `records.rs`
- `waiters.rs`
- tests under `runner_capacity/tests/` with the `support.rs` builders

The Python binding is `py_runner_capacity_snapshot` in
`crates/sase_core_py/src/provider_policy/mod.rs`.

### Output: `accounting` and `accounting_valid` on `RunnerCapacitySnapshotWire`

Add `accounting: Vec<RunnerCapacityAccountingWire>` with **exactly one entry per request
record**. The candidate is excluded and is still reported only through
`candidate_decision`. Each entry has these fields:

- `artifact_dir`, `project_name`, `agent_name` (optional).
- `state`, one of:
  - `holding`: this record is its claim's charged member;
  - `riding_claim`: occupying, but its claim is charged through another member;
  - `waiting`: in `waiters`;
  - `not_holding`;
  - `unknown`.
- `charged_weight: f64`: what this record adds to `occupied_capacity`. It is 0 for
  riders and for every non-holding state.
- `effective_weight: Option<f64>`: `None` when invalid.
- `weight_source`, one of `default`, `inherited`, `explicit`, `explicit_zero`, or
  `invalid`.
- `claim_owner_key` and `claim_holder_artifact_dir` (both optional). Riders point at
  their claim's holding member.
- `reason_code` (optional; `None` for a plain positive-weight holder or a rider), one
  of:
  - `not_started`
  - `queued`
  - `pending_question`
  - `pending_gate`
  - `explicit_zero_weight`
  - `invalid_weight`
  - `not_live`
  - `no_pid` (a live real monitor member without a PID, reported as `unknown`)
  - `done`
  - `not_agent_record`

Spell all three enums in snake_case, matching `claim_kind`. Map reasons from the actual
predicates in `records.rs` (`is_user_agent_record`, `is_occupying_record`, the pending
gate check, and `is_real_monitor_member`) and from the waiter selection in `waiters.rs`.
Do not keep a second copy of those predicates. Records in a claim dropped for an invalid
weight are reported as `unknown`/`invalid_weight`.

Add a top-level `accounting_valid: bool` that is `false` whenever the snapshot fails
closed. That covers invalid live claim weight, capacity overflow, and an invalid limit;
the `diagnostics` already exist. Display code reads this flag instead of guessing which
diagnostic codes are fatal.

The charged member of a claim is the maximum-weight member. Break ties by
`owner_artifact_dir`, then lexicographically, so the choice is deterministic.

Invariants, each with a test:

- every request record appears exactly once;
- when `accounting_valid` is true, the compensated sum of `charged_weight` equals
  `occupied_capacity`;
- the claim owner keys of `holding` entries equal the set of `claims[].owner_key`, with
  exactly one holder per claim;
- `waiting` entries equal `waiters`.

### Input: `queue_weight_inherited`

Verify monitors inherit the starter's `queue_weight` (`src/sase/monitor/member.py`
copies it) unless the monitor was started with an override. An override writes
`monitor_queue_weight_overridden: true` into the monitor's `agent_meta.json`.

- Add `queue_weight_inherited: bool` (default false) to `RunnerCapacityRecordWire`. Rust
  reports `weight_source = inherited` for it.
- If the scan's `AgentMetaWire` does not yet carry `monitor_queue_weight_overridden`,
  add it there in Rust (agent_scan) and in the Python mirror
  `src/sase/core/agent_scan_wire_markers.py`.
- Populate the input in sase's `capacity_record_from_scan`
  (`src/sase/core/runner_slots/_admission_capacity_records.py`): true for real monitor
  members without the override marker.
- The TUI roster builder may leave it false.

### Versioning and the sase side

- Bump `RUNNER_CAPACITY_POLICY_SCHEMA_VERSION` (7 → 8) and the hard-coded `7`s in the
  binding round-trip tests
  (`crates/sase_core_py/src/editor_completion/tests/surfaces.rs`).
- Decide breaking-change marking (`feat!`) per sase-core `AGENTS.md`, given that every
  wire struct is `deny_unknown_fields`.
- In sase:
  - Add `queue_weight_inherited` to `_supports_legacy_capacity_fallback` and to the
    fields `_drop_queue_capacity_alias` strips
    (`src/sase/core/runner_slots/_admission_snapshot.py`), so older cores keep working.
  - Every Python reader must tolerate a missing `accounting` key. That means "accounting
    unavailable", never "holds nothing".
- Admission behavior must not change. Existing admission, waiter, and candidate tests
  must pass unchanged.
- Add a Python test through the real binding for the screenshot shape: six records give
  5.0 and six dispositions, with the zero-weight monitor as `holding` /
  `explicit_zero_weight` / `explicit_zero`.
- Move `sase-core-revision.txt` past the core commit. When one turn changes both repos,
  the host writes the pushed SHA itself; see `docs/contribute/rust-backend.md`, "The CI
  source revision pin".

## Phase machine-report: shared authoritative machine capacity report

Add a read-only adapter under `src/sase/core/runner_slots/` (export it from the
package). Its typed `MachineCapacityReport` is the only input the CLI, the list command,
and the TUI use. It carries:

- **Identity and timing:**
  - `machine`: the local machine label. Reuse or move the non-TUI core of
    `local_machine_label()` from `src/sase/ace/tui/modals/machines_pane_rendering.py` so
    core code does not import the TUI.
  - `computed_at`, `scanned_at`, and the scan `token`.
  - `scan_current`: whether the scan matches the live `runner_slots.token`.
- **Limit and totals:**
  - `effective_limit` plus limit provenance: the configured limit, and the active
    override's limit, source, `created_at`, and `expires_at`. Reuse
    `EffectiveRunnerLimitSnapshot` / `get_active_runner_limit_override` in
    `src/sase/config/runner_limit_override.py`.
  - `occupied_capacity`, `occupied_lanes`, and `over_limit` (`occupied > limit` beyond
    rounding).
- **Snapshot contents:** `claims`, `waiters`, `accounting`, `diagnostics`,
  `accounting_valid`, and `accounting_available` (false on an older core).
- **Raw and inputs:**
  - `raw_snapshot`, for TUI row application and export.
  - Per-record display inputs that the snapshot does not echo: turn kind/role, monitor
    id, `slot_requested_at`, authored `queue_capacity`, and `run_started_at`. These are
    attributes for presentation, not occupancy decisions. The authored budget lets
    displays attribute over-limit load to `%queue(capacity=N)` launches (C8).

### Computation

1. Get records from the shared scan cache through `load_or_refresh_runner_slot_scan`,
   with admission's own collector and scan options. Move `_collect_runner_slot_records`
   and `record_liveness_probe` out of `src/sase/axe/run_agent_wait_slots.py` into the
   core package if core would otherwise import axe. Admission must keep calling the same
   code.
2. Probe liveness freshly on every computation. Process deaths do not bump the token.
3. Take a **read-only** active-holds snapshot (no pruning, no notifications).
4. Use the config deference settings and the current time.
5. Call `runner_capacity_snapshot` with no candidate. That is the same Rust call and the
   same inputs as admission, minus the candidate and the lock.

### Bounded rescan policy

This policy keeps the TUI from causing steady-state scans or SSD writes:

- While the cache's token matches the live token, reuse the cached records regardless of
  age.
- When the token differs, rescan. Display-initiated rescans are rate-limited to one per
  30 s and only one may be in flight. Between them, serve the old records with
  `scan_current=False` and an honest age.
- A caller may force a rescan (the CLI's `-r`).
- If needed, add a cache read that returns the stale entry and its token instead of
  `None`.

> [!decision] publish_snapshot_file = no Do not write any new file. The shared scan
> cache is the published input, and every reader computing the same Rust snapshot over
> it with a live probe agrees with admission.

> [!decision] publish_snapshot_file = yes After each computation, atomically write the
> report JSON to `~/.sase/runner_slots.snapshot.json`, keyed by token, only when its
> content (ignoring timestamps) changed. Readers reuse a published report whose token
> matches and whose age is under 10 s instead of recomputing. Admission does not
> publish, because its snapshot excludes the candidate.

Tests (fake clock, fake liveness, fixture scan cache):

- the screenshot shape gives 5.0 / 8, six accounting entries, and override provenance;
- a token change inside the rate-limit window serves stale records with
  `scan_current=False`;
- a dead PID drops out without a rescan;
- an older-core snapshot without `accounting` yields `accounting_available=False`;
- an invalid weight yields `accounting_valid=False` with the known subtotal;
- the report never takes the admission lock or bumps the token.

Extend `tests/test_capacity_snapshot_parity.py` so the report and admission's snapshot
agree.

## Phase load-cli: the `sase agent load` command and shared capacity view model

Read the `cli_rules` and `docs_writing` memory notes first.

### Shared view model

Write a pure function (no Textual imports) from `MachineCapacityReport`, plus an
optional `running_nodes` count supplied by the TUI, to typed sections. The inspector
reuses it. The sections:

- **Summary lines:**
  - `5 of 8 units in use · 3 headroom`;
  - limit provenance
    (`limit 8: ACE override since 2026-09-13, no expiry · config default 10`);
  - snapshot time and age, and the diagnostics count;
  - `6 running nodes · 5 holding capacity · 1 holding 0` when `running_nodes` is known;
  - the over-limit explanation when `over_limit`.
- **Holding capacity:** one row per claim, with name, kind, charged amount, weight
  source, since, and authored budget if any. Kinds are worker, verify monitor, wait
  monitor (`-w 0`), gate, and parallel member, derived from record attributes. Shared
  claims appear once, with `shared with <holder>` on riders. A total line follows.
- **Live, holding nothing:** grouped by `reason_code`, with counts. Every group is
  complete or explicitly says it is truncated. An unresolvable record shows a fallback
  identity (the artifact dir basename) and is never silently dropped.
- **Queued for capacity:** each waiter's real blocker, from the snapshot's blockers
  (`requests 2; headroom 1; blocked by capacity`, a hold, an error). Never promise a
  start time.
- **Diagnostics:** with `admission blocked` when `accounting_valid` is false.
- **Accounting unavailable:** an explicit line when the core is too old.

### Command

Add `sase agent load` in `src/sase/agents/`, registered alphabetically in the
`sase agent` group:

- The default output is colored Rich, laid out like the research's inspector mockup.
- `-j/--json` prints the exact report (raw snapshot plus provenance and metadata).
- `-r/--rescan` forces a fresh scan.
- Every option is sorted and has a short alias, and the help text is excellent.

Test the rendering and the JSON shape for the screenshot shape and for the invalid,
stale, and over-limit cases.

### Docs

- Regenerate the CLI reference with `just fix`.
- Add a "What holds capacity" Reference section to `docs/reference/runner-slots.md`.
  Give one entry per row kind from the research table, saying what it adds and why:
  - a default worker;
  - a verify monitor;
  - a `-w 0` wait or epic-launch monitor;
  - a serial rider;
  - `WAITING` not started;
  - `QUEUED`;
  - a question or gate;
  - clan, proc, and step rows.

  Point the section at `sase agent load`.

## Phase list-holder-parity: retire the Python holder predicate behind `sase agent list`

C6: `runner_slot_holder_dirs` in `src/sase/agent/_running_listing_common.py` is a third
Python implementation of occupancy. It is used from `_running_listing_running.py`.
`_holds_runner_slot` in `src/sase/integrations/agent_list_entries.py` falls back to
`agent.status == "RUNNING"`, so `TESTING` and `WAITING CUTOVER` rows miss it.

- Source `holds_runner_slot` from Rust accounting (`state == holding`) over the
  listing's own records through the shared snapshot-from-records path.
- Source `runner_slot_holders` and `runner_slots_in_use` from the snapshot's claims and
  `occupied_lanes`.
- Remove the status fallback. Integrations that hand-build `RunningAgentInfo` must pass
  the flag explicitly; check every caller, including
  `src/sase/core/agent_hold_pending.py`.
- Delete `runner_slot_holder_dirs` once it is unused (see the `symvision` memory note).
- Add additive per-entry JSON fields to `sase agent list -j`: `capacity_state`,
  `capacity_charged`, `capacity_reason`.
- Parity test: for the screenshot shape, including `TESTING` and `WAITING CUTOVER`
  statuses, the list's holders equal the report's holding entries.
- If Rust's unweighted `running_agent_slot_count` / `is_runner_slot_occupying_record`
  (`agent_runtime.rs`) is now redundant, record a `PROPOSED FOLLOW-UP:` note on the
  phase bead. Do not change it here.

## Phase tui-report-feed: feed the TUI from the authoritative report

Read the `tui` and `tui_perf` memory notes first. This phase is event-loop-sensitive.

### Refresh cadence

- Add the `runner_slots.token` path to `SurfaceTokenRoots`
  (`src/sase/ace/tui/actions/event_refresh/_surface_tokens.py`). The token is stat-only.
  A token change, and an override-file change (already watched), schedules a
  **capacity-only** report refresh, never an Agents reload.
- Add a 10 s interval for the liveness re-probe.
- Timer and token callbacks stay thin and synchronous. They launch the computation with
  `spawn_pump_free_task` plus `asyncio.to_thread`.
- Coalesce with scheduled/running/pending guards and cancel at teardown. Apply the
  result on the UI thread, re-capturing tab and selection state after the await.
- An idle host with an unchanged token must do no scans and no new file writes. Verify
  with `SASE_TUI_TRACE=1` (`refresh.auto_tick`).

### Applying the report

- Refactor `refresh_runner_slot_context`
  (`src/sase/ace/tui/models/agent_runner_slots.py`) so its row application
  (`_apply_runner_capacity_snapshot`) can take the authoritative report's
  `raw_snapshot`. Row QUEUED status, queue positions, blockers, and admission limits
  then come from admission's waiters.
- On each roster apply (`prepare_loaded_agents_apply_boundary` in
  `actions/agents/_loading_compute.py`, and `_run_agents_capacity_refresh_from_roster`
  in `_loading_filter.py`), apply the latest report. On a new report, re-apply it to the
  current rows.
- Patch changed rows selectively (`patch_row`). Never trigger a full list rebuild for a
  capacity change.
- Feed `AgentLoadIndicator.update_load` (via `_display_detail_info.py`) from the report.
  The holders come from accounting with agent names. This phase keeps the existing chip
  chrome; `cap-chip` restyles it.
- Expose to the chip whether the value is authoritative or an estimate, and the report's
  age, validity, over-limit flag, and limit provenance.

### Roster projection's remaining roles

Before the first report lands, the roster projection is the **estimate**, flagged as
such. After that it is only a **drift witness**. First paint never waits on a scan.

### Telemetry (F8)

Append to `tui_agent_loads.jsonl` through the existing background writer
(`src/sase/logs/tui_telemetry.py`):

- `capacity_gauge` (token, occupied, limit, claim owner keys, validity, source) when the
  displayed value changes;
- `capacity_drift` (both totals and the differing artifact dirs) when the roster
  estimate and the report disagree for two or more consecutive comparisons.

Both are rate-limited and never written per tick.

### Tests

Use an injectable fake report source; visual and unit tests must not scan the real
filesystem. Cover:

- **Scope:** search, fold, project, tribe, and remote machine tab changes leave the chip
  unchanged.
- **Roster completeness:** removing both verify-monitor rows from the roster still
  shows 5.
- **Lifecycle:** first-paint estimate → authoritative.
- **Coalescing:** a token change during an in-flight compute.
- **Teardown:** cancellation.
- **Telemetry:** emission.

> [!decision] capacity_decision_record When accepted, this phase adds a `decisions` web
> strand through `/sase_memory_write` (Claim / Why / Cost / Reopens when). It records
> that capacity displays (chip, marks, inspector, CLI, list) read the Rust snapshot
> computed over admission's scan cache, never a display roster, and that Rust owns every
> hold/no-hold reason. Then run `sase memory init`. When declined, record nothing.

## Phase cap-chip: machine-labelled `cap` chip with provenance, staleness, fault, and overage

Restyle `src/sase/ace/tui/widgets/agent_load_indicator.py` (F5, F7, C8):

```text
full:      ⌂athena cap 5/8* · model: …      * = limit from an override (dim)
compact:   cap 5/8* · …                     never a bare "5/8"
estimate:  ⌂athena cap ~5/8                 roster estimate before the first report
stale:     ⌂athena cap 5/8 ⟳41s             report older than ~2.5 poll cycles (25 s)
fault:     ⌂athena cap ?/8 !                accounting invalid; admission blocked
over:      ⌂athena cap 8.25/8 ▲             over the default limit (per-launch budget)
```

### Chip text and styling

- Keep the free-capacity color palette. The chip always describes the local machine's
  full scope, including while a remote machine tab is showing.
- The override marker `*` turns to the warning style when the override has no expiry and
  is older than 14 days.
- Keep `full_cells`, `compact_cells`, and host-fit density working.

### Tooltip

The tooltip shows, in order:

- the summary and headroom;
- `6 running nodes · 5 hold capacity (5.0) · 1 holds 0`, with running nodes from the
  existing left-strip counts;
- limit provenance;
- snapshot time and age;
- for a fault: the known subtotal, the diagnostics, and `admission blocked`;
- for overage: an explanation naming per-launch `%queue(capacity=N)` budgets, plus the
  holders admitted under one;
- the holder list, complete or explicitly truncated;
- `,l opens the capacity inspector`;
- on a remote tab: a note that remote capacity is not shown.

### Help and docs

- Add a `?` help line in the Agents help sections (`modals/help_modal/`): "cap is this
  machine's reserved runner capacity against `max_running_agents`. _running_ counts
  Running-bucket nodes. Serial sessions share one claim; `-w 0` waits hold 0."
- Docs (read the `docs_writing` memory note first):
  - Rewrite the header paragraph in `docs/reference/runner-slots.md` near line 19. It
    currently claims the header shows "the same global capacity state" as `load: 8/10`
    next to `8 [8 running · 1 queued]`. Use a split example (`6 running` / `cap 5/8`)
    and list what holds 0.
  - Fix the matching text in `docs/reference/tui/agents-tab-grouping.md` (around lines
    254–271).

### Goldens

The label change touches most Agents-tab PNG goldens.

- Pin the machine label and the report in visual fixtures so goldens never depend on the
  host.
- Run `just fix-tui-screenshots` through `/sase_monitor` (`TESTING`/`TESTED`), then
  inspect the run report and the golden diff before finalizing.

Tests cover each state above, the override set/cleared/expired flip within one refresh,
and the fault state never rendering free capacity.

## Phase occupancy-marks: occupancy marks on Running rows and a Capacity detail line

### Attaching accounting to rows

- Attach each accounting entry to its agent row by `artifact_dir`: agent rows, never
  workflow-step rows. Use new `Agent` fields (state, charged weight, weight source,
  reason, claim owner, claim holder).
- Add them to `agent_render_key` (`widgets/_agent_list_render_cache.py`) so cached rows
  repaint.
- Repaint changed rows selectively.

### The mark

Draw one dim mark on every Running-bucket agent node and clan container, including
collapsed clans:

- **holding N:** always show the number, including the default 1.
- **holding 0:** an explicit zero claim.
- **in Running but holding nothing:** a rider, an unstarted row, or a pending human.
- **`?`:** no accounting entry, invalid weight, missing PID evidence, or accounting
  unavailable.

A clan container's mark sums `charged_weight` over its members' **distinct**
`claim_owner_key`s. With all clans visible, the marks sum to the chip's total (test this
for the screenshot shape). Expected rows:

```text
bob-cli (RUNNING) ×6 [R2 W1 D3] <mark 2>    bob-cli-ct
sase (RUNNING) ×6 [R1 W4 D1] <mark 1>       sase-1oh.5.1
sase (RUNNING) ×4 [R1 W2 D1] <mark 1>       sase-1oh.4.1
sase (TESTING) ×5 [R1 W2 D2] <mark 1>       sase-1oe
sase (WAITING CUTOVER) ×11 [R1 W1 D9] <mark 0>   sase-1n6
```

Placement:

- Clan containers: put the mark after the R/W/D chip in `format_agent_option`
  (`widgets/_agent_list_render_agent.py`).
- Agent nodes: put it beside the queue badges (`append_agent_queue_badges` in
  `widgets/_agent_list_render_agent_status.py`).
- Keep the authored `wN`/`cN`/`pN` badges. The mark shows what is **charged now**, which
  is a different thing.

> [!decision] mark_style = glyph Pick one glyph for the mark family. It must render in
> the TUI font and the visual-snapshot renderer stack (Fira Code, DejaVu, Noto Emoji).
> It must not collide with the wave/bead dots (`●`, `○`, `◐`), the `▪` separators, or
> any existing row token; grep the row renderers. It must not be emoji-presentation (no
> width ambiguity). Hollow and unknown variants follow from the same family plus `?`.
> Document the chosen glyphs.

> [!decision] mark_style = text Use `cap1`, `cap0.25`, `cap0`, `cap-` (holding nothing),
> and `cap?` (unknown).

### Detail pane

Add a `Capacity:` line to agent and clan context cards
(`widgets/prompt_panel/_agent_display_header_metadata_identity.py`). It is the one
capacity field that is **not** suppressed for monitors and clans. Examples:

- `Capacity: 1 (verify monitor, session sase-1oe.3)`
- `Capacity: 0 (wait monitor, explicit w0)`
- `Capacity: none (not started — waiting on deps)`
- `Capacity: shared with <holder>`

### Docs and goldens

- Document the marks and the detail line in the Agents-tab Reference.
- Update goldens through `/sase_monitor` as in `cap-chip`.

## Phase capacity-inspector: keyboard-first capacity inspector

Build a modal that renders the `load-cli` view model from the TUI's current report, with
`running_nodes` taken from the display counts.

### Rows and lists

- Every agent row is selectable and jumps to the agent. Reuse `navigate_to_agent_tab`,
  as `action_show_runners` does in `actions/axe_service.py`.
- Lists are complete or explicitly truncated; a truncated group can expand.
- Queued rows show real blockers.
- The modal updates live when a new report arrives.

### Copy action

Add a copy action that puts the exact `sase agent load -j` JSON on the clipboard, so the
next incident is not reconstructed from a screenshot.

### Ways to open it

- **Chip click:** add a click handler on `AgentLoadIndicator`.
- **Leader `,l`:** add `capacity_inspector: "l"` under `leader_mode.keys` in **both**
  `src/sase/default_config.yml` and `src/sase/ace/tui/keymaps/mode_keymaps.py`. Wire it
  in `actions/agent_workflow/_leader_mode.py`. Add a label to `_LEADER_LABELS`
  (`commands/_mode_commands.py`) so it reaches the command palette. List it in the
  leader-mode listings (`widgets/_keybinding_modes.py`, help modal sections).
- **Command palette:** reached through the `_LEADER_LABELS` entry above.

Follow the `ace` memory note's footer and help-modal rules.

### Docs and tests

- Add docs: a Reference entry for the inspector, with keybinding docs regenerated by
  `just fix`.
- Tests:
  - each way of opening the modal;
  - jumping;
  - the truncation labels;
  - the copy payload equals the CLI JSON;
  - an invalid or unavailable accounting state.
- Add a modal golden through `/sase_monitor`.

## Regression matrix ownership

- **Monitor PID cases** (weight 1 and explicit 0, every loader path): `roster-fixes` for
  the loaders; `tui-report-feed` for the chip.
- **Screenshot shape** (6 running, 5.0, six dispositions): `roster-fixes`,
  `core-accounting`, `machine-report`, `occupancy-marks` (marks sum), and
  `list-holder-parity`.
- **Exclusions** (dead supervisor, reused PID, pending gate, terminal monitor):
  `roster-fixes` for the roster; `core-accounting` for the reason codes.
- **Claim grouping** (parallel owners give two charges; serial overlap gives one charge,
  explained): `roster-fixes` and `core-accounting`.
- **Invalid weight** (a visible fault, never "free"): `core-accounting`,
  `machine-report`, and `cap-chip`.
- **Scope invariance:** `tui-report-feed`.
- **Limit changes** (the marker flips within one refresh): `machine-report` and
  `cap-chip`.

## Verification

Every phase:

- runs `just fix` and then `sase tool run check` in the sase repo;
- runs `sase tool run check` inside the linked sase-core checkout when it changes Rust.

Phases that change rendered TUI output also run `just fix-tui-screenshots` through
`/sase_monitor`. Read the `lint_and_test` memory note first.

After `tui-report-feed` lands, a live TUI on a busy host must show a chip equal to
`sase agent load -j`'s `occupied_capacity` across refreshes. Note the comparison on the
phase bead.
