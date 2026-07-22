---
tier: tale
title: Complete the toobig split clan summary
goal: The toobig split chop authors one compact summary that survives dedupe and appears
  exactly once on the concrete clan declaration.
bead: sase-8l.2
parent: sase/repos/plans/202607/toobig_clan_summary.md
create_time: 2026-07-22 12:57:14
status: done
---

- **PROMPT:** [202607/prompts/toobig_clan_summary_1.md](prompts/toobig_clan_summary_1.md)

# Complete the `toobig_split` Clan Summary

## Goal

Finish phase bead `sase-8l.2` in `bugyi-chops` by authoring the compact Rich summary specified by the parent epic,
carrying the identical summary on every actionable `toobig-@` proposal, documenting the behavior, and verifying the full
plugin-to-SASE directive path without changing scan, dedupe, priority, wait-chain, or launch ownership behavior.

## Context and Constraints

- The prerequisite SASE contract from `sase-8l.1` is already landed: `ChopResultBuilder.propose()` accepts
  `clan_summary`, preparation propagates one agreed value across a raw clan before dedupe, and planning emits it only on
  the surviving concrete clan declarer.
- The renderer must be pure and must interpolate only integer scan facts: file count, scan-root count, and the three
  configured limits. No repository paths, environment values, or scanner output may enter Rich markup.
- The plain-text layout must use a 76-cell target width, singular/plural `FILE` handling, thousands separators for
  limits, the fixed mission copy, and the existing clan-summary palette: warm bold header, cyan section label, soft
  lavender mission, and muted gray facts.
- The plugin remains a proposal producer. Axe alone allocates concrete clan names, injects `%clan`/`%id` directives,
  performs dedupe, and launches agents.

## Implementation

1. In `src/bugyi_chops/toobig_split.py`, add descriptive constants for the 76-cell width and four Rich styles, plus a
   small pure summary renderer beside `build_result()`. Build the five-line fixed layout with `rich.text.Text`,
   serialize it to valid Rich markup, and derive only the dynamic integer fields from `files`, `trees`, and `limits`.
2. In `build_result()`, leave no-op scans unchanged. For actionable scans, compute the summary once after the file list
   is known and pass that exact `clan_summary` string to every `result.propose(...)` call using `CLAN_TEMPLATE`, while
   preserving prompts, agent names, priorities, dedupe keys, scan order, and `wait_on` values byte-for-byte.
3. Expand `tests/test_toobig_split.py` with focused presentation coverage. Assert the canonical plain text, valid Rich
   parsing, intended style spans, plural and singular file headings, custom limits with thousands separators, and a
   maximum of 76 terminal cells per rendered line. Strengthen the existing multi-proposal test to prove every structured
   proposal carries the same authored summary while retaining all prior assertions.
4. Add a SASE-path integration test in the plugin suite. Feed the plugin result through
   `prepare_chop_proposals()`/`plan_chop_proposals()`, parse the exact scaffolded prompts with
   `extract_prompt_directives()`, and prove that the first survivor declares one concrete
   `%clan(..., tribe=chop, summary=[[...]])`, joiners contain no summary declaration, directive extraction recovers the
   authored markup, and planning an accepted subset after dropping the original head promotes a summarized tail. Isolate
   name-reservation reads so the test is deterministic and side-effect free.
5. Update the README's `toobig_split` contract and example text to describe the ACE summary contents, deliberate
   per-proposal metadata repetition, and Axe's sole ownership of concrete clan allocation and agent launch.

## Validation and Completion

1. Run the focused `test_toobig_split.py` suite against the active SASE development environment while iterating.
2. Run the plugin's full `just check` with `BUGYI_CHOPS_VENV_BIN` pointing to that active SASE `.venv/bin`; address
   formatting, lint, typing, coverage, test, build, and artifact-validation failures until clean.
3. Run the configured `toobig_split` chop with `--dry-run --chop-verbose`, verify that no agent launch occurs, and
   inspect the proposal metadata and exact scaffolded prompts for one summary-bearing clan declaration and summary-free
   joiners. If the live scan is a no-op, create an isolated temporary dry-run fixture/configuration that exercises the
   same configured Axe runner path without launching agents.
4. Recheck both repositories for unintended changes, update only bead `sase-8l.2` with concise implementation notes, and
   close it after all validations pass. Leave parent epic `sase-8l` open and create no beads.
