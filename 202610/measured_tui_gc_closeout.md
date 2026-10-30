---
tier: tale
size: medium
title: Re-land the lost graph-allocation fix and close the measured TUI GC epic
goal:
  Restore the graph-allocation phase whose commit was lost, remove the reporter's false
  delayed-heartbeat gap, and close epic sase-1nf.14.1 with an honest FAIL/INCOMPLETE
  acceptance record.
decisions:
  heartbeat_delay:
    ask:
      Should the reporter count a late heartbeat as covered when window_s and uptime
      still match?
    choices:
      accept:
        Late but fully accounted intervals up to 450 s count as covered and are reported
        as delayed
      keep:
        The 302 s cap stays, so one late heartbeat on a loaded host leaves coverage
        INCOMPLETE
    default: accept
    why:
      window_s/uptime agreement detects lost accounting; a skipped beat (600 s+) cannot
      fit 450 s
    answer: accept
proposed_by: bbugyi200.athena.sase-1nf.14.1.land
proposed_with: claude/opus
bead: sase-1nf.14.1
decided_by: auto
create_time: 2026-10-29 23:12:38
status: wip
---

- **BEAD:**
  [sase-1nf.14.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nf/sase-1nf.14.1.md)

# Plan: Re-land the lost graph-allocation fix and close the measured TUI GC epic

## Context

This is the closeout tale for epic `sase-1nf.14.1`
(`plan:202610/measured_tui_gc_fixes.md`). Its land agent verified the epic and found
work that still belongs to it. Read the epic and its notes first:
`sase bead read sase-1nf.14.1 -r "Need the land triage note and plan path"`. Read note
`LAND TRIAGE` on the epic and note `LAND CORRECTION` on `sase-1nf.14.1.3`.

Already verified, so do not redo it:

- Phases `.1` (`0189f13f7b`), `.2` (`3d70e0a81b`, `98668bf431`), `.4` (`0d11a115db`),
  `.5` (`720b2a22bc`), and `.6` (`353d833bf0`) landed as reported.
  `sase bead epic-symbols sase-1nf.14.1` reports no entries.
- Phases `.7` and `.8` closed, at the user's explicit choice, on an honest
  FAIL/INCOMPLETE acceptance matrix (`research:202610/tui_gc_fixed_capture_20261028.md`,
  `research:202610/tui_gc_fixed_soak_20261029.md`). The user asked for no further
  waiting. **Do not start a new capture, soak, monitor wait, or question gate.**
- Follow-up triage is done: new task `sase-1pv`, plus `+1` corroborations on `sase-1nc`
  and `sase-1f3`. Do not create task beads.
- Integration review is done. Only the capacity-path audit in step 1 remains.

Two defects remain, and this epic caused both:

1. **Phase `.3` graph-allocation never landed.** Its agent finished, but then crashed
   (provider-timer reuse). A workspace lease then force-reset the uncommitted changes.
   That crash chain is fixed on master by `9012f46cf9`. The bead is closed, but no ref
   contains `classify_cached_graph_needs`. The fixed-code captures therefore measured
   code **without** the top-ranked producer fix. The land agent recovered the diff by
   replaying the agent's 20 completed `edit_file` calls onto base `602ff33c1b`. It is
   stored as `file:explicit:cc1fcd3f9d4bd944a9c454b5` (sha256 `d62501a94ee8...`) and
   applies cleanly to master `c927c40860`.
2. **The reporter rejects delayed heartbeats as missing.** `HEARTBEAT_MAX_GAP_S = 302`
   in `tools/_tui_freeze_report_acceptance.py` comes from phase `.1`. Under Athena's
   load, one heartbeat per host arrived 304.094 s and 302.768 s after its predecessor.
   Each had `window_s` within 0.40 s and 0.06 s of the wall delta, and its full-GC
   counters agreed with the emitted rows, yet each was flagged
   `missing heartbeat interval`. That alone makes any four-hour window on a loaded host
   INCOMPLETE. `tools/_tui_freeze_report_activity.py` repeats the same `<= 302` bound.

Before editing, read these through `/sase_memory_read`: `ace.md`, `tui.md`, `tools.md`,
`docs_writing.md`, and `lint_and_test.md`. Also read `tui_perf.md` if that note exists.
Do not edit any memory file. Keep every change inside TUI cache ownership, diagnostics
tooling, and docs, as the epic's common rules require.

## Step 1: Re-land graph-allocation

1. Audit-read the recovered diff:
   `sase artifact read file:explicit:cc1fcd3f9d4bd944a9c454b5 "Re-land lost sase-1nf.14.1.3 diff"`.
   Then apply it with
   `git apply --3way "$(sase artifact path file:explicit:cc1fcd3f9d4bd944a9c454b5)"`. It
   touches `_loading_graph.py`, `_loading_compute.py`, and `_loading_compute_merge.py`
   under `src/sase/ace/tui/actions/agents/`. It also touches
   `tests/_agents_tab_incomplete_merge_helpers.py` and
   `tests/test_agents_tab_graph_isolation.py`. If it does not apply, rebuild it by
   replaying the completed `edit_file` items (find/replace JSON in `item/*` records)
   from
   `~/.local/share/muse/sessions/.msp-view-g6/9780b2e4-4d07-4634-8ab8-26ac075796a5/journal-00000000.bin`
   onto `602ff33c1b`, then rebase that result onto master.
2. The diff adds `CachedGraphNeeds`, `classify_cached_graph_needs()` (read-only
   classification that mirrors the merge's early-return predicates), and
   `own_required_apply_snapshot()`. The new function copies only the needed roots with
   one shared memo: cached roster when the merge patches history, cached named procs
   only for an authoritative load with `proc_projection is None`, capacity only when the
   capacity merge recursion runs, and fleet rows whenever present. The worker boundary,
   the apply boundary, and the merge path all call it. Check it against every rule in
   the epic plan's "Phase graph-allocation" section: an authoritative load with a
   projection copies zero cached rows, an empty projection is authoritative, standalone
   `copy_agent_graph()` semantics are unchanged, and live mutable rows never reach a
   mutating step.
3. **Re-audit the code that landed after `602ff33c1b`.** For each case below, prove that
   a root left live (uncopied) by the classifier never reaches a step that writes rows:
   - `f843bd8100` (capacity header from machine report): `_loading_compute.py` now calls
     `refresh_runner_slot_context` twice when `snapshot.capacity_report` is set, on
     `prep.filtered_agents` and on estimate copies, with
     `capacity_agents=prep.capacity_agents`.
   - `8d41b72218` (queued sessions and clans stay QUEUED across capacity refreshes):
     added `copy_agent_rows_relinked`, and updated runner-slot and status handling.
   - `212a476786` (capacity accounting fields written onto rows).

   Also trace named-proc carryover (`merge_named_proc_agents` into
   `prep.capacity_agents`), dismissed-agent lists, relationship rebuild, and fleet
   projection. If any path can mutate a row the classifier left live, extend the
   predicate so that root is copied. Never fall back to shallow copies.

4. Add a regression for the capacity-report path. Use an authoritative load with a proc
   projection, a non-empty cached capacity roster, and an installed `capacity_report`.
   Assert that cached and capacity rows are unchanged, compared through
   `row_observation`-style snapshots, after real worker preparation. Also assert that no
   discarded cached history is copied. Keep the four recovered tests and confirm at
   least one fails when `_loading_graph.py` is temporarily reverted to the full-copy
   behavior. Restore the change afterwards.
5. Clean up symbols. After the patch, `own_prepared_apply_snapshot` may have no non-test
   caller. Resolve any Symvision finding per `symvision.md`: delete or privatize the
   symbol rather than whitelisting it. Update the docstrings in
   `_loading_apply_snapshot.py` and `_loading_compute_types.py` that name it.
6. Re-measure the producer, using the method from the phase note. Benchmark repeated
   real worker preparation at 100 and 1,000 rows, including cyclic session rows and a
   delta/partial update. Compare the old full copy (`own_prepared_apply_snapshot`
   behavior) with the new classified copy. Record clone counts, tracemalloc peak and
   retained bytes, and objects released after dropping the result, before and after
   `gc.collect()`. Also record Python, Textual, and Rich versions and the source
   revision. Put the scratch script under `/tmp` and do not check it in. Store the
   results with
   `sase artifact create -p <results.md> -l "graph-allocation re-land benchmark" -b sase-1nf.14.1.3`.
   Do not claim that all graph garbage is gone.

## Step 2: Separate delayed heartbeats from missing ones

> [!decision] heartbeat_delay = accept
>
> In `tools/_tui_freeze_report_acceptance.py`, replace the absolute cap with one named
> missing-heartbeat bound, `HEARTBEAT_MISSING_GAP_S = 1.5 * HEARTBEAT_INTERVAL_S`.
> Comment that a skipped heartbeat yields at least about twice the cadence, so this
> bound cannot bridge one. Keep the `window_s` and `uptime_s` agreement checks (within
> `HEARTBEAT_INTERVAL_TOLERANCE_S`) as the accounting-continuity witness. An interval
> that is longer than `HEARTBEAT_INTERVAL_S + HEARTBEAT_INTERVAL_TOLERANCE_S` but passes
> every check stays valid coverage. Count it as `delayed_interval_count`, with the
> longest delay, in the coverage diagnostics, the text output, and the JSON output. Bump
> the JSON schema version only if the reporter's versioning rule requires it for an
> added field. Make `tools/_tui_freeze_report_activity.py` use the same bound, imported
> from one place, so prompt-state bucketing agrees with coverage.
>
> Update `tests/test_tui_freeze_report_acceptance.py`. Each new case must fail on old
> code where applicable:
>
> - A 304.094 s interval whose `window_s` is 303.696331 counts as valid and delayed.
> - A 302.1 s interval with matching `window_s` is now valid. Rewrite
>   `test_adjacent_window_s_uses_two_second_tolerance` to test a `window_s` mismatch of
>   more than 2 s instead.
> - `test_missing_heartbeat_gap_is_not_bridged_by_matching_window_s` (a 600 s gap with
>   matching `window_s`) still fails coverage.
> - A 600 s gap whose `window_s` is about 300 fails.
> - A delayed interval inside a prompt-state bucket is grouped like an on-time one.
>
> If the runbook's reporter section describes coverage rules, add one sentence about
> delayed intervals there.

> [!decision] heartbeat_delay = keep
>
> Leave the 302 s cap and its tests unchanged. Add one sentence to the runbook's
> reporter section: an interval more than 2 s late is reported as missing even when its
> `window_s` matches, so on a loaded host the capture owner must diagnose such an
> interval from the retained `window_s` comparison.

Do not change any acceptance threshold. Do not re-run or republish the retained capture
or soak bundles: their observed spans (about 4,520 s and 7,229 s) are under four hours,
so the verdicts stay FAIL/INCOMPLETE either way.

## Step 3: Runbook clarification

In `docs/contribute/perf-runbook.md`, "Capture preflight" step 2 already says "Every
restart defines a new instance even if exec reuses the PID". After that sentence, add
one sentence: an exec-based restart, including the TUI's own re-exec (for example
`sase tui --restart-service`), keeps the PID, `/proc` start ticks, and boot ID. Any
watcher that follows a long capture must therefore key on `app_instance_id` from startup
and heartbeat rows. That is the failure that ended the `.8` soak. Follow
`docs_writing.md`, and keep the line width and style of the surrounding text.

## Step 4: Verify

Follow `lint_and_test.md`:

- Run the focused suites: `tests/test_agents_tab_graph_isolation.py`,
  `tests/test_agent_graph_copy.py`, the other loading-pipeline tests the phase note
  named (`tests/test_agents_tab_*` and `tests/ace/tui/test_*capacity*`), and
  `tests/test_tui_freeze_report_*.py`.
- Run `sase tool run check`. If it escalates, join the printed run with `/sase_monitor`
  instead of rerunning it. Do **not** run `just check-full`.
- No rendered output changes, so no PNG goldens should move. If any does, stop and
  diagnose it; do not rebaseline.
- Run `git diff --check`.

## Step 5: Close the epic (final step)

1. Append to the lost phase:
   `sase bead note sase-1nf.14.1.3 "RE-LANDED by the sase-1nf.14.1 closeout tale: <files>, <tests incl. new capacity-report regression>, benchmark <artifact ref>, check <ToolRun id/result>."`
2. Run `sase bead epic-symbols sase-1nf.14.1`. Resolve every listed entry by wiring it
   up, privatizing it, or deleting it. Re-key an entry only to a still-open bead that
   needs it. It currently lists none.
3. Close the epic normally. Never use `--force`:

   ```bash
   sase bead close sase-1nf.14.1 --note "<verification>"
   ```

   The note must say all of the following:
   - Producer and policy phases `.1`, `.2`, `.4`, `.5`, and `.6` verified at their
     commits.
   - `.3` graph-allocation re-landed in this tale from
     `file:explicit:cc1fcd3f9d4bd944a9c454b5`, with the capacity-path audit result,
     tests, and benchmark ref.
   - The heartbeat-gap fix (or the kept cap, per the decision) and the runbook sentence.
   - `sase tool run check` result.
   - Acceptance is **not** met. Fixed source `196690f6cb` (without graph-allocation)
     measured FAIL/INCOMPLETE on both hosts: idle full-GC p95 4.476/1.051 s, GC share
     12.60%/4.28%, hitch share 15.29%/0.71%, Athena VmSwap max 631,345,152 B, no
     four-hour, busy-hour, or 24-hour coverage. The user closed `.7`/`.8` on that matrix
     (gates `6faea19e`, `4cadd2a2`). Cite both research refs.
   - Follow-ups: `sase-1pv` new; `sase-1nc` and `sase-1f3` corroborated; soak_wait
     instance pinning declined as a task and handled by the runbook sentence.

4. Run `just symvision` and confirm it is clean.
5. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path printed
   by `sase bead read sase-1nf.14.1`, in the plans sidecar, for
   `plan:202610/measured_tui_gc_fixes.md`. Change only that field.
6. Handle the parent. `sase-1nf.14` is a **phase** bead ("Fix the measured GC freeze
   causes"). Its contract is to prove `sase-1f3`'s targets on restarted TUIs and to
   close `sase-1nc`. That contract is **not** met, so **do not close `sase-1nf.14`**.
   Record the blocker instead:

   ```bash
   sase bead note sase-1nf.14 "BLOCKED after child epic sase-1nf.14.1 closed: <targets still failing with numbers and refs>; the measured fixed code lacked graph-allocation, which re-landed only now; acceptance needs a fresh user-preflighted capture on the re-landed code (sase-1f3) and further producer work (sase-1nc). No phase close; the user decides whether to schedule that or to settle this phase."
   ```

   Leave `sase-1nf`, `sase-1nc`, and `sase-1f3` open. In your final response, report
   this parent blocker explicitly.
