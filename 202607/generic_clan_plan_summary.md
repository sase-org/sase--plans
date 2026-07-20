---
tier: tale
title: Generic clan plan summaries and script arguments
goal: 'Clan declarations can pass quoted arguments to launch-time summary scripts,
  and a reusable console script can render any valid authored plan with the shared
  PLAN-lane presentation while preserving launch safety and compatibility.

  '
create_time: 2026-07-20 15:34:15
status: done
prompt: 202607/prompts/generic_clan_plan_summary.md
---

# Plan: Generic clan plan summaries and script arguments

## Context and scope

This implements phase `sase-8d.2` of the plan-lane clan-summary epic. Phase 1 is the prerequisite: its shared
plan-display module owns plan-file loading and the logical, width-aware PLAN-lane rendering that this work must consume.
This phase adds generic script invocation and a generic plan-summary executable; it does not rewrite
`sase_clan_summary_epic`, add epic plan-reference root discovery, or change clan-panel visuals, which belong to phase 3.

Launch-time summary generation remains decorative and non-fatal. It still runs during directive extraction, inherits the
declaring process environment, uses the initial workspace as its working directory, writes stderr to the agent log,
times out after 20 seconds, and persists at most 32 KiB of UTF-8 output.

## Summary-script argv support

Extend `src/sase/axe/clan_summary_script.py` so a `summary_script=` value can describe an executable plus shell-style
quoted arguments. Preserve existing values by probing the complete literal first: an executable path whose filename
contains spaces must still run unchanged. Only when that literal is not an executable should the resolver use `shlex`
parsing, discover the first token with the current absolute/relative-path and installed-command rules, and pass the
remaining tokens directly as subprocess argv without invoking a shell.

Malformed quoting, an empty argv, discovery failures, execution errors, timeouts, non-zero exits, and empty output must
keep the existing warning-and-omit behavior so summary generation can never block an agent launch. Add focused
persistence or smoke coverage for bare commands with arguments, quoted arguments, relative paths, and literal executable
paths containing spaces, alongside the current failure, environment, timeout, stderr, and output-cap cases.

## Generic plan summary executable

Add `sase_clan_summary_plan` under `src/sase/scripts/` and register it in `pyproject.toml`. It accepts a plan reference
as its first positional argument, falling back to `SASE_EPIC_PLAN_REF` when no argument is supplied. Resolve an ordinary
relative reference from the script's working directory, then use the shared phase-1 plan-display loader and logical
renderer rather than duplicating plan validation, normalization, styles, or PLAN-lane layout.

Serialize the shared Rich `Text` lines at the established summary width. Fit the document below the existing summary
byte budget by retaining complete leading sections and phase blocks, dropping phase blocks only from the tail, and
adding a parseable omission line; never return arbitrarily truncated Rich markup. Missing, unreadable, or invalid plans
should emit a small escaped plain fallback and exit successfully, with diagnostics on stderr, so this executable is safe
under the same launch-time contract as other clan summary scripts.

Add script-level tests for explicit-argument and environment fallback selection, tale and epic plan rendering,
width-aware PLAN-lane parity, valid Rich markup, whole-block omission under a large plan, the UTF-8 budget, and
successful fallback for missing or invalid input. Keep all plan parsing and display assertions aligned with the public
shared module introduced by phase 1.

## Documentation

Update the launch-time clan-summary sections in `docs/agent_families.md` and `docs/xprompt.md` to show quoted
`summary_script=` argv syntax and the generic `sase_clan_summary_plan` usage. Document that scripts inherit the complete
launch environment, with SASE overriding `SASE_CLAN_NAME`, `SASE_CLAN_GENERATION`, and the optional `SASE_CLAN_TRIBE`;
call out the epic launch variables `SASE_EPIC_PLAN_REF`, `SASE_EPIC_BEAD_ID`, and `SASE_EPIC_CLAN_TRIBE`. Correct the
stale 10-second statement to the implemented 20-second timeout and retain the ordering, working-directory, stderr,
byte-cap, and non-fatal guarantees.

## Validation

Run targeted tests for summary-script persistence/launch behavior and the new generic plan-summary executable, including
console-entry-point discovery where practical. Confirm serialized output round-trips through `Text.from_markup` and
never exceeds its document budget. Because this phase changes repository files, run `just install` followed by the
complete `just check` gate, addressing any formatting, typing, unit, visual-snapshot, or Symvision regressions without
updating unrelated visual goldens.
