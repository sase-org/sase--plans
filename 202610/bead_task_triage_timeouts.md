---
tier: tale
title: Fix recurring bead_task_triage timeouts and gate failures
goal:
  bead_task_triage finishes well inside its 2m budget with no recurring warnings, and
  snoozed beads get wake gates again.
size: medium
decisions:
  retire_drifted_gates:
    ask:
      Should gate cancellation check only bundle integrity so contract-drifted legacy
      gates can be retired?
    choices:
      retire:
        Cancel checks identity and hashes only; the stuck sase-cj gate is canceled next
        tick
      skip:
        Leave cancel_gate as is; the sase-cj cancel warning keeps repeating every tick
    default: retire
    why:
      Cancel runs no option commands and still checks integrity; it only tolerates
      contract drift
    answer: retire
proposed_by: bbugyi200.athena.0zr
decided_by: reviewer
decided_via: tui
create_time: 2026-10-12 04:57:42
status: wip
---

# Fix the recurring `bead_task_triage` failures (timeouts, missing snooze wake gates, uncancellable legacy gate)

## Diagnosis

The `checks` routine's `bead_task_triage` chop runs every ~5 minutes with a 2-minute
budget (`timeout: "2m"` in `src/sase/default_config.yml`). It has timed out about 50
times since Sep 10. 42 of those are in October, and the rate jumped sharply on Oct 10.
During slow windows every run times out, so the digests show one failure every ~7
minutes. Every run, including the ones that succeed, also logs the same three warnings:

```
[bead_task_triage] Failed to create gate for gh_bobs-org__bob-cli:bob-cli-3o: create_bead_snooze_gate() got an unexpected keyword argument 'closed_at'
[bead_task_triage] Failed to cancel stale gate bead-task-triage-sase-cj-a255163035a8-g2: input_schema must equal the schema compiled from 'inputs'
[bead_task_triage] Failed to create gate for gh_sase-org__sase:sase-po: create_bead_snooze_gate() got an unexpected keyword argument 'closed_at'
```

There are three independent defects. The timeout is the error in the digest. The other
two are the "Subprocess Output" lines that appear in every digest.

### 1. Timeout: the live-agent check walks the entire agent history every tick

Each tick, `bead_work_in_flight()` calls `beads_with_live_agents()` in
`src/sase/bead/work_liveness.py`. That calls
`scan_agent_artifacts(sase_projects_dir(), AGENT_BEAD_SCAN_OPTIONS)`, which is a full
filesystem walk over **every `ace-run` artifact directory ever created**: 17,995 dirs
and 34,743 marker files today, growing by ~300 per day. The walk only exists to find the
~23 agents that are currently alive.

Read-only timings of the chop's phases on the live host:

| phase                                                                                        | time                                                                          |
| -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `bead_work_in_flight` (full artifact walk)                                                   | **15.5–16.0 s** (14.6 s user CPU; single-threaded, CPU-bound, 0 major faults) |
| sase bead store `list_issues` (first call per process)                                       | 1.1–5.2 s                                                                     |
| everything else (inventory, gate polling for 300+ gates, fingerprints, untracked-gate sweep) | < 0.5 s                                                                       |
| successful production runs (`script_duration_ms`)                                            | 15–41 s                                                                       |

The walk is about 85% of the runtime. It grows linearly with history, and because it is
CPU-bound it stretches directly with host load (load average is 30+ during agent and
test fan-outs).

The digests' partial output places the kill site. In ~45 of the 50 timeouts, the bob-cli
warning had already been printed. That warning comes right after the walk finishes, and
the kill landed shortly afterwards. In the rest, nothing was printed, so the kill landed
inside the walk itself. Readers of the Rust bead read model are bounded (250 ms / 5 s
SQLite busy waits, and they never take the bead flock), so the bead read cannot account
for the missing ~100 s.

`bead_claim_checks` had the same full-walk problem and was moved onto the persistent
agent artifact index in `4924c8b9b9` (sase-wn.4,
`src/sase/scripts/_chop_incremental_index.py`). `bead_task_triage` was never migrated. I
measured the index alternative on the live host:

| source                                                                                           | time          | rows   | live `(project, bead) → agent` map |
| ------------------------------------------------------------------------------------------------ | ------------- | ------ | ---------------------------------- |
| full filesystem walk (today)                                                                     | 16.05 s       | 17,995 | 23 entries                         |
| index, full history (`freshness="cached"`, the bead_claim_checks query)                          | 18.05 s       | 17,791 | identical                          |
| index, **active rows only** (`include_active=True`, no completed/history, `include_hidden=True`) | **2.4–2.8 s** | 4,883  | **identical**                      |

The index's active tier is
`has_done_marker = 0 OR workflow_status NOT IN ('completed','failed','cancelled','noop')`
(`sase-core` `agent_scan/index/selection.rs::active_where`). A live worker cannot have
both a done marker and a terminal status, so the live set is always contained in it. The
runner writes `agent_meta.json` (pid, `process_identity`, `bead_id`) through
`write_agent_meta_atomic(..., index_updater=update_agent_artifact_index_for_marker_mutation)`,
so new workers appear in the index at launch. The `launching` half (active launch procs)
already covers the window before that.

### 2. Every snoozed bead's wake gate fails to be created (since Aug 10)

`create_gate()` in `src/sase/scripts/_bead_task_triage_gates.py` builds one `common`
kwargs dict and passes it to both the TaskTriage factory and the BeadSnooze factory.
Commit `187085a80b` ("surface withheld post-close corroboration") added
`"closed_at": issue.closed_at` to `common` for the TaskTriage gate.
`create_bead_snooze_gate()` (`src/sase/bead/snooze_gate.py`) has never accepted
`closed_at`, so every BeadSnooze creation raises `TypeError`. The chop logs a warning
and retries on every tick.

Impact: the only two snoozed beads, `bob-cli-3o` (until 2026-10-12 09:00) and `sase-po`
(until **2026-08-21**, so seven weeks overdue), have no wake gate. Their wakes never
surfaced. Tests missed it because every snooze-gate fake is `lambda **kwargs: ...`
(`tests/_axe_chop_bead_task_triage_helpers.py::patch_snooze_gate` and the inline fakes
in `tests/test_axe_chop_bead_task_triage_snooze.py`), and those accept any keyword.

### 3. A legacy TaskTriage gate can never be cancelled

`sase-cj` was closed on 2026-08-10, so the chop tries to cancel its pending gate
`task_triage/bead-task-triage-sase-cj-a255163035a8-g2` on every tick. The gate was
created 2026-08-07 23:49, just before `0a13ffed4d` ("close the three input-enforcement
gaps") changed the single-line input pattern from `^[^\n]*$` to `^[^\n]*(?![\s\S])`.
`cancel_gate()` (`src/sase/notification_gates/executor_cancellation.py`) calls
`load_and_verify_bundle()`. That function checks integrity (identity plus request and
resource hashes, all of which pass here), and then it **re-validates the stored options
against today's contract**. The stored snooze option's `input_schema` no longer equals
the freshly compiled one, so it raises `conflicting_input_declaration`. The gate looks
pending forever, can never be cancelled, and the lane-state entry is never dropped. A
scan of every pending bundle found exactly one task_triage bundle (and one `custom`
bundle) in this state. Its notification is already gone from the live inbox.

## Changes

### A. Read live agents from the artifact index (fixes the timeout)

`src/sase/bead/work_liveness.py`:

- Add a module constant for the active-tier query, next to `AGENT_BEAD_SCAN_OPTIONS`:
  `AgentArtifactIndexQueryWire(include_active=True, include_recent_completed=False, include_full_history=False, active_limit=None, recent_completed_limit=0, include_hidden=True)`.
  Keep the default `freshness="revalidate"` so stored marker state is rechecked.
- Add a private helper that returns the candidate records:
  - if `use_index` is true and `default_agent_artifact_index_path()` is a file, return
    `query_agent_artifact_index(index_path, root, <active query>, AGENT_BEAD_SCAN_OPTIONS).records`;
  - on any exception from the index path, fail open to the walk, using the same style
    and `# noqa: BLE001` comment as
    `_chop_incremental_index.query_ace_run_index_records`;
  - otherwise return `scan_agent_artifacts(root, AGENT_BEAD_SCAN_OPTIONS).records`.
- Change the signature to
  `beads_with_live_agents(projects_root=None, *, use_index=True)`. The existing body,
  which keeps the newest record per `(project, agent_name)` and then calls
  `agent_record_is_alive`, runs unchanged over the helper's records. Import
  `default_agent_artifact_index_path` and `query_agent_artifact_index` from
  `sase.core.agent_scan_facade` at module level so tests can monkeypatch them on
  `sase.bead.work_liveness`.
- Update the module and function docstrings. They should say that the index's active
  tier always contains every live worker (no done marker plus terminal status while the
  process lives). They should also say that "newest record per agent name" is now
  evaluated among active rows. The only effect is that a record which has already
  written its done marker stops counting a little earlier. The full walk remains the
  fallback when the index is missing or unusable.
- `sase.bead` must not import `sase.scripts`, so the env escape hatch stays in the chop.

`src/sase/scripts/sase_chop_bead_task_triage.py`:

- Import `chop_scan_full_walk` from `sase.scripts._chop_incremental_index`.
- In `_reconcile`, pass
  `live_agents=lambda: beads_with_live_agents(use_index=not chop_scan_full_walk())`. The
  lambda must look up the module-global `beads_with_live_agents` at call time so the
  existing
  `monkeypatch.setattr(task_triage, "beads_with_live_agents", lambda **_kwargs: ...)`
  tests keep working.

`src/sase/scripts/_chop_incremental_index.py`: update the module docstring so it says
`SASE_CHOP_SCAN_FULL_WALK=1` also sends `bead_task_triage`'s live-agent read back to the
full walk. Do not add a new config key. `default_config.yml` needs no change.

### B. Stop passing `closed_at` to the snooze gate factory (fixes the missing wake gates)

`src/sase/scripts/_bead_task_triage_gates.py::create_gate`: remove `"closed_at"` from
`common`. Pass it only to the TaskTriage factory, as
`task_gate_factory(**common, closed_at=issue.closed_at)`. The BeadSnooze call stays
`snooze_gate_factory(snooze=issue.snooze, **common)`. Do not add `closed_at` to the
snooze gate contract; its preview has no post-close rendering.

### C. Let cancellation retire contract-drifted bundles (decision `retire_drifted_gates`)

> [!decision] retire_drifted_gates = skip Do not implement section C or its tests.
> Mention the stuck `sase-cj` gate in the final summary as a known leftover.

`src/sase/notification_gates/hashing.py`:

- Extract the first half of `load_and_verify_bundle()` into a new public
  `verify_bundle_integrity(bundle_path) -> tuple[dict[str, Any], GateAdapter]`. That
  half covers owned-root assertion, schema_version check, kind and request_id identity,
  the adapter lookup with the bundle-path identity check, the request hash, resource
  hashes and modes, and the unexpected-hash check.
- `load_and_verify_bundle()` becomes `verify_bundle_integrity()` followed by the
  existing canonical-structure checks (`normalize_gate_structure`, query, branches,
  groups, primary branch, and the rest). Its behavior and error codes must stay
  byte-for-byte identical. Add the new name to `__all__` if the module defines one.

`src/sase/notification_gates/executor_cancellation.py::cancel_gate`: call
`verify_bundle_integrity()` instead of `load_and_verify_bundle()`. Add a short comment
explaining why. Cancellation is a write-once terminal record that runs no option
commands, so it needs proof that the bundle is the one SASE wrote (integrity). It does
not need proof that the bundle is still answerable under today's contract. Answering,
acceptance, and execution keep using `load_and_verify_bundle()` and still fail closed.
`cancel_gate` uses nothing from the adapter or the normalized structure. It reads only
envelope fields such as `kind`, `request_id`, `hashes.request`, and `notification_id`.
Confirm that by reading `collect_gate_lifecycle_facts` and `settle_gate_notification`
before changing anything. If either one turns out to depend on normalized options, stop
and keep the full verification for the receipt branch only.

With this in place, the next triage tick cancels the `sase-cj` g2 gate with reason
`task_bead_no_longer_ready` and removes it from lane state, using the existing reconcile
code.

## Tests

Add or adjust these tests, following the existing helpers in each file:

1. **Snooze factory signature regression**
   (`tests/test_axe_chop_bead_task_triage_snooze.py`, or a new focused test module).
   Drive `_bead_task_triage_gates.create_gate` once per kind (TaskTriage, BeadSnooze,
   FlagTriage). Use factories built with `unittest.mock.create_autospec(<real factory>)`
   for `create_task_triage_gate`, `create_bead_snooze_gate`, and
   `create_flag_triage_gate`, so an unexpected keyword raises `TypeError` just as it
   does in production. Use an issue that has a non-empty `closed_at` and `close_history`
   (a reopened bead) and, for the snooze case, a snooze record. Assert that the
   TaskTriage factory receives `closed_at` and the BeadSnooze factory does not. Also
   switch `patch_snooze_gate` in `tests/_axe_chop_bead_task_triage_helpers.py` (and any
   sibling task/flag factory patch helpers there) to an autospec'd fake that still
   records kwargs, so the whole suite enforces real signatures. Confirm the new test
   fails against the old `common` dict before you fix it.
2. **Live-agent source selection** (`tests/test_bead/test_work_liveness.py`).
   Monkeypatch `sase.bead.work_liveness.default_agent_artifact_index_path`,
   `query_agent_artifact_index`, and `scan_agent_artifacts`:
   - index file present: the index records are used and the walk is never called (make
     the walk fake raise), and the result matches the walk-based expectation for the
     same records;
   - index file missing: falls back to the walk;
   - index query raises: falls back to the walk;
   - `use_index=False`: walks even though the index exists;
   - the query passed has `include_active=True`, `include_recent_completed=False`,
     `include_full_history=False`, and `include_hidden=True`. The existing tests call
     `beads_with_live_agents(tmp_path)` under the isolated SASE home, where no index
     exists, so they keep exercising the walk unchanged.
3. **Chop wiring** (`tests/test_axe_chop_bead_task_triage_agent_liveness.py`): with the
   real `work_liveness.bead_work_in_flight`, patch `task_triage.beads_with_live_agents`
   to record its kwargs. Assert `use_index=True` by default and `use_index=False` when
   `SASE_CHOP_SCAN_FULL_WALK=1` is set via `monkeypatch.setenv`.
4. **Drifted-bundle cancellation** (only if `retire_drifted_gates = retire`; place it
   with the existing cancellation tests, e.g.
   `tests/test_notification_gate_execution.py` or `tests/test_gate_operations.py`).
   Create a real gate through the service, using a `custom` or `task_triage` kind with a
   single-line (`line`) input. Rewrite its stored option `input_schema` pattern to the
   legacy `^[^\n]*$`, then recompute `hashes.request` with the same request-hash helper
   the service uses, so integrity still holds. Assert:
   - `load_and_verify_bundle` raises `GateError` with code
     `conflicting_input_declaration`;
   - `cancel_gate` succeeds, writes `cancellation.json`, and `poll_gate` reports
     `cancelled`;
   - with the request hash left stale instead, `cancel_gate` still raises
     `hash_mismatch` (integrity is still enforced). Also run the existing cancel,
     answer, and conformance suites (`tests/gate_conformance/`,
     `tests/gate_turn/test_cancel.py`, `tests/test_gate_decision_acceptance.py`) to
     confirm nothing else depended on cancel re-validating structure.

## Verification

1. `just check` (the agent default per the `lint_and_test` memory; do not run
   `just check-full`).
2. One-off host measurement. Put a throwaway script in a temp directory, never inside
   the repo, and run it with the uv-tool interpreter
   (`~/.local/share/uv/tools/sase/bin/python -I`). Compare
   `beads_with_live_agents(use_index=True)` against
   `beads_with_live_agents(use_index=False)` on the real `~/.sase`. They must return
   identical dicts, and the index path should take a few seconds versus ~15 s or more
   for the walk. Report both timings and the entry count in the final summary.
3. Do not edit `~/.sase` state by hand (lane state, gate bundles, notifications). After
   the fix reaches the editable install the scheduler uses, the next `bead_task_triage`
   tick should log no warnings and should:
   - create BeadSnooze wake gates for `bob-cli-3o` and `sase-po`. `sase-po`'s wake time
     is in the past, so its notification surfaces right away; this is expected.
   - with `retire`, cancel the `sase-cj` g2 gate (`canceled=1` once) and remove it from
     lane state.

## Out of scope (note in the final summary, do not implement)

- Of the index's 4,883 "active" rows, about 23 are live. The rest are killed agents that
  still have markers but never wrote a done marker. Startup maintenance only drains rows
  with no markers at all. It does not affect correctness, and the active query stays
  fast (~2.5 s), but it is the next thing to look at if this read slows down again.
- `wait_checks` (routine `agent_waits`) timed out 3 times in October and also reads
  filesystem rows. It may benefit from the same index treatment.
- One `custom` gate bundle (`custom-fc8b6e5b-…`) has the same drift. With `retire` it
  becomes cancellable through the normal cancel path. No automatic action is needed.
