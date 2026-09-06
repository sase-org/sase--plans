---
tier: tale
title: Define portable fleet identity and operation contracts in sase-core
goal: Provide one validated, versioned, transport-free domain contract for remote
  agent identity, owner-resolved records, counts, cursors, operation replay, connection
  plans, and elapsed/freshness semantics, exposed to Python through sase_core_rs.
size: medium
proposed_by: bbugyi200.athena.sase-xe.2
bead: sase-xe.2
status: done
---

- **PARENT:** [202609/remote_dispatch_fleet.md](remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.2.md)

# Portable fleet identity and operation contracts

## Scope and tier

Implement phase `sase-xe.2` as one medium tale. The approved epic has already fixed the
product and crate boundaries, and the remaining work is a cohesive contract surface
whose types depend on each other. Splitting identity, projections, counts, cursors, and
operation keys among separate agents would create incompatible version-1 wire shapes.
One coding agent can implement the transport-free Rust module, its PyO3 exposure, the
cross-language smoke tests, and the core revision integration as one reviewable change.

Open the linked `sase-core` repository through `/sase_repo` and use only the path that
it returns. Repository paths below are relative to either `sase-core` or the primary
`sase` repository, as named explicitly. Do not add Tokio, Reqwest, Axum, a network
client, or another transport/runtime dependency to `crates/sase_core`. Do not implement
gateway routes, enrollment authentication, durable operation journals, provider plugins,
follow persistence, or TUI presentation in this tale; later epic phases own them.

The authoritative design is `plan:202609/remote_dispatch_fleet.md` and its research
source
`research:202609/remote_dispatch_and_fleet_focus/remote_dispatch_and_fleet_focus.md`.
Its settled rules apply throughout: provider metadata is not identity, human names are
labels, only an owner resolves local liveness and content availability, viewers never
receive actionable PIDs or paths, feed cursors are distinct from mutation preconditions,
and one running-count definition must serve every frontend.

## Contract organization and invariants

Add a focused public module under `crates/sase_core/src/` (prefer `fleet_contract.rs`,
or a small `fleet_contract/` module if separation materially improves readability),
export it from `lib.rs`, and expose its public wire types, constants, validators, and
pure decision functions. Use the repository's established serde conventions: snake_case
enum values, `deny_unknown_fields` on new externally supplied records, explicit schema
versions, deterministic ordered collections, and actionable typed errors. Keep the
version-1 JSON shape stable and make invalid or future-version input fail closed rather
than being silently coerced.

Define a single fleet contract schema-version constant and, where independent evolution
requires it, named sub-contract version constants. Every top-level record crossing a
process or protocol boundary carries a version. The PyO3 boundary remains dict-in /
dict-out and calls core validators/logic; it must not duplicate validation in binding
code. Reuse the existing generic Python/JSON conversion helpers and release the GIL with
`py.allow_threads` for filesystem work such as installation-identity persistence.

Document these distinctions in rustdoc next to the wire types, not only in tests:

- installation ID versus machine network identity, configured `id.machine_name`,
  provider reference, and display alias;
- stable logical agent/family identity versus exact shell/run instance identity;
- lifecycle status versus owner-resolved process liveness, connection health, and viewer
  observation freshness;
- event-feed position versus resource/action revision used as a mutation precondition;
- operation idempotency within an acceptance window versus a promise of indefinite or
  exactly-once execution.

## Implementation

### 1. Installation identity and hierarchical locators

Create a versioned per-user installation identity store under the supplied SASE home,
using a dedicated filename, a bounded file size, the existing store-lock conventions,
mode-restricted directories/files, and crash-safe atomic replacement. A missing store is
created once; concurrent creators converge on one persisted identity. A malformed,
oversized, unsupported-version, or unexpectedly replaced store is never overwritten by
an ordinary ensure/read call.

Installation IDs are opaque, normalized, stable values with a recognizable versioned
prefix and enough entropy/collision resistance for cross-machine use. They are identity,
not credentials: do not embed username, hostname, machine name, provider, endpoint, or
filesystem location. Reuse existing dependencies and standard-library facilities; do not
add a runtime or transport dependency merely to generate IDs. Make generation injectable
at the internal boundary so tests are deterministic.

Expose explicit lifecycle operations rather than encouraging callers to edit the file:

- ensure/load returns the current record and whether it was newly created;
- rotate creates a distinct identity only when the caller supplies the expected current
  identity, records rotation provenance/reason, and returns old/new identity metadata;
- migrate/adopt preserves a supplied valid installation identity for reinstall/clone
  recovery only under an explicit expected-state precondition.

Rotation and migration are hooks, not automatic clone detection. Preserve enough
metadata (creation/rotation time and prior identity where appropriate) for later repair
UX without treating the history as a credential or alias.

Define composable locator wires rather than one overloaded name string:

- origin locator: installation identity;
- project locator: origin plus a portable project ID (not a local checkout path);
- logical locator: project plus logical agent ID and optional stable family ID;
- exact instance locator: logical locator plus shell ID and run/attempt ID.

Validate every segment as non-empty, bounded, control-character-free identifiers. The
logical locator has a deterministic canonical key for set/map deduplication. Exact
mutation targets require the instance locator. Keep machine-hood names from
`machine_hood.rs` as validated labels/lookup conveniences only; add an explicit helper
that associates an owner-qualified display name with a locator without deriving origin
identity from that name.

### 2. Owner-resolved summary/detail records and capabilities

Define versioned resolved summary and detail wires as safe projections over existing
`AgentMetaWire` / `AgentArtifactRecordWire`, not a duplicate artifact source of truth.
Provide a pure projection request that combines an existing artifact record with an
origin/project identity and explicit owner-resolution facts. The owner facts carry the
result of liveness/content checks; the projection function performs no PID check, path
open, marker cleanup, or index mutation itself.

The compact summary must contain the exact and logical locators, human display labels,
safe project/model/provider/status/intent fields needed for a fleet row, observation
time, row/resource revision, distinct lifecycle and liveness values, capability set, and
safe bounded content metadata. The lazy detail extends or embeds the summary and adds
only protocol-safe detail fields and opaque content handles. A handle may carry an
opaque ID, kind, revision/digest, byte length, and range/growth support; it never
carries an absolute/relative filesystem path, PID, process-group ID, raw bearer token,
or auth header. Do not serialize the source `artifact_dir`, `project_dir`,
`project_file`, `workspace_dir`, marker paths, response/output paths, or local process
IDs into either resolved record.

Use closed enums for settled version-1 lifecycle/liveness/health/freshness states and a
normalized deterministic capability set for actions/content. Capability names must be
validated and deduplicated; resource capabilities remain separate from host/protocol
capabilities so later authorization can intersect them honestly. Unknown future enum
values fail the version-1 validator rather than being interpreted as healthy/running.

Make the projection reject inconsistent owner facts, including a live terminal record,
an actionable capability without an exact instance, a content capability without its
opaque handle, a row revision from a different logical identity, or a handle containing
path-like/secret-bearing data. Cover the representable stale case explicitly: a record
can say it was running when observed while current connection health is offline and
freshness is stale/unknown.

### 3. One logical-agent count definition

Add a pure count input/output contract over owner-resolved summaries. Centralize the
running predicate so later CLI, gateway summaries, Focus, and Fleet do not each
reimplement it. Mirror the existing `sase_agent_status_counts` semantics:

- consider agent shells only; tree/container headers, monitors, gates, procs, and
  historical shells never count as running LLM agents;
- count the effective `Running` bucket only when it is not dismissable;
- count `Starting` only when it is explicitly the concrete agent projected from a
  container;
- select the current exact instance for each logical locator and deduplicate sequential
  family handoffs, rejecting ambiguous equal-revision competing current instances;
- report waiting/attention agents and occupied runner slots as separate numbers rather
  than folding either into `running`.

The result should expose at least distinct logical-agent total, running, waiting,
attention, and occupied-slot counts plus the revision/observation basis used. Define and
test deterministic current-instance ordering using the explicit row revision and exact
locator, not display name or vector order. Do not add Focus/Fleet membership or
partial-host aggregation here; the later follow-store phase applies this shared host
count to those scopes.

### 4. Cursors, operation keys, connection plans, and time rules

Define a store cursor as `(store_generation, sequence)`. Add a pure replay classifier
whose request includes the current generation, newest sequence, oldest replayable
sequence, and whether deletion/tombstone history is complete. It returns a typed result
such as current/replayable/resync-required and the reason. Generation mismatch, sequence
ahead of the authority, replay-ring gap, or incomplete deletion history must request a
bounded authoritative resync. A zero/initial cursor is explicit. Do not reuse the cursor
as an action precondition; define a separate resource revision type.

Define operation-key and payload-fingerprint types with strict validation. The key is
scoped by authenticated controller identity and has an opaque operation ID; the payload
fingerprint is a full lowercase SHA-256 digest over canonical JSON with a domain
separator. Define acceptance-window and durable-record/receipt wires plus a pure
decision function covering:

- unseen in-window key and payload: accept as new;
- same scoped key and fingerprint: return the original accepted/pending/settled receipt;
- same scoped key and different fingerprint: conflict;
- same operation ID under another controller: a distinct scoped key;
- expired/tombstoned key: reject as expired, never accept as a fresh operation.

Keep persistence and crash-boundary enforcement out of this phase; `op-journal` will
store these records and act on the decision. Validate finite, ordered timestamps and
exact target/resource-revision preconditions in the typed request.

Define a serializable connection-plan wire containing only routing data: provider
reference, validated absolute HTTPS endpoint, credential reference, pinned installation
identity, connection kind, and explicit TLS/trust settings. Support system roots and
opaque CA/server-name references as data, while rejecting inline credentials, raw auth
headers, URL userinfo, fragments, insecure HTTP, unsupported schemes, missing identity
pins, and ambiguous trust modes. Provider reference and credential reference are bounded
identifiers; provider choice never participates in locator equality.

Finally, expose pure elapsed/freshness helpers with two separate inputs:

- owner-relative runtime duration uses owner wall-clock start/stop/observation values,
  clamps only documented clock anomalies, and never subtracts clocks from two hosts;
- cache age uses viewer-supplied monotonic elapsed time. When a viewer restarts and the
  monotonic basis is unavailable, classify freshness conservatively as unknown/stale
  instead of reconstructing it from the owner's wall clock.

Reject NaN/infinity and negative acceptance windows. Put threshold choice in request
data so the core returns a classification rather than hard-coding a UI refresh period.

### 5. PyO3 exposure and primary-repository integration

In `crates/sase_core_py/src/lib.rs`, add dict-in/dict-out bindings for the
schema-version query and each stateful or pure operation above: installation identity
ensure/rotate/ migrate, resolved-record projection/validation, count aggregation, cursor
replay classification, operation fingerprint/decision, connection-plan validation, and
time classification. Prefer a small shared typed conversion helper with wire names in
error messages. Map domain validation to `ValueError` and filesystem/locking failures to
`RuntimeError` consistently with nearby binding code. Register every binding in the
module and add it to the module-level exposed-surface documentation.

In the primary `sase` repository, add focused Python smoke tests that obtain every new
entry point through `sase.core.rust.require_rust_binding`; do not introduce a Python
fallback or duplicate Rust models. Exercise representative nested dict round trips and
error paths against the locally built extension. Add the new mandatory binding names to
`tools/validate_sase_core_rs` / its fixture-driven tests only if that validator's
contract requires explicit inventory; preserve the existing missing-binding diagnostic
style.

Integrate the linked core revision according to the repository's host-owned two-repo
workflow: the final `sase-core-revision.txt` must identify the committed core containing
these bindings, and `tools/validate_sase_core_rs` must exercise that build. Do not hand
edit package versions or dependency version pins; release-plz owns them. If the
host-owned finalizer must commit/publish the core before the pin can be resolved, follow
the established linked-repository finalization/ratchet sequence rather than pinning an
unrelated remote HEAD.

## Regression and contract coverage

Keep most semantic coverage in `crates/sase_core` next to the new module, with PyO3
conversion/registration tests in `crates/sase_core_py`, and a compact end-to-end Python
smoke suite in `sase`.

Cover at least:

1. Installation store creation/read idempotency, deterministic injected generation, two
   concurrent creators, mode/atomic-write behavior, rotate/migrate expected-current
   fencing, corrupt/oversized/future files left unchanged, and distinct identity from
   machine/provider/display fields.
2. Locator canonical keys and serde round trips; same human name at two origins remains
   distinct; provider/alias changes leave identity stable; malformed/oversized/control
   character segments and name-as-instance targeting are rejected.
3. Projection from representative `AgentArtifactRecordWire` values, including active,
   waiting/question, terminal, family handoff, monitor/gate, stale-offline, and missing
   optional metadata. Serialize every output and assert forbidden local fields/values
   are absent recursively. Test inconsistent owner facts and invalid capabilities/
   handles fail closed.
4. Count parity matrices for running/dismissable running/container-projected starting,
   waiting/attention, monitor/gate/proc exclusion, occupied-slot separation, duplicate
   rows, current family handoff, and ambiguous revisions. Assert input order does not
   affect output.
5. Cursor initial/current/replay/gap/ahead/generation-change/tombstone-loss cases, with
   resync reasons and boundary sequences.
6. Operation canonical fingerprint stability under JSON object-key reordering;
   controller scoping; same-key replay; conflicting payload; pending and settled receipt
   replay; exact-target/precondition mismatch; expiry boundary; and expired tombstone
   rejection.
7. Connection plan round trip and provider-switch identity invariance; reject HTTP,
   userinfo, fragments, inline secrets/headers, invalid pins, and conflicting trust
   options.
8. Owner-relative elapsed duration, clock anomaly behavior, viewer monotonic fresh/stale
   boundaries, restart-without-basis conservative classification, and non-finite input
   rejection.
9. Real `require_rust_binding` calls for every registered function and representative
   dict shapes, plus an assertion that importing the primary Python package does not
   import any network/provider module as a side effect.

Prefer structural bounds and deterministic clocks/entropy over sleeps. Snapshot the
version-1 JSON examples where that makes accidental wire drift obvious.

## Verification and bead completion

Before changing any implementation file, re-read the linked repository's `AGENTS.md`.
After changing any primary `sase` file, read `lint_and_test.md` through the audited SASE
memory workflow and bootstrap the ephemeral checkout with `just install` if required. Do
not edit `sase/memory/`.

Run focused Rust tests while iterating, but do not treat `cargo test -p sase_core` as
final verification because it omits the binding tests. Final verification requires:

- `just check` from the `sase-core` repository root, which includes PyO3 tests;
- focused primary Python smoke and validator tests against the rebuilt local extension;
- `tools/validate_sase_core_rs` green;
- primary-repository `just check` after installation, per `lint_and_test.md`.

Inspect both repositories' final diffs and statuses. Before closing, run
`sase bead epic-symbols sase-xe.2`; resolve every symbol or re-key its Justfile
ownership to the parent epic or a later still-open phase. Then close only `sase-xe.2`
with a note that names the core, binding, cross-language, revision-pin, and validation
evidence. Never close `sase-xe` or any ancestor. Do not create task beads; record
genuinely out-of-scope discoveries as `PROPOSED FOLLOW-UP:` notes on `sase-xe.2` for the
epic land agent to triage.

## Acceptance criteria

- Downstream phases can serialize and validate one versioned contract for origin,
  project, logical agent/family, and exact shell/run identity without using names,
  providers, endpoints, paths, or PIDs as identity.
- Resolved summary/detail records are safe owner projections with explicit lifecycle,
  liveness, health, observation, revision, capabilities, and opaque content handles;
  recursively serialized output contains no actionable local path/process/auth data.
- One order-independent core function produces the documented running, waiting,
  attention, and occupied-slot counts with family-handoff deduplication.
- Cursor replay explicitly requests resync for generation/gap/tombstone failures;
  operation replay distinguishes original receipt, payload conflict, controller scope,
  and expiry; connection plans remain serializable data with strict TLS/secret hygiene.
- Installation identity persists atomically, survives normal machine/provider/alias
  changes, and supports explicitly fenced rotation/migration.
- Every contract is reachable through the real `sase_core_rs` extension with no Python
  fallback, no new transport/runtime dependency in `sase_core`, and both repositories'
  required checks pass against the integrated core revision.
