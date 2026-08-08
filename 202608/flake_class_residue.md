---
tier: epic
title: Finish epic sase-h8 by landing the never-implemented clock phase, clearing
  the four nodes that failed its exit criterion, and closing the wait-idiom gate gaps
goal: 'Epic sase-h8 reaches the exit criterion it wrote for itself and could not meet.
  The `clock` phase, whose bead closed `done` with zero commits after its agent stalled
  on a soak, is actually implemented: the stall watchdog runs off an injectable time
  source so its five F2 nodes assert exact episode counts instead of tolerances, and
  the contract-set budget guard stops depending on real elapsed wall clock. The four
  nodes that failed `just test-contention` at the land attempt are fixed by mechanism
  under the same discipline. The wait-helper gate stops missing the idiom it was built
  to retire, so post-epic commits cannot silently reintroduce it. Only then does sase-h8
  close and sase-ct close on the measured, enforced criterion the epic promised.

  '
phases:
- id: clock
  title: Actually implement the clock phase that sase-h8.5 closed without landing
  depends_on: []
  size: medium
  description: 'clock: drive `EventLoopStallWatchdog` from an injectable time source
    so the five F2 `test_stall_watchdog.py` nodes plus `test_nested_pause_requires_final_resume_before_detection`
    produce hitch and stall episodes deterministically, restore exact episode counts
    in place of the `>= 1` tolerances two prior fixes installed, keep one real-timer
    end-to-end test, and move `test_contract_set_serial_runtime_stays_within_budget`
    off wall clock now that sase-h8.7''s F6 fix unblocked it.'
- id: residue
  title: Fix the four nodes that failed the sase-h8.9 exit criterion
  depends_on:
  - clock
  size: medium
  description: 'residue: fix the four nodes `just test-contention` failed on at the
    land attempt — three wall-clock-shaped (`test_first_page_paints_before_full_extension`,
    `test_lowered_threshold_soak_keeps_fixed_paths_responsive`, `test_timed_out_summary_script_never_blocks_launch`)
    and one off-pump (`test_apostrophe_enters_jump_mode_with_hints_skipping_headers`)
    — using the conventions `clock` establishes and the sase-h8.2 wait primitive.'
- id: gate-gaps
  title: Close the wait-idiom gate gaps that let the retired pattern back in
  depends_on: []
  size: medium
  description: 'gate-gaps: widen `tools/check_test_wait_helpers` past the two roots
    and the one function name it currently matches so it catches the sixth `_wait_until`
    copy, the inline `for _ in range(N): await pilot.pause()` bounded-wait loops that
    landed after the epic''s waits phase, and the raw ACE panel injections sase-h8.6
    asked for; migrate every call site the widened check reports.'
- id: land
  title: Meet the exit criterion, close sase-ct, and close the epic
  depends_on:
  - clock
  - residue
  - gate-gaps
  size: small
  description: 'land: re-run all four sase-h8 exit criteria on the combined tree,
    file genuinely distinct residue with /sase_new_task, close sase-ct with the note
    its plan specifies, then close sase-h8, run `just symvision` and remove what it
    reports, and mark both plan files done.'
proposed_by: bbugyi200.athena.sase-h8.land
parent_bead: sase-h8
create_time: 2026-08-08 10:56:16
status: wip
bead_id: sase-h8.10
---

- **PROMPT:** [prompts/202608/flake_class_residue.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/flake_class_residue.md)
- **BEAD:** [sase-h8.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-h8/sase-h8.10.md)

# Plan: Finish epic sase-h8

## Why this plan exists

Epic `sase-h8` was chartered to retire the `sase-ct` flake class by making it
reproducible, fixing it by mechanism, and gating regressions. Eight of its nine phases
did that. The land agent's verification found two holes that keep the epic from closing
honestly, and one that keeps it from staying closed.

**The `clock` phase never landed.** `sase-h8.5` is closed with resolution `done`, but it
has no closing note, and `git log --all --grep=sase-h8.5` returns nothing. Its agent
transcript (`~/.sase/chats/202608/gh_sase_org__sase-ace_run-sase_h8_5-260807_200435.md`)
shows it establishing a baseline soak, correctly refusing to edit files mid-measurement,
and then stalling — its last words are "I'll pick up automatically when it completes."
It never did. The bead was closed 15 hours later with no work in the tree. The evidence
is visible in the source today: `tests/ace/tui/util/test_stall_watchdog.py` still
contains `time.sleep(0.14)` blocks and real `time.monotonic()` deadlines, and
`test_watchdog_keeps_hitch_and_stall_state_machines_independent` is still in the durable
health store's reproducible-flake set. The `sase-h8.3` triage table called this file
"five nodes in one file failing by one shared mechanism, which is the strongest argument
in this triage for the structural change the plan asks for" — and the structural change
was never made.

**The exit criterion failed.** `sase-h8.9` ran criterion 1 on the combined tree at
`c902dd71c` and `just test-contention` failed repeat 1 with four nodes red. That agent
did the right thing: it did not close `sase-ct`, exactly as the parent plan instructed
("If the exit criterion is not met, do not close it — report what is left and leave it
open. A seventh reopen is a worse outcome than an honest open bead."). Three of those
four nodes are wall-clock shaped, which is to say they are `clock`'s family arriving
late — further evidence the missing phase is the load-bearing gap, not an incidental
one.

**The gate does not catch the idiom it was built to catch.**
`tools/check_test_wait_helpers` matches exactly one function name, `_wait_until`, in
exactly two directory roots. It therefore misses a sixth copy that survives at
`tests/test_agent_group_revival_e2e.py:409`, and it misses inline bounded-wait loops
entirely. Commits that landed during and after the epic reintroduced that inline idiom
in `tests/ace/tui/test_custom_gate_modal.py:259` (from `010b01a41`, the newest commit on
master) and `tests/ace/tui/test_notification_plan_gate.py:117,156,181`. A gate that a
week of ordinary commits already walked around is not a gate.

Everything else verified clean. The six `ff0b765a4` notification-gate nodes, the Muse
`test_setup_hint_points_script_installs_at_the_install_subcommand` doctor node, and the
`test_content_layout.py` schema-version node — all filed as `PROPOSED FOLLOW-UP` by four
different phases as "blocks a clean `just check` for every agent" — are fixed on master
by later commits; 35 tests across those files pass at HEAD. The harness
self-contamination that `sase-h8.3` found, and the fifth `_wait_until` copy in
`tests/fakey/harness.py`, were both fixed inside the epic. Those need no further work.

## What is already true (do not redo it)

| Deliverable                                                             | Where                                                                                                       | Commit           |
| ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ---------------- |
| `just test-contention` + `run_pytest` contention mode + per-node tally  | `Justfile`, `tools/run_pytest`, `tests/_contention.py`, `tests/_contention_plugin.py`                       | `2bac5ad9e`      |
| `wait_for` bounded-wait primitive                                       | `src/sase/ace/testing/wait.py`                                                                              | `6476ec65c`      |
| Triage table, 45 nodes, family + fix shape + owning phase               | `research:202608/parallel_suite_flake_triage.md`                                                            | research sidecar |
| F1 off-pump fixes (clipboard delivery, path-inventory worker)           | `tests/ace/tui/modals/test_artifact_files_modal_copy.py`, `.../widgets/test_prompt_at_prefix_completion.py` | `4dc323117`      |
| `set_agent_prompt_document` pinned-document injector                    | `src/sase/ace/testing/prompt_document.py`                                                                   | `f980248c1`      |
| Ambient env pinning + fakey `LOAD_TOLERANT_TIMEOUT` + `hold_retry_wait` | `tests/_run_pytest_fixtures.py`, `tests/fakey/harness.py`                                                   | `0a1502a04`      |
| Flake baseline + `--fail-on-new-flake` gate + CI contention lane        | `tests/reproducible_flake_baseline.txt`, `tools/selection_health`, `.github/workflows/ci.yml`               | `c902dd71c`      |

Read `research:202608/parallel_suite_flake_triage.md` before starting any phase. It
measures family membership empirically and corrects the original epic plan in several
places.

The mechanism families it uses, referenced throughout below: **F1** off-pump settle gap,
**F2** real-wall-clock threshold, **F3** injected fixture state lost to a repaint,
**F4** cross-test leakage, **F5** subprocess/pipe race, **F6** ambient env leakage.

## Phase `clock`: Actually implement the clock phase

**Scope.** The five `tests/ace/tui/util/test_stall_watchdog.py` nodes the triage table
assigns to `clock`:

- `test_watchdog_records_one_stall_with_stack_and_context` (`assert 3 == 1` — three
  `tui_stall` records where one was asserted)
- `test_watchdog_writes_loop_recovery_record` (spurious extra stall before
  `stall_recovered`)
- `test_watchdog_keeps_hitch_and_stall_state_machines_independent` (spurious hitch cycle
  during the file's own settle; **recurred after `156cac833` already widened it**)
- `test_watchdog_records_compact_loop_hitch_and_recovery` (unbalanced hitch/recovery
  pairs)
- `test_watchdog_records_compact_pump_hitch_and_recovery` (same)

plus `test_nested_pause_requires_final_resume_before_detection`, which `sase-h8.7`
measured at 1/8 under its post-fix soak and explicitly handed to `clock` as a sixth F2
node in the same file, and
`test_contract_manifest.py::test_contract_set_serial_runtime_stays_within_budget`.

**Method — the watchdog.** The seam is already visible.
`src/sase/ace/tui/util/_stall_watchdog_monitor.py` calls `time.monotonic()` directly in
five places (`_run`, `_mark_loop_progress`, `pause`, `resume`,
`_reset_pump_state_locked`) and its `_run` method is a
`while not self._stop_event.wait(...)` daemon loop whose body is one complete poll. Take
the shape the `sase-h8.5` agent drafted before it stalled: an injectable `monotonic=`
source on the watchdog plus a `_poll_once()` seam extracted from that loop body, so
tests advance an explicit clock and call `_poll_once()` to produce episodes
deterministically. A production change to introduce the seam is acceptable and expected
— the original plan said so.

That structural change is what earns back the exact assertions. Two prior fixes
(`156cac833` loosened exact hitch/stall counts to `>= 1`; `b473a10d0` waited for
balanced pairs) treated the symptom, and the health store recorded
`test_watchdog_keeps_hitch_and_stall_state_machines_independent` failing on a head that
already contained `156cac833`. Do not widen a tolerance a third time. Restore `== 1`
counts and full event ordering.

**Method — the budget node.** `sase-h8.7` cleared this node's blocker: its F6 cascade is
gone and
`SASE_TEST_SELECTION_HEALTH_DISABLED=1 pytest -p no:randomly tests/test_contract_manifest.py::test_contract_set_serial_runtime_stays_within_budget`
now passes where it previously asserted `1 == 0`, so the harness can finally falsify a
fix here. Two normalization attempts already exist — the `budget` phase of
`plans:202608/test_selection_landing.md` and `08d0e0476` (reshaped CPU probe). Note what
`sase-h8.5` established before stalling: all 19 of this node's recorded full-run
failures **predate** `08d0e0476`, verified with `git merge-base --is-ancestor`, so that
guard is already off elapsed wall clock and asserts normalized child CPU. Its fix is
unfalsified rather than confirmed-broken. Establish which of those is true under the
harness before changing it; if the measurement shows the normalization holds, say so
with evidence and change nothing rather than adding headroom a third time.

**Acceptance.** Under `just test-contention` restricted to the watchdog file, the full
repeat count passes with zero failures in the tally. Every fixed node's assertions are
independent of host load — no `>= N` count that was `== N` before, no bare real-elapsed
threshold. At least one test still exercises the real timer path end to end.
`just check` green.

**Watch out for.** The watchdog exists to detect real event-loop stalls in production.
Do not reach determinism by making it untestable against real time. Keep the real-timer
coverage, and make sure the injectable source defaults to `time.monotonic` so production
behavior is unchanged.

## Phase `residue`: Fix the four nodes that failed the exit criterion

**Scope.** The tally `sase-h8.9` recorded at `c902dd71c` (artifact
`.pytest_cache/sase-contention/repeat-01.json` in that agent's workspace; 4 failed /
27624 passed / 10 skipped in 1230.75s):

- `tests/ace/tui/test_artifacts_files_loading.py::test_first_page_paints_before_full_extension`
  — contains `release_full.wait(timeout=5)`, a fixed real-wall-clock ceiling on a
  `threading.Event`. This is the same shape `sase-h8.7` fixed in
  `tests/fakey/harness.py`, where the finding was that the ceiling is a deadlock
  diagnostic, not a speed assertion.
- `tests/ace/tui/test_residual_freeze_soak.py::test_lowered_threshold_soak_keeps_fixed_paths_responsive`
  — a responsiveness soak against a lowered threshold; F2.
- `tests/test_clan_summary_script_failures.py::test_timed_out_summary_script_never_blocks_launch`
  — asserts a 0.3s subprocess timeout fires against a script that sleeps 60s. Under 13x
  oversubscription the subprocess may not reach its own `sys.stderr.flush()` inside the
  window; F2.
- `tests/ace/tui/test_xprompt_browser_jump.py::test_apostrophe_enters_jump_mode_with_hints_skipping_headers`
  — a single `await page.pause()` before asserting rendered option text; F1 off-pump
  settle gap.

**Measured and out of scope: the visual-lane node.** The epic bead carries a note from
`sase-h7.13`'s land agent measuring
`tests/ace/tui/visual/test_ace_png_snapshots_agents_slow_tools.py::test_agents_slow_tool_calls_fold_levels_png_snapshots`
failing **3 of 3** full `just test-visual` runs at `20752def2` (one on a stashed clean
tree), with a 0.30% changed-pixel diff at `max_diff_ratio 0.0` — a content difference
under parallel execution, not renderer drift. That note argued the class is not confined
to the default lane, and no phase ever addressed it. This plan's author re-measured it:
a full `just test-visual` at HEAD `010b01a41` is green, **562 passed / 1 skipped in
64.33s**, with the node itself passing in 4.81s inside that run. One green full-lane run
does not formally falsify a 3/3 reproducer, but it is the same measurement that produced
the original finding, and it inverts it. Treat the node as fixed by an intervening
commit and do not open work on it. If `land`'s criterion 3 run disagrees, it comes back
into scope there.

**Method.** Classify each node into the triage table's families before touching it, then
fix at the mechanism. Use `wait_for` (`src/sase/ace/testing/wait.py`) for the F1 node
and the conventions `clock` establishes for the F2 ones — do not invent a third
wall-clock idiom alongside `clock`'s and `tests/fakey/harness.py`'s
`LOAD_TOLERANT_TIMEOUT`. That is why this phase depends on `clock`.

**Acceptance.** Each node shown failing before and passing after: under the contention
harness where it reproduces, or under an injected delay at the identified boundary where
it does not, which is the falsification standard `sase-h8.4` and `sase-h8.6` both met. A
`just test-contention` soak restricted to the affected files is green across the full
repeat count. `just check` green.

**Watch out for.** A bounded wait that polls for a condition already true when the test
starts verifies nothing. Where the end state is indistinguishable from the initial
state, wait on a transition, not a value — `sase-h8.4` caught two unsound predicates
this way and only found them because it tested under an injected delay.

## Phase `gate-gaps`: Close the wait-idiom gate gaps

**Scope.** `tools/check_test_wait_helpers` currently walks two roots (`tests/ace/tui`,
`tests/fakey`) and flags AST function definitions named exactly `_wait_until`. Three
classes of the very idiom the epic retired slip past it.

1. **A sixth surviving copy.** `tests/test_agent_group_revival_e2e.py:409` defines
   `async def _wait_until(predicate, *, timeout=5.0)` — the same raw-pilot bounded wait
   `sase-h8.2` retired in four other files and `sase-h8.3` found a fifth of. It lives in
   `tests/`, outside both scanned roots. Retire it onto `wait_for` and widen the check's
   roots so the next one cannot hide in the same way.

2. **Inline bounded-wait loops.** The check matches a function _name_, so the same wait
   written inline is invisible. Two live examples landed during and after the epic:
   `tests/ace/tui/test_custom_gate_modal.py:259` (from `010b01a41`, the newest commit on
   master) and `tests/ace/tui/test_notification_plan_gate.py:117,156,181` (from
   `20752def2` / `e1da6d1b7`). Both are `for _ in range(N): await pilot.pause()` with a
   condition-and-break — a hand-rolled `wait_for` with a frame-count budget instead of a
   deadline. Detect that shape and migrate these call sites onto `wait_for`.

3. **Raw ACE panel injections.** `sase-h8.6` filed this as a follow-up and `sase-h8.8`
   did not implement it: the check should flag raw `panel.update(Text(...))` writes into
   app-derived ACE panels (`#agent-prompt-panel` and friends) and point at
   `sase.ace.testing.set_agent_prompt_document`, the settle-verified injector that
   suppresses the competing repaint.

**Method.** Extend the existing AST walker rather than bolting on a regex pass; it
already parses every file it scans. Keep the diagnostics naming the supported
replacement, which the current tool does well. Every new detection needs a test in
`tests/test_check_test_wait_helpers_tool.py` covering both a positive and a
deliberately-excluded negative — a `for` loop that presses a button N times
(`tests/ace/tui/test_typed_input_form.py:200`) is a legitimate repetition, not a wait,
and must not be flagged.

**Acceptance.** `tools/check_test_wait_helpers` exits non-zero on each of the three
shapes before the migration and zero after it, with every reported call site migrated
rather than suppressed. The migrated tests pass under the contention harness.
`just check` green.

**Watch out for.** This check runs in every agent's `just check`. A false positive on a
legitimate loop costs every agent on the repo, so bias the inline-loop detector toward
precision: require the loop body to contain both a pause-shaped await and a conditional
break before flagging.

## Phase `land`: Meet the exit criterion and close

**Exit criterion.** The four criteria `sase-h8`'s plan set, unchanged, on the combined
tree:

1. `just test-contention` at the baseline pinning and worker count, repeated, with
   **zero** failures in the tally.
2. `just check-full` green.
3. `just test-visual` green. Already met once at HEAD `010b01a41` (562 passed, 1
   skipped, 64.33s) before this plan's phases ran; re-run it on the combined tree to
   confirm the phases did not disturb the PNG lane, which is what the criterion is for.
4. The flake gate passing against `tests/reproducible_flake_baseline.txt`, which must
   still contain no ACE node and no node this epic was scoped to fix.

Note on criterion 4: the gate passes today only vacuously — it reports "0 current, 0
allowed" because no full-lane record has landed after the baseline's
`effective-after: 2026-08-08T14:10:42Z` marker. Confirm it is judging real post-baseline
records before treating it as met, or say plainly in the close note that it is not yet
exercised.

**Then, in order.**

- File genuinely distinct residue with `/sase_new_task`, naming the proposing bead.
  Known candidates the land agent's verification identified, all needing a
  duplicate-check first: (a) `sase-h8.4`'s residual — the two
  `test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_*` nodes where F1 was
  falsified, instrumentation landed, and the residual work is confirming the raiser once
  it recurs; (b) `sase-h8.3`'s second `reproducible_flake_nodeids` false-positive mode —
  a deterministic break on master is still indistinguishable from a flake, since
  different workspaces hit it with disjoint change sets, and `sase-h8.8` handled only
  the catastrophic-run threshold. Do not file the six `ff0b765a4` gate nodes, the Muse
  doctor node, or the `test_content_layout.py` schema node: all three were verified
  fixed on master at HEAD. Do not file the `sase-e2` bead-lock node; it is already
  tracked there.
- Close `sase-ct` with a note that states, in this order: the root cause of the _class_
  (no reproducer and no gate, so remediation was driven by whoever tripped over a node
  next and closure was a judgment call); the mechanism families and which phase fixed
  each; the before/after red rate; and the enforced criterion that now replaces
  hand-adjudication. Reference `just test-contention` and
  `tests/reproducible_flake_baseline.txt` by name so the next reader can re-run the
  measurement.
- Close `sase-h8` with `sase bead close sase-h8 --note "..."`.
- Run `just symvision` and remove the stale epic-symbol whitelist entries and unused
  code it reports — those entries expire when the epic closes.
- Set `status: done` in the frontmatter of **both** plan files: this one and
  `plans:202608/parallel_suite_flake_class.md`.

**Watch out for.** `sase-ct` has been closed six times and reopened six times, and
`sase-h8.5` shows what a phase closed `done` without landing looks like from the
outside: indistinguishable from a real closure until someone reads the tree. If
criterion 1 fails again, do not close `sase-ct` — report the tally and leave it open. If
a criterion cannot be run at all, say which and why rather than declaring it met. Verify
against the source and the commits, not against the closing notes.
