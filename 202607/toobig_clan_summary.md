---
tier: epic
title: Beautiful toobig split clan summaries
goal: Carry one safe, polished scan summary through the structured chop contract and
  attach it to the declaring toobig clan.
phases:
- id: chop-clan-summary-contract
  title: Add group-safe clan summaries to chop proposals
  depends_on: []
  description: Extend sase-core and SASE so a structured chop can provide one literal
    Rich summary that follows the surviving clan declarer through validation, deduplication,
    preview, and launch.
  size: large
- id: toobig-summary-presentation
  title: Author and verify the toobig clan summary
  depends_on:
  - chop-clan-summary-contract
  description: Use the new contract in bugyi-chops to render a compact, width-bounded
    toobig scan summary, document it, and verify the complete plugin-to-directive
    path.
  size: medium
create_time: 2026-07-22 11:17:03
status: wip
bead_id: sase-8l
---

# Beautiful `toobig_split` Clan Summaries

## Outcome

When `bugyi_chop_toobig_split` finds oversized Python files, the concrete `toobig-<token>` clan will open with a concise
Rich summary in the ACE clan panel. The summary will explain the mission, quantify the scan, and show the configured
limits while the existing clan roster continues to show individual workers.

The declaring launch must contain exactly one directive shaped like:

```text
%clan(toobig-0, tribe=chop, summary=[[
<Rich summary>
]])
```

Joiners must continue to use only `%id(<member>, clan=toobig-0)`. The summary is metadata, not part of any split agent's
work prompt.

## Design Decisions

### Use the literal `summary=` argument

Use `%clan(..., summary=[[...]])`, not `summary_script=`. The chop has already completed the scan and knows the stable
file count, tree count, and limits. A literal summary is deterministic, adds no launch-time subprocess or timeout,
remains valid when the workspace is prepared later, and is visible in an exact dry-run prompt.

The plugin must not emit its own `%clan` directive. It will submit structured clan metadata, and Axe will remain the
sole owner of concrete clan allocation and the declaring/joining scaffold.

### Make the summary clan-scoped and dedupe-safe

Add an optional `clan_summary` string to a structured launch proposal. It is valid only when `clan` is also present.
Within one result document, all non-null summaries for the same raw clan template must be identical; conflicting values
fail the whole result before launch.

During proposal preparation, resolve that one agreed summary per raw clan and copy it onto every prepared member before
once-per filtering. This preserves the summary when the original head is deduplicated and a later surviving member is
promoted to declarer. Planning attaches the summary only to the first accepted member of the concrete clan; joiners
never redeclare or override it.

Keep the field optional so existing chop producers and non-clan proposals retain their current behavior. Keep the
current result schema version because this is an additive part of the in-development SASE 0.12 contract; do not
hand-edit release versions.

### Encode it safely through the directive grammar

Treat `clan_summary` as literal Rich markup, not executable text. The shared Rust validator should reject blank
summaries, NUL, values over the established clan-summary budget, and sequences that the current xprompt text-block
grammar cannot round-trip exactly (notably a literal `]]` terminator or `+` substitution). Surface field-specific
diagnostics rather than allowing malformed markup to escape the `summary=[[...]]` argument.

Centralize the Python formatting of the `%clan` declaration so the concrete name, `tribe=chop`, and summary argument are
assembled once. Parse the resulting prompt in tests and compare the recovered summary with the submitted value; do not
rely only on substring assertions.

### Keep the presentation compact and purposeful

Render a fixed-layout summary using the existing clan-summary palette and a 76-cell target width. Dynamic values are
integers only, so repository paths and other untrusted scanner output never enter Rich markup or the directive grammar.
The intended plain-text composition is:

```text
◆ TOOBIG SPLIT · 3 FILES
MISSION
Decompose oversized Python modules into focused, reviewable units
without changing behavior.
2 scan roots · limits 1,000 / 850 / 700 lines · sequential queue
```

Use a warm bold accent for the header, cyan for the section label, soft lavender for the mission, and muted gray for
scan facts. Handle `1 FILE`/plural correctly, format limits with thousands separators, and keep every rendered line
within 76 cells. The queue remains sequential exactly as it is today.

Do not change ACE layout, the `chop` tribe, proposal priorities, wait dependencies, dedupe keys, scanner behavior,
Athena configuration, or the compact top-level Axe run summary.

## Phase 1: Add Group-Safe Clan Summaries to Chop Proposals

Work in the SASE repository and its linked `sase-core` repository.

1. Extend `ChopLaunchProposalWire` with optional `clan_summary` data and retain strict unknown-field handling for
   everything else. Add Rust validation for the field-level representation rules, the `clan_summary`-requires-`clan`
   relationship, and conflicting summaries within one raw clan group. Cover JSON round trips, legacy omission,
   field-specific failures, and multiple independent clans.
2. Thread `clan_summary` through the public `sase.chops.launch_proposal`, `ChopResultBuilder.propose`, and
   `add_proposal` helpers. Update SDK tests so authored results prove the value survives Rust validation and atomic
   serialization.
3. Extend the prepared/planned chop proposal model. Resolve a single summary for each raw clan before dedupe, preserve
   it across accepted-subset planning, and render it only on the concrete declarer's
   `%clan(<name>, tribe=chop, summary=[[...]])` directive. Keep non-clan and summary-free prompt bytes unchanged.
4. Include the effective declaration summary in dry-run proposal metadata so operators can inspect both the structured
   value and exact scaffolded prompt. Do not add it to joiner launch descriptors as though they had declared it.
5. Add focused SASE tests for normal clan batches, summary-free compatibility, exact directive extraction, conflicting
   summaries, a deduplicated head that promotes a summarized tail, multiple clan templates, and launch preflight proving
   that only one segment declares the summary. Retain all current name reservation and wait-chain assertions.
6. Document `clan_summary` in the Axe structured-result contract and chop-package authoring guide, including its group
   semantics, safety limits, dedupe behavior, and translation to the literal `%clan` `summary=` argument.
7. Rebuild the local Rust binding after the wire change. Run the focused Rust and Python tests during iteration, then
   run `just rust-check`, `just install`, and the mandatory `just check` from the SASE repository.

## Phase 2: Author and Verify the `toobig` Clan Summary

After Phase 1, open `gh:bbugyi200/bugyi-chops` through `/sase_repo` and use the active SASE development environment so
the plugin tests exercise the new SDK contract.

1. Add a small, pure summary renderer beside `build_result()` in `toobig_split.py`. Give the Rich styles and 76-cell
   width descriptive constants, generate the exact composition above from file/tree counts and limits, and avoid
   including raw paths, environment values, or command output.
2. Compute the summary once for an actionable scan and pass the identical `clan_summary` value on every
   `result.propose(...)` call for `CLAN_TEMPLATE`. This repetition is deliberate: any surviving proposal can safely
   become the declarer after dedupe.
3. Expand `test_toobig_split.py` to assert identical structured summaries on every proposal, singular/plural and
   custom-limit rendering, valid Rich markup, the intended color/style spans, and the 76-cell maximum. Preserve all
   existing prompt, name, workspace, dedupe, scanner, and wait-chain assertions.
4. Add an integration assertion using the SASE planning/directive path: the first surviving member gets one concrete
   `%clan(..., summary=[[...]])`, directive extraction recovers the authored markup exactly, later members contain no
   clan summary declaration, and promoting a later member does not lose the summary.
5. Update the plugin README's `toobig_split` contract and example to explain the ACE clan summary without suggesting
   that the script launches agents or owns concrete clan names.
6. Run the plugin's focused tests, then `just check` with `BUGYI_CHOPS_VENV_BIN` pointed at the active SASE development
   environment. Finally run the configured `toobig_split` chop in dry-run/verbose mode and inspect the exact scaffolded
   prompt to confirm the summary is attached once and no agents launch.

## Acceptance Criteria

- Existing structured chop results without `clan_summary` validate and launch unchanged.
- A summarized clan result remains all-or-nothing under strict Rust validation, including safe text-block representation
  and same-clan consistency.
- Once-per filtering can remove the original head without removing the summary; the first surviving member declares it
  exactly once.
- The concrete declaring prompt uses the literal `%clan` `summary=` argument, and directive extraction yields the
  original Rich markup byte-for-byte after documented normalization.
- The `toobig_split` summary clearly communicates mission, file count, scan-root count, limits, and sequential
  execution; it parses as Rich markup and stays within 76 cells.
- No file path or other scanner-controlled text is interpolated into the markup.
- SASE passes `just rust-check`, `just install`, and `just check`; `bugyi-chops` passes its full `just check` against
  that SASE environment; the final Axe dry run launches nothing.

## Rollout and Compatibility

Land the SASE/sase-core contract before the plugin begins emitting `clan_summary`. The plugin already targets the SASE
0.12 line that introduced clan-scoped chop proposals, so no compatibility shim, schema-version fork, or runtime-specific
branch is needed. Existing installed plugin versions remain valid because the new field is optional on the consumer
side.
