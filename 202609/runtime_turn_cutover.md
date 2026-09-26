---
tier: tale
title: Runtime, syntax, and CLI cutover for sase turns
goal:
  Non-TUI SASE runtime and public syntax use canonical turn and named-proc vocabulary
  while legacy durable data remains readable and retired authored syntax is controlled
  by a sunset flag.
size: medium
proposed_by: bbugyi200.athena.sase-1ab.3
bead: sase-1ab.3
create_time: 2026-09-26 12:09:42
status: wip
---

- **PARENT:**
  [202609/sase_turn_rename.md](https://github.com/sase-org/sase--plans/blob/main/202609/sase_turn_rename.md)
- **BEAD:**
  [sase-1ab.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ab/sase-1ab.3.md)

# Runtime, syntax, and CLI cutover for sase turns

Implement the `runtime-cutover` scope of bead `sase-1ab.3` and the parent design
`plan:202609/sase_turn_rename.md`. The existing `sase-core` additive rename and Python
persistence/wire phase are prerequisites already complete. This plan is scoped to the
`sase` repository and non-TUI code; another phase owns `src/sase/ace/tui/**` and
`tests/ace/**`. Make only import-following TUI edits here. Do not close the parent epic
or any other ancestor bead.

The new terminology distinguishes agent turns, monitor turns, and gate turns as session
members; stand-alone named procs use `named-proc` and `proc_name`. Keep Unix shell,
completion-shell, and UI chrome uses of "shell" unchanged. Preserve permanent readers
for pre-rename metadata, gate bundles, and proc rows. Writers use canonical keys. Avoid
replacing the substring `turn` in `return` when updating handoff errors: pass the member
noun to `TurnHandoffError`.

## Runtime

Move `src/sase/gate_shell/` to `gate_turn/`, `shells/` to `turns/`, `plan_shell/` to
`plan_gate_turn/`, and `question_shell/` to `question_gate_turn/`. Also rename
`main/gate_shell_handler.py`, `main/gate_shell_render.py`,
`notification_gates/model_shell.py`, and the reclaim script module. Update all non-TUI
imports, APIs, identifiers, comments, messages, and the corresponding test
packages/files. Keep legacy data readers explicit; avoid internal alias modules made
solely for the rename. Match the parent design's mapping of `shell_name` to `proc_name`,
`shell_kind` on a proc to `proc_role`, and member `shell_kind` to `turn_kind`. Run
focused runtime tests and fix import fallout before handing off.

## Syntax

Implement `sase gate create -G/--turn`, `-g/--turn-status`, `-E/--turn-stop-status`, and
`--next-fork {session,turn,none}`; gate specs use a `turn` block and `fork: turn`.
Implement `sase proc list/run -N/--name` and `gate.turn.reclaim_grace_seconds`. Create
the `legacy_sase_shell_syntax` sunset flag only with `sase flag new`, using the exact
enabled/disabled/removal semantics in the parent design. Centralize hidden legacy
aliases and replacement errors in `src/sase/agent/legacy_sase_shell_syntax.py`, and test
both flag states. Keep durable legacy readers unconditional. Update CLI help with sorted
options and short aliases.

## Surfaces

Rename gate and proc CLI JSON keys and display labels, scheduler job and entry points,
telemetry metrics and catalog entry, `%wait` help, routing prefix, fork xprompt, and
`sase_gate`, `sase_run`, `sase_questions`, `sase_plan`, and `sase_monitor` skill source
templates. Regenerate the CLI completion snapshot. Update renamed test paths in the
contract manifest, shard timings, reproducible flake baseline, and terminology-test
allowlist. Do not deploy generated skills to chezmoi; a later parent phase owns that.
Include a `feat!:` Conventional Commit message and `BREAKING CHANGE:` footer listing CLI
flags, JSON keys, config key, scheduler job (including unmigrated old run history), and
metric names in the final declaration.

## Integration and verification

Read `lint_and_test.md` before finishing tracked changes. Run `just install` in this
fresh workspace, focused tests during implementation, then `sase tool run check` (not
`check-full`). Classify every remaining non-TUI `shell` hit as Unix/completion/UI
chrome, deliberate legacy reader/fixture, or sunset branch, and repair unintended
concept hits. Coordinate renamed API imports with concurrent TUI work without rewriting
its owned surfaces. Run `sase bead epic-symbols sase-1ab.3`; resolve each remaining
symbol or re-key its Justfile line to an open bead. If a failure reproduces on the clean
base tree, note `PROPOSED FOLLOW-UP:` on this phase and close anyway. Record other
discovered work only as `PROPOSED FOLLOW-UP:` notes on `sase-1ab.3`. Close only
`sase-1ab.3` with `sase bead close sase-1ab.3 --note "<what was verified>"`.
