---
tier: tale
size: medium
title: Retire detached proc options and enforce supervisor-only semantics
goal:
  Remove detached as a selectable execution mode while preserving historical proc
  readability, make session scope purely attribution-based, and add durable-argv and
  ACE-ownership invariants that prevent the retired model from returning.
proposed_by: bbugyi200.athena.sase-m9.3.1.5
bead: sase-m9.3.1.5
create_time: 2026-08-15 21:16:58
status: wip
---

- **PARENT:** [202608/ace_proc_ownership.md](ace_proc_ownership.md)
- **BEAD:**
  [sase-m9.3.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.3.1.5.md)

# Plan: Retire detached proc options and enforce supervisor-only semantics

## Context

Every newly submitted proc now runs under the independent proc supervisor. The remaining
`-d/--detached` flags and `detached`/`tui` execution kinds expose an ownership choice
that no longer exists. Historical rows must remain inspectable and controllable, but new
writers and public help must describe only supervisor-owned work whose optional session
id is attribution.

This tale completes phase `detach-retirement-and-enforcement` of the approved
`ace_proc_ownership` epic. It changes only the current `sase` repository and closes only
the assigned phase bead after verification.

## Implementation

1. Retire the public CLI options without losing the migration diagnostic.
   - Remove `-d/--detached` from `sase proc run`, `sase proc list`, and the legacy
     `sase task` alias help/parser surface, including examples and completion-visible
     option inventories.
   - Keep the legacy `--kind command|tui|detached` history filter parseable but suppress
     it from public help and completion surfaces.
   - Before public argparse handling, recognize obsolete `-d` and `--detached` tokens
     only in the proc/task list and run option regions, never inside the command after
     `--`, and fail for one release with exactly:
     `all procs are detached; remove --detached (use --session none for no attribution).`

2. Make all new proc writers use the single supervisor-owned command kind.
   - Have `sase proc run` always submit durable argv through the ordinary proc API,
     resolving `--session` only for attribution and using `--session none` for an
     unattributed proc.
   - Make legacy programmatic detached entry points and monitor/epic/task fallbacks
     create ordinary supervisor-owned command rows with no session attribution while
     continuing to recognize existing active `detached` rows for deduplication,
     recovery, display, and control.
   - Remove active behavior branches that choose submission or visibility from semantic
     kind; retain constants, parsing, glyphs, and compatibility paths only where needed
     to read historical `command`, `tui`, and `detached` rows.

3. Align scope, JSON, ACE observation, and documentation with attribution semantics.
   - An explicit session filter must always include unattributed rows as well as rows
     attributed to the selected session; `--all` continues to include every session.
   - Update scope titles, empty states, JSON scope language/schema, and focused tests so
     they describe unattributed visibility rather than a privileged detached kind.
   - Update the CLI, ACE, integration, SDD/launch, and getting-started references that
     still instruct users or plugins to select detached mode, while clearly marking
     historical kind filters and compatibility APIs as legacy where they remain.

4. Add regression and static invariants.
   - Cover proc and task help, the exact obsolete-token error (including separator
     safety), `--session none`, explicit-session inclusion of unattributed procs, hidden
     legacy-kind history filtering, new-writer kind normalization, and legacy-row
     readability/control.
   - Extend AST/static conformance so durable proc submission APIs cannot accept or
     invoke Python callables, production writers cannot emit `tui` or `detached` kinds,
     and ACE cannot record its own PID as the owner of an active proc. Add focused
     process-level evidence that a proc submitted by ACE is owned by its supervisor.

## Verification

1. Run the focused parser, proc handler/render, proc service/runner, monitor/bead
   launch, ACE observer/pane, and static inventory tests affected by the change.
2. Exercise real CLI help and obsolete-token invocations for both `proc` and `task`,
   including a launched command whose argv contains `--detached` after `--`.
3. Run `just install`, then `just check` as required for repository changes. If scoped
   selection escalates or reports unusual coverage, use `/sase_monitor` for
   `just check-full` with an explicit next action.
4. Recheck the diff for unintended memory/generated-instruction changes, record any
   out-of-scope discovery as a `PROPOSED FOLLOW-UP:` note on `sase-m9.3.1.5`, and close
   only `sase-m9.3.1.5` with a note summarizing the verified behavior.

## Acceptance criteria

- Public proc/task help and completions expose no detached selector, and obsolete flag
  use returns the required exact one-release diagnostic without consuming command argv.
- Every new proc is a supervisor-owned durable-argv command row; session is attribution
  only, and unattributed rows remain visible under explicit session filters.
- Historical `tui` and `detached` rows remain filterable through the suppressed legacy
  kind option and remain readable and controllable.
- Static and process-level tests prevent callable submission, semantic legacy-kind
  writes, and active ACE-PID ownership from regressing.
- Focused verification and `just check` pass, and only the assigned phase bead is
  closed.
