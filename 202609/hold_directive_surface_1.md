---
tier: tale
title: Parse the hold directive across Rust and Python
goal: "The beta-gated %hold directive has one canonical Rust contract, survives typed
  agent and proc launch planning, and is parsed identically by Python with the required
  validation and plan-time safety diagnostics.

  "
size: medium
proposed_by: bbugyi200.athena.sase-11l.5.1.1
bead: sase-11l.5.1.1
status: done
---

- **PARENT:**
  [202609/hold_directive_surface.md](https://github.com/sase-org/sase--plans/blob/main/202609/hold_directive_surface.md)
- **BEAD:**
  [sase-11l.5.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11l/sase-11l.5.1.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-11l.5.1.1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-11l.5.1.1.md)
- **COMMITS:**
  - [a685c07](https://github.com/sase-org/sase-core/commit/a685c0725fba4b6391bfb8541a06f935480253d4)
    — feat(core): add hold directive contracts

# Plan: Parse `%hold` everywhere behind `agent_holds`

## Scope and invariants

Implement only bead `sase-11l.5.1.1`: directive parsing, typed launch wires and
diagnostics, shared editor metadata, Python adapters, and flag propagation. Do not arm
or release holds, capture pending targets, add confirmation UI, synthesize Hood
completion candidates, or edit memory. Shared parsing and selector semantics live in
`sase-core`; Python remains a thin adapter. Hold-free launch payloads must retain their
current serialized shape and digest, while a unit carrying a hold must round-trip and
change its digest.

The approved epic design explicitly sanctions the one `sase flag new agent_holds`
operation needed to create the beta flag's dedicated removal bead. Use that workflow
exactly once if the flag does not already exist, and create no other beads. Preserve the
flag's explicit disabled branch for the later removal phase.

## Implementation

1. In the linked `sase-core` repository, add a pure `hold_directive` module modeled on
   `queue_directive`. Define occurrence, argument, field, scope, error, and collection
   wires with serde defaults/omissions; parse and union repeated `%hold` occurrences;
   sort and deduplicate names, tribes, and hoods; enforce duplicate `ttl`/`scope`
   detection; and emit the approved typed errors for missing selectors, `all`, invalid
   scope or duration, unknown keywords, plus suffixes, malformed forms, and a disabled
   non-bare directive. Keep bare `%hold` inert while `agent_holds` is off. Reuse the
   shared proc-duration grammar, moving it to a neutral helper only if the module
   dependency requires it.

2. Add canonical formatting and selector expansion in Rust. The formatter must emit
   fields in the specified stable order and parse back to the same `HoldFieldsWire`.
   Selector expansion must map every authored name to names/clans/workflows, add it to
   families only when `parse_agent_family_name` accepts it, normalize tribes, validate
   hoods through the hold store's component-boundary rules (exposing
   `normalize_hood_vec` within the crate), and include supplied pending artifact
   directories only when `pending` is selected. Re-export the API from
   `crates/sase_core/src/lib.rs`.

3. Extend the Rust editor contract with `DirectiveValueRole::Hood` and a beta-gated
   `hold` directive directly after `queue`. Give it colon/parenthesized syntax,
   repeatability, Agent positionals, `pending`/`future` suggestions, and the exact
   `hood`, `scope`, `ttl`, and `tribe` keyword metadata. Update feature-flag lookup,
   examples, snippets, colon support, positional/keyword mixing, and the audited
   contract matrix. Do not add Hood candidate synthesis in this bead; the new role must
   fall through to static suggestions until the completion phase lands.

4. Thread `HoldFieldsWire` through the typed Rust planner. Collect and strip `%hold` for
   both agent and proc units, except for flag-off bare prose; add omitted-by-default
   `hold` fields to `AgentUnitWire` and `ProcUnitWire`; include canonical hold text in
   approval preview and agent dispatch-prompt reconstruction; and update all wire
   literals. Reject `%hold` with `%dispatch` and with repeat fan-out or retained
   `%repeat`. Add `hold-self` checks against an explicitly named unit's identity,
   family, or clan. Extend the existing wait-cycle graph with target-to-holder edges
   when another explicit unit matches an authored name, tribe, or hood, excluding the
   holder's kin, and report `hold-cycle` distinctly from `wait-cycle`.

5. Add pyo3 bindings for `collect_hold_fields`, `format_hold_directive`, and
   `hold_fields_to_selectors`, document them in the binding index, register them in the
   module, and cover their Python-visible JSON contracts. Land these Rust changes first.
   Then update the primary repository's core pin with `tools/ratchet_core_revision` and
   keep both binding-integrity tools green.

6. Scaffold and propagate the `agent_holds` beta flag using the exact description,
   enabled, disabled, and removal conditions from the approved epic design. Paste the
   generated registry entry, sync the feature-flag schema, add an
   `agent_holds_enabled()` helper beside the Python directive adapter, include the flag
   in launch/editor flag snapshots and the fixed completion catalog, and pass an
   `SASE_AGENT_HOLDS` environment pin from the Python LSP launcher into the Rust LSP
   server's enabled flag list. Verify name completion hides `hold` while the flag is
   off.

7. Add `src/sase/xprompt/hold_directive.py` with the frozen `HoldDirective` value,
   disabled-message constant, Rust-backed collector, formatter, and selector adapter.
   Add `hold` without an alias to the known and multi-value directive tables, collect
   occurrence payloads through a shared queue/hold occurrence helper, and attach
   `PromptDirectives.hold`. Preserve fenced, backticked, and disabled-region text; keep
   flag-off bare `%hold` as inert prose while rejecting every non-bare form with the
   flag-specific error. Extend dispatch scanning and repeat validation with the
   prescribed `%hold` rejections.

8. Mirror the optional hold payload in the Python agent/proc launch dataclasses,
   from-dict hydrators, and JSON conversion. Update directive contract parity and the
   focused Python/Rust tests for both flag states, every selector spelling, unions and
   duplicates, canonical round-trips, selector expansion (including a `--` shell name
   excluded from families), agent/proc wires, preview and digest behavior,
   dispatch-prompt restoration, self/cycle/repeat/dispatch diagnostics, and literal
   zones. Keep all existing hold-free snapshots byte-compatible.

## Verification

Run targeted Rust tests for `hold_directive`, `agent_launch`, `agent_launch::admission`,
and `editor`, plus the pyo3 binding checks. In the primary repository, run the focused
xprompt, dispatch, repeat, wire-contract, LSP, feature-flag, and typed-launch tests. Run
`just fix`, then the required `just check`; use `just check-full` only if the scoped
gate escalates or reports unusual selection, because the parent land agent owns the
combined epic's exhaustive gate.

Before closing, run `sase bead epic-symbols sase-11l.5.1.1` and resolve every remaining
entry or re-key its Justfile ownership to a still-open parent/later phase. Close only
`sase-11l.5.1.1` with a note naming the successful Rust, binding, targeted Python, and
`just check` verification; leave `sase-11l.5.1` and all ancestors open.
