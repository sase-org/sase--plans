---
tier: epic
title: Make Files spread page-anchor navigation converge on CI (finish sase-1a7)
goal: Files spread ctrl+j and cycle_card navigation always converges on the live page
  anchor, including on CI's Python 3.12 runner with Textual 8.2.8. The test_deck_spread_pilot
  ctrl+j node and every test_files_anchor_ownership node stay green in scheduled Full
  CI. sase-1a7 closes done, and the interrupted sase-1ou landing resumes.
parent_bead: sase-1ou
decisions:
  ci_proof_runs:
    ask: Should the land agent require two post-fix scheduled Full CI runs, not one,
      before closing sase-1a7?
    choices:
      two: Waits about 12 hours; a surviving 25%-per-run race passes about 56% of
        the time
      one: Waits about 6 hours; a surviving 25%-per-run race passes about 75% of the
        time
    default: two
    why: One green run already failed to distinguish fixed from unfixed after 691ef46153
    answer: two
phases:
- id: files-landing
  title: Remove the stall in Files spread navigation and make CI timeouts diagnosable
  size: medium
  depends_on: []
  description: 'files-landing: rebuild the CI-like Python 3.12 / Textual 8.2.8 environment,
    prove with a deterministic real-widget test that pending Files navigation stalls
    once its bounded refresh-retry chain runs out after the last layout publish, make
    navigation resume on scroll-extent change instead, add state to the anchor-wait
    timeout message, and prove it with isolated and loaded loops in both Textual versions.'
proposed_by: bbugyi200.athena.sase-1ou.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-28 17:04:13
status: wip
bead_id: sase-1ou.6
---

- **PROMPT:** [prompts/202610/files_spread_ci_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/files_spread_ci_landing.md)
- **PARENT:** [202610/backlog_epic_2_top_five_task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202610/backlog_epic_2_top_five_task_beads.md)
- **BEAD:** [sase-1ou.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ou/sase-1ou.6.md)

# Make Files spread page-anchor navigation converge on CI (finish sase-1a7)

## Why this child epic exists

This is a child epic of `sase-1ou`, the epic plan
`plan:202610/backlog_epic_2_top_five_task_beads.md` ("Task-bead backlog, epic 2"). Read
that plan with `sase artifact read`. Read `sase-1ou`, its phase `sase-1ou.4` and task
`sase-1a7` with `sase bead read <id> -r "<why>"` before starting.

The `sase-1ou` land agent verified everything else in that epic on 2026-10-28 at sase
master `86c3250e69`:

- It closed `sase-12f`, `sase-18t` and `sase-1p6`.
- `sase-1ot` stays open, because the owner rejected the purge gate.
- It filed or corroborated every follow-up (see the `LANDING TRIAGE` note on
  `sase-1ou`).

Two pieces of that landing commit after its turn and exist before this plan launches:

- the reconciled `tests/reproducible_flake_baseline.txt` in sase;
- the sase-core change to `crates/sase_core/src/tool_run/store/connection.rs` and
  `.../store/tests/contention.rs`, which fixed sase-1p6.

`sase-1a7` could not be closed. The fix `691ef46153` (phase `sase-1ou.4`, "resolve Files
ctrl+j page-anchor scroll race") fixed two real races that were reproduced
deterministically. But the same failure signature came back in CI after the fix:

- Scheduled Full CI `40658194643` at `ac23f63ee4` (2026-10-28) contains the fix. It
  failed
  `tests/ace/tui/widgets/decks/test_files_anchor_ownership.py::test_hidden_paged_restore_preserves_spread_scroll`
  in the **Python 3.12** job only.
- The failure was
  `wait_for(wait_for_anchor_scroll.<locals>.settled_on_live_target) timed out after 5.0s`,
  at the test's first landing wait after `panel.cycle_card(1)` (line 81). That is the
  same production path as sase-1a7's node
  `tests/ace/tui/widgets/decks/test_deck_spread_pilot.py::test_files_ctrl_j_scrolls_page_anchor_to_top`.
- CI installs **Textual 8.2.8**. `uv.lock` pins 8.0.1; that drift is task `sase-1cs`,
  which this epic does not fix.
- The original node passed four post-fix scheduled runs (`40580694556`, `40614969473`,
  `40658194643`, `40728370252`). Before the fix it failed in at least `40288313398` and
  `40259042876` out of ten scheduled runs. The race was always Python-3.12-only in CI.
- Local evidence is all green. The lander ran:
  - 74 local full-lane records with 0 failures for each node;
  - 48 isolated node pairs on a Python 3.12 + Textual 8.2.8 venv;
  - 5 runs of `tests/ace/tui/widgets/decks/` at `-n 6` under 48 CPU spinners;
  - 3 runs of that directory pinned to 4 CPUs (`taskset -c 0-3`, `-n 4`).

  None of it reproduced the race.

The fix cannot be proven locally; it needs scheduled Full CI after it lands. A tale
coder cannot wait on CI for its own commit, but this epic's land agent can. That is why
this is a child epic and not a tale.

## Constraints (inherited from the parent plan)

- Never weaken an assertion, raise a timeout, cap or budget (keep the 5.0 s
  `wait_for_anchor_scroll`), add a symvision pragma, `xfail`/skip a node, or add a
  baseline entry without an open bead.
- Fix production code under `src/sase/ace/tui/widgets/decks/`. A test-only change does
  not close sase-1a7.
- Read the `tui` memory note (`/sase_memory_read`) before any TUI or pilot-test work,
  and `lint_and_test` before finishing. Do not edit `sase/memory/**`.
- Run `sase tool run check`. `just check-full` is explicit-only.
- Phase workers never create beads. They append `PROPOSED FOLLOW-UP:` notes to their
  phase bead.

## Phase `files-landing`

**Code to read first.**

- `src/sase/ace/tui/widgets/decks/_panel_spread_files_scroll.py`: `_cycle_files_spread`,
  `_schedule_files_scroll`, `_apply_files_scroll`, `_confirm_files_scroll` and
  `_reschedule_files_scroll_after_layout`. This code was split out of `panel_spread.py`
  by `f8a8b5a4f6` after the fix landed. Confirm the split preserved `691ef46153`'s
  behavior.
- `src/sase/ace/tui/widgets/decks/files_spread.py`: `_publish_section_layout`, which
  calls `_reschedule_files_scroll_after_layout`.
- `src/sase/ace/tui/widgets/decks/panel_interaction.py`: the scroll watcher hold.
- `tests/ace/tui/widgets/decks/_deck_settle.py` and
  `tests/ace/tui/widgets/decks/test_files_anchor_ownership.py`.

**Working hypothesis (prove or refute it first).**

1. The navigation chain is still bounded. `_schedule_files_scroll` returns once
   `_files_scroll_retry >= 8` or `attempt >= 8`. `_apply_files_scroll` retries while the
   anchor row is `None` or `max_scroll_y < row`. `_confirm_files_scroll` retries while
   the landing is not yet confirmed.
2. When the chain gives up, `_files_pending_scroll_index` stays set, but nothing
   reschedules it until the next `_publish_section_layout`.
3. Suppose the spread's scroll extent (virtual size and `max_scroll_y`, which grow when
   `enable_section_layout_reserve()` takes effect) or the landing settles more than
   about 8 refreshes after the last layout publish. Then navigation stalls permanently,
   and the anchor wait times out.
4. A slower runner or a different Textual version (8.2.8 coalesces or defers layout and
   virtual-size updates differently) would make that ordering more likely. That would
   explain a CI-only, Python-3.12-only failure.

**Steps.**

1. **Rebuild the CI-like environment.**
   - Create a scratch Python 3.12 venv outside the repository.
   - Install `uv export --frozen --no-hashes --all-extras --no-emit-project` with
     `textual==8.2.8` and without `sase-core-rs`.
   - Copy the workspace venv's abi3 `sase_core_rs` package and its dist-info into the
     new venv.
   - Add `.pth` entries for the checkout's `src/` and root.
   - Check that `python -c "import textual, sase_core_rs, sase"` reports 8.2.8.
   - Also try reproduction routes the lander did not: random test order,
     `tests/ace/tui/` at `-n 4` pinned to 4 CPUs, and the full non-visual suite on the
     same pinned set. CI runs the decks tests after thousands of other tests.
2. **Prove the stall deterministically** with a real-widget test in
   `test_files_anchor_ownership.py` (or a sibling file):
   - Hold the Files scroll extent (or the anchor) short of the target for more than the
     retry bound after the last layout publish. Use controlled callback ordering, as the
     existing file does, not sleeps.
   - Then let the extent grow. Assert that navigation lands on the live anchor.
   - It must fail on the current code and pass on the fix.
   - If the hypothesis is refuted, find the real stall the same way. Record the evidence
     in a phase note before changing course.
3. **Fix it in production code.** Resume pending Files navigation when the Files scroll
   container's extent changes. For example, watch its `virtual_size` / `max_scroll_y` or
   its resize. That way convergence no longer depends on a fixed number of
   `call_after_refresh` hops. Keep a bounded backstop only if it re-arms on the next
   extent change. Keep:
   - the superseding generation and pending-index semantics from `691ef46153`;
   - the hidden-paged guard;
   - unmount clearing.
4. **Make the next CI failure diagnosable.** When `wait_for_anchor_scroll` times out,
   its message (or the deck tests' failure output) should report:
   - the last `scroll_y`, live target and `max_scroll_y`;
   - the pending index, retry/attempt counters, generation and confirmed flag;
   - the Textual version.

   It must not change any timeout or success condition.

5. **Prove it.** Record before/after counts for each of these in your close note:
   - the deterministic regression;
   - at least 50 isolated runs of both nodes, in the default venv (Textual 8.0.1) and in
     the 3.12 / 8.2.8 venv;
   - 5 runs of `tests/ace/tui/widgets/decks/` at `-n 6` under a concurrent CPU load, in
     both venvs;
   - `sase tool run check`.

   No baseline entry exists for either node; add none. Leave sase-1a7 open for the land
   agent, which needs post-commit CI.

## Landing (this child epic's land agent)

1. **Verify the phase.** Verify the phase's notes, its commit on sase master, and the
   regression test. Check that no timeout, assertion or baseline was weakened.
2. **Wait for CI proof.** Use `/sase_monitor` to wait (sleep plus
   `gh run list --workflow "Full CI"`) for scheduled Full CI runs whose head contains
   the phase commit.

   > [!decision] ci_proof_runs = two Require two such runs, consecutive or not. In
   > neither run may any Python job fail
   > `test_deck_spread_pilot.py::test_files_ctrl_j_scrolls_page_anchor_to_top` or any
   > `test_files_anchor_ownership.py` node.

   > [!decision] ci_proof_runs = one Require one such run with the same condition.

   Other reds owned elsewhere do not block (for example `sase-1cs`'s Textual drift,
   `sase-1p5`, `sase-1pf`, `sase-1pg`). If a watched node fails, record the new
   diagnostic output on `sase-1a7` and the phase bead. Reopen the phase, and rerun
   `sase bead work` on this epic rather than closing.

3. **Close sase-1a7** `done`, with the CI run ids and the phase evidence. Then close the
   reopened parent phase `sase-1ou.4` with a note naming this epic and sase-1a7's
   closure. Close this child epic normally. Run `just symvision`, and set `status: done`
   in this plan file.
4. **Resume the interrupted `sase-1ou` landing** (`parent_bead`):
   - Re-read `sase-1ou`, its `LANDING TRIAGE` note and its plan's Landing section.
   - Re-check the five task beads. sase-12f and sase-18t are closed done. sase-1a7 is
     closed by step 3. sase-1ot stays open with the owner's rejection recorded, as that
     plan allows.
   - Re-run `.venv/bin/python tools/selection_health --fail-on-new-flake`. It must not
     exit 2, and every exceedance must still be owned.
   - Run `sase tool run check`.
   - Retire any `sase bead epic-symbols sase-1ou` entries. Close `sase-1ou` with
     `sase bead close sase-1ou --note "<verification>"`, run `just symvision`, and set
     `status: done` in `plan:202610/backlog_epic_2_top_five_task_beads.md`.
   - `sase-1ou` has no parent bead.
   - As the last action, run that plan's Landing step 5 ("Start the next backlog epic").
     Follow its chaining rules: re-triage, the `dormant_flakes = close` sweep, rank, and
     propose the next epic with `/sase_plan`. `sase plan propose` ends the turn.
