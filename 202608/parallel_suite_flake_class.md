---
tier: epic
title:
  Retire the parallel-suite flake class (sase-ct) by making it reproducible, fixing it
  by mechanism, and gating regressions
goal: 'The default parallel test lanes stop producing unattributable one-node failures.
  The flake class becomes reproducible on demand instead of only under accidental host
  load, every node the durable health store calls a reproducible flake is fixed at its
  mechanism rather than one-at-a-time as it surfaces, and a committed baseline gate
  fails the build when a new flake node appears — so sase-ct can close on a measured,
  enforced criterion instead of on "the node named in the latest reopen is fixed".

  '
phases:
  - id: harness
    title: A contention harness for the default (non-visual) lane
    depends_on: []
    size: medium
    description:
      "harness: add a `just test-contention` recipe and `run_pytest` mode that
      oversubscribes a pinned CPU set the way `test-visual-contention` already does for
      the PNG lane, plus a repeat/soak knob and a per-node failure tally, so the class
      can be reproduced and a fix can be falsified on demand."
  - id: waits
    title: One bounded-wait primitive for raw-pilot tests
    depends_on: []
    size: small
    description:
      "waits: publish a single supported bounded-wait helper for tests that drive a raw
      Textual pilot instead of `AcePage`, retire the four ad-hoc `_wait_until` copies
      onto it, and document when a bare `pause()` is and is not sufficient."
  - id: triage
    title: Measured classification of every flake node
    depends_on:
      - harness
    size: medium
    description:
      "triage: run the contention harness to produce an empirical failure corpus,
      reconcile it against the durable health store's reproducible-flake set, and commit
      a triage table that assigns every node a mechanism family and a named fix shape,
      so the remediation phases work from measurement rather than from the bead's
      anecdotes."
  - id: pump
    title: Fix the off-pump settle-gap family
    depends_on:
      - triage
      - waits
    size: medium
    description:
      "pump: fix every triaged node whose failure is a single `pause()` standing in for
      work that runs off the Textual message pump, by waiting on the observable end
      state with the shared bounded-wait primitive."
  - id: clock
    title: Fix the real-wall-clock-threshold family
    depends_on:
      - triage
      - waits
    size: medium
    description:
      "clock: remove the dependence on real elapsed wall-clock time from the stall
      watchdog tests and the other triaged deadline-shaped nodes, by driving them from
      an injectable time source or a load-normalized budget instead of loosening
      assertions again."
  - id: fixture
    title: Fix the ACE fixture-state and cross-test-leakage family
    depends_on:
      - triage
      - waits
    size: medium
    description:
      "fixture: fix the triaged ACE nodes whose injected fixture state is overwritten by
      a queued repaint or leaks across tests, by making state injection settle-verified
      and the affected panes isolated per test."
  - id: tooling
    title: Fix the non-ACE store, tooling, and subprocess family
    depends_on:
      - triage
      - waits
    size: medium
    description:
      "tooling: fix the triaged non-ACE nodes — bead-store clusters, selection and
      run_pytest tooling tests, the coverage-context cache, and the subprocess/pipe
      races — while leaving the bead-mutation lock timeout tracked separately under
      sase-e2 alone."
  - id: gate
    title: A committed flake baseline that fails the build on new flakes
    depends_on:
      - pump
      - clock
      - fixture
      - tooling
    size: medium
    description:
      "gate: commit a flake baseline file and a gate that fails when the health store's
      reproducible-flake set exceeds it, wire the gate into the exhaustive lane and a CI
      contention job, and add the lint check that stops the retired ad-hoc wait helpers
      from coming back."
  - id: land
    title: Land the epic and close sase-ct on a measured criterion
    depends_on:
      - gate
    size: small
    description:
      "land: run the exit criterion on the combined tree, file any residue with
      /sase_new_task, and close sase-ct with a note that states the root cause of the
      class and the enforced criterion that replaces hand-adjudication."
proposed_by: bbugyi200.athena.v5
bead_id: sase-h8
create_time: 2026-09-09 19:51:01
status: wip
---

- **PROMPT:**
  [prompts/202608/parallel_suite_flake_class.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/parallel_suite_flake_class.md)
- **BEAD:**
  [sase-h8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-h8/README.md)

# Plan: Retire the parallel-suite flake class

## Why this plan exists

`sase-ct` has been closed six times and reopened six times. Every closure was honest
about the node it fixed and every reopen was a different node. The bead's own triage
note (2026-08-05) already said the scope is "making the default parallel suite reliable,
not fixing one node", but no closure has ever been able to act on that, because the
class has never been reproducible on demand. Every fix so far was verified by
hand-injecting a delay into one specific code path with a throwaway probe, then
declaring victory when a single `just check` went green — a run that passes roughly 57%
of the time even with the bug present.

That is the actual defect this plan fixes: **there is no reproducer and no gate**, so
remediation is driven by whoever happens to trip over a node next, and closure is a
judgment call rather than a measurement.

Two concrete demonstrations that node-at-a-time remediation does not converge, both
verified in this workspace at `0c084068c`:

- `156cac833` (2026-08-07 12:48 EDT) loosened
  `test_watchdog_keeps_hitch_and_stall_state_machines_independent` specifically to fix
  it under this bead. `156cac833` is an ancestor of `28d40c5a8`. The health store
  records that same node failing again on `28d40c5a8` at 2026-08-07T21:33:22Z. The
  targeted fix did not hold.
- The same file's sibling nodes were then fixed one at a time in three separate closures
  (`test_watchdog_records_compact_loop_hitch_and_recovery`,
  `test_watchdog_records_compact_pump_hitch_and_recovery` reported but not fixed), each
  with its own bespoke remedy, because the shared cause — the test asserts on real
  elapsed wall-clock time — was never removed.

## Measured state

All numbers below were measured in this workspace at `0c084068c` on the 64-core
development host, from the durable selection-health store at
`~/.sase/test-selection/gh_sase-org__sase/`.

| Measurement                                                      |                              Value |
| ---------------------------------------------------------------- | ---------------------------------: |
| Full-run health records in the store                             |                                256 |
| Full runs with at least one failure                              |                                109 |
| Red rate, all recorded full runs                                 |                              42.6% |
| Red rate, most recent 80 full runs                               |                              42.5% |
| Nodes satisfying `reproducible_flake_nodeids` over the store     |                                 30 |
| Nodes `just selection-health` currently reports flake-suppressed | 17 (79 scoped run/failure matches) |
| ACE TUI test files                                               |                                866 |
| `.pause()` call sites under `tests/ace/tui/`                     |             1,887 across 173 files |

The red rate is the number that matters. A mandatory gate that fails 43% of the time for
reasons unrelated to the change under test is not a gate; it is a coin flip that costs a
re-run. The bead's `+1` evidence is full of exactly that cost — `sase-h2` paid three
consecutive re-runs of a ~3-minute gate on a change that touches no `src/` code at all,
and `sase-gy.2` was intercepted on a Markdown prose-width flip.

The class is also **not confined to ACE TUI**, despite the bead's title. Of the 30
reproducible-flake nodes, roughly a third are ACE and the rest are bead-store, selection
tooling, coverage-context, and `fakey` subprocess nodes. And it is not confined to
contended dev hosts: `sase-fq.land` and `ci_fix.sase.8`/`ci_fix.sase.f` recorded the
same one-node-per-run signature on GitHub-hosted runners after `9672c5602` restored full
worker parallelism on 4-vCPU runners.

## The precedent this plan copies

This repository has already retired one flake class of exactly this shape, and it did
not do it by fixing nodes as they surfaced. `just test-visual-contention` pins a
26-worker pool to two CPUs (13x oversubscription) to make PNG convergence flakes
deterministic. The Justfile comment records the arc: **116 failed** at the pre-fix
baseline, 15 failed after a convergence-only fix, then **363 passed** once the real
cause was fixed — under the same oversubscription, without regenerating goldens, and
with exact pixel equality retained.

That worked because the harness came first. This plan applies the identical method to
the default non-visual lane, which has no such harness today.

## Mechanism families

The bead's own notes, the health store, and the code already establish four distinct
mechanisms. They are listed here so the remediation phases have a starting hypothesis;
`triage` is what assigns each node to one on measured evidence, and is allowed to add a
family or move a node.

**F1 — off-pump settle gap.** A test does one `pilot.pause()` and asserts, but the work
it is waiting on runs off Textual's message pump (a worker thread, `asyncio.to_thread`,
a pump-free task). `pilot.pause()` calls `textual._wait.wait_for_idle(0)`, which
guarantees only ~20–40 ms of in-loop idle and never awaits a worker thread. Diagnosed
concretely for `test_bulk_waiting_agents_mount_forced_artifact_prompts` (relaunch prompt
resolution via `schedule_relaunch_prompt_resolution`,
`src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py:61`) and for
`test_on_mount_refines_title_to_resolved_version` (off-thread title refinement).

**F2 — real wall-clock thresholds.** The assertion depends on how much real time
elapsed. The stall watchdog tests construct thresholds (`hitch_threshold_seconds=0.03`,
`threshold_seconds=0.08`, `poll_interval_seconds=0.01`) that collide with the file's own
`await asyncio.sleep(0.03)` settle idiom, so contention during the settle registers a
spurious hitch cycle. `test_codeblock_band_replaces_cursor_line_fill_but_not_cursor`
depended on Textual's 0.5 s cursor blink phase.
`test_contract_set_serial_runtime_stays_within_budget` is a fixed serial wall-clock
ceiling a loaded runner misses by ~4%.

**F3 — lost writes on injected fixture state.** The test injects state and a queued
render overwrites it during the pause. Diagnosed for
`test_agent_metadata_search.py::_set_prompt_text`: a queued
`_fire_debounced_detail_update → AgentDetail.update_display → AgentPromptPanel.update_display`
repaints `#agent-prompt-panel` and drops the injected corpus, after which the search
overlay legitimately renders empty.

**F4 — cross-test and cross-process shared state.** State leaks between tests or between
concurrently running suites. `test_rapid_navigation_loads_only_the_final_detail`
observed `rows[1].id` where `rows[2].id` was asserted. The `tests/test_bead/` snooze,
close-history, and migration nodes fail in clusters, which is the signature of shared
store state rather than per-node timing.
`test_installing_prunes_the_cache_to_the_keep_limit` had a real mtime tie-break defect
(fixed by `aec67f31c`) of the same family.

**F5 — subprocess and pipe races.**
`test_tracked_executor_reports_terminal_and_extra_commands_live` fails with
`cannot start command: [Errno 32] Broken pipe` under load; it is the single most
frequent ACE node in the store (13 occurrences). `aaa8245df` guarded streaming stdin
close against broken pipe on 2026-08-07 at 17:34 EDT, which **postdates** the node's
most recent recorded failure (2026-08-07T21:08:05Z on `28d40c5a8`), so that fix has
never been exercised against the class. The `fakey` retry-pipeline and
`test_hook_wrapper_retry` nodes are the same shape.

## Non-goals

- **`sase-e2`.**
  `tests/test_bead/test_cli_work_contention_regressions.py::test_concurrent_bead_mutations_wait_past_the_old_lock_timeout`
  is the concurrent bead-mutation lock timeout, tracked separately and already in
  progress. The 2026-08-05 triage explicitly excluded it. It stays excluded here even
  though it is the single most frequent node in the store (20 occurrences); `triage`
  records it as out-of-scope rather than silently absorbing it.
- **Suite scaling and test selection.** The two-speed verification architecture, the
  selection engine, and the contract set are delivered work (`sase-fp`,
  `@plan:202608/test_suite_tier1.md`). This plan does not revisit them.
- **PNG visual convergence.** Already retired via `test-visual-contention`; this plan
  reuses its method, not its scope. If the contention harness surfaces visual-lane
  nodes, they are recorded for a follow-up, not fixed here.
- **Retrying or quarantining flaky tests.** Auto-retry hides the defects this plan
  exists to remove. The gate in the `gate` phase is a baseline that must shrink, not a
  suppression list that may grow.

## Ground rules for every phase

- Do not fix a node by loosening an assertion until the mechanism is understood and the
  loosened assertion still pins the behavior the test exists to verify. `156cac833` is
  the cautionary example: the loosened counts were correct, and the node still recurred,
  because the wall-clock dependence was left in place.
- Every claimed fix must be falsified against the `harness` reproducer, not against a
  single green `just check`. A fix that cannot be shown to fail before and pass after
  under the harness is not verified.
- Record measurements in the phase bead note: the node, the mechanism, the before/after
  under the harness, and the family assignment.
- Run `just install` before other `just` targets in a freshly claimed workspace.

---

## Phase `harness`: A contention harness for the default (non-visual) lane

Add the reproducer the class has never had.

**Deliverables.**

- A `contention` mode in `tools/run_pytest`, and a `just test-contention` recipe modeled
  directly on the existing `test-visual-contention` recipe in the `Justfile`: `taskset`
  onto a small pinned CPU set with a heavily oversubscribed worker count, overridable
  via environment variables in the same style as `SASE_VISUAL_CONTENTION_CPUS` /
  `SASE_VISUAL_CONTENTION_WORKERS`. Guard on `taskset` being available, as the visual
  recipe does.
- A repeat/soak knob so a run executes the selection R times and a per-node failure
  tally is emitted at the end (node id, failure count, run indices). One pass is not
  enough signal for a class whose base rate is under one node per run; the tally is what
  makes a fix falsifiable.
- The mode must **not** take a suite-gate lease and must not be reachable from
  `just check` or `just check-full`. It is an opt-in diagnostic lane. Keep it out of
  `FULL_LANE_MODES` and out of the health-store recording modes so soak failures do not
  pollute the durable selection-health record that `triage` and `gate` read.
- Recipe comments recording the measured pre-fix baseline, matching the convention the
  `test-visual-contention` comment block already sets.

**Acceptance.** Starting from `0c084068c` with no test fixes applied, the harness
reproduces at least three distinct known nodes across a bounded run — including at least
one from F1 and one from F2 — and reports them in the tally. Reproducing
`test_watchdog_keeps_hitch_and_stall_state_machines_independent` and
`test_tracked_executor_reports_terminal_and_extra_commands_live` is the concrete target;
they are the two most frequent ACE nodes in the store. If a full-suite contention run is
too slow to iterate on, also support restricting it to a node-id or path subset, because
that is the loop the remediation phases will live in.

**Watch out for.** The harness deliberately starves the machine. Do not let it acquire
the shared suite gate, and note in the recipe comment that other agents' runs on the
same host will be affected while it runs.

## Phase `waits`: One bounded-wait primitive for raw-pilot tests

The primitive largely exists already and is under-used.
`src/sase/ace/testing/ace_page.py` ships `AcePage.wait_for`, `expect_state`,
`expect_modal`, `expect_no_modal`, and `expect_screen_contains`, all bounded polling
with a 5 s default and a diagnostic timeout message. 207 ACE TUI test files use
`AcePage`; 100 drive a raw pilot from `run_test()` and have no such helper, which is why
four near-identical private copies exist:

- `tests/ace/tui/test_agent_bulk_kill_edit.py:35`
- `tests/ace/tui/_config_center_tabs_helpers.py:51`
- `tests/ace/tui/widgets/test_prompt_format.py:34`
- `tests/ace/tui/test_family_member_relaunch.py:64`

**Deliverables.**

- One shared, supported bounded-wait helper for raw-pilot tests, placed where ACE test
  helpers already live and named consistently with the `AcePage` methods. It takes a
  pilot and a predicate, polls via `pilot.pause()`, and raises with a message that names
  the predicate and the elapsed timeout — the ad-hoc copies' generic "condition timed
  out" strings are part of why these failures were expensive to diagnose from a CI log.
- The four copies above retired onto it, with no behavior change.
- A short docstring or helper-module comment stating the rule the remediation phases
  apply: a bare `pause()` is sufficient only for work that completes on the message
  pump; anything crossing a thread, a worker, or a pump-free task must be waited on by
  observable end state.

**Deliberately not in scope.** Mass-migrating all 1,887 `.pause()` call sites. Most are
fine. Only nodes `triage` implicates get migrated, in `pump`.

**Acceptance.** `just check` green; the four call sites now import one helper; no new
public API surface beyond the single helper.

## Phase `triage`: Measured classification of every flake node

Turn the bead's anecdote pile into a table the remediation phases can execute against.

**Deliverables.**

- A soak run of the `harness` reproducer at the epic's base commit, long enough to
  produce a per-node failure tally with more than one observation for the frequent
  nodes. Record the exact command, CPU pinning, worker count, repeat count, and wall
  time.
- The union of that tally with the durable health store's reproducible-flake set.
  Compute the store side with `reproducible_flake_nodeids`
  (`tests/_test_selection_health.py:156`) over the full-run records, and cross-check
  against `just selection-health`'s flake-suppressed section. Note that the two sets
  differ legitimately — the report's set is additionally restricted to nodes matched by
  a scoped run — and record both.
- A committed triage table (a research note under `sdd/research/`, or the epic's own
  bead notes if the phase judges a file unwarranted) with one row per node: node id,
  occurrences, mechanism family, the specific observed symptom, the named fix shape, and
  the owning remediation phase. Nodes already fixed by an earlier `sase-ct` closure are
  included and marked, so the remediation phases can confirm rather than re-derive them.
- An explicit out-of-scope list: the `sase-e2` bead-lock node, any visual-lane node, and
  any node the phase determines is a real regression rather than a flake — a cluster of
  nodes in one file failing together in the same run is more likely a genuine break at
  that commit than a timing race, and the `tests/test_bead/` snooze cluster in
  particular should be checked against that hypothesis before being assigned to F4.

**Acceptance.** Every node in the union has a family and an owning phase, or an explicit
out-of-scope reason. The remediation phases can start without re-running the soak.

**Watch out for.** The health store is a shared, host-local, 30-day-retention artifact
under `~/.sase`. Read it; do not mutate or prune it.

## Phase `pump`: Fix the off-pump settle-gap family

Fix every F1 node from the triage table.

**Expected membership** (confirm against the table; do not treat this as the list):
`test_on_mount_refines_title_to_resolved_version`, the three
`modals/test_artifact_files_modal_copy.py` copy nodes,
`test_rows_apply_and_loading_clears_while_cleanup_is_blocked`, the two
`test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_*` nodes,
`test_config_jump_hint_moves_cursor_and_repaints_detail`,
`TestAtPrefixIntegration::test_at_prefix_directory_drilldown`,
`test_updates_pane_sase_dev_update_shows_all_commit_groups`,
`test_xprompt_highlight_overlay_marks_spans_and_registers_styles`, and
`test_prompt_catalog.py`'s implicated node.
`test_bulk_waiting_agents_mount_forced_artifact_prompts` is already fixed this way
(`bde727ecc`) and serves as the reference shape; confirm it under the harness rather
than re-fixing it.

**Method.** For each node, identify the asynchronous boundary the test is racing —
worker thread, `asyncio.to_thread`, pump-free task, debounced timer — then replace the
bare `pause()` with a bounded wait on the observable end state, using `AcePage.wait_for`
/ `expect_*` for `AcePage` tests and the `waits` helper for raw-pilot tests. Name the
boundary in a comment at the call site, as `test_agent_bulk_kill_edit.py` already does.

**Acceptance.** Each fixed node is shown failing before and passing after under the
`harness` reproducer with an injected delay at the identified boundary, not merely
passing a clean run. `just check` green.

**Watch out for.** A bounded wait that polls for a condition already true when the test
starts silently verifies nothing. Where the end state is not distinguishable from the
initial state, wait on a transition, not a value.

## Phase `clock`: Fix the real-wall-clock-threshold family

Remove real elapsed time from the assertions, rather than widening tolerances a third
time.

**Expected membership**: the four `tests/ace/tui/util/test_stall_watchdog.py` nodes
(`test_watchdog_keeps_hitch_and_stall_state_machines_independent`,
`test_watchdog_writes_loop_recovery_record`,
`test_watchdog_records_compact_pump_hitch_and_recovery`,
`test_watchdog_records_compact_loop_hitch_and_recovery`),
`test_contract_set_serial_runtime_stays_within_budget`, and any node the triage table
adds.

**Method.** The watchdog file is the priority and the hard case: it has now absorbed
three separate targeted fixes under this bead and one of them demonstrably did not hold.
Prefer a structural change over another tolerance widening — drive
`_EventLoopStallWatchdog` from an injectable time source or an explicitly advanced clock
in tests so hitch and stall episodes are produced deterministically, and keep at most
one test that exercises the real timer path end to end with generous margins. Review
`src/sase/ace/tui/util/stall_watchdog.py` and `_stall_watchdog_monitor.py` for the seam;
if introducing one requires a production change, that is acceptable and expected here.

For `test_contract_set_serial_runtime_stays_within_budget`, note that two prior attempts
already exist — the `budget` phase of `@plan:202608/test_selection_landing.md`
(calibration-probe-normalized CPU measurement) and `08d0e0476` (reshaped CPU probe,
restored headroom under xdist contention). It failed again after both, most recently at
2026-08-07T20:01:12Z on `43250ffb6`. Establish why the normalization is not holding
before adding headroom a third time; if a wall-clock budget cannot be made sound under
oversubscription, propose moving the guard off wall clock entirely.

**Acceptance.** Under the harness, the watchdog file passes across the full repeat count
with zero failures, and the fixed nodes no longer contain an assertion whose truth
depends on how loaded the host is. `just check` green.

**Watch out for.** The watchdog's purpose is to detect real event-loop stalls in
production. Do not make the tests deterministic by making the watchdog untestable
against real time — keep coverage that the real timer path works.

## Phase `fixture`: Fix the ACE fixture-state and cross-test-leakage family

Fix the F3 and F4 nodes in the ACE lane.

**Expected membership**: the `tests/ace/tui/test_agent_metadata_search.py` nodes
(`test_inline_metadata_search_commit_repeat_q_and_passthrough`,
`test_inline_metadata_search_reverse_key_override`,
`test_inline_metadata_search_yank_and_frozen_refresh`) and
`test_rapid_navigation_loads_only_the_final_detail`.

**Method.** `_set_prompt_text` in the metadata-search file was already repaired once by
re-applying the injected text until it survives a settling turn. Generalize that:
injection into a panel that a debounced render can repaint needs to be either
settle-verified or performed after the competing render is quiesced. Prefer suppressing
or awaiting the competing render over retry-until-it-sticks where the pane makes that
possible — a retry loop fixes the symptom and leaves the race.

For `test_rapid_navigation_loads_only_the_final_detail`, the reported symptom is a
neighbouring row's id, which is state leaking across tests rather than a timing budget.
Establish whether the leak is module-level cache, a shared loader, or an unreset
reactive, and isolate it with a fixture — check whether `tests/ace/tui/conftest.py`
(which already autouse-resets the ACE shutdown signal and isolates the commits project
inventory) is the right home for the reset.

**Acceptance.** Each node shown failing before and passing after under the harness. The
injected-state helper's contract is documented at its definition. `just check` green.

## Phase `tooling`: Fix the non-ACE store, tooling, and subprocess family

Fix the non-ACE nodes. This is the larger half of the reproducible-flake set and the
half the bead's title has always undersold.

**Expected membership** (confirm against the triage table): the `tests/test_bead/`
clusters (`test_cli_snooze.py` ×4, `test_snooze_lifecycle.py`, `test_snooze_gate.py`,
`test_snooze_close_regression.py` ×3, `test_close_history_*` ×3, `test_db_migrations.py`
×2, `test_cli_golden.py::test_bead_cli_golden_contract[list_json]`), the selection and
runner tooling nodes (`tests/test_run_pytest_scoped.py` ×2,
`tests/test_select_tests_tool.py` ×2, `tests/test_suite_gate_integration.py`,
`tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection`,
`tests/test_install_coverage_contexts_tool.py::test_installing_prunes_the_cache_to_the_keep_limit`,
`tests/test_plan_display.py::test_malformed_header_block_leaves_authored_metadata_visible`,
`tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor`),
and the subprocess races
(`tests/ace/tui/test_notification_custom_gate.py::test_tracked_executor_reports_terminal_and_extra_commands_live`,
`tests/fakey/test_retry_pipeline_e2e.py` ×2, `tests/test_hook_wrapper_retry.py`).

**Method notes.**

- The bead-store clusters fail together, which points at shared state or a shared temp
  root rather than per-node timing. `tw.f1`'s report of three close-history nodes
  failing together, all passing in isolation with and without the reporter's diff, is
  the clearest evidence. Check for a store or `SASE_HOME` that is not per-test, and
  check the interaction with the `TMPDIR`/`SASE_PYTEST_TMP_REDIRECTED` isolation that
  `sase-fq.8.2` landed.
- `test_tracked_executor_reports_terminal_and_extra_commands_live` is the most frequent
  ACE node in the store and is a subprocess test, not a pilot test. `aaa8245df` guarded
  streaming stdin close against broken pipe _after_ the node's last recorded failure, so
  the first job is to determine under the harness whether that fix already closed it. If
  it did, say so with evidence; if it did not, the `assert False is True` and `Errno 32`
  symptoms are two different failures and both need accounting.
- `test_malformed_header_block_leaves_authored_metadata_visible` was previously observed
  failing identically across five unrelated workspaces because of an out-of-date
  `sase_core_rs` binding. `5a039ef14` now fails loudly when the built binding is behind
  the pyproject floor. Confirm whether that node is already resolved by it rather than
  assuming.

**Acceptance.** Each fixed node shown failing before and passing after under the
harness, or documented with evidence as already resolved by a named commit. `sase-e2`'s
bead-lock node is untouched and explicitly named as out of scope in the phase note.
`just check` green.

## Phase `gate`: A committed flake baseline that fails the build on new flakes

Make the class self-reporting so it can never again be discovered only by an agent
tripping over it.

**Deliverables.**

- A committed flake baseline file — an empty-or-near-empty allowlist of node ids, in the
  style of `tests/contract_manifest.txt`, with a header comment explaining that entries
  are debts to remove and that adding one requires a filed bead.
- A gate that compares the durable health store's reproducible-flake set against that
  baseline and fails when the set exceeds it. `just selection-health` already computes
  and reports the set; extend it with a fail-on-new-flake mode rather than building a
  parallel tool. Wire the gate into `just check-full` — not `just check`, which must
  stay fast — and ensure it degrades to a pass with a clear message when the local store
  has too few records to judge, so a fresh workspace is not blocked.
- A CI job that runs the `harness` reproducer on a schedule (not per-PR — it is
  deliberately slow and starves its runner), so CI-side members of the class surface as
  a named job failure rather than as an unattributable one-node red in `test (3.12)`.
  Model it on the existing `visual-test` and `perf-floors` jobs in
  `.github/workflows/ci.yml`.
- A lint check that fails on a re-introduced private bounded-wait helper in `tests/`, so
  the four copies `waits` retired do not come back. Add it alongside the repo's existing
  custom gates (`tools/pyscripts-*`, `tests/test_justfile_lint.py`) in whichever style
  fits.

**Acceptance.** With the baseline committed and the remediation phases landed,
`just check-full` passes; hand-adding a known-flaky node id to a scratch copy of the
store makes the gate fail with a message naming the node; removing it makes the gate
pass.

**Watch out for.** The gate reads a shared host-local store that several workspaces
write. It must not become a cross-workspace flake generator itself — that would be a
spectacular irony. Prefer failing only on nodes with enough observations to satisfy
`reproducible_flake_nodeids`, which already requires two full runs with disjoint change
sets.

## Phase `land`: Land the epic and close sase-ct on a measured criterion

**Exit criterion.** On the combined tree:

1. `just test-contention` at the baseline pinning and worker count, repeated, with
   **zero** failures in the tally — the same bar `test-visual-contention` cleared when
   it went from 116 failures to 363 passed.
2. `just check-full` green.
3. `just test-visual` green, confirming the fixes did not disturb the PNG lane.
4. The flake gate from `gate` passing against a baseline file that contains no ACE node
   and no node this epic was scoped to fix.

**Then.**

- File any residue with `/sase_new_task` — in particular any node `triage` marked
  out-of-scope, and any visual-lane node the harness surfaced.
- Close `sase-ct` with a note that states, in this order: the root cause of the _class_
  (no reproducer and no gate, so remediation was driven by whoever tripped over a node
  next and closure was a judgment call); the four mechanism families and which phase
  fixed each; the before/after red rate; and the enforced criterion that now replaces
  hand-adjudication. Reference the harness recipe and the baseline file by name so the
  next reader can re-run the measurement.
- Mark this plan done.

**Watch out for.** `sase-ct` has been closed six times already. If the exit criterion is
not met, do not close it — report what is left and leave it open. A seventh reopen is a
worse outcome than an honest open bead.
