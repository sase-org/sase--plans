---
tier: tale
title: Land durable Agent CLI update history with configured-timezone integration
goal:
  Close epic sase-el only after its history panel follows the configured-timezone display contract and every phase
  follow-up has a recorded outcome.
proposed_by: bbugyi200.athena.sase-el.land
bead: sase-el
create_time: 2026-08-03 10:43:39
status: done
---

- **PARENT:**
  [202608/agent_cli_update_history.md](https://github.com/sase-org/sase--plans/blob/main/202608/agent_cli_update_history.md)
- **BEAD:** [sase-el](https://github.com/sase-org/sase--beads/blob/main/pages/sase-el/README.md)

# Land `sase-el` with configured-timezone integration

## Goal

Finish epic `sase-el` by integrating its Agent CLI history renderer with the configured-timezone display contract that
landed while the epic was in progress, independently revalidating the completed feature, recording every proposed
follow-up outcome, closing the epic normally, running the post-close Symvision cleanup, and marking the canonical epic
plan done.

## Audit baseline

The land audit established the following facts and they should remain the basis for implementation:

- `sase-el.1` through `sase-el.4` are closed with resolution `done`. The epic commits are `55eb24331` (journal),
  `e4ad93916` (load/config/session plumbing), `a086b0f30` (renderer/scope toggle), and `d55db39c9` (docs/help/visual
  goldens).
- The journal implementation centralizes recording in `execute_agent_cli_updates`, labels all three supported trigger
  paths, filters no-op runs, isolates append failures, bounds/rotates JSONL storage, and tolerates malformed records.
  ACE reads it only in the existing off-thread Updates load worker and renders from the in-memory load result, so the
  keystroke path performs no disk I/O.
- Config defaults/schema, help, ACE/provider/configuration docs, and the selected-CLI, all-CLI, and empty visual
  snapshots are present. Before this plan, 80 focused agent-CLI/history tests passed and the three history PNG tests
  passed exact comparison.
- The checkout has no ChangeSpec/PR and `HEAD` equals `origin/master`. Review of every non-epic commit after
  `55eb24331^` found one required integration: commits `2c70516`, `c449ce2`, and `f0e562b` established
  `sase.core.time.format_local` as the configured-timezone display path, but the newly added
  `plugins_browser_agent_clis_history._relative_time` still calls bare `datetime.fromtimestamp()` for records at least
  seven days old. On a host whose timezone differs from SASE's configured timezone, the history panel therefore shows
  the wrong absolute wall time.
- Three phase notes proposed unrelated follow-ups. The lock-timeout reports from `sase-el.1`, `sase-el.2`, and
  `sase-el.3` are one semantic duplicate of in-progress task `sase-e2`. The separate `sase-el.3` @-prefix directory
  drilldown report is a recurrence of canceled task `sase-ea`. The active-epic audit found no credible causal owner for
  either flake.

## Phase 1: Integrate configured-timezone rendering

1. In `src/sase/ace/tui/modals/plugins_browser_agent_clis_history.py`, remove the direct `datetime` dependency and
   format the absolute branch of `_relative_time` through `sase.core.time.format_local(epoch, "%b %d %H:%M")`. Preserve
   all relative thresholds, future-clock handling, and output shape.
2. Strengthen `tests/ace/tui/test_plugins_browser_pane_agent_clis_history.py` so the absolute-age case runs under the
   `tz_divergence` fixture (configured `America/New_York`, host `UTC`) and asserts the explicit configured-timezone wall
   time, not an expectation calculated with the same helper or the host clock. Keep the existing 30-second, 90-second,
   two-hour, two-day, eight-day, and future-clock boundary coverage.
3. Search the epic's source paths for any remaining bare system-clock display conversion. Production journal creation
   may continue to use `datetime.now(get_timezone())`; the prohibition is against argument-less display conversion.

## Phase 2: Revalidate the complete epic

1. Run the focused journal, execution, pane-plumbing, renderer, and scope-toggle tests:

   ```bash
   .venv/bin/python -m pytest tests/agent_clis \
     tests/ace/tui/test_plugins_browser_pane_agent_clis.py \
     tests/ace/tui/test_plugins_browser_pane_agent_clis_history.py -q
   ```

2. Run the three committed Agent CLI history PNG cases with `just test-visual -k agent_clis_history`; do not update
   goldens unless the configured-timezone fix intentionally changes a pinned fixture, and inspect any unexpected diff.
3. Run the mandatory repository gate with `just check`. If an unrelated contention flake recurs, preserve its exact
   evidence for the already identified task rather than weakening the test or treating the epic as incomplete; any
   deterministic failure caused by these changes remains in scope and must be fixed before landing.

## Phase 3: Land and close

This is the final phase and must be completed only after phases 1-2 are green.

1. Record the phase proposals through the already-invoked `/sase_new_task` duplicate workflow:
   - Add one `sase-el.land` corroboration to `sase-e2` consolidating the independent reproductions from proposing beads
     `sase-el.1`, `sase-el.2`, and `sase-el.3`, including their full-suite failures and immediate isolated passes. Do
     not create a duplicate task.
   - Add one corroboration to `sase-ea` identifying proposing bead `sase-el.3` and its full-suite @-prefix directory
     drilldown failure followed by an immediate isolated pass. The first valid corroboration should promote the canceled
     task back to `ready`; do not create a duplicate task.
2. Build a close note that records: every phase/commit and source area reviewed; the focused, visual, and full-gate
   results; the post-start commit audit and configured-timezone integration; why the three lock-timeout proposals were
   consolidated into `sase-e2`; why the directory proposal corroborated `sase-ea`; and that no proposal was silently
   declined.
3. Close normally with `sase bead close sase-el --note "<audit note>"`. Do not use `--force`; all four phases are
   already complete, so a rejection must be investigated and resolved deliberately.
4. After the close succeeds, run `just symvision`. If it reports stale `sase-el` whitelist entries or unused symbols,
   first read `symvision.md` through `/sase_memory_read`, then remove/fix what it reports and rerun Symvision until
   clean. Confirm no `sase-el` whitelist entry remains.
5. Use `/sase_repo` to open the plans sidecar, change only the canonical `plans:202608/agent_cli_update_history.md`
   frontmatter from `status: wip` to `status: done`, and verify the plan diff. This status update happens last, after
   the epic is actually closed and post-close Symvision is green.

## Acceptance criteria

- Absolute Agent CLI history timestamps render in SASE's configured timezone even when the host timezone differs.
- Focused behavior tests, all three history PNG snapshots, `just check`, and post-close `just symvision` pass.
- `sase-e2` contains the consolidated `sase-el.1`/`.2`/`.3` corroboration, and `sase-ea` contains the distinct
  `sase-el.3` corroboration and is ready for triage.
- `sase-el` is closed with resolution `done` and a complete audit/follow-up note; no force close is used.
- The canonical epic plan frontmatter reads `status: done`.
