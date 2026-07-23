---
tier: epic
title: AXE whole-system status command
goal: 'Operators can run `sase axe status` to get one read-only, trustworthy, beautiful
  snapshot of AXE lifecycle intent, orchestrator coherence, lumberjack health, runner
  occupancy, and actionable recovery guidance in either human or stable JSON form.

  '
phases:
- id: axe-status-domain
  title: Portable AXE runtime status contract
  depends_on: []
  size: medium
  description: '"Define the portable AXE runtime status contract" section: add the
    Rust-owned status wire, deterministic classifier, PyO3 binding, and exhaustive
    domain tests.

    '
- id: axe-status-snapshot
  title: Side-effect-free host status snapshot
  depends_on:
  - axe-status-domain
  size: medium
  description: '"Collect one side-effect-free host snapshot" section: collect authoritative
    Python host signals, call the shared classifier, tolerate races, and align deep
    doctor with the snapshot.

    '
- id: axe-status-cli
  title: Polished AXE status CLI and docs
  depends_on:
  - axe-status-snapshot
  size: medium
  description: '"Ship the polished CLI, automation contract, and docs" section: register
    and render the human/JSON command, enforce exit semantics, test terminal layouts,
    and document the feature.

    '
create_time: 2026-07-23 07:31:15
status: done
bead_id: sase-8t
---

# Add an intuitive, reliable, and beautiful `sase axe status`

## Objective

Add a read-only `sase axe status` command that gives operators an immediate, trustworthy answer to three questions:

1. Is AXE doing what the operator last requested?
2. Is the orchestrator and each configured lumberjack actually healthy now?
3. If something is wrong, what concrete command should the operator run next?

The default terminal view should be a polished Rich dashboard, while `-j/--json` should expose the same snapshot as a
stable, versioned machine contract. The command must not start, stop, heal, clean, or otherwise mutate AXE state.

## User Experience and Semantics

### Command surface

- Register `status` alphabetically under `sase axe`, with help text that describes the command as a read-only,
  whole-system health snapshot.
- Add `-j/--json` for a stable JSON object. Keep the short alias because this is a public long option.
- Keep `sase axe lumberjack status` as a compatibility/debugging surface; do not silently redirect or remove it.
- Use these exit codes:
  - `0`: AXE is running and healthy, intentionally in maintenance, intentionally stopped, or has never been started.
  - `1`: AXE is down despite an explicit running request, has incoherent orchestrator lock/PID evidence, has an
    unhealthy configured lumberjack, or has a live unconfigured/orphan lumberjack.
  - `2`: the snapshot could not be collected because configuration or another required input is invalid/unreadable.

### Status model

Keep lifecycle state separate from health so the output does not call an intentional stop an error. The top-level state
is one of:

- `running`: a coherent live orchestrator and healthy configured lumberjacks;
- `maintenance`: the orchestrator is live and a structurally valid maintenance marker is active;
- `stopped`: no live AXE runtime and the last explicit desired state is stopped;
- `not_started`: no live runtime and no valid desired-state marker exists;
- `down`: the desired state explicitly says running but the orchestrator is absent;
- `degraded`: AXE is partly live or internally inconsistent;
- `error`: required status inputs could not be collected.

Classify current health from authoritative evidence, not legacy `status.json` alone:

- Require lifecycle-lock and PID evidence to agree for a healthy orchestrator. A held lock with no resolvable live PID,
  a live published PID without the lock, or conflicting live orchestrator identities is degraded and should explain the
  exact inconsistency.
- Preserve the existing fresh-install distinction: a missing desired-state marker plus no process is `not_started`, not
  an outage, even though `sase axe ensure` historically treats a missing marker as an implicit request to run.
- Treat a well-formed maintenance marker as an intentional paused state and show its reason, owner PID, and age. Status
  collection must not call stale-marker cleanup or mutate the marker.
- For every configured lumberjack, distinguish `running`, `not_reporting`, `stale_process`, `stale_heartbeat`, and
  `error`. A heartbeat is stale after `max(60 seconds, 3 * configured interval)`, matching the existing deep-doctor
  rule.
- Show cumulative error counts as historical information, but do not mark an otherwise live/fresh lumberjack unhealthy
  solely because it encountered an error in the past.
- Inspect state-directory lumberjack names in addition to configured names. Include an unconfigured lumberjack only when
  its recorded PID is still live, label it `orphaned`, and degrade the overall result.

### Human output

Render one compact `AXE Status` summary panel followed by a lumberjack table:

- Start with a strong colored badge and plain-language headline (`RUNNING`, `MAINTENANCE`, `STOPPED`, `NOT STARTED`,
  `DOWN`, `DEGRADED`, or `ERROR`).
- Show desired state with source/time, orchestrator PID and lock coherence, maintenance reason when applicable, current
  hook/agent runner occupancy versus configured limits, and the most recent valid lifecycle-journal event.
- In the sorted lumberjack table show name, state, PID, heartbeat age, cycles, errors, and configured chops. Use concise
  symbols and a restrained palette: green for healthy, cyan for neutral runtime facts, yellow for intentional pauses or
  warnings, red for actionable failures, and dim text for unavailable data.
- Add an `Attention` panel only when there are problems. Render deterministic issue summaries and deduplicated next
  steps such as `sase axe ensure`, `sase axe stop --force`, or `sase doctor --deep`.
- Ensure the layout folds cleanly at narrow widths, produces readable plain text when color is disabled or stdout is
  redirected, and never emits Rich markup/ANSI in JSON mode.

### Stable JSON

Emit a schema-version-1 object containing:

- `schema_version`, `generated_at`, `state`, `health`, `summary`, and `exit_code`;
- the explicit desired-state marker (or `null`);
- raw and derived orchestrator evidence;
- maintenance information (or `null`);
- current/max hook and agent runner counts;
- sorted lumberjack records with configured/live/heartbeat facts and derived state;
- the most recent valid lifecycle event (or `null`);
- ordered issue records with stable codes, severity, summary, and suggested command;
- a nullable collection error.

Human rendering must consume this same object/model so terminal and JSON answers cannot drift.

## Architecture

Follow the Rust-core boundary:

- Python remains responsible for host I/O: loading AXE config, reading marker/status/journal files, probing process
  liveness and lifecycle-lock/PID artifacts without cleanup, calculating timestamp ages against an injected clock, and
  counting live runners.
- A new pure `sase-core` AXE-runtime-status module owns deterministic classification: orchestrator coherence,
  desired/live reconciliation, lumberjack health, overall state/health, stable issue codes, ordering, and exit-code
  selection.
- Expose the classifier through `sase_core_rs`; add a typed, Textual-free Python facade that builds the request wire and
  rehydrates the response.
- Make deep doctor consume the shared collected/classified snapshot for overlapping desired/orchestrator/lumberjack
  checks, while retaining its deeper log-cap, orphan-temp, and historical-error diagnostics. This prevents the new CLI
  and doctor from disagreeing without weakening doctor.
- Keep the existing TUI-oriented `get_axe_status()` compatibility shape for this feature; migrating the live TUI to the
  new contract is a follow-up unless it can be done without widening the phase.

## Non-goals

- Do not make `status` start, stop, restart, ensure, repair, or clean AXE state.
- Do not query or manage the optional systemd watchdog; `sase axe ensure install/uninstall` remains its management
  surface, and avoiding subprocess/systemd calls keeps status fast and portable.
- Do not turn this into chop history, log viewing, or the full deep-doctor report.
- Do not remove or change the documented exit semantics of `sase axe maintenance status` or
  `sase axe lumberjack status`.
- Do not edit release-owned versions in `sase-core`.

## Phase axe-status-domain: Define the portable AXE runtime status contract

Dependencies: none

In the linked `sase-core` repository:

1. Add an `axe_status` module with schema-versioned serde wire types for desired state, orchestrator probe inputs,
   maintenance, lumberjack observations, runner occupancy, lifecycle event, derived issue records, and the complete
   snapshot.
2. Implement a pure deterministic classifier for the state matrix and issue ordering described above. Validate request
   schema and reject structurally invalid inputs clearly; normalize/sort worker and issue output so repeated snapshots
   are stable.
3. Cover the full matrix in Rust tests: healthy running; intentional stop; fresh/not-started; explicit-running outage;
   active maintenance; each lock/PID inconsistency; missing, dead, stale-heartbeat, error, and orphan lumberjacks;
   cumulative historical errors without current degradation; and deterministic ordering/exit codes.
4. Re-export the contract from `sase_core`, add the PyO3 conversion/binding and schema-version helper in `sase_core_py`,
   document the binding, and add binding round-trip/error tests that prove the Python dictionary shape.
5. Verify the linked core with formatting, clippy, and workspace tests (or the shell repository's `just rust-check`
   wrapper) without changing Cargo release versions.

Acceptance:

- Given the same raw host observations, every frontend can obtain the same stable AXE state, health, issues, and exit
  code without filesystem/process access in Rust.
- All domain and PyO3 tests pass, including schema and serialization assertions.

## Phase axe-status-snapshot: Collect one side-effect-free host snapshot

Dependencies: axe-status-domain

In the SASE shell repository:

1. Add typed Python request/response models plus a thin Rust-backed facade for the new binding. Keep it independent of
   argparse, Rich, and Textual.
2. Add a collector that:
   - calls `probe_orchestrator(cleanup=False)`;
   - reads desired state, maintenance, the latest valid lifecycle event, configured lumberjacks, and only live
     unconfigured state-directory lumberjacks;
   - checks process liveness and calculates heartbeat/marker/event ages using an injectable timezone-aware clock;
   - reads current hook/agent runner counts and configured limits;
   - handles files disappearing or changing between reads as a coherent best-effort snapshot rather than crashing;
   - never invokes lifecycle cleanup, ensure, systemd, or any state-writing helper.
3. Give required config/read failures a typed collection error that can become the stable `error` snapshot/exit 2, while
   treating optional missing or malformed historical files as absent data plus a diagnostic where appropriate.
4. Refactor the overlapping part of `check_axe_state()` to consume this shared snapshot, retaining deep-only checks for
   pinned logs, orphan rotation temp files, and historical error totals.
5. Add focused tests using redirected state/config roots, fake process probes, and a fixed clock. Assert exact state
   matrices, race-tolerant missing files, no cleanup/write calls, orphan filtering, heartbeat thresholds, and doctor
   parity.

Acceptance:

- Snapshot collection is read-only, deterministic under a fixed clock, and does not hide partial/incoherent runtime
  evidence.
- The CLI-ready model and deep doctor agree on current desired/orchestrator/lumberjack health.

## Phase axe-status-cli: Ship the polished CLI, automation contract, and docs

Dependencies: axe-status-snapshot

In the SASE shell repository:

1. Register `status` in sorted `sase axe` parser/help output with `-j/--json`; dispatch it from `sase.main.axe_handler`,
   and keep usage strings/help inventory sorted.
2. Add separate human and JSON render functions. Build the Rich summary panel, lumberjack table, and conditional
   attention panel from the shared snapshot; use width-aware/folding columns and injectable consoles for tests.
3. Serialize JSON with stable keys/types and no ANSI, and exit with the snapshot's documented 0/1/2 result. Ensure
   collection errors are actionable in both human and JSON forms.
4. Add parser/help tests, handler/exit-code tests, exact JSON schema tests, and deterministic no-color render tests at
   wide and narrow terminal widths. Exercise healthy, maintenance, intentionally stopped, not-started, down, degraded,
   orphan, and collection-error views.
5. Document the command, fields, state meanings, JSON option, exit codes, read-only guarantee, and relationship to
   `ensure`, deep doctor, maintenance status, and lumberjack status in `docs/axe.md`, `docs/cli.md`, and relevant CLI
   configuration/help reference tables.
6. Run `just install` followed by `just check`; run targeted AXE/parser/render tests during development and
   `just rust-check` as needed after integrating the linked-core binding.

Acceptance:

- `sase axe status` is immediately understandable and visually polished in a terminal, remains readable without color,
  and gives a specific next action for every unhealthy state.
- `sase axe status -j` emits the documented schema with the same classification and exit code as human output.
- Help output is sorted and complete, documentation is current, existing AXE lifecycle behavior is unchanged, and all
  shell/core checks pass.

## Revalidation Checklist

- Re-run the state matrix against explicit desired `running`, explicit `stopped`, and no marker.
- Re-run orchestrator coherence cases with every lifecycle-lock/PID combination, including disappearing processes.
- Re-run lumberjack cases at exactly the heartbeat threshold and one second beyond it.
- Confirm status collection makes no writes and does not call cleanup/ensure/systemd helpers.
- Compare human and JSON output from the same snapshot and assert matching state, health, issues, and exit code.
- Render at narrow and wide widths, with color enabled, `NO_COLOR`, and redirected output.
- Confirm `sase axe -h` remains alphabetically sorted and public long options retain short aliases.
- Run both repositories' required checks and preserve unrelated worktree changes.
