---
tier: epic
title: Plan-lane-style rich clan summaries
goal: 'Epic agent clans present the same rich plan rendering in their clan summary
  that epic lander agents show in the PLAN lane, backed by generic %clan summary-script
  machinery that any clan declaration can reuse.

  '
phases:
- id: shared-renderer
  title: Shared plan display module extracted from the PLAN lane
  depends_on: []
  size: medium
  description: '''Shared plan display module'' section: extract the PLAN lane''s value
    types, plan-file metadata loading, and logical line rendering into a shared presentation
    module that ResponsivePlanSection delegates to, with parity tests and unchanged
    PNG snapshots.'
- id: generic-machinery
  title: Generic plan summary script and summary_script arguments
  depends_on:
  - shared-renderer
  size: medium
  description: '''Generic clan summary machinery'' section: add argument support to
    %clan summary_script values, ship a sase_clan_summary_plan console script that
    renders any plan file in PLAN-lane style, and document the summary-script environment
    contract.'
- id: epic-summary
  title: Plan-first epic clan summary rendering
  depends_on:
  - generic-machinery
  size: medium
  description: '''Plan-first epic clan summary'' section: rewrite sase_clan_summary_epic
    to render the epic''s plan file through the shared renderer with multi-root path
    resolution and bead-store fallback, then refresh script tests and clan-panel visual
    fixtures.'
create_time: 2026-07-20 14:39:47
status: done
bead_id: sase-8d
---

# Plan: Plan-lane-style rich clan summaries

## Context

Selecting an epic agent clan row in `sase ace` shows the clan's `clan_summary` markup between the CLAN header fields and
the fold header (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`). Epic launches declare
`%clan(<epic>, tribe=epic, summary_script=sase_clan_summary_epic)` (`src/sase/bead/work.py`), and that script
(`src/sase/scripts/sase_clan_summary_epic.py`) renders the epic from the **bead store**: title, capped goal, phase list
with status icons and size chips, child epics, and a plan reference line. On any load failure it falls back to plain
`[bold]EPIC <id>[/]`.

Two problems observed on a live epic clan:

1. **The fallback fires in practice.** The summary script runs while launch directives are extracted
   (`src/sase/axe/run_agent_runner.py`, via `extract_directives_and_write_meta`), which is _before_ dependency waits and
   workspace preparation. For an epic launched moments after its beads were created, the declaring agent's
   not-yet-refreshed workspace bead store does not contain the epic, `refresh_current_bead_store()` races the bead push,
   and the clan panel permanently shows the plain fallback (confirmed in the declaring agent's persisted
   `agent_meta.json`: `clan_summary: "[bold]EPIC <id>[/]"`).
2. **Even the successful rendering is the wrong shape.** The much richer rendering users want is the PLAN lane shown for
   epic lander agents in the SASE CONTEXT panel (`ResponsivePlanSection` in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_plan_section.py`): a `PLAN · epic · N phases` header,
   `Title:`/`Goal:`/`Path:` rows, then numbered phases rendered as `N ◆ <title>`, a metadata row
   `<id> · after <deps> | no dependencies · model <m>`, and the full phase description. That lane is built from
   plan-_file_ data parsed by the Rust-core-backed `validate_plan` binding (`src/sase/sdd/plan_validate.py`) and cached
   per-file in `src/sase/ace/tui/models/_agent_associated_plan_cache.py`.

The fix follows from the data flow: the plan file reference is already available to the summary script at launch time.
Epic work segments export `SASE_EPIC_PLAN_REF` (plus `SASE_EPIC_BEAD_ID`, `SASE_EPIC_CLAN_TRIBE`) into the launched
agent's environment (`epic_work_segment_env` in `src/sase/bead/work.py`), and `resolve_clan_summary_script`
(`src/sase/axe/clan_summary_script.py`) hands summary-script subprocesses a copy of the full process environment. The
plan file itself is committed to the plans sidecar before the epic launches, so rendering from the plan file is both
richer and immune to the bead-store race.

## Design overview

Three layers, one per phase:

1. **Shared presentation module.** The PLAN lane's logical rendering moves into a shared, TUI-independent module so the
   lane and clan summaries render through the same code and cannot drift.
2. **Generic machinery.** Any `%clan` declaration can produce a plan-lane summary: `summary_script=` gains optional
   arguments, and a new `sase_clan_summary_plan` console script renders an arbitrary plan file. The summary-script
   environment contract gets documented.
3. **Epic wiring.** `sase_clan_summary_epic` becomes plan-file-first (via `SASE_EPIC_PLAN_REF`) with the current
   bead-store rendering as fallback, eliminating the observed race for plan-backed epics.

Boundary and performance notes that apply throughout:

- **Rust core boundary.** Plan parsing already lives in the Rust core (`plan_validate` binding); the `clan_summary`
  field already flows through the agent-scan wire. Everything in this epic is Rich-text presentation and launch-time
  script plumbing, so no `../sase-core` changes are needed.
- **TUI performance.** All rendering stays pure: the shared module performs no I/O in render paths, the TUI keeps its
  existing mtime-keyed plan-file cache, and clan summaries remain launch-time-static markup parsed once with
  `Text.from_markup`. No new refresh paths, no per-keypress disk access.
- **Memory files.** Documentation updates go to `docs/` only; `sase/memory/` and generated instruction shims are out of
  scope.

## Shared plan display module

Create a shared presentation module (suggested home: `src/sase/sdd/plan_display.py`, following the
`sase.agent.bead_display` precedent of display helpers shared between agent code and the TUI) that owns:

- **Value types.** Move `AssociatedPlanPhaseSummary` and the plan-shaped subset of `AssociatedPlanSummary` (title, goal,
  tier, display path, phases) into the shared module. `src/sase/ace/tui/models/_agent_associated_plan_types.py`
  re-exports them so existing TUI imports keep working; TUI-only fields (paths, readability, availability states) stay
  in the TUI types.
- **Plan-file loading.** A pure `load_plan_display(path)`-style helper built on `validate_plan` / `validate_plan_file`
  (launch mode) that returns the display data plus tier detection, extracted from the metadata-construction logic the
  associated-plan cache uses today. The TUI cache keeps its signature/TTL behavior and delegates the parse-to-metadata
  step here.
- **Line rendering.** Pure functions producing the lane's _logical_ fixed styled lines from display data: the
  `PLAN · <tier> · N phases` header details, the `Title:`/`Goal:`/`Path:` field rows, and per-phase blocks
  (`N ◆ <title>`, the id/dependencies/model metadata row, wrapped full description). The color constants these lines use
  (`COLOR_PLAN_PRIMARY`, `COLOR_PLAN_SUBHEADER`, `COLOR_SUMMARY`, `COLOR_REASON`, `COLOR_EMPTY`) move to (or are
  imported from) the shared module so both consumers use one definition. Rendering accepts a target width so script
  consumers can emit fixed-width serialized lines while the TUI keeps responsive reflow.

`ResponsivePlanSection` keeps its responsive table-based `__rich_console__` behavior but builds its rows and phase
blocks from the shared functions, so the widget's output is unchanged. Guardrails: existing widget tests plus the ACE
PNG snapshot suite must pass without golden updates for the agent detail panel.

## Generic clan summary machinery

Make the rich-summary capability reusable by any `%clan` user:

- **`summary_script=` arguments.** `resolve_clan_summary_script` currently treats the whole value as one script
  name/path. Teach it to split the value into argv (shlex) while preserving backward compatibility: first try the full
  literal as an executable path, then fall back to argv splitting where the first token is discovered as today
  (`_discover_summary_script` / `discover_chop_script`) and the remaining tokens pass through as arguments. This lets
  any clan declare, for example,
  `%clan(myclan, summary_script="sase_clan_summary_plan sase/repos/plans/202607/foo.md")`.
- **`sase_clan_summary_plan` console script.** New script (registered in `pyproject.toml` like `sase_clan_summary_epic`)
  that resolves a plan reference from its first argument or from `SASE_EPIC_PLAN_REF`, loads it with the shared module,
  and prints the plan-lane rendering serialized as Rich markup at the summary width, reusing the byte-budget/omission
  fitting approach from `sase_clan_summary_epic` (`_fit_document`-style: drop whole phase blocks from the tail and
  append an omission line rather than emitting a truncated document). Missing/invalid plan files print a small plain
  fallback and exit 0 so launches never block, matching existing summary-script guarantees.
- **Documented contract.** Update the clan summary sections of `docs/agent_families.md` and `docs/xprompt.md`: summary
  scripts inherit the declaring launch's full environment (including `SASE_EPIC_PLAN_REF`, `SASE_EPIC_BEAD_ID`, and
  `SASE_EPIC_CLAN_TRIBE` for epic launches) in addition to `SASE_CLAN_NAME`/`SASE_CLAN_GENERATION`/`SASE_CLAN_TRIBE`;
  `summary_script=` argument syntax; the new generic script; and fix the stale "capped at 10 seconds" claim to match the
  actual `CLAN_SUMMARY_TIMEOUT_SECONDS` (20 seconds).

## Plan-first epic clan summary

Rewrite `sase_clan_summary_epic` to be plan-file-first while keeping its launch-safety guarantees:

- **Resolution order.** (1) `SASE_EPIC_PLAN_REF` resolved against, in order, the current working directory (the
  declaring agent's workspace) and the project's primary checkout roots — the plans sidecar may not be cloned into a
  fresh workspace yet when directive extraction runs, but the primary sidecar checkout already contains the committed
  plan. (2) If no plan ref is set or no root yields a readable, valid plan file, fall back to today's bead-store
  rendering (epics can exist without plan files). (3) Final fallback stays `[bold]EPIC <id>[/]`.
- **Rendering.** The plan-backed document is: a `◆ EPIC <id>` header line (keeping the clan's at-a-glance identity; the
  title lives in the `Title:` row), followed by the shared plan-lane body — `Title:`/`Goal:`/`Path:` rows and the
  numbered phase blocks with id, dependencies, model, and full wrapped descriptions — at the existing 76-column summary
  width. Launch-time status icons, done-counts, and size chips are dropped from this rendering: they go stale the moment
  a phase closes, and the PLAN lane the user wants parity with does not show them. The bead-store fallback path keeps
  its current rendering unchanged. Document-level fitting keeps the 32 KiB cap with whole-block omission lines.
- **Tests and fixtures.** Extend `tests/test_bead/test_clan_summary_epic_script.py` for plan-first behavior (env-driven
  resolution, multi-root fallback, invalid plan file falls back to bead store, byte budgeting) and add coverage
  asserting the epic launch env reaches summary scripts (alongside `tests/test_axe_smoke_clan_summary.py`). Add or
  refresh a clan-panel PNG snapshot fixture (`tests/ace/tui/visual/`) showing a plan-lane-style summary so the end state
  in the clan detail panel is pinned.

## Testing

- Parity: unit tests assert the PLAN lane's logical output and the shared renderer produce identical styled lines for
  the same display data, so lane and summary cannot drift.
- Behavior preservation: agent-panel widget tests and PNG snapshots pass without golden changes after the
  shared-renderer extraction.
- Script contracts: round-trip tests that serialized summaries stay valid Rich markup (`Text.from_markup`) and within
  the 32 KiB cap for large plans; fallback-chain tests per resolution step; smoke tests for `summary_script` argument
  parsing including the literal-path-with-spaces compatibility case.
- Visual: `just test-visual` covers the new clan-panel golden; local runs use exact pixel equality.

## Risks and constraints

- **Refactor drift on the agent panel.** Extracting rendering from `ResponsivePlanSection` risks subtle visual changes;
  the PNG suite is the backstop, and the extraction must be a pure code move (same styles, same spacing).
- **Argument-splitting compatibility.** Existing `summary_script=` values that are paths containing spaces must keep
  working; the literal-path-first probe handles this, and tests pin it.
- **Long plans.** A many-phase plan produces a long summary above the clan panel's fold header. The document budget plus
  whole-block omission keeps it bounded, and the panel scrolls; if this proves noisy in practice, making the summary a
  foldable section is a natural follow-up outside this epic's scope.
- **Symbol moves.** Relocated value types and color constants must keep re-export shims so Symvision and existing
  imports stay clean; run the full `just check` gate in every phase.
