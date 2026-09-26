---
tier: tale
title: Finish the TUI turn-surface cutover
goal:
  The TUI names and renders session turns and named procs consistently, with updated
  goldens and unchanged navigation performance.
size: medium
proposed_by: bbugyi200.athena.sase-1ab.4
bead: sase-1ab.4
create_time: 2026-09-26 15:45:12
status: wip
---

- **PARENT:**
  [202609/sase_turn_rename.md](https://github.com/sase-org/sase--plans/blob/main/202609/sase_turn_rename.md)
- **BEAD:**
  [sase-1ab.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ab/sase-1ab.4.md)

# Finish the TUI turn-surface cutover (sase-1ab.4)

The runtime and CLI cutover is already in place, but the TUI still renders and names
session members as shells. Complete only the TUI scope of `bead:sase-1ab.4`, following
`plan:202609/sase_turn_rename.md` and its shared vocabulary and compatibility policy.
This is one coordinated presentation change: changing the model and section names, their
callers, assertions, and visual fixtures together keeps the UI coherent.

1. Rename TUI-owned modules, imports, tests, fixture helpers, and public-internal
   identifiers for session turns and named procs. Cover the five modules called out in
   the epic plan: `actions/agents/_proc_shell_dismiss.py`,
   `models/agent_proc_shells.py`, `models/_agent_session_shell_membership.py`,
   `widgets/prompt_panel/_agent_proc_shell_section.py`, and
   `widgets/prompt_panel/_agent_shell_section.py`. Update the lane/count/roster
   identifiers in `models/agent_session_members.py`, `ResponsiveShellSection`,
   `include_monitor_shells`, `is_proc_shell`, style constants, and every importer.
   Retain unrelated Artifacts pane chrome, Unix-shell and completion identifiers.
2. Move section IDs from `shells` and `proc-shell` to `turns` and `named-proc`, plus
   `proc-shell:{id}` output source, `proc-shell-dismiss` worker group, and producer
   method-name strings. Check whether those IDs survive in persisted fold state; if they
   do, read old IDs while writing only the new ones. Add focused coverage for a
   pre-rename stored ID if applicable.
3. Apply the epic vocabulary to every TUI-visible surface: `SESSION TURNS`,
   `AGENT TURN`, `GATE TURN`, `MONITOR TURN`, `NAMED PROC`, row counts/tails, Node
   Finder kinds and previews, the `0-9 turn` footer, help legend, modal title,
   notifications, display-name fallbacks, statistics and artifact descriptions, and the
   TUI comment in `src/sase/default_config.yml`. Distinguish a session monitor turn from
   a standalone named proc. Update assertions and tests in `tests/ace/**`,
   `tests/perf/**`, and related TUI tests along with their code. No keymap value is
   expected to change; if one does, synchronize the default config.
4. Rename shell-named visual tests and fixtures and the seven shell-named golden files,
   including their window titles. Run the full `just fix-tui-screenshots` update through
   `/sase_monitor` with `TESTING`/`TESTED`, because stale golden removal requires a full
   pass. Inspect the report for every creation/removal and each update group;
   investigate any `partial` result before accepting it. Then run the strict visual
   check and inspect representative generated PNGs.
5. Preserve the TUI performance contract: no new filesystem work, awaits, subprocesses,
   or full rebuilds in navigation/render handlers. Run the existing j/k navigation
   benchmark, compare with a clean baseline if numbers suggest a regression, and run
   `sase tool run check` after `just fix`. Classify every remaining `shell` hit in the
   scoped TUI and tests as Unix shell, UI chrome, completion, or a deliberate legacy
   reader. Run `sase bead epic-symbols sase-1ab.4`; resolve entries or re-key their
   Justfile lines to an open bead before closing. Record out-of-scope findings on this
   phase bead as `PROPOSED FOLLOW-UP:` notes rather than creating beads. Close only
   `sase-1ab.4` with a note stating the verified results, even if a check failure
   reproduces identically on the clean base tree; do not close any ancestor.
