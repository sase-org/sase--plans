---
tier: epic
title: Fix ToolRun SQLite contention at the store boundary
goal: ToolRun launch, settlement, and queries survive ordinary parallel contention
  with bounded waits and no duplicate writes.
parent_bead: sase-1ou.2
phases:
- id: core-contention
  title: Diagnose and fix the Rust store contention policy
  size: medium
  depends_on: []
  description: 'core-contention: capture the clean-base failure and lock timeline,
    fix the responsible connection or transaction paths in sase-core, and test bounded
    contention and safe replay through the bindings.'
- id: caller-integration
  title: Adopt the core policy and prove the affected ToolRun flows
  size: medium
  depends_on:
  - core-contention
  description: 'caller-integration: consume the landed core policy in Python, preserve
    explicit latency deadlines, ratchet the core pin, repeat the same isolated and
    loaded stress runs, and retire only proven fixed baseline nodes.'
proposed_by: bbugyi200.athena.sase-1ou.2
proposed_with: codex/gpt-6-astra
create_time: 2026-10-27 10:41:24
status: done
bead_id: sase-1ou.2.1
---

- **PROMPT:** [prompts/202610/toolrun_sqlite_contention.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/toolrun_sqlite_contention.md)
- **PARENT:** [202610/backlog_epic_2_top_five_task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202610/backlog_epic_2_top_five_task_beads.md)
- **BEAD:** [sase-1ou.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ou/sase-1ou.2.1.md)

# Fix ToolRun SQLite contention at the store boundary

## Scope and ownership

This is a child epic for `sase-1ou.2`, the ToolRun contention phase of
`plan:202610/backlog_epic_2_top_five_task_beads.md`. Read that artifact with
`sase artifact read` and read `sase-1ou.2` and `sase-18t` with `sase bead read` before
implementing. The final parent decision `dormant_flakes = close` stands; this child adds
no reviewer decisions and authorizes no memory edits.

The assigned phase was in progress with no prior-work notes when inspected. `sase-18t`
remained ready with nine corroborations. Its most recent recorded failure was the
settlement-notification node in Master Gate on 2026-10-19. There is no verified
completed fix to reuse. This planning turn inspected source and history; it did not
execute a reproduction or change implementation files.

Two medium phases separate Rust diagnosis and the core API contract from Python
integration and application-level flake proof. The second phase depends on the first so
it can pin an actual host-committed core revision. Each phase must be implementable by
one coder, using monitor continuations for long commands.

Child phase workers close only their assigned child phase after checking
`sase bead epic-symbols <assigned-id>`. They do not close this child epic, `sase-1ou.2`,
`sase-1ou`, or any task bead. The child land agent owns completion of this child epic
and then `sase-1ou.2`. Leave `sase-18t` closure to the outer epic's land agent: the
current user explicitly limited closure to the assigned phase, overriding the parent
design's instruction to close that task here. Provide that land agent complete closure
evidence.

Do not create task beads. Record discovered work as
`PROPOSED FOLLOW-UP: <summary — evidence>` on the assigned phase; consolidate relevant
proposals on `sase-1ou.2` for the outer land agent. Do not hand-edit bead status. Do not
modify `sase/memory/**`, versions, or changelogs.

## Findings and design constraints

The inspected sase base was `e608de2db6`, containing phase `sase-1ou.1`'s flake-baseline
repair. The opened sase-core checkout was `89576266`; sase's source pin was
`800caef4c70039ccf299a0ecca379142f80234c2`. Recheck all three before work because other
epics can advance them.

Open sase-core with `/sase_repo` and use only its printed path. Read its `AGENTS.md`.
Paths below are relative to their respective repository roots.

In sase-core:

- `crates/sase_core/src/tool_run/store/connection.rs` owns connection setup.
  `open_write_store` sets WAL, executes `SCHEMA_SQL`, checks additive columns, and
  unconditionally upserts `meta.schema_version` on every write connection. This adds
  schema work and a separate metadata write before the operation's transaction.
  Determine which statements actually take locks; do not assume every
  `CREATE ... IF NOT EXISTS` causes a write.
- `open_read_store` uses `SQLITE_OPEN_READ_ONLY`, sets a busy timeout and checks schema
  version. It neither migrates nor starts a write transaction. `with_read_store` has no
  outer retry. `enable_wal_mode` alone has an explicit busy retry loop, with its own
  timer and timeout reset afterwards.
- The main lifecycle, handoff, reconciliation, triage and receipt writers already use
  `TransactionBehavior::Immediate`. Adding it everywhere is not the missing fix. Trace
  connection open, schema setup, BEGIN, COMMIT and close separately, including WAL
  recovery/checkpoint/last-connection cleanup.
- `store/retention.rs` calls its `inspect` closure inside an immediate write
  transaction. That closure calls `stage_output_files` and
  `select_aggregate_log_candidates`, which inspect the filesystem. Quarantine scanning
  is already outside the transaction. This is a concrete long-lock candidate, but is not
  proof that retention caused the tiny ad-hoc handoff failure. That failure was reported
  even without settle-time triage.
- `tool_run/wire.rs` defines a 250 ms default and a 30 s maximum. Duration zero already
  means the core default. `crates/sase_core_py/src/telemetry/mod.rs` repeats
  `busy_timeout_ms=250` for ToolRun bindings and also contains unrelated telemetry
  bindings that must retain their existing behavior.
- Existing tests in `tool_run/store/tests.rs` include concurrent writers and
  `busy_failure_is_bounded`; the former only requires one competing operation to
  succeed, and the latter supplies an explicit 25 ms budget. Add coverage that requires
  all eligible operations to succeed under releasable contention, while preserving the
  bounded-failure guarantee.

In sase:

- `src/sase/core/tool_run.py` repeats the 250 ms default in its thin adapters. The same
  number appears in `src/sase/tool/receipts.py`,
  `src/sase/ace/tui/tool_runs/{snapshot,summaries}.py`, and
  `src/sase/history/chat_catalog_provenance/artifacts.py`.
- Other values have different purposes: `src/sase/tool/triage_stage.py` has a 2 s
  contention workaround; `ownership.py` uses 1 s; `executor_triage.py` passes the
  remaining operation deadline. Tests have an isolated 5 s override in `test_detach.py`.
  Audit each call; do not replace every explicit timeout indiscriminately.
- Read `docs/contribute/rust-backend.md`, especially SQLite ownership and the source
  pin. Never open one ToolRun database from both Python sqlite3 and the bundled Rust
  SQLite in the same process. Use Rust-only fixtures or separate processes for lock
  experiments and inspection.

## Shared reproduction and evidence protocol

Freeze the harness, selected nodes, repetition count, load source, and process cleanup
before production changes. Record sase/core SHAs, loaded extension path/version and
build provenance, Python version, workers requested and granted, host load, elapsed
times, and failures by exact signature. An unrelated timeout is not automatically
evidence for the SQLite bug.

Use the original node plus all nine current baseline entries under `# sase-18t`:

```text
tests/tool/test_lifecycle_controls.py::test_stop_live_inline_run_settles_stop_requested
tests/tool/test_demand_runs.py::test_foreground_run_records_context_usage_and_grant
tests/tool/test_detach.py::test_watchdog_reports_ended_join_monitor
tests/tool/test_detach.py::test_watchdog_stops_unjoined_run_when_starter_dies
tests/tool/test_executor.py::test_nested_run_has_parent_and_no_duplicate_logs
tests/tool/test_inline_escalation.py::test_fast_success_matches_inline
tests/tool/test_known_continuation.py::test_agent_run_stops_on_a_new_item
tests/tool/test_settlement_retention.py::test_handoff_end_to_end_publishes_one_notification
tests/tool/test_wrapper_fidelity.py::test_inline_caller_group_sigkill_leaves_no_survivors
tests/tool/test_wrapper_fidelity.py::test_owner_child_shares_wrapper_group_and_group_kill_leaves_no_survivors
```

Run the selection through `.venv/bin/python -m pytest`: initially ten serial iterations
and twenty iterations requesting `-n 8`, with a fixed bounded concurrent CPU/I/O load
source during the loaded runs. Keep load data in managed temporary storage and always
reap the load processes. Use the same commands and counts on the before and after trees;
expand both sets comparably if needed. No full-suite run is needed. An isolated pytest
home separates tests from the real machine ledger, while each tested launcher and worker
still share their test ledger. Preserve that isolation.

Use `/sase_monitor` for long loops and builds, with a continuation that reads the
results and resumes the assigned phase. Never abandon or duplicate an in-flight command.
Keep logs and the reusable harness as audited artifacts, record their refs in phase
notes, and use `sase artifact read` to consume them. A new permanent `tools/` harness is
optional, not required; read `tools.md` before adding one.

If the natural flake does not recur, report zero failures honestly. Capture a controlled
contention schedule reproducing the observed failing store path on the unchanged
implementation, then prove that the same schedule passes after the fix. Synthetic proof
supplements the natural before/after rates; it must not be presented as a naturally
observed CI failure. A static candidate or a green stress run alone is insufficient
root-cause evidence.

## Phase core-contention

1. Establish the clean-base measurements before changing production behavior. Use
   temporary Rust instrumentation or SQLite tracing to identify the actual lock holder,
   operation, extended result code, journal mode and hold duration when a
   launcher/show/settle call fails. Sample uninstrumented runs as well. Explain the
   read-only `show` failure specifically: an ordinary WAL writer lock alone is not an
   adequate explanation. Probe concurrent first-open, schema initialization, and
   last-close/reopen overlap. Record enough detail that phase caller-integration can
   repeat the exact schedule.
2. Remove the measured source of unnecessary contention. Keep an already-current store's
   write-open path free of redundant schema writes; ensure first-create and additive
   migration remain atomic and safe for concurrent openers. A process-local initialized
   flag alone cannot establish readiness after file replacement. Preserve newer-schema
   refusal before destructive changes, old-store read compatibility, permissions, and
   corruption handling. If retention filesystem work is implicated, move it outside the
   writer critical section while revalidating eligible rows inside the transaction.
   Preserve unsettled-run protection and accurate receipt/log selection; do not split
   atomic settlement just to shorten a transaction.
3. Centralize contention handling at the Rust store boundary. Use one bounded
   elapsed-time budget for connection/setup and any retries belonging to one store
   operation. Recompute remaining time before blocking; do not multiply the allowance
   across retries or reset it at WAL setup. Retry only classified transient SQLite
   contention, with bounded backoff, and propagate unrelated errors immediately. Keep
   Busy distinct from corruption and never quarantine a healthy locked store. For reads,
   any retry restarts the complete safe read attempt rather than returning an incomplete
   result. For writes, retry only when no effect committed or rollback/replay safety is
   established. Do not blindly replay arbitrary `FnMut` operations after an uncertain
   commit.
4. Choose the default from the measured hold/recovery durations and product latency
   needs, documenting the reason in code and the evidence note. Increasing a timeout
   alone is not this fix. Preserve explicit positive caller budgets and the existing
   maximum. Prefer the existing zero-means-core- default contract over a new settings
   API: ToolRun PyO3 defaults can pass zero to the core, and phase caller-integration
   will do the same in Python. Leave unrelated telemetry bindings untouched. No new wire
   schema is needed for this policy unless the actual implementation changes serialized
   data.
5. Add focused Rust tests using barriers/channels to establish lock order rather than
   hoping two threads overlap. Cover ordinary WAL reader/writer coexistence, the
   reproduced transient failure, concurrent creation/migration, a releasable lock
   succeeding under the production policy, and an unreleased lock exhausting an explicit
   budget. Verify no duplicated runs/events/settlement, no quarantine on busy, and
   unchanged invalid/read-only/newer-schema errors. Keep scheduler tolerance in timing
   assertions; the controlled event ordering is the oracle. Add PyO3 round-trip tests
   for omitted, zero, and explicit timeout arguments and continued GIL release. Extend
   existing compatibility tests as needed.
6. Run targeted tests with the repo's `just test -p ...` recipes, including the relevant
   `sase_core_py` tests, then `sase tool run check` in sase-core. Never run bare cargo
   or a global install. Remove temporary instrumentation. Use host-owned finalization
   for the core change. Record the committed SHA, exact policy contract, trace evidence,
   clean-base rates and reusable harness artifact refs for the next phase. Do not retire
   Python baseline entries yet.

## Phase caller-integration

1. Read phase core-contention's evidence and reopen sase-core through `/sase_repo`.
   Verify the core fix landed before redoing it. Use the documented local venv build
   flow to test the fixed extension; record which build each parent and subprocess
   actually imports. Do not run global installs or modify release version pins. If
   setup/build is long, use `/sase_monitor`.
2. Update ToolRun adapters and default-only callers to delegate to the Rust policy,
   preferably by passing the existing zero sentinel. Preserve the public explicit
   `busy_timeout_ms` argument. Remove historical test-only contention overrides once the
   production default covers them. Retain real deadlines, especially
   `executor_triage.py`'s remaining budget. Review the 2 s stage and 1 s ownership
   overrides individually against phase one's evidence.
3. Read `ace.md`, `tui.md` and `tui_perf.md` before changing TUI callers. Keep explicit
   bounded latency at interactive/provenance call sites if necessary; centralizing
   policy does not authorize a multi-second UI stall. Verify any affected snapshot work
   stays off the event loop and message pump. No layout, keybinding, or screenshot
   changes are expected. Add focused adapter contract tests in
   `tests/core/test_tool_run_store.py` and affected caller tests for default delegation
   and explicit-budget preservation, not copies of Rust logic.
4. Refresh the base through the normal host workflow, then run
   `just ratchet-core-revision` for the already-landed core change. Exit 2 means an
   applied/pending bump, not an error. Verify the selected SHA contains the fix and does
   not regress a concurrently advanced pin. If this phase also needs core edits, declare
   both repos so the host commits core first and writes its SHA into sase's pin; never
   invent a future SHA or manually commit.
5. Repeat the frozen serial and loaded stress runs against the integrated
   implementation. Run the relevant complete test files and core adapter tests. Confirm
   the controlled regression schedule now passes, the target SQLite failures disappear,
   and the failing-lock latency still respects its bound. Record denominators and
   outcomes for every selected node, not only a combined pass count. Trace any residual
   target contention and fix it before claiming this objective complete; do not relabel
   it as unrelated.
6. For each baseline node actually shown to share the fixed mechanism, replace its entry
   with `# fixed-at: <UTC instant> <node id>` and a preceding comment naming `sase-18t`,
   `sase-1ou.2` and the actual fixing core commit. Use the verified integration/fix
   instant, not a future cutoff. Leave any differently failing node tracked with
   explicit evidence and a proposed follow-up. Never retire all nine solely because they
   share a file prefix or baseline heading.
7. Run `.venv/bin/python tools/selection_health --fail-on-new-flake` and verify the
   retired nodes no longer appear as exceedances. Run `sase tool run check` in every
   repo changed after reading `lint_and_test.md`; do not run `just check-full`. Preserve
   exact results even if the gate has unrelated reds. A failure reproduced identically
   on a clean base is a proposed follow-up, with existing owner bead if known, and does
   not prevent phase closure.
8. Supply the child land agent an evidence table: base/fixed SHAs and extension builds,
   commands, per-node before/after rates, controlled regression outcome, check results,
   pin ancestry, retired nodes and remaining unrelated failures. Investigate `sase-19t`
   only enough to determine whether the same fix covers it; report a proposed follow-up
   for the outer land agent instead of closing it.

## Landing and completion

The child land agent verifies that both phases landed, the core pin includes the fix,
and the final integration evidence supports the original launcher/settle/ show failure.
Recheck any affected tests after conflict resolution and run the required guarded checks
for changes made during landing. Known parent-plan reds are leads, not automatic
waivers: use current clean-base evidence.

Consolidate the results on `sase-1ou.2`, including evidence suitable for the outer land
agent to close `sase-18t` and consider `sase-19t`. Check epic-symbol entries for this
child epic and `sase-1ou.2`, resolving them or re-keying real remaining work to an
appropriate still-open bead. Close this child epic after its children, then close only
the assigned outer phase with
`sase bead close sase-1ou.2 --note "<verified behavior, rates, checks, pin and evidence refs>"`.
Do not close `sase-1ou` or any other ancestor plan. Finish through `/sase_final`;
commits, branches, pushes and pin coordination remain host-owned.
