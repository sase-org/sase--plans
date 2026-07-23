---
tier: tale
title: Ship the polished AXE status CLI
goal: Operators can inspect one read-only AXE whole-system snapshot in a polished
  terminal view or stable JSON form.
bead: sase-8t.3
parent: sase/repos/plans/202607/axe_status.md
create_time: 2026-07-23 08:43:10
status: done
---

- **PROMPT:** [202607/prompts/axe_status_cli.md](prompts/axe_status_cli.md)

# Ship the polished `sase axe status` CLI

## Goal

Complete bead `sase-8t.3` by exposing the already-implemented, read-only AXE status snapshot as an operator-friendly
terminal dashboard and a stable schema-version-1 JSON contract. The command must report the classifier-owned exit code,
stay consistent across human and JSON output, preserve the existing `lumberjack status` and `maintenance status`
surfaces, and document how status relates to recovery and deeper diagnostics.

## Current State

- `src/sase/axe/status_models.py` defines the frozen schema-version-1 `AxeStatusSnapshot`, including lifecycle state,
  health, desired state, raw and derived orchestrator evidence, maintenance, runner occupancy, sorted lumberjacks,
  lifecycle journal event, stable issues, collection errors, and the classifier-selected exit code. Its `to_wire()`
  method is the canonical JSON shape.
- `src/sase/axe/status_collector.py` already produces the shared snapshot without cleanup or state writes and converts
  required input failures into `error` snapshots with exit code 2.
- `src/sase/main/parser_ace.py` currently registers the AXE commands in alphabetical order but has no top-level
  `status`; `src/sase/main/axe_handler.py` similarly has no dispatch branch and its fallback usage inventory omits it.
- Existing Rich output follows the injectable-console pattern in `src/sase/axe/chop_render.py`. Parser ordering is
  enforced generically by `tests/main/test_parser_command_defaults.py`, while focused help assertions live in
  `tests/main/test_parser_command_help.py`.
- `docs/axe.md`, `docs/cli.md`, and the AXE CLI reference in `docs/configuration.md` describe the compatibility/debug
  status commands but not the whole-system snapshot.

## Implementation

1. Add the public command surface and dispatch.
   - Register top-level `sase axe status` between `maintenance` and `start` in `src/sase/main/parser_ace.py`.
   - Describe it in help as a read-only, whole-system health snapshot and add the public `-j/--json` flag with
     machine-readable schema-version-1 wording.
   - Add the sorted `status` branch and fallback usage entry in `src/sase/main/axe_handler.py`.
   - Collect exactly one snapshot, select JSON or human rendering from that same object, and terminate with
     `snapshot.exit_code` so healthy/intentional states return 0, actionable degradation returns 1, and collection
     failure returns 2.

2. Add dedicated, testable status rendering in `src/sase/axe/status_render.py`.
   - Provide separate human and JSON render functions. JSON must serialize only `snapshot.to_wire()` with deterministic
     formatting/key ordering and must never invoke Rich or emit markup/ANSI.
   - Render the human view through an injectable Rich `Console` as one `AXE Status` summary panel, a sorted lumberjack
     table, and an `Attention` panel only when issues or a collection error exist.
   - Use a state-to-badge/style mapping that keeps lifecycle separate from health: green for healthy running, yellow for
     maintenance and warnings, cyan/dim for intentionally stopped or not-yet-started facts, and red for down, degraded,
     or collection errors.
   - Include desired state with source/time, orchestrator live PID and lock coherence, maintenance reason/owner/age when
     present, hook and agent runner occupancy, and the newest lifecycle event in the summary.
   - Show each lumberjack’s name, derived state, PID, heartbeat age, cycle count, historical error count, and configured
     chops. Use compact duration/placeholders plus foldable or width-adaptive columns so narrow terminals remain
     readable without dropping contract facts.
   - Preserve the classifier’s issue order, display each issue summary, and deduplicate suggested commands in first-seen
     order so every unhealthy view has a concrete next action without repeated instructions.

3. Add focused parser, handler, contract, and terminal-render coverage.
   - Extend `tests/main/test_parser_command_help.py` to include `status` in the sorted AXE inventory and assert both
     aliases and the read-only help wording.
   - Add `tests/test_axe_status_cli.py` with immutable snapshot builders that exercise handler dispatch and exact exit
     codes 0/1/2 without reading host state.
   - Assert JSON is exactly the snapshot wire contract, is stable across repeated renders, contains no ANSI, and carries
     the same state, health, issues, and exit code as the human view’s source snapshot.
   - Render deterministic wide and narrow consoles with color disabled and redirected output. Cover healthy running,
     maintenance, intentional stop, not-started, down, degraded orchestrator/lumberjack, live orphan, and collection
     error snapshots; assert the attention panel is conditional, next steps are deduplicated, all lumberjack fields are
     represented, and narrow output folds rather than truncating essential facts.
   - Include a color-enabled console assertion and a `NO_COLOR`/non-terminal assertion so ANSI behavior is deliberate
     while JSON remains plain under every mode.

4. Update operator and automation documentation.
   - Add `sase axe status` and `sase axe status --json` to the command table/examples in `docs/axe.md`, followed by a
     focused whole-system status section covering the seven lifecycle states, health interpretation, visible fields,
     issue/next-step behavior, read-only guarantee, JSON schema version, and exit codes.
   - Explain when to use `status`, `ensure`, `doctor --deep`, `maintenance status`, and `lumberjack status`, explicitly
     retaining the latter two as compatibility/debugging surfaces.
   - Add the command to the automation inventory in `docs/cli.md`.
   - Add the `-j/--json` option and 0/1/2 exit-code contract to the AXE CLI reference in `docs/configuration.md`,
     without introducing new configuration values.

## Validation

1. Run `just install` before repository checks so the workspace uses the current linked Rust binding and development
   dependencies.
2. During development, run focused tests for status models/collector, the new CLI renderer/handler suite, AXE parser
   help, and global parser ordering/defaults.
3. Manually capture or test human output at narrow and wide widths with normal color, `NO_COLOR`, and redirected output;
   run JSON mode for representative exit-0, exit-1, and exit-2 snapshots and verify it has no ANSI.
4. Run `just rust-check` to revalidate the already-integrated status binding used by the new public automation surface,
   without changing Rust release versions.
5. Run the required final `just check`, fix every failure, and rerun it until clean. Recheck `git diff --check`, verify
   only intended files changed, close only `sase-8t.3`, and confirm parent epic `sase-8t` remains open.

## Acceptance Criteria

- `sase axe -h` lists `status` alphabetically, and `sase axe status -h` clearly documents the read-only snapshot and
  both JSON aliases.
- Human and JSON modes render the same single classified snapshot and exit with its documented 0/1/2 code.
- The human dashboard is understandable, actionable, color-aware, and readable at narrow widths or when redirected.
- JSON is the exact stable schema-version-1 wire object with no Rich markup or ANSI.
- Every designed state family, conditional attention behavior, parser contract, and terminal layout has deterministic
  automated coverage.
- AXE documentation and CLI references explain the command, fields, states, exit codes, read-only behavior, and
  relationship to existing recovery/debugging commands.
- The full Python and linked-core checks pass, existing AXE lifecycle behavior is unchanged, `sase-8t.3` is closed, and
  parent epic `sase-8t` is not closed.
