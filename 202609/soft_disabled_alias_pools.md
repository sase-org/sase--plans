---
tier: tale
title: Enforce soft-disable policy throughout pooled model launches
goal:
  Keep soft-disabled providers out of automatic pool launches when an eligible
  non-soft-disabled pool member can serve the request, including reservation redemption
  and provider-priority interactions.
size: medium
proposed_by: bbugyi200.athena.0jw
create_time: 2026-09-12 05:25:13
status: wip
---

# Enforce soft-disable policy throughout pooled model launches

## Scope and size

Implement this as one medium-size tale spanning `sase` and `sase-org/sase-core`. The
investigation has isolated two related, reproducible policy gaps. One coding agent can
implement their shared eligibility rules, update the Python adapters, and add
integration coverage. An epic is unnecessary.

Open `sase-core` through the repository skill before reading or modifying it. If the
linked name is unavailable, use `sase repo open gh:sase-org/sase-core` with an audit
reason and use the returned checkout. Shared routing policy belongs in
`crates/sase_core`; expose it through `crates/sase_core_py` and thin Python glue.

## Evidence and diagnosis

The shipped `@large` primary pool contains Claude Opus and Codex Sol, with Grok in a
last-resort tail. `@xlarge` has the same structure using Claude Fable and Codex Astra.
Read the actual expressions from `src/sase/llm_provider/model_alias_defaults.yml`; do
not change these defaults.

At investigation time, the authoritative and read-only routing snapshots agreed: Claude
and Grok had active manual soft disables, Codex was registered with an available CLI,
and provider priority was absent. There were no active temporary alias overrides. Fresh
resolution selected Codex for both aliases.

Recent launch metadata establishes that these were pooled selections:

- Agent `0jq.f0` recorded `model_alias=large`, origin `default_model`, and a redeemed
  reservation for `claude/opus`. Its prompt-step record and `exec_llm_provider` also
  recorded Claude.
- Agent `sase-zr.land` recorded `model_alias=xlarge`, origin `directive`, and an
  unredeemed reservation for `claude/claude-fable-5`. It was still waiting during the
  metadata inspection; this record establishes reservation, not invocation.

The metadata files were under the project's runtime artifacts in
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/12/20260912050157/` and
`20260912050851/`. Published `agent:` artifact pages were unavailable, so the live
metadata and CLI detail views supplied the evidence. Neither metadata record preserves
the complete routing context at reservation creation. Do not claim that historical
priority, Codex availability, or the original reason for reserving Claude has been
established.

Two defects are independently reproducible without invoking a provider or changing
machine-global routing/cursor state:

1. **Reservation redemption bypasses soft-disable exclusion.** In
   `src/sase/llm_provider/launch_selection.py`, `launch_selection_from_reservation()`
   peeks the current selection but compares only cursor owner, alias trail, and origin.
   It then accepts the old target whenever its availability is not `UNAVAILABLE`. A
   Claude reservation remains redeemable when Claude is `SPARING` and Codex is
   `PREFERRED`. With one shared live context, both aliases produced
   `fresh: codex, reserved: claude`. Bootstrap reserves before dependency/runner-slot
   waits in `src/sase/axe/run_agent_directives.py`, so reservations must be checked
   again immediately before use.
2. **Provider priority loses the distinction between real soft disables and ordinary
   backups.** Rust's `classify_provider_availability()` in
   `crates/sase_core/src/provider_priority.rs` classifies both as `Sparing`, while
   preserving their distinct provenance and actual-disable records. Python's
   `_selector_member_states()` discards that distinction. For Claude soft-disabled,
   Codex enabled, and priority assigned to Grok in the last-resort tail, the primary
   pool becomes `[SPARING, SPARING]`. `pool_availability_mask()` therefore admits both,
   selecting Claude at cursor zero for both shipped aliases. The same issue occurs when
   the priority provider is absent or unavailable and healthy backups compete with
   actually soft-disabled providers. This reproduction used a synthetic priority
   context; it is not evidence that priority was active during the reported launches.

The existing focused suites all passed: 38 tests across soft-disable routing, priority
routing, last-resort aliases, and composed reservation launches. They cover ordinary
soft filtering and hard-unavailable reservation rejection, but miss the combinations
above.

## Required behavior

Apply policy to the primary pool independently of any last-resort tail:

| Situation                                                                              | Expected behavior                                                                      |
| -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Claude actually soft-disabled; Codex usable without a soft disable                     | Fresh selection and redemption use Codex.                                              |
| Priority provider is only in the tail, absent, or unavailable                          | Usable non-soft-disabled primary members still outrank actually soft-disabled members. |
| A usable, non-soft-disabled priority provider belongs to the primary pool              | Fresh selection prefers it, preserving the existing priority behavior.                 |
| All usable primary members are actually soft-disabled                                  | They remain usable and rotate normally; do not divert to the tail.                     |
| Only a soft-disabled primary member is usable                                          | Use that member even if the tail is healthy.                                           |
| Every primary member is unavailable                                                    | Use the existing ordered tail behavior without consuming the pool cursor.              |
| An existing reservation becomes only a priority backup, without an actual soft disable | Keep the reservation redeemable, as the current priority-reservation test requires.    |
| Direct provider/model, concrete override, or ordered `A                                |                                                                                        | B` fallback selects a soft-disabled provider | Preserve the existing explicit/fallback behavior; soft disable is not a global execution ban. |

An unchanged valid reservation redeems without consuming again. A reservation
invalidated by a real soft disable is marked spent and replaced by exactly one consuming
resolution. Keep the original spent cursor slot; no global rewind is safe when other
agents may have reserved intervening slots.

## Implementation

1. **Add narrow Rust-owned eligibility operations.** Use full classified provider
   records, including actual disable and provenance, rather than tri-state values alone.
   Keep the existing provider-level wire classification and UI labels compatible. A pure
   pool-mask operation should prefer usable members without an actual soft disable,
   apply priority within those members, and fall back to actual-soft members only when
   that pool has no usable non-soft member. A corresponding reservation eligibility
   operation must reject an unavailable reserved member and an actually soft-disabled
   reservation when a usable non-soft primary member exists. It must preserve the
   priority-only reservation contract above. Share the underlying policy primitives
   rather than reproduce these comparisons in Python. Validate malformed inputs and
   member indices at the Rust/PyO3 boundary; keep these operations free of filesystem
   and cursor I/O.

   Keep fresh-selection and reservation admission distinct: a fresh priority mask must
   not itself invalidate an already-reserved ordinary backup. Conversely, priority
   provenance must never disguise a real soft disable as an ordinary backup. Additive
   binding functions are sufficient; avoid a global enum or wire-version change merely
   to represent this pool-specific policy.

2. **Preserve routing information through all selector paths.** Update
   `model_alias_resolution_resolve.py`, `model_alias_resolution_selector.py`, and the
   raw-selector lane in `model_launch_settings.py` to use the Rust-derived pool
   eligibility consistently. Update `load_balancing.py` and its adapters as needed so
   fresh selection, diagnostics, and cursor-owner reporting use the same pool mask. Do
   not evaluate tail members when deciding whether a primary soft member can be spared.
   Preserve weighted schedules, alias-owner cursor sharing, fingerprints, and ordered
   fallback behavior. Keep display peeks non-consuming and reuse a captured routing
   context rather than recapturing provider state for each member.

3. **Revalidate the actual reserved pool member.** In
   `launch_selection_from_reservation()`, retain the existing structural, alias-origin,
   and cursor-owner checks. Resolve the owning pool's members with the same captured
   context and call the Rust reservation policy for the reserved target. Account for
   delegated alias owners and namespaced scalar setting pools; use the active launch
   overrides and effective expression, not a separate config lookup that ignores them.
   Reject a reservation that cannot be matched to its current primary pool. Do not
   compare the reserved target directly with the next round-robin winner: intervening
   reservations legitimately change the cursor. Preserve reserved effort and provenance
   when accepting it.

4. **Exercise the existing replacement path.** `resolve_prompt_step_launch_selection()`
   in `src/sase/xprompt/workflow_executor_steps_prompt_launch.py` already marks an
   unredeemed invalid reservation spent, skips stale metadata inheritance, and performs
   one fresh consuming resolution. Reuse that path. Verify that root metadata,
   prompt-step markers, saved chat metadata, and the provider call all agree on the
   replacement provider/model/effort. Preserve the explicit pipe inheritance contract
   and ordinary direct-provider dispatch semantics.

5. **Document the precedence and reservation boundary.** Update the temporary provider
   disable, provider priority, and pool-reservation explanation in `docs/llms.md`: real
   soft disables remain distinct from priority backups, and a queued pool reservation
   can be invalidated before invocation. This is a routing correctness fix; no new CLI
   flags, model defaults, or live provider state changes are required. Do not restart or
   drain the user's agents as part of implementation.

## Regression tests and verification

First add failing tests demonstrating the two reproductions; then implement the fix. Pin
provider installation facts and use isolated runtime state. Do not invoke actual
provider CLIs or advance the user's machine-global pool cursor.

- In Rust and PyO3 tests, cover the behavior table, mask/index validation, actually soft
  priority targets, and preservation of existing wire records.
- Extend `tests/llm_provider/test_provider_disable_soft_routing.py` and
  `test_provider_priority_routing.py` for a non-soft backup competing with a real soft
  disable when the priority provider is in the tail, outside the pool, or unavailable.
  Assert fresh selection and selector diagnostics agree.
- Cover both shipped `@large` and `@xlarge` expressions, explicit alias directives,
  default-model alias resolution, delegated owners, and a raw scalar selector. Use the
  existing `real_model_alias_defaults` fixture where shipped expressions matter; the
  normal test environment freezes alternate defaults.
- Extend `tests/test_pooled_alias_single_consumption.py` with bootstrap followed by a
  soft-disable change before redemption. Include Claude already soft while Codex is
  initially unavailable and then becomes available, and stale Claude reservations
  injected under the reported live context. Assert replacement, reservation marking,
  cursor consumption, and metadata/provider agreement.
- Keep tests proving unchanged reservations redeem without consuming, intervening
  launches do not invalidate otherwise eligible reservations, priority-only backups
  remain redeemable, weighted accounting stays correct, all-soft pools remain usable,
  and hard-disable/last-resort/direct-selection behavior remains intact. Test that a
  healthy tail alone does not invalidate a soft primary reservation.

Run the targeted Python suites, including weighted-pool, launch-setting, and
alias-origin coverage as affected. Build/install the modified Rust binding into the
coding workspace's Python environment using the repository's supported
`SASE_CORE_DIR`/`just rust-install` workflow, then test the Python adapters against that
binding. Do not mistake an installed older wheel for verification of the new API.

Run `just check` in `sase-core` (or `./scripts/check.sh`) so PyO3 binding tests are
included; `cargo test -p sase_core` alone is insufficient. Run `just check` in `sase` as
required by the lint/test memory. Read the TUI performance memory before changing shared
display-path behavior, and keep new eligibility calls pure and bounded. Use the monitor
skill for long checks and for `just check-full` if the repository's broadening rules
require it. Do not hand-edit release versions.

Completion requires both reproductions to choose Codex while all compatibility cases
pass, with no implementation changes to the configured alias pools or the user's runtime
disable settings. Report the confirmed defects separately from the historical context
that the old metadata cannot reconstruct.
