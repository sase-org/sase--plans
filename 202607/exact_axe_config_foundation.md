---
tier: tale
title: Exact AXE config composition and mutation planning
goal: AXE runtime loading and editor previews share one exact-key, provenance-aware
  Rust composition and mutation contract, while source-preserving writes reject stale
  plans and replace targets atomically.
bead: sase-8m.1
parent: sase/repos/plans/202607/axe_config_editor.md
create_time: 2026-07-22 12:04:55
status: done
---

- **PROMPT:** [202607/prompts/exact_axe_config_foundation.md](prompts/exact_axe_config_foundation.md)

# Plan: Exact AXE config composition and mutation planning

## Context and outcome

AXE currently has two configuration authorities. Generic layer discovery, schema validation, inventory, and scalar edit
planning already live in `sase-core`, but keyed lumberjack/chop composition remains in `src/sase/axe/_config_layers.py`.
Runtime loading conditionally invokes that Python implementation only when a map-form chop appears, while Config
Center's generic edit wire still derives key segments by splitting a dotted display path. The shared Python apply path
then writes previewed text directly, without proving the file is unchanged or using an atomic replacement.

Complete the `sase-8m.1` foundation by making Rust the single deterministic authority for AXE layer composition and
entry mutation planning, exposing exact key segments end to end, routing runtime loading through the same composed
result used for previews, and hardening the shared source-preserving apply transaction. Preserve the existing public
Python patch/import surfaces where tests and callers rely on them; this phase supplies frontend-neutral contracts for
later AXE modal work and does not add TUI actions, forms, or keymaps.

## Contract and behavior

- Treat lumberjack and base-chop identities as mapping keys carried in `Vec<String>`/segment arrays. Dotted text is a
  display/legacy scalar-path representation only; names containing dots, dashes, spaces, and other YAML mapping-key
  characters must round-trip without being split or normalized into a new name grammar.
- Compose the ordered AXE layer stack in Rust. Normalize legacy string/object chop lists into stable keyed entries for
  composition, retain lower-layer entries when a higher concatenate/patch layer contributes a map, honor replace-layer
  legacy-list behavior, deep-patch matching keyed entities, and retain the effective runtime shape plus exact-path
  provenance. Surface deterministic duplicate, invalid legacy entry, identity mismatch, migration, and strict AXE
  diagnostics with the responsible layer.
- Expose an inventory for every effective lumberjack and base chop, including its exact identity, effective object,
  enabled state, base-versus-expanded relationship metadata, field provenance, and each writable layer's raw sparse
  contribution and representation (`absent`, legacy list, or keyed map). Generated `for_each` instances refer back to
  their base chop; they are not independently mutable.
- Plan add/edit mutations from an exact entry selector, writable target layer, and ordered per-field set/unset
  operations. Apply those operations only to the target layer's raw sparse contribution. Reset removes that layer's
  field override; inherited fields are not copied. Promote a target legacy chop list to an equivalent keyed map only
  when an edit requires exact keyed mutation, preserving all in-layer entries and comments outside the rewritten
  subtree. Return a standard config write plan, effective entity before/after preview, candidate composed config,
  generic schema diagnostics, strict AXE diagnostics, and enough exact target information for the existing YAML diff
  path. Adding a map-form chop above built-in list defaults must leave every lower-layer chop effective.
- Make generic config edit requests accept exact `key_path` segments while preserving the current dotted `path` request
  for existing callers. Reject missing, empty, or contradictory path inputs rather than guessing. Preserve the current
  response fields and Python scalar/list/map behavior.
- Bind a preview to the target bytes it was derived from. Immediately before apply, compare a strong byte token and
  refuse a stale or mismatched target with a typed conflict result/exception. On a match, write a same-directory
  temporary file, preserve applicable existing mode bits, flush it, atomically replace the target, and clear config
  caches only after success. Failed writes or replacements leave the original file and cache state intact and clean up
  temporary artifacts; newly created targets receive normal secure/default creation semantics.

## Implementation

### Rust AXE composition domain

Add focused AXE composition/inventory/planning modules under `crates/sase_core/src/config/` (or a cohesive sibling
module if that keeps the domain clearer), reusing `ConfigLayerInputWire`, generic merge/path helpers, schema validation,
and the existing strict `axe_chop` validator rather than duplicating them. Introduce serde wire records for exact paths,
provenance contributions, entry selectors and representations, field operations, inventory entities, composition
results, and mutation plan results. Keep domain code free of PyO3 and file I/O.

Port the semantics from `src/sase/axe/_config_layers.py`, but store provenance and selectors as exact segment vectors
internally. Define deterministic diagnostic ordering and de-duplication. Preserve legacy-list identity/order while
normalizing, make cross-layer duplicate detection sensitive to concatenate versus replace behavior, and ensure strict
validation runs on both raw contributions (for attributable migration/duplicate errors) and the final effective AXE
section. Feed exact provenance into validation through an exact-path-aware lookup, retaining readable diagnostic paths
for compatibility.

Extend `ConfigEditRequestWire` with a backwards-compatible exact `key_path` option and centralize request path
resolution. Update the generic planner, preview lookup, write plan, and tests so explicit segments win only when valid
and dotted-only callers behave exactly as before. Export the new Rust APIs from `config::mod`/`lib.rs`, expose thin
JSON-in/JSON-out functions from `crates/sase_core_py`, register them in the extension module, and add binding contract
tests.

### Python facades and runtime parity

Add typed, Textual-free Python dataclasses/adapters for AXE composition, inventory, selectors, raw contributions, field
operations, and mutation plans. Serialize discovered `ConfigLayer` objects with the same helper used by generic config
inventory so runtime, Config Center, and future AXE editors pass identical layer inputs. Convert exact provenance to the
small per-chop field view expected by runtime models without parsing dotted identity strings.

Replace Python composition logic in `src/sase/axe/_config_layers.py` with compatibility wrappers over the Rust binding,
keeping patchable names needed by existing tests. Route `load_axe_config()` through the Rust composition result for all
layer stacks, rather than consulting `load_merged_config()` first and conditionally recomposing only map-form data. Keep
caching keyed to the existing config token and composed layer inputs, and keep expensive discovery off unrelated hot
paths. Ensure the data parsed into `AxeConfig` is exactly the same effective data returned in mutation previews.

Extend `plan_config_edit()` with an optional exact-path argument while retaining its dotted positional API. Add an AXE
entry planning facade that invokes the new Rust mutation planner, maps its standard write plan through the existing
chezmoi target resolver, applies the source-preserving YAML transformation, and produces a target-file unified diff. For
a promoted legacy chop list, rewrite only the necessary `chops` subtree; for map-form sparse edits, write the exact
entry contribution. Do not add editor state or UI-specific policy.

### Conflict-safe atomic apply

Capture the exact target bytes (including the absent-file state) and a strong digest/token in `EditPlanResult` when
planning. Introduce a typed conflict subtype/result distinguishable from validation and I/O failures. In
`apply_config_edit()`, re-read bytes immediately before mutation, compare them with the plan token, and abort before any
write or cache clear when stale.

Implement the successful path with an explicitly managed same-directory temporary file and `os.replace`: create parent
directories as needed, write and flush encoded bytes, preserve the existing target's permission bits where applicable,
fsync the file before replacement, and best-effort fsync the directory after it. Clean up the temp on every pre-replace
failure. Clear config caches only after the replacement completes. Keep Config Center callers compatible; the later
shared-editor phase can turn the typed conflict into reload/re-plan UI while the user's in-memory draft remains intact.

## Verification

In `sase-core`, extend `crates/sase_core/tests/config_parity.rs` or add focused AXE config contract tests covering:

- exact lumberjack/chop keys containing dots, dashes, spaces, and other valid mapping characters;
- default legacy list plus overlay keyed-map composition, field-level provenance, sparse inherited edits and resets,
  enabled state, and generated-instance-to-base identity;
- concatenate and replace layers, cross/in-layer duplicates, invalid legacy entries, map/list identity mismatches,
  migration diagnostics, deterministic diagnostic attribution, and target-layer list-to-map promotion;
- add and edit previews that preserve lower-layer defaults, generic schema validation plus strict AXE validation, and
  equality between the composition used by runtime and the planner candidate;
- generic exact `key_path` edits and unchanged dotted-path compatibility, including malformed/contradictory requests;
- serde/PyO3 binding round trips for every new request/result.

In `sase`, extend `tests/test_axe_lumberjack_config.py`, add focused AXE inventory/planner adapter tests, and expand
`tests/test_config_edit.py` for comment-preserving exact-key output, sparse map edits, legacy-list promotion, runtime
versus preview parity, stale existing-file and absent-file conflicts, mode preservation, cache-clear timing, atomic
replace failure, and temporary-file cleanup. Keep established generic Config Center planning/apply tests green.

Run Rust formatting, `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace` in the linked
`sase-core` repository. Then run `just install` so the local extension is rebuilt, run focused AXE/config tests, and run
the repository-required `just check`. Inspect both repositories' final diffs/statuses, close only bead `sase-8m.1` after
all checks pass, and leave parent epic `sase-8m` and all sibling beads untouched.
