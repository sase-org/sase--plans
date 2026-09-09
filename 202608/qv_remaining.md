---
tier: epic
status: done
title: Finish monitor-status landing integration
goal: "Dismissed monitors with a custom stop label resolve waits from the bundle's
  recorded pair, and the family-conversation monitor PNG matches the pair-accent
  rendering this epic already shipped.

  "
parent_bead: sase-qv
phases:
  - id: waits
    title: Honor recorded stop status in dismissed-archive wait resolution
    depends_on: []
    size: small
    description: "waits: classify dismissed-bundle monitor outcomes from the recorded
      monitor_stop_status instead of only the literal MONITORED default.

      "
  - id: goldens
    title: Refresh the remaining monitor golden and re-check later surfaces
    depends_on: []
    size: small
    description: "goldens: regenerate the stale family-conversation monitor PNG if it
      still mismatches, confirm the completion spec, and wire any later-landed surface
      that should show the pair.

      "
proposed_by: bbugyi200.athena.sase-qv.land
bead_id: sase-qv.8
create_time: 2026-09-09 19:51:19
---

- **PROMPT:**
  [prompts/202608/qv_remaining.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/qv_remaining.md)
- **BEAD:**
  [sase-qv.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-qv/sase-qv.8.md)

# Plan: Finish monitor-status landing integration

This is leftover landing work for epic `sase-qv` (Required custom monitor statuses with
deterministic pair colors). All seven original phases shipped. A previous land pass
closed the epic and marked `plan:202608/monitor_custom_statuses.md` `status: done` after
claiming this integration was committed. Those commits are not on `master`. This child
epic finishes that work; its land agent then resumes `sase-qv` via `parent_bead`.

Do **not** re-implement the original epic. The contract module, required `-s`/`-S`
flags, status-pair plumbing, Agents-tab coloring, family mirroring, Procs-tab chip, and
guidance already exist. Do **not** re-file the phase follow-ups: `sase-r4`, `sase-r5`,
and the `+1`s on `sase-p9` / `sase-oz` / `sase-oe` are already in the bead store.

## Already on master (do not redo)

- `src/sase/monitor_status.py` owns the 20-character clamp, `MonitorStatusPair`, the
  12-color OKLCH accent band, and the style/glyph/effective-label rule.
- `src/sase/palette_hash.py` is shared by project, artifact-tab, and monitor-pair
  accents.
- `sase monitor start` requires `-s/--start-status` and `-S/--stop-status` (handler exit
  2 with teaching text; `StartMonitorRequest` clamps in `__post_init__`).
- Both labels ride the wire and filesystem loaders onto `Agent`, `RunningAgentInfo`,
  `AgentListEntry`, and the mobile summary.
- `date_anchor_time` keys monitor rows on `monitor_state_is_terminal`.
- Coloring is live on the agent list (`monitor_status_presentation`), the prompt panel
  MONITOR section, family containers (`_agent_status_apply` /
  `_agent_status_family_planner`), the Procs tab `MonitorStatusChip`, `sase agent list`,
  and monitor list/show/markdown/JSON (schema v2).
- Guidance is in `sase/memory/build_and_run.md`, `AGENTS.md` plus the four provider
  shims, `docs/monitors.md`, `docs/ace.md`, and the `sase_monitor` skill.
- `tests/completion/snapshots/cli_spec.json` already carries the post-qv.2
  `sase monitor start` `description_digest` `076adb65014057c7` (rewritten by a later
  `tmux-agent` commit). Confirm it; do not rewrite it if the snapshot tests are green.

## waits: dismissed-archive wait resolution

`src/sase/core/dismissed_agent_completion.py` `_archived_outcome_from_bundle` still
treats only the literal `DEFAULT_MONITOR_STOP_STATUS` (`MONITORED`) as a monitor stop
label. Once custom stop labels are mandatory, that branch is dead for every new monitor:
a dismissed `TESTED` / `SLEPT` / `DEPLOYED` row falls through to fail-closed wait
resolution even when `monitor_state` is `completed`.

The bundle already persists the recorded pair. `Agent.monitor_stop_status` is an init
field, and `Agent.to_bundle_dict` serializes every init field that is not on its
denylist, so production dismissed JSON carries `monitor_stop_status`.

### Change

In `_archived_outcome_from_bundle`, after the known-status table:

1. If `status` is not a string, fail closed (unchanged).
2. Read the bundle's `monitor_stop_status`. Clamp it with
   `clamp_monitor_status_or_default(..., default=DEFAULT_MONITOR_STOP_STATUS)` so a
   missing/invalid historical field still projects to `MONITORED`.
3. Compare `status` to that recorded stop label case-insensitively (strip + upper).
   Match → `effective_done_outcome` with `outcome=MONITOR_OUTCOME` and the bundle's
   `monitor_state`, exactly as today's `MONITORED` branch does.
4. Mismatch or an unrecorded custom label → `None` (fail closed). Arbitrary user
   statuses must not become monitor successes.

Keep the known-status table (`DONE`, `FAILED`, `EPIC APPROVED`, …) first so non-monitor
dismissed rows do not consult the pair.

Import `clamp_monitor_status_or_default` from `sase.monitor_status`. The existing
`DEFAULT_MONITOR_STOP_STATUS` import from `sase.monitor_state` can stay (it re-exports
the contract constant) or move next to the clamp import; do not duplicate the literal
`"MONITORED"`.

### Tests

`tests/test_dismissed_agent_completion.py` plus `tests/_dismissed_completion_helpers.py`
(`write_dismissed_completion(..., extra=)` already merges arbitrary bundle fields).

Keep `test_archived_default_monitor_status_uses_monitor_state` (status `MONITORED`, no
recorded pair → still uses `monitor_state`). Add:

- `test_archived_recorded_stop_status_uses_monitor_state` — status `TESTED`,
  `extra={"monitor_stop_status": "TESTED", "monitor_state": ...}` for the same five
  `monitor_state` cases (`completed` / `stopped` resolve; `failed` / `timeout` / `None`
  fail). Optionally one mixed-case `tested`/`TESTED` row so the compare is
  case-insensitive.
- Split today's `test_archived_custom_monitor_stop_status_remains_fail_closed` into two
  fail-closed cases: (a) status `SLEPT` with no `monitor_stop_status` (unrecorded custom
  label), (b) status `SLEPT` with `monitor_stop_status: "TESTED"` (mismatch). Both must
  leave `archived_completion is None` / unresolved.

Do not broaden wait resolution past this recorded-pair match.

## goldens: PNG, completion spec, later surfaces

### Family-conversation monitor PNG

`tests/ace/tui/visual/snapshots/png/agents_family_conversation_monitor_120x40.png` was
last rewritten by `sase-qv.4` (`91c432385`) and is still behind the rendering that
commit plus `sase-qv.5` (`18dcf6b8d`) shipped. Reproduced at `4950f060c` by the
`sase-qw` land agent: the golden shows `visual-family-root (MONITORED)` in the default
status color; the code renders `(MONITORED ✓)` in the pair accent on the mirrored family
container. Isolated failure:

```bash
just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_agents_family_panel.py::test_family_conversation_monitor_phase_png_snapshot
```

If that node is still red, regenerate **only that golden**:

```bash
just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_agents_family_panel.py::test_family_conversation_monitor_phase_png_snapshot --sase-update-visual-snapshots
```

Then confirm both monitor visual nodes pass:

- `test_family_conversation_monitor_phase_png_snapshot`
- `test_settled_monitor_lane_badge_png_snapshot`

Do **not** rebaseline the rest of `just test-visual`. Those 36 unrelated stale goldens
are ready task `sase-r5`. After this golden is green, leave a note on open task
`sase-q1` that its remaining node now matches master; do not close `sase-q1` (owner
decision).

### Completion spec

Run `tests/completion/test_snapshot.py`. The checked-in digest for `sase monitor start`
is already `076adb65014057c7`. If both snapshot tests pass, make no spec change. If they
fail, `just sync-completion-spec` / `tools/sync_completion_spec --write` and commit only
the resulting `tests/completion/snapshots/cli_spec.json` drift that this epic still
owns.

### Later-landed surfaces

Commits since `3e3c93774` that are not this epic's stitches include tmux Agent, Launch
Control, the Update panel, Logs jump, Memory panel, and filter-bar persistence.
Spot-check already shows tmux Agent / Launch Control do not render a monitor status
token. Still grep those trees (and anything newer than this plan) for
`sase monitor start`, `StartMonitorRequest`, `start_monitor`, and agent-status rendering
that hard-codes `MONITORING` / `MONITORED` or omits `monitor_status_presentation`. Wire
any surface that should now show the pair; if none do, say so in the close note and make
no drive-by edits.

## Verification

`just install` first in the ephemeral workspace. After each phase, `just check`. The
`goldens` phase also runs the two monitor visual nodes above. Do not run
`just check-full` here; the resumed `sase-qv` land agent owns that.

## Out of scope

- Closing `sase-qv`, the symvision whitelist pass, and flipping
  `monitor_custom_statuses.md` to `status: done` (resumed land agent).
- The 36 non-monitor PNG goldens (`sase-r5`).
- The `WorkspaceOccupiedError` flake (`sase-r4`) and the other already-filed follow-ups.
- The core-floor-probe `could_not_determine` warning (noted on in-progress epic
  `sase-qx`).
- Plugin-repo callers: `sase-qv.2` already grepped `sase-github` / `sase-telegram` /
  `sase-nvim` / `sase-research-artifacts` and found none. Re-grep only if a later commit
  in those repos is in the later-surface audit.
