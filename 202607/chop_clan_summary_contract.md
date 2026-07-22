---
tier: tale
title: Add group-safe clan summaries to structured chop proposals
goal: A validated literal Rich summary survives dedupe and appears exactly once on
  the surviving chop clan declarer.
bead: sase-8l.1
parent: sase/repos/plans/202607/toobig_clan_summary.md
create_time: 2026-07-22 12:04:36
status: wip
---

- **PROMPT:** [202607/prompts/chop_clan_summary_contract.md](prompts/chop_clan_summary_contract.md)

# Add Group-Safe Clan Summaries to Structured Chop Proposals

## Goal

Extend the additive schema-v1 structured chop contract so a proposal may carry an optional literal Rich-markup
`clan_summary`. Validate that value centrally in `sase-core`, preserve one agreed summary for each raw clan template
through once-per filtering, and put it on exactly the surviving concrete clan declarer's
`%clan(<name>, tribe=chop, summary=[[...]])` directive. Existing result documents and summary-free generated prompts
must remain compatible byte-for-byte.

This implements only bead `sase-8l.1`. Do not implement the later `bugyi-chops` presentation phase, create beads, or
close the parent epic.

## Contract and Invariants

- Keep `CHOP_RESULT_SCHEMA_VERSION` at `1`; `clan_summary` is optional and strict unknown-field rejection remains in
  force.
- `clan_summary` is legal only on a proposal with `clan`. When present, reject blank values, NUL bytes, UTF-8 payloads
  larger than the existing 32 KiB clan-summary budget, and text that cannot round-trip through the xprompt text-block
  grammar (at minimum literal `]]` terminators and `+`, whose argument decoding changes it to a space). Diagnostics must
  point at the proposal's `clan_summary` field.
- Within a full result, all non-null summaries for one raw clan template must be identical. Null members inherit that
  one agreed value during preparation; conflicting non-null values invalidate the whole result before dedupe or launch.
  Different raw clans may use different summaries.
- Resolve/copy the raw-clan summary onto every prepared member before once-per filtering. Plan the effective summary
  only on the first accepted member of the allocated concrete clan, so deduplicating the original head promotes a tail
  without losing the summary. Joiners use only `%id(<member>, clan=<concrete-clan>)` and never redeclare metadata.
- Treat the value as literal Rich markup. Do not execute a summary script, interpolate scanner output, or place the
  summary in any proposal work prompt.
- Preserve current tribe ownership (`chop`), naming/reservation, wait relinking, batching, priorities, and launch
  behavior.

## Implementation

1. Extend the shared Rust wire and validation contract in `sase-core`.
   - Add `clan_summary: Option<String>` to `ChopLaunchProposalWire` in `crates/sase_core/src/axe_chop/wire.rs` with
     serde defaulting so legacy omission continues to decode.
   - In `crates/sase_core/src/axe_chop/validation.rs`, add a reusable literal-summary validator using the established 32
     KiB UTF-8 byte limit. Enforce the field-level representation rules and `clan_summary`-requires-`clan` in
     `validate_chop_proposal`.
   - During `validate_chop_result`, track each raw clan template's non-null summary and fail on conflicts before the
     validated document can reach Python. Keep the check scoped by raw clan, not by concrete allocated name.
   - Expand `crates/sase_core/src/axe_chop/tests.rs` (and the existing PyO3 round-trip test in
     `crates/sase_core_py/src/lib.rs` where useful) for JSON round trips, legacy omission, each field-specific invalid
     case, missing clan, same-clan agreement/conflict, null-plus-present inheritance semantics, and independent clans.

2. Thread author-supplied values through the public Python SDK in `src/sase/chops/sdk.py`.
   - Add the optional keyword to `launch_proposal`, `ChopResultBuilder.propose`, and `add_proposal`, forwarding it in
     the same omission-preserving style as the other optional proposal fields.
   - Extend `tests/test_chop_sdk.py` so both helper construction and `ChopResultBuilder.write()` prove the summary
     survives real Rust validation and the atomic JSON write unchanged. Retain the absence behavior for legacy authors.

3. Preserve and plan effective clan metadata in `src/sase/axe/chop_proposals.py`.
   - Add `clan_summary` to the prepared proposal model and resolve one agreed non-null value per raw clan after all
     proposals are normalized, copying it onto every prepared member before the runner calls once-per filtering.
   - Carry an effective declaration summary on the planned proposal model only when `declares_clan` is true. This makes
     accepted-subset planning promote the summary together with a deduped clan head.
   - Centralize `%clan` declaration formatting in one helper. Emit the existing declaration exactly when no summary is
     present; otherwise append the safe literal `summary=[[...]]` argument. Keep joiner scaffolding unchanged.
   - Add `clan_summary` to proposal preview metadata as the effective declaration value (summary for the declarer,
     null/absent for joiners and non-clan proposals), alongside the exact scaffolded prompt. If launch descriptors
     expose it, expose it only for the declarer rather than implying joiners declared the summary.

4. Add focused planning, dry-run, and launch-preflight coverage.
   - Extend `tests/test_axe_chop_result_protocol.py` for summary-free byte compatibility, normal summarized clans,
     conflicting full results, multiple raw clans, and a once-per-deduplicated head whose first surviving tail declares
     the inherited summary.
   - Parse the scaffolded declarer's prompt with `sase.xprompt.directives.extract_prompt_directives` and compare
     `PromptDirectives.clan_summary` to the submitted Rich markup after the grammar's documented normalization; do not
     rely only on substring assertions.
   - Extend `tests/test_axe_chop_clan_launch.py` to exercise dry-run metadata and multi-prompt launch preflight: exactly
     one segment declares the literal summary, later segments are summary-free joiners, full member names/waits remain
     correct, and no agents launch during dry run.
   - Retain the existing assertions for concrete clan reservation, accepted-subset allocation, wait-chain relinking,
     environment propagation, partial launch recording, and summary-free prompts.

5. Document the additive author/runner contract.
   - Update the structured-results section of `docs/axe.md` to list `clan_summary`, its validation and same-raw-clan
     agreement rules, its pre-dedupe inheritance/promotion behavior, the 32 KiB and text-block safety restrictions, and
     its exact translation to the declaring `%clan(..., summary=[[...]])` directive.
   - Update the chop-package guidance and example in `docs/plugins.md` so authors repeat one identical summary on all
     members that share a clan template, while Axe remains the sole owner of concrete clan allocation and declaration.
   - Do not change release versions, ACE layout, or the later `toobig_split` plugin implementation.

## Validation

1. During iteration, run focused Rust tests for `axe_chop` and the PyO3 chop binding, rebuild/install the local Rust
   binding, and run the focused Python SDK, result-protocol, clan-launch, proposal-launch, and directive-extraction
   tests.
2. Run `just rust-check` from the SASE repository to format-check, lint, and test the linked Rust workspace.
3. Run `just install` so the editable environment and compiled `sase-core-rs` binding match the changed wire contract.
4. Run the mandatory full `just check` from the SASE repository and fix/re-run until it passes.
5. Recheck both the SASE and linked `sase-core` worktrees to ensure only intentional source/test/doc changes remain,
   then close only `sase-8l.1` with `sase bead update sase-8l.1 --status closed`. Verify the child is closed and the
   parent `sase-8l` is still open/in progress as originally managed.

## Acceptance Checks

- Legacy structured results without `clan_summary` validate, serialize, preview, and launch without generated-prompt
  changes.
- Unsafe or group-conflicting summaries fail atomically in Rust with a field-specific error before any launch.
- A summarized tail promoted after head dedupe declares the same summary exactly once on the allocated concrete clan.
- Directive extraction recovers the authored Rich markup, while joiner prompts and joiner metadata never claim a
  declaration.
- Dry runs expose the effective declaration summary and exact prompt but reserve no names and launch no agents.
- `just rust-check`, `just install`, and `just check` pass; `sase-8l.1` is closed and its parent epic is not closed.
