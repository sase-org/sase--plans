---
tier: tale
title: Sequential multi-target sase bead work
goal:
  Allow one sase bead work invocation to process any ordered mix of plan files and bead
  IDs with shell-&& short-circuit semantics.
size: medium
proposed_by: bbugyi200.athena.w0
create_time: 2026-08-08 15:48:46
status: wip
---

# Plan: Sequential multi-target `sase bead work`

## Scope and behavioral contract

Extend `sase bead work` from one required positional target to one or more required
targets. Preserve the caller's order and send every target through the existing
single-target plan-file/task-bead/epic-bead dispatch path with the same command-wide
options.

Treat the invocation as an ordered, non-atomic sequence equivalent to running the
single-target commands with `&&`:

- Finish one target before beginning the next.
- Stop at the first target that raises the command's existing error/exit path; do not
  validate, mutate, or launch any later target.
- Preserve successful work and output from earlier targets rather than adding a batch
  rollback or all-target preflight.
- Preserve the existing meaning of a user-declined launch: because the current
  single-target command returns success after printing `Aborted.`, the sequence may
  continue just as an `&&` chain would.
- Apply command-wide flags independently to every processed target. Plan-only flags
  (`--parent`, `--artifacts-dir`, and `--cl-name`) and task-only launch feedback keep
  their existing target-local validation, so a mixed sequence may complete earlier
  compatible targets and then stop when it reaches an incompatible one.
- Keep one code-swap reader lock around the complete invocation so the already-running
  Python process cannot observe an editable source swap between targets. If that lock
  cannot be acquired, process no targets and retain the existing single-target error
  shape using the first requested target. Continue to create one launch timing record
  per target inside the locked sequence.
- In human mode, retain the natural concatenation of each existing target's output. In
  JSON mode, emit one complete JSON object per processed target, one object per line;
  the one-target JSON contract remains exactly one object and one trailing newline.

This is a `tale`: the parser, ordered dispatch wrapper, compatibility behavior, tests,
and documentation form one tightly coupled CLI change that one implementation agent can
complete and verify together. Do not change the underlying plan-file, task-bead, or
epic-bead orchestration, add parallel execution, introduce per-target option groups, or
expand the TUI's single-row launch action.

## Implementation

1. Update `register_bead_work_parser` in `src/sase/main/parser_bead_lifecycle.py` so the
   positional argument accepts one or more values while preserving their authored order.
   Keep at least one target required. Refresh the metavar, description, JSON help, and
   examples to advertise multiple plan files, multiple bead IDs, and a mixed invocation.

2. Refactor `src/sase/bead/cli_work_entry.py` around the current single-target boundary:
   normalize parser-produced target lists while continuing to accept the scalar
   `argparse.Namespace(target=...)` and legacy `id` shapes used by library callers and
   existing tests; acquire the code-swap reader lock once; then call the unchanged
   locked single-target dispatcher once per normalized target in order. Let its existing
   `SystemExit` and unexpected exceptions escape immediately to implement
   short-circuiting. Do not collect all results before printing, reopen one project for
   the whole batch, or otherwise couple target-local locks, timers, cleanup,
   publication, confirmation, and rollback behavior.

3. Add focused regression coverage at the parser and dispatch boundaries:
   - A single positional value still parses, multiple values (including a `.md` path
     mixed with bead IDs) parse in order, and omitting all targets remains a usage
     error.
   - Scalar namespaces remain compatible, protecting existing direct callers.
   - A successful mixed sequence invokes the real single-target boundary in exact
     argument order and reuses the command options for every target.
   - A failure in the middle leaves the earlier call observed and prevents the later
     call, including the JSON case where completed result lines and the failing result
     line remain valid newline-delimited objects.
   - The code-swap lock still prevents any target from starting, and one lock covers the
     ordered sequence.
   - Existing single-plan, task, epic, validation, confirmation, timing, and JSON tests
     continue to pass without changing their behavior.

4. Update the user-facing contract in `docs/beads.md` and the command reference table in
   `docs/configuration.md`: show the plural target syntax, explain order and first-error
   short-circuiting, clarify that earlier side effects are not rolled back, document
   command-wide flag behavior for mixed targets, and describe JSON Lines output for a
   multi-target run. Align the parser's help test and concise command summaries such as
   the bead onboarding text and sidecar bead README where they display a singular
   `sase bead work` synopsis.

## Verification

Run `just install` before repository checks, then run the focused parser and bead-work
test files covering the new sequence plus the existing plan-file, task, epic, and
code-swap-lock paths. Finish with `just check`; if its selection escalates or reports an
unusual selection, follow repository guidance and run `just check-full`.

Acceptance requires that one target remains backward compatible, arbitrary plan/bead
mixes execute strictly in the supplied order, the first failure prevents every later
target without undoing prior successes, JSON output is independently parseable one line
per processed target, and all updated help/documentation describes those semantics.
