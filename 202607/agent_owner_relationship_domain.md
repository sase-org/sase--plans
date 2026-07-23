---
tier: tale
title: Explicit agent ownership and portable relationship domain
goal: SASE core and Python expose validated owner-aware name, family, hood, link-target,
  and whole-batch relationship operations that later global-hood phases can consume
  without dotted-string ownership inference.
bead: sase-8v.1
parent: sase/repos/plans/202607/global_agent_hoods.md
create_time: 2026-07-23 13:05:35
status: wip
---

- **PROMPT:** [202607/prompts/agent_owner_relationship_domain.md](prompts/agent_owner_relationship_domain.md)

# Plan: Explicit agent ownership and portable relationship domain

## Goal

Complete bead `sase-8v.1` by replacing machine-prefix inference at the shared backend boundary with explicit, validated
owner identity operations; canonical agent/family/hood classification; and a portable, whole-batch relationship
validation and destination-ID rewrite contract. Expose the domain through typed `sase_core_rs` bindings and a focused
Python facade while retaining the existing machine-hood APIs as deprecated migration shims for the later
local-persistence phase.

This is a tale because `sase-8v.1` is already one dependency-free phase of the parent epic. Its pure-Rust API, pyo3
surface, Python facade, and cross-language tests form one compatibility seam that should be implemented and validated
together by one coding agent. Do not close the parent epic or create follow-up beads.

## Current state and constraints

- `sase-core/crates/sase_core/src/machine_hood.rs` currently validates only `machine_name` and infers ownership by
  prepending, stripping, or classifying a known leading machine segment.
- `sase-core/crates/sase_core_py/src/lib.rs` exposes those helpers directly, and `src/sase/core/machine_hood_facade.py`
  adds current configuration discovery plus compatibility lookup policy.
- There is no explicit username-bearing owner value, no source-versus-target ownership classifier, and no canonical
  backend contract for global/local name transforms, archive decoration, family/hood structure, link targets, or batch
  relationship rewrite.
- Agent artifacts already carry relationship-like fields (`parent_agent_timestamp`, `parent_timestamp`,
  `retry_of_timestamp`, `retried_as_timestamp`, `wait_for`, family, and clan), but Phase 1 must define a portable,
  host-independent v2 validation/rewrite seam rather than copying runtime paths, PIDs, or workspace state.
- Shared domain behavior belongs in the linked Rust core. Python must remain a thin typed facade with no parallel
  parsing or graph implementation.
- Do not edit release-plz-owned versions, SASE memory files, generated instruction shims, configuration, persistence,
  sidecar publication/import code, commit footers, or TUI behavior in this bead.

## Design

### 1. Introduce a validated explicit owner model in `sase-core`

Add a dedicated pure-Rust agent identity/relationship module (split into focused submodules if that keeps the code
clear) and re-export its public contract from `sase_core::lib`.

Define:

- `AgentOwnerIdentity { username, machine_name }`, with construction/validation that always validates both fields.
- A source-owner representation that can carry either a v2 owner or a v1 machine with explicitly unknown username; never
  infer a username by splitting a dotted agent name.
- A serialized ownership classification enum with exactly these cases: `exact_owner`, `same_user_other_machine`,
  `other_user`, and `username_unknown_v1`.
- Typed, actionable validation errors that preserve which username, machine, agent name, run ID, container, or edge
  failed.

Username validation is a conservative, path-safe, dot-free identity segment:

- accept lowercase ASCII letters/digits, with `-` and `_` only internally;
- reject empty values, leading/trailing separators, uppercase, dots, path separators, `--`, control characters, empty
  segments, and reserved sidecar/internal namespace names;
- report the expected syntax and the rejected value.

Continue to reuse the existing canonical machine-name validator rather than creating a competing rule. Validate every
deserialized owner at API entry; deriving `Deserialize` must not provide an unchecked public path into domain
operations.

### 2. Add explicit global/local name operations

Implement operations whose owner inputs make provenance unambiguous:

- globalize a locally owned semantic name to `<username>.<machine_name>.<local-name>`;
- verify and globalize legacy input qualified by the supplied current machine, rejecting a different/malformed legacy
  hood instead of guessing;
- validate and strip a global prefix only against the explicitly supplied source owner;
- localize a verified global name for a supplied source and target owner:
  - exact owner -> bare local semantic name;
  - same username, other machine -> `<source-machine>.<local-name>`;
  - different username -> `<source-username>.<source-machine>.<local-name>`;
  - username-unknown v1 -> a conservative machine-qualified foreign spelling;
- classify source versus target through the four-case enum independently of name parsing.

Treat dismissal decoration as archive presentation state: strip at most one canonical `YYMMDD.` prefix before global
identity construction and expose that normalization directly. All transformations must reject empty/unsafe semantic
names, path traversal/components, malformed owner/name agreement, and accidental empty remainders. Exact-owner
verification, already-normalized inputs, localization, and dismissal normalization should be idempotent where their
typed input contract permits it.

### 3. Centralize family, hood, ancestry, and link-target classification

Add canonical helpers that operate on semantic names after archive normalization:

- parse only a terminal `--<role>` suffix into family base/member role; reject empty base/role and embedded or repeated
  family delimiters;
- return the solo name unchanged when no terminal family suffix exists;
- extract the first dotted local hood segment;
- test recursive membership by exact-or-`hood.` boundary so `foo` includes `foo.bar` but never `foobar`;
- synthesize ordered dotted ancestors (`foo`, `foo.bar`, `foo.bar.baz`) from solo or family-member names;
- select a deterministic transport link target: family member -> `families/<global-family>.md` plus a stable member
  anchor; solo agent -> `agents/<global-agent>/README.md` with no anchor.

The result types should be serialization-friendly and contain explicit kind/path/anchor data so later publication code
does not reproduce suffix or path rules. Validate every generated path component; do not use directory spelling as
ownership proof.

### 4. Define and validate a portable v2 relationship batch

Create a versioned, serde-backed wire with explicit owner, runs, structural containers, and relationships. The exact
Rust type decomposition may vary, but the JSON contract must make these concepts unambiguous:

- each run has a non-empty stable `source_run_id`, explicit `global_name`, and source owner agreement;
- containers have a typed `family` or `clan` kind, explicit global name/owner, and stable member references;
- relationships have a typed kind covering parent, workflow-parent, retry, and wait edges; targets are either stable
  source run IDs or explicitly global names, with required-versus-optional semantics where an external reference may
  legitimately be absent from the hood;
- no absolute paths, workspace numbers, PIDs, credentials, or arbitrary machine-local state enter this wire.

Whole-batch validation must happen before rewrite and must:

- enforce the supported schema version and conservative count/string limits;
- validate owners, global/local name agreement, run IDs, global reference names, family/clan container names, and path
  safety;
- reject duplicate source run IDs, duplicate global identities, duplicate/inconsistent containers, and duplicate
  one-to-one structural edges;
- reject dangling required run targets, illegal cross-owner references, self-edges, and container members absent from
  the run set;
- detect directed cycles for parent/workflow/retry/wait relationships where a cycle is invalid, with diagnostics that
  identify the relation and involved IDs;
- return a canonical validated summary/order suitable for later export/import callers.

Add a batch destination-ID rewrite operation that accepts a complete source-to-destination map, rejects missing or
duplicate destination IDs and unknown mapping keys, and rewrites every run-ID-bearing relationship/container member
through that one map. Preserve global-name targets and stable ordering. The result must clearly distinguish rewritten
destination IDs from source IDs so import code cannot accidentally persist the wrong namespace.

### 5. Expose typed pyo3 bindings

Extend `sase_core_rs` with bindings for:

- owner/username validation and ownership classification;
- globalize, legacy-globalize/verify, and localize operations;
- archive normalization, family parsing, hood/ancestor helpers, membership, and link-target selection;
- relationship schema version, whole-batch validation, and destination-ID rewrite.

Use ordinary Python scalars for scalar transforms and serde-converted dict/list payloads for typed result/wire values.
Convert every domain/serde failure to `ValueError` with the original actionable Rust diagnostic. Add module
registration, API-list documentation, and direct pyo3 tests that prove the functions are actually exported and preserve
the intended Python shapes.

Keep `validate_machine_name`, `qualify_machine_agent_name`, `strip_machine_agent_name`, and `machine_hood_of` exported
unchanged. Mark them as migration shims in Rust/Python documentation only; do not emit runtime deprecation warnings or
change current callers in Phase 1.

### 6. Add a focused Python identity facade

Create `src/sase/core/agent_identity_facade.py` as a thin application boundary:

- frozen dataclasses/enums for owner/source identity, classification, parsed family data, link targets, validated
  relationship summaries, and rewritten batches as useful;
- statically named `require_rust_binding(...)` calls so the required-binding audit can see every new dependency;
- `asdict`/mapping conversion only—no Python regex, dotted-name inference, graph traversal, relationship validation, or
  rewrite fallback;
- clear type hints and exported API names aligned with the Rust operations.

Do not replace `MachineHoodIdentity.current()` or migrate persistence callers yet. This facade is the explicit API that
Phases 2–5 will consume once nested identity configuration and local persistence exist.

## Test matrix

### Pure Rust

- Username/owner matrix: valid lowercase/digit/internal `-`/`_`; empty, uppercase, dot, slash/backslash, leading or
  trailing separator, `--`, control, and reserved namespace failures; reuse of all existing machine-name cases.
- Ownership/localization: exact owner, same user/other machine, other user with the same machine name, and
  username-unknown v1; explicit owner verification prevents dotted-string guessing.
- Globalization: dotted local names, family members, dismissal decoration, legacy current-machine qualification,
  malformed/foreign legacy qualification, owner/name mismatch, unsafe components, and round-trip/idempotence invariants.
- Family/hood: `foo.bar.baz--code`, solo/family parsing, ordered ancestors, top-level hood, recursive membership, and
  boundary false positives such as `foo` versus `foobar`.
- Link targets: family path/member anchor and solo README path with unsafe inputs rejected.
- Relationships: valid mixed parent/workflow/retry/wait and family/clan batch; duplicate IDs/names; dangling required
  targets; optional global targets; cross-owner references; missing container members; self/cyclic edges; unsafe and
  oversized payloads; complete rewrite; incomplete/duplicate/extra destination maps; deterministic output.

### Binding and Python facade

- Direct pyo3 tests for scalar errors/results, typed dict/list shapes, schema version, validation summary, and rewrite.
- Python facade tests with mocked binding lookup to prove every operation delegates without reimplementing logic.
- Integration tests against the rebuilt local extension for the full owner matrix, family example, malformed payloads,
  and a relationship rewrite round trip.
- Existing `tests/test_machine_hood_facade.py` and Rust machine-hood tests remain unchanged and passing, proving shim
  compatibility.

## Implementation sequence

1. Implement the validated owner/name/family domain and exhaustive unit tests in the linked `sase-core` checkout.
2. Implement the versioned relationship wire, whole-batch validator, graph checks, rewrite, and unit tests.
3. Re-export the new pure-Rust API, add/register pyo3 functions, and add direct binding tests.
4. Run Rust formatting early, then build/install the local extension into the primary repository virtual environment.
5. Add the thin Python facade and focused mocked/integration tests.
6. Run focused Rust/Python tests while iterating, then the full required validation below.
7. Confirm diffs are limited to the linked core and primary Python facade/tests, add concise implementation notes to
   `sase-8v.1`, close only `sase-8v.1`, and verify parent `sase-8v` remains open.

## Validation

Because this ephemeral primary workspace may have stale dependencies, run `just install` before repository-wide checks.
The final gate is:

1. `just install` (rebuilds/installs the linked local `sase_core_rs`);
2. `just rust-fmt-check`;
3. `just rust-clippy`;
4. `just rust-test` (workspace tests include the pyo3 binding tests);
5. focused Python facade/integration tests;
6. `just check`.

If formatting changes Rust sources during iteration, rerun all Rust gates after the final edit. Run the facade-focused
Python tests again after `just check` to guard against a stale extension or order-dependent state. No visual tests are
required because this phase changes no TUI rendering.

## Acceptance criteria

- Explicit validated username/machine ownership replaces dotted-name inference in every new API.
- Global/local transformations produce the specified exact-owner, same-user/other-machine, other-user, and unknown-v1
  spellings, including the same machine name under two usernames.
- Dismissal decoration never enters a canonical global identity.
- Family parsing, hood extraction/membership, ancestors, and family/solo link targets handle dotted names and only a
  terminal family role without boundary false positives.
- Portable v2 relationship batches reject malformed ownership, collisions, dangling required targets, illegal
  references, unsafe components, and forbidden cycles before returning any rewritten state.
- A complete destination map deterministically rewrites every run relationship and container member; incomplete or
  ambiguous maps fail atomically.
- New functionality is callable through `sase_core_rs` and the thin Python facade; legacy machine-hood symbols retain
  their existing behavior as documented migration shims.
- Rust fmt/clippy/workspace tests, binding tests, focused Python tests, and `just check` pass.
- Only `sase-8v.1` is closed; parent epic `sase-8v` remains open and no new beads are created.
