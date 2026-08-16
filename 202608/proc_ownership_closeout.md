---
tier: tale
title: Finish supervisor-owned proc acceptance and close sase-m9.3
goal:
  The three confirmed supervisor-proc closeout regressions are fixed without weakening
  the durable observer boundary, the existing sase-ml task is dispositioned, the
  integrated tree passes focused and exhaustive verification, and epic sase-m9.3.1 plus
  parent phase sase-m9.3 close with exact evidence.
size: medium
proposed_by: bbugyi200.athena.03b
bead: sase-m9.3.1
create_time: 2026-08-16 09:29:03
status: wip
---

- **BEAD:**
  [sase-m9.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.3.1.md)

# Plan: Finish supervisor-owned proc acceptance and close `sase-m9.3`

## Objective

Complete the closeout work that remains after all five implementation phases of
`sase-m9.3.1` closed. Preserve the architectural result already on `master`: durable ACE
operations are supervisor-owned argv submissions observed read-only by ACE, while
genuinely session-local work remains a normal Textual worker and never becomes a durable
proc row. Fix the three regressions that currently make the epic's own notes and
required landing verification unresolved, then close `sase-m9.3.1` and its parent phase
`sase-m9.3` only after the combined tree passes acceptance.

## Verified starting state

Recheck all of this against current `origin/master` before editing; the audit was made
on `30c9ba23b` after a successful `just install`.

- All five child phases `sase-m9.3.1.1` through `.5` are closed. Their implementation
  commits are ancestors of the audited tree: `07e254a42` (durable operation contracts),
  `0835b38d2` (Patch and agent argv migration), `7d7581a21` (remaining producers),
  `8c4840458` (read-only observer), and `ac5d95810` (detached-option retirement).
- The stale `Justfile` exemption for `sase-m9.3.1.2(compare_inventory_to_source)` is
  gone. The inventory helper is now private, so the repeatedly reported post-close
  Symvision blocker has already been resolved.
- The epic has no successful land-agent close or exhaustive verification record. Phase
  `.5` also closed without a verification note. The current epic notes contain three
  still-reproducible findings that must be handled before closure.
- In a live SASE agent environment,
  `tests/test_gate_cli_answer.py::test_set_types_every_declared_input_field` reads the
  current agent's `run.launch` request sidecar and fails with an operation mismatch. The
  exact node passes when the inherited `SASE_PROC_REQUEST_PATH`,
  `SASE_PROC_RESULT_PATH`, `SASE_PROC_ID`, `SASE_PROC_OPERATION`, `SASE_PROC_LOG_PATH`,
  and `SASE_PROC_SESSION_ID` variables are unset. Ready task bead `sase-ml` already
  tracks this defect with seven independent reports; do not create a duplicate bead.
- `sase monitor list --all --json` fails globally because an old `ace-run` artifact has
  `agent_family_role="monitor"` but no `monitor_id`. `active_monitor_for_lane` and
  reconciliation already tolerate records that `MonitorRecord.from_record` rejects, but
  `list_monitors` converts the same malformed row without a guard. No separate task bead
  tracks this exact defect; it is recorded repeatedly on `sase-m9.3.1` and prevents use
  of the required monitor workflow.
- `_submit_session_worker` stores its running `ObservedProc` only in
  `_session_completion_callbacks`. Observer snapshots replace `_proc_projection`, the
  Procs pane and top-bar indicator read only that projection, and
  `running_background_procs()` uses it for update/restart gating. Consequently a live
  session worker is invisible to all three surfaces. This is recorded on `sase-m9.3.1`;
  no separate task bead covers it.

## Implementation

### 1. Make the test process hermetic against the launching proc

Keep the production operation-sidecar fallback intact: supervisor-launched commands must
continue to resolve request/result paths and operation identity from their `SASE_PROC_*`
environment when explicit CLI paths are absent.

Extend the existing autouse environment-isolation fixture in
`tests/_conftest_environment.py` so every ambient `SASE_PROC_*` variable is removed
before a test runs. Tests that intentionally exercise the supervisor environment may
still set those variables explicitly with `monkeypatch` after fixture setup. Prefer one
named contract set or a prefix-based scrub over duplicating an incomplete hand-written
subset, and retain the fixture's teardown guarantees so a test cannot leak proc context
to the next node.

Add a focused regression that seeds all six live-proc variables before fixture setup or
in a nested pytest process, proves ordinary gate/ops/launch handlers do not consume the
caller's sidecars, and proves an intentional per-test environment override still
exercises `resolve_request_path`, `resolve_result_path`, `resolve_proc_id`, and result
operation identity. Run every file family named by `sase-ml`, not just one exemplar,
under the inherited live agent environment. Close `sase-ml` as done only after those
tests pass without an outer `env -u` workaround, citing the exact verification in its
close note.

### 2. Exclude false monitor-family rows consistently

Define one validity boundary for monitor artifact records. A row whose metadata merely
claims `agent_family_role="monitor"` but lacks the durable `monitor_id` is not a monitor
member and must be ignored consistently by listing, lookup, reconciliation, lane
conflict checks, and `has_any_monitor`; it must not poison global enumeration or invent
a monitor.

Implement that boundary in the shared record-selection/conversion path rather than
adding a CLI-only exception. Keep true malformed-monitor behavior deliberate: skip
historical false positives that cannot satisfy `MonitorRecord.from_record`, while not
hiding I/O or reconciliation failures for valid monitor members. Add store tests with a
legacy false-positive row beside valid running and terminal monitors, covering both
all-project and project-scoped listing plus lane/lookup behavior. Re-run the real
`sase monitor list --all --json` command against the host artifact that currently
reproduces the crash.

### 3. Overlay session-local work into the ACE read model

Do not register session workers with `ProcObserver`, write them to the durable proc
store, or make ACE an owner of durable state. Instead, introduce one UI-side projection
composition path that combines the latest immutable durable `ProcProjection` with the
currently running session-worker `ObservedProc` rows. Every consumer must see that same
effective projection:

- top-bar active count and lifecycle running-task count;
- Procs pane and runners modal rows;
- plugin/update restart gating;
- deduplication and exclusive-scope conflict checks.

Give local rows the current projection's session attribution, add them immediately on
successful worker creation, and remove them on success or error before refreshing the
affected UI surfaces. Merge them again whenever an observer snapshot arrives so a
durable refresh cannot erase active local rows. Keep durable rows authoritative for
stored status, logs, results, restart reconstruction, and settlement; session rows
remain intentionally non-reconstructable after ACE exits.

Extend `tests/ace/tui/test_proc_actions_session_workers.py` and focused pane/update
tests to prove active-count changes, Procs visibility, current-session scoping, restart
blocking, dedup/exclusion across local and durable rows, snapshot preservation, and
removal after both successful and failed completion. Add a negative assertion that the
overlay never calls durable observer registration or writes a proc record.

## Integrated verification

1. Start from a clean, current tree and run `just install` before every repository
   verification lane required by workspace instructions.
2. Run the focused operation-I/O, gate CLI/conformance, ops-command, launch/prompt,
   monitor store/CLI, session-worker, proc observer, Procs-pane, plugin-update, and
   lifecycle suites. Include the producer inventory and
   `tests/test_proc_submission_static_invariants.py` so the fix cannot reintroduce
   callable submissions, semantic `tui`/`detached` writers, or ACE-owned durable rows.
3. Exercise public `proc` and `task` help plus the obsolete detached-token diagnostic,
   `--session none`, explicit-session inclusion of unattributed durable rows, and a
   supervisor-launched command settling after its ACE starter exits. This supplies the
   missing phase `.5` and end-to-end epic evidence rather than relying only on the
   existence of its commit.
4. Because session workers become visible in rendered proc surfaces, run
   `just test-visual`, inspect any PNG actual/expected/diff artifacts, and update
   goldens only for intentional output.
5. Run `just check`. Then run `just check-full` only through `/sase_monitor` with an
   explicit next action that fixes causally related failures and reports unrelated
   failures against their existing beads. A failing or timed-out full lane is a hard
   stop for closure; do not repeat the earlier phase-level carve-outs now that `sase-ml`
   is part of this closeout.

## Closure

After every acceptance step is green, append one note to `sase-m9.3.1` that names the
three remediations, the `sase-ml` resolution, focused counts, visual disposition,
`just check`, monitored `just check-full` id/result, and the end-to-end detached
supervisor evidence. Re-audit all child notes for any later `DISCOVERED ISSUE` or
`PROPOSED FOLLOW-UP` and disposition each under the bead policy.

Close `sase-m9.3.1` normally as done; the user's current request explicitly authorizes
that direct close in place of the missing land agent. Then close parent phase
`sase-m9.3` normally as done with a note linking the child epic's verification. Do not
use `--force`, do not close or modify parent epic `sase-m9`, and stop if either close
reports an unfinished descendant. Confirm both bead statuses after closure so the
waiting `sase-m9.land` agent can continue.
