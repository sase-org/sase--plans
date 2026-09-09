---
tier: epic
title: Finish toobig_split release integration and skipped-first live acceptance
goal:
  Published installs carry the repaired typed identity contract and durable live
  evidence proves skipped-first chop promotion.
parent_bead: sase-so
phases:
  - id: release_contract
    title: Ratchet the published core contract
    depends_on: []
    size: small
    description:
      "release_contract: ratchet SASE to the newest complete compatible core release and
      verify the minimum-wheel path."
  - id: live_skip
    title: Perform the skipped-first live drill
    depends_on:
      - release_contract
    size: xsmall
    description:
      "live_skip: run a controlled live AXE batch and capture durable skipped-first
      clan-promotion evidence."
proposed_by: bbugyi200.athena.sase-so.land
bead_id: sase-so.5
create_time: 2026-09-09 19:51:55
status: wip
---

- **PROMPT:**
  [prompts/202608/toobig_split_final_integration.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/toobig_split_final_integration.md)
- **PARENT:** [202608/toobig_split_identity_tribe.md](toobig_split_identity_tribe.md)
- **BEAD:**
  [sase-so.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-so/sase-so.5.md)

# Finish `toobig_split` release integration and skipped-first live acceptance

## Context

Epic `sase-so` restored grouped identity in the Rust/Python typed-launch wire, promoted
the first eligible AXE chop proposal to clan declarer, and changed `bugyi-chops` to emit
keyed basename member templates. The land audit verified the implementation commits and
current source in all three repositories:

- `sase-core` commit `8d51bd8` added the grouped `AgentUnitWire` contract and was
  released in v0.31.10. Later commit `1d3c9c6` fixed the shared `[[...]]` closing rule
  used by `%clan(..., summary=[[...]])`, and was released in v0.31.11.
- SASE commits `abefcc4fb` and `4041c17e4` mirror/reconstruct the wire and durably
  promote the first proposal that actually reaches dispatch.
- `bugyi-chops` commit `22b3db5` emits `<basename>.{@<path-digest>}` templates.
- Current focused verification passed: 126 Rust agent-launch tests, 23 SASE
  identity/admission tests, and 95 plugin tests against the current SASE/core tree.
  `just check` also passed and escalated its scoped lane to the full suite.

The land audit found two remaining acceptance/integration items. First, SASE still
declares `sase-core-rs>=0.31.0,<0.32.0`; the published v0.31.0 wheel predates this
epic's wire, and `tools/probe_core_floor` reports the declared floor as
`stale_actionable`. Second, rollout run `20260824T091645_021963` proved a live 4/4
eligible clan, but could not exercise skip-to-declarer promotion because every target
was still above the 700-line condition floor and an overlapping `toobig-*` clan was
active. There is no active `toobig-*` agent at the time of this audit.

Two unrelated coordinator defects proposed by `sase-so.4` were deliberately routed as
`DISCOVERED ISSUE` notes to their causal still-open epic `sase-s6`: stale sidecar PID
liveness and archive-scale agent-wait polling. Do not absorb those fixes into this plan.

## Implementation

### Ratchet the published core contract

1. Recheck the newest complete `sase-core-rs` PyPI release with
   `tools/ratchet_core_window --report-only`; it should be at least v0.31.11 and must
   contain both the grouped typed identity wire and the text-block parser fix.
2. Use the repository's ratchet tool (not hand-edited lock entries) to update
   `pyproject.toml` and `uv.lock` to the newest complete compatible 0.31.x release.
   Review the generated diff and refuse unrelated transitive churn unless the tool's
   documented transitive-lock refresh is required and justified.
3. Run `just install`, the focused identity/admission suites, the published-minimum core
   validation/probe, and `just check`. The minimum-wheel path must expose the grouped
   identity bindings and parse the clan summary forms used by this epic; an editable
   checkout alone is not sufficient evidence.

### Perform the skipped-first live drill

1. Recheck that no live or queued `toobig-*` clan exists immediately before the drill.
   If one exists, wait through the supported SASE mechanism; do not dismiss another run,
   edit clan state, or overlap swarms.
2. Run one controlled, genuinely live AXE typed-admission batch through the production
   dispatcher and real agent launcher, shaped like `toobig_split` with a shared
   `toobig-@` clan, keyed basename members, repeated `tribe=chop`/summary metadata, and
   a sequential wait chain. Arrange at least three units so the first unit's `%if`
   returns skip, the second and third are eligible, and their prompts are harmless,
   bounded verification work. Do not substitute a mocked launcher or a pure unit test.
3. Verify from durable request/journal/receipt data and live agent metadata that:
   - unit 1 is `skipped` and allocated no agent, runner slot, or workspace;
   - unit 2 is the sole declarer, named `toobig-<token>.<basename>.<token>`, with the
     concrete clan, `tribe=chop`, `clan_tribe=chop`, and the intended summary;
   - unit 3 joins that exact clan and is not placed under `@default`;
   - admission completes with one skipped unit, two launched units, no condition or
     launch errors, and no second declaration after restart/reconciliation.

4. Let the verification agents settle through normal lifecycle controls. Do not
   hand-edit run history, receipts, clan registry state, or once-per state.
5. Capture the command, run/request ID, receipt, relevant journal rows, and filtered
   agent metadata as explicit artifacts attached to the child plan bead. Add a close
   note that identifies the core floor selected and the exact live identities/outcomes.

## Acceptance

- A normal published SASE install can no longer resolve a core wheel older than the
  grouped identity and clan-summary behavior this epic requires.
- Current-core and published-minimum verification both pass, including the focused Rust,
  SASE, and plugin bridge suites.
- Durable live evidence proves skipped-first promotion with one declarer and one joiner
  in the same concrete `@chop` clan, with no skipped-unit resources and no member under
  `@default`.
- The child plan contains no fixes for the two `sase-s6` coordinator follow-ups.
